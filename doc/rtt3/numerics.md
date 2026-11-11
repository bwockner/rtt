# Numerics: SIMD, floating point and agreement with the simulator

Status: design notes for `rtt3`, with a short section on what applies to
RTT 2.x today. `realtime-internals.md` keeps a summary. Part of the
`rtt3` plan; see [README.md](README.md) for the index.

Decided so far: performance comes first, and the simulator and the real
deployment are expected to agree within a tolerance, not bit for bit.
Open points are in `../open-questions.md` and at the end of this document.

SIMD matters to the framework in four places: how data is laid out and
copied, how threads are set up, how blocks are compiled, and how results
are compared with the simulator. The framework does not vectorise user
algorithms; it must not get in the way of code that is vectorised, and it
must make the consequences visible.

## Alignment is preserved end to end

- Channels, buffers, data objects and log rings honour the alignment of
  the value type, including over-aligned types (32 bytes for AVX, 64 for
  AVX-512, vector types from linear-algebra libraries). Static deployment
  storage is laid out with each member's alignment, and the configuration
  arena and block pools take an alignment argument.
- `rtt3::port_data` accepts over-aligned types. A compile-time check
  rejects a type whose alignment exceeds what a given channel or pool was
  built for, instead of producing a misaligned object.
- The wire encoding for the UDP bridge and the log encoder copy bytes and
  make no alignment assumption about the destination buffer.

## Layouts that vectorise

For per-axis data the recommended form is a
struct of fixed-size arrays (`std::array<double, N>` for positions, one
for velocities, and so on), not an array of per-axis structs. The
compiler can vectorise loops over it, and it is still plain port data.
Reflection treats both forms alike. Repeated ports (`components-and-ports.md`) are for wiring
convenience; a block that processes all axes at once should take one
array-valued port.

## `std::simd`

GCC 16 ships the C++26 data-parallel types as an
experimental feature. They may be used inside blocks. They are not
allowed in port data types or block signatures for now: their size and
layout depend on the target's vector width, which would make a type differ
between the x86-64 desktop and the ARM target and break the shared schema.
A block converts to and from fixed-size arrays at its boundary. This is to
be revisited once the feature is stable.

## Per-thread floating-point state

Each real-time thread sets its
floating-point control state at start, from the deployment:

- Subnormal numbers are flushed to zero by default on real-time threads
  (the flush-to-zero and denormals-are-zero bits on x86-64, the flush-to
  -zero bit on AArch64). Arithmetic on subnormals can be many times slower
  on some processors, which shows up as latency spikes when a filter or
  integrator decays towards zero. This is a numerical change, so it is a
  stated default that a node can opt out of, and the simulator has to use
  the same setting for results to match.
- The rounding mode is set explicitly and floating-point exceptions are
  masked, so behaviour does not depend on what the creating thread had.
- The settings are part of the start-up report.

## Instruction sets and builds

- The target CPU is set in the toolchain file per machine class (`os-layer.md`),
  so the compiler can use the vector instructions the deployment hardware
  has. One binary per machine class, not run-time dispatch, is the
  default; it keeps code paths identical between test and production.
- Where one binary must serve several CPUs, function multi-versioning is
  permitted for individual blocks. Its resolvers run at load time, which
  immediate binding (`realtime-internals.md`) already
  moves to start-up.
- 512-bit vectors on x86-64 are off unless enabled deliberately: on some
  processors they lower the clock frequency of the core for a while and
  enlarge the register state saved at every context switch. The
  benchmark suite reports context-switch cost with and without them.
- On AArch64 the baseline is NEON. Scalable vectors are not used in port
  data, for the same reason as `std::simd`.

## Eigen

Eigen is the linear-algebra library used downstream for state estimation
and control, so its types will appear inside blocks, in block signatures
and on ports. The statements about Eigen below are from knowledge of the
library and were not checked against the version in use; the ones that
matter are written as compile-time checks so they verify themselves.

### Fixed-size types as port data

- Fixed-size Eigen types (`Vector3d`, `Matrix4d`, `Quaterniond`, and
  fixed-size `Matrix<double, N, M>`) are the intended port data for
  vectors and matrices: no allocation, size known at compile time.
- They are classes with private storage, so the generic "reflect the
  public members" rule does not describe them. An optional adaptor header
  in the block package teaches the schema, the wire encoding, the logger
  and the `rtt3::finite` check about them: a fixed-size matrix is a named
  array of scalars with its shape. The adaptor is separate so that neither
  the framework nor blocks that do not use Eigen include it.
- Quaternions are encoded with named fields. Eigen stores coefficients in
  the order x, y, z, w but its constructor takes w first, which is a
  classic source of silent errors when data crosses a boundary; a schema
  with names removes the ambiguity for the simulator and tools.
- The adaptor asserts at compile time what it relies on (for example that
  the type is trivially copyable in the Eigen version in use) and falls
  back to coefficient-wise copying where an assumption does not hold.

### Alignment depends on compiler flags

This is the main hazard. The alignment of fixed-size vectorisable Eigen
types follows the instruction set the translation unit is compiled for:
16 bytes with SSE or NEON, 32 with AVX. Consequences:

- **Within one program**, every translation unit that sees a given struct
  must be compiled with the same vector flags, or the struct has two
  different layouts and the program is broken in ways that are hard to
  diagnose. The block package exports its flags as usage requirements, and
  each library that carries port data types embeds the alignment it was
  built with, so a mismatch is a link error, not a crash.
- **Between the desktop and the ARM target**, layouts of structs
  containing Eigen members can differ (32-byte alignment with AVX on
  x86-64, 16 with NEON). So raw bytes of such a struct are never sent
  between machines. The wire encoding writes scalars member by member,
  and the schema hash is computed from names, scalar types and shapes, not
  from offsets and padding. A type whose layout differs is then still the
  same type to the simulator and the tools.
- **Capping Eigen's static alignment at 16 bytes is not recommended.**
  An earlier draft of this document suggested it. The owner has seen a
  crash with a 16-byte setting on x86, and that is the expected outcome
  whenever the cap does not reach every piece of code that touches the
  same Eigen objects. The setting changes the size and layout of types,
  so it has to be identical in every translation unit **and in every
  library that takes or returns Eigen types in its interface**, including
  prebuilt ones. A dependency compiled with AVX and Eigen's defaults
  expects 32-byte alignment; handing it an object from code capped at 16
  faults on the first aligned load or silently reads the wrong offsets.
  The same applies in reverse, and to any mix of vector flags without a
  cap. The exact cause of the crash seen has not been established; see the
  open points.

### Boundary types that do not depend on flags

The robust way out is to stop the layout of shared types from depending
on compiler flags at all, instead of trying to keep the flags identical
everywhere:

- Types that cross a boundary (port data, block signatures, anything
  shared with the simulator or stored in a recording) use Eigen's
  unaligned fixed-size form, `Eigen::Matrix<double, R, C, Eigen::DontAlign>`
  and the corresponding quaternion. Their alignment is that of the scalar,
  their layout is the same under every vector flag and on both
  architectures, and no cap or project-wide macro is involved.
- The block package provides aliases for the common ones (for example
  `blk::Vec3`, `blk::Mat4`, `blk::Quat`), so that writing a boundary type
  is no more effort than writing the aligned one, and a concept that
  rejects aligned Eigen types in port data and block signatures with a
  message pointing to the alias.
- Inside a block, code may copy into Eigen's default aligned types for
  heavy computation, or compute directly on the unaligned ones; Eigen
  still vectorises with unaligned loads, which cost little on current
  x86-64 and AArch64 processors. For the small fixed sizes typical of
  control and estimation the difference is unlikely to be measurable, and
  the benchmark suite includes a case to confirm it.
- Aligned Eigen types remain fine for a block's internal state and local
  variables, where only one translation unit sees them.
- Third-party libraries that expose Eigen types in their interfaces are
  called from inside a block with whatever types they expect, converted at
  the call. Their build flags then only have to agree with the block that
  calls them, which is the constraint that exists today anyway.

With this rule the two bullets above become checks on internal code only,
and the layout of anything exchanged between machines or libraries is
fixed by construction.

### Allocation

- Dynamic-size types (`MatrixXd`, `VectorXd`) allocate. On ports they are
  `dynamic_data` and need the explicit opt-in and data sample
  (`components-and-ports.md`).
- Where the size is only known at configuration but has a bound, use
  Eigen's bounded form (dynamic size with fixed maximum rows and columns).
  It stores inline and qualifies as bounded data.
- Inside blocks, the usual sources of hidden allocation are temporaries
  from expressions on dynamic-size operands, and decomposition objects.
  Construct decompositions with their size during configuration and reuse
  them; use `noalias()` where a product is assigned to a distinct
  destination.
- The framework's allocation guard catches any of this in tests. Eigen's
  own switch for forbidding allocation at run time can be turned on in
  debug builds as a second net that reports the Eigen expression
  responsible.

### Threads and back ends

- Eigen can parallelise large products with OpenMP when the program is
  built with it. That must be off (`EIGEN_DONT_PARALLELIZE`) for anything
  running in a real-time thread: worker threads would be created outside
  the deployment's scheduling and affinity.
- For the same reason, Eigen is not configured to delegate to an external
  BLAS or LAPACK library in real-time code; those may start threads and
  allocate.
- Eigen's assertions are disabled in deployment builds and enabled in
  test builds, where they catch size and aliasing errors.

### Compiler options

- Eigen follows the compiler's vector flags, so the per-machine target
  CPU setting above is all that is needed to get vectorised code.
- The exclusion of `-ffinite-math-only` matters for Eigen in particular:
  its own finiteness tests (`allFinite()`, `hasNaN()`) and the checks
  inside some decompositions rely on NaN semantics.

### Numerical robustness in estimation and control

Tolerance-based agreement with the simulator assumes the algorithm is
well conditioned. The places in estimation and control code where small
differences are amplified, and where simulation may not reveal the
problem at the same operating point as the real machine:

- Solve with a decomposition, not with an explicit inverse, and check the
  decomposition's status before using the result.
- Covariance matrices lose symmetry and positive definiteness through
  rounding; use an update form that preserves them (Joseph form or a
  square-root formulation) and compare them with a tolerance.
- Regularise near kinematic singularities; behaviour there is dominated by
  conditioning, not by the arithmetic.
- A failed decomposition or a division near zero produces non-finite
  values that then spread through the state. Mark estimator outputs and
  commands `rtt3::finite` so that this stops at the block boundary.

### Versions

One Eigen version is pinned for both the simulator and the target and
recorded in the block package, since alignment defaults and
trivial-copyability of fixed-size types have changed between releases.

## Kalman-style filtering: factorisation and structure

Factorisation helps, but for the state sizes usual in control (a handful
to a few tens of states) it is rarely the largest gain. The options
below are ordered by how much work they typically remove. They are
standard results from the estimation literature, stated from general
knowledge; none has been measured on the code in question.

### 1. Do not run the covariance recursion at all, where that is valid

For a time-invariant model with constant noise covariances and every
measurement present at every step, the covariance and the gain converge
to constants. The steady-state gain can be computed once, during
configuration, by solving the discrete Riccati equation (an iterative
calculation, which is acceptable there). The run-time filter is then the
state prediction and `x + K (z - H x)`: no covariance propagation, no
factorisation, no matrix inverse. Many drive-level observers qualify.
Where the model varies slowly with an operating point, a table of gains
indexed by that point is the same idea.

It does not apply when measurements drop out or arrive at varying rates,
when the model is linearised about the current state, or when the
covariance itself is consumed downstream.

### 2. Exploit the structure of F and H in the propagation

`P = F P Fᵀ + Q` is usually the most expensive line, and F is rarely
dense: integrator chains, block structure between subsystems, identity
blocks. H very often selects or differences a few states, so `H P Hᵀ` is
a sub-block of P, not a product.

- With a symbolic model (`blocks.md`, "Generated kinematics"), the
  generator emits the propagation element by element, using the sparsity
  of F and H, computing only one triangle of P, and sharing common
  subexpressions. For filters of this size this is typically the biggest
  single saving available, and it needs no factorisation.
- Without generation, the same structure can be used by hand with block
  operations, at the cost of the code complexity that generation avoids.
- Subsystems that are not coupled through F, Q or H are separate, smaller
  filters. Cost grows with the cube of the state size, so two filters of
  half the size cost about a quarter.

### 3. Sequential scalar updates in place of a matrix solve

When the measurement noise covariance is diagonal, the measurement
update can process one scalar measurement at a time. Each scalar update
needs one division and work proportional to the square of the state
size; no innovation covariance matrix is formed and nothing is
factorised. Correlated measurement noise can be brought to this form by
whitening with a factor of R that is computed once.

This form suits a real-time system for reasons beyond speed:

- a missing or rejected measurement is simply skipped, so a varying
  number of valid measurements needs no dynamically sized matrices;
- gating of outliers is a per-measurement comparison;
- the operation count per measurement is fixed.

### 4. When a matrix solve is needed, use the right factorisation

- The innovation covariance is symmetric positive definite. Compute the
  gain by a Cholesky factorisation and two triangular solves, never by an
  explicit inverse. It costs roughly half of a general factorisation and
  is better conditioned.
- Plain Cholesky has no pivoting, so its operation count is fixed, which
  matters for timing. A non-positive pivot means the matrix has lost
  positive definiteness; that is reported as an error from the block, not
  ignored.
- Compute and store one triangle of symmetric matrices, and use
  symmetric rank updates for the covariance, so that symmetry holds by
  construction instead of being restored afterwards.
- For very small measurement dimensions (up to three or four) a
  closed-form, unrolled solve beats a general routine; generated code
  produces this automatically.
- Factor constant matrices once. Whitening factors of R and Q, for
  parameters fixed at build time, can be compile-time constants
  (`compile-time.md`); otherwise they are computed during configuration.
- If measurements far outnumber states, the information form inverts a
  matrix of the state size instead of the measurement size.

All of this stays within fixed-size types with in-place factorisation, so
there is no allocation.

### 5. Square-root and UD forms

Propagating a factor of the covariance (a Cholesky factor, or the UD
form of Bierman and Thornton) instead of the covariance itself does not
reduce the operation count much. What it buys:

- the covariance is symmetric and positive semi-definite by construction,
  with no symmetrisation or Joseph-form update needed;
- roughly twice the effective numerical precision, since the factor has
  the square root of the condition number.

The second point is where a speed-up can come from: a square-root filter
can often run in single precision where a conventional one needs double.
On the ARM target that doubles the number of values per vector register
and halves memory traffic. This is a design decision per filter and needs
the filter's consistency checked in simulation, not assumed.

The UD measurement update is naturally sequential and combines with
option 3.

### Suggested order of attack

1. Check whether a steady-state or scheduled gain is valid.
2. Generate the propagation from the symbolic model, using structure and
   symmetry.
3. Use sequential scalar updates where the measurement noise is, or can
   be made, uncorrelated.
4. Where a solve remains, use Cholesky on fixed-size types.
5. Consider a square-root form if robustness is a problem or single
   precision is wanted.

### Checking a restructured filter

A restructured filter is compared with the reference implementation on
recorded data, state and covariance both, within tolerance. In
simulation, where the true state is known, the usual consistency
statistics (normalised estimation error and innovation) show whether the
filter is still correctly tuned; a speed-up that makes the filter
overconfident is not a gain.

## Agreement between the simulator and the target: within tolerance

Decided: performance comes first, and the simulator and the real
deployment are expected to agree within a tolerance, not bit for bit. A
controller whose behaviour changes materially with last-bit differences
is fragile, and real sensors and plants perturb it far more than the
compiler does. The plan therefore does not promise or test for
bit-identical results across machines, and blocks are built for speed:
vectorisation on, fused multiply-add allowed, target-specific
instructions used.

What remains worth guarding against is not small numerical difference but
the few compiler options that change behaviour in kind:

- **Finite-math-only removes safety checks.** With `-ffast-math` (through
  `-ffinite-math-only`) the compiler may assume no value is ever NaN or
  infinite and delete tests such as `std::isnan(x)` or `x != x`. A guard
  on a sensor value or on a command about to reach an actuator can vanish
  without warning. This is the one fast-math component that is excluded.
- **Reassociation breaks compensated arithmetic.** Code that depends on
  the exact order of operations (compensated summation, error-free
  transformations, some angle-wrapping tricks) can be simplified into
  something wrong, not something slightly different. Such functions are
  marked to be compiled without reassociation.
- **Process-wide side effects.** Linking with fast-math can switch the
  floating-point mode of the whole process at start-up on some toolchain
  versions. The mode is set explicitly per thread (above), so this neither
  surprises nor is relied on.

Recommended build for blocks, as the default of the block package:
optimise fully, allow contraction, drop `errno` and trapping-math
semantics (`-fno-math-errno -fno-trapping-math -ffp-contract=fast`), and
optionally allow reciprocal and signed-zero relaxations. Not
`-ffinite-math-only`, and reassociation only where the author of a block
turns it on.

## Finite checks that cannot be optimised away

Because non-finite
values are the failure that matters on a real machine, the framework
offers the check itself: a port or a block output annotated
`rtt3::finite` is tested on every write, by inspecting the bit pattern
with integer operations, so the test survives any floating-point option.
A non-finite value puts the node into its error state and is not
delivered. It is on by default for links into nodes marked as actuators
and off elsewhere.

## Comparing against the simulator, done so that tolerance works

- *Per-step replay.* A block is fed recorded inputs step by step and its
  outputs compared within a bound. Inputs are taken from the recording at
  every step, so differences do not feed back and accumulate through the
  rest of the system.
- *State drift.* A block with internal state (an integrator, a filter)
  still accumulates its own difference. Stable filters stay within a
  bound; a pure integrator drifts without limit, so the comparison uses a
  bound that grows with time or resynchronises the state periodically.
- *Discrete decisions.* A threshold, a mode switch or a rounding to an
  integer can fall on the other side for a last-bit difference, and the
  two runs then differ by a whole event for a short time. Comparisons
  around such events allow a shift of a sample or two in time, not just a
  bound in value, or compare the event sequence separately from the
  signals.
- *Closed loop.* End-to-end comparison of simulation and real machine is
  about behaviour (tracking error, settling, limits respected), not about
  matching traces sample by sample; open-loop traces of a system with
  contact diverge whatever the arithmetic.

## Where bit-exact results would still be needed

None of these is
known to apply to the current system; they are listed so the decision can
be revisited if one appears:

- two computers computing the same value and cross-checking each other
  for safety, or voting, with an equality comparison;
- checksums, hashes or signatures taken over floating-point results;
- regulatory or certification evidence that requires reproducing a
  recorded run exactly on different hardware.

Results on the same binary and the same machine are repeatable in any
case, which is what regression tests on one platform need.

## Checking it

Replay tests compare within stated tolerances on x86-64
and AArch64. The benchmark suite includes a block over array data built
with and without vectorisation, so a lost vectorisation shows as a
regression. A test confirms that the `rtt3::finite` check still fires in
a build with every fast-math option on.

## What applies to RTT 2.x today

- **Over-aligned types and the real-time allocator.** TLSF aligns blocks
  to twice the pointer size, 16 bytes on 64-bit targets
  (`rtt/os/tlsf/tlsf.c:138`). A type that needs 32-byte alignment is not
  guaranteed it when allocated through `rt_allocator`. Check any such
  types used with real-time strings, containers or `send`.
  With Eigen this is a concrete risk on x86-64 builds that enable AVX:
  fixed-size vectorisable types then need 32 bytes, and an object holding
  one that is allocated from the pool can be misaligned, which faults on
  an aligned load. Builds limited to SSE or NEON need only 16 bytes and
  are unaffected.
- **Eigen on RTT 2.x ports.** Dynamic-size vectors need `setDataSample`
  with a correctly sized value before connections are made, or the first
  writes allocate. Keep OpenMP off in real-time components.
- **Subnormal numbers.** Flush-to-zero can be set at the start of each
  real-time thread, in the component's `configureHook` or `startHook`,
  since it is per-thread state.
- **Build flags** are chosen by each component, not by RTT. Avoid plain
  `-ffast-math` for the reason given above; prefer
  `-fno-math-errno -fno-trapping-math -ffp-contract=fast`, or add
  `-fno-finite-math-only` after it.

## Open points

- Is flushing subnormals to zero on real-time threads acceptable as the
  default?
- Eigen is used for state estimation and control (answered). Which
  version, and are Eigen types on ports today fixed-size, dynamic, or
  both?
- What were the circumstances of the crash seen with a 16-byte alignment
  setting on x86: which macro or flag was set, was it applied to every
  target, was AVX enabled (`-mavx`, `-march=native`), and were prebuilt
  libraries that use Eigen involved? That would confirm whether the cause
  was a mismatch between translation units or something else.
- Is using unaligned Eigen types at boundaries acceptable, with aligned
  types kept for internal computation?
- Is any Eigen code built with OpenMP or linked against an external BLAS
  today?
- For the existing filters: how many states and measurements, is the
  model time-varying or linearised about the state, is the measurement
  noise diagonal, do measurements drop out, and is the covariance used
  downstream? Those answers pick among the options under "Kalman-style
  filtering".
- Is single precision acceptable for any filter if a square-root form
  makes it numerically safe?
- Is there any cross-checking between two computers, or any checksum over
  computed values, that would need identical results?
- `std::simd` in GCC 16 is experimental and unverified here, like the
  rest of the GCC 16 feature set; revisit its exclusion from port data
  once it is stable.
