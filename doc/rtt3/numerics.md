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

### Single precision with a factored covariance

The idea of factorising so that the filter fits 32-bit lanes without
losing precision is sound, with two qualifications: which factorisation,
and which part of the filter the precision argument covers.

**Which factorisation.** For a covariance the right one is the symmetric
form of LU: `P = U D Uᵀ` with unit-triangular U and diagonal D (or
equivalently `L D Lᵀ`), or the Cholesky factor. A general LU
factorisation with pivoting is not suitable for carrying the covariance:
it does not preserve symmetry or positive definiteness, and it gives no
precision benefit. For solving with a sensor's innovation covariance the
same holds; Cholesky or `L D Lᵀ`, not general LU.

**Why it recovers the precision.** A conventional filter squares the
dynamic range of the problem: the condition number of P is the square of
that of its factor. Single precision carries about seven decimal digits,
so a conventional covariance recursion starts to fail when the ratio
between the best and worst known directions approaches what seven digits
can hold. Working on the factor halves the exponent, which is why a
factored filter in single precision behaves, for the covariance and the
gain, about as well as a conventional one in double.

**What the argument does not cover: the state and the residual.** The
factor only protects the covariance arithmetic. The state vector and the
innovation `z - h(x)` are still limited by seven digits if they are held
in single precision. A position of a kilometre is then resolved to about
a tenth of a millimetre, and subtracting two nearly equal large numbers
to form a residual can lose most of the remaining digits. This, not the
covariance, is usually where single precision actually hurts.

**So: mixed precision.** The parts split cleanly by cost.

| Part | Work per step | Precision |
|---|---|---|
| Nominal state, prediction of the state, measurement prediction and residual | Proportional to the state size: a few scalar operations | Double |
| Covariance factor, its propagation, gains | Proportional to the square or cube of the state size: the bulk | Single |

- An error-state formulation makes the split natural: the nominal state
  is held and propagated in double, the filter estimates a small
  deviation, and only the deviation's covariance factor and the gains are
  single precision. The residual is formed in double and converted.
- Scale the error states so that their standard deviations are of
  comparable magnitude. This improves conditioning in any precision and
  costs nothing at run time when the scaling is part of the symbolic
  model.
- Keep coordinates relative to a local origin where large offsets would
  otherwise eat digits.

**Which factored form.**

- *UD (Bierman and Thornton).* No square roots. The measurement update
  is scalar and sequential, taking each measurement's variance directly,
  which matches the per-sensor structure above. The time update is a
  weighted Gram-Schmidt procedure that is inherently sequential, so it
  vectorises less well than a plain product.
- *Cholesky factor with a QR-based time update.* More regular
  arithmetic, which suits vector code better; needs square roots, which
  are a single cheap instruction on both target architectures.
- Either way, the sparsity of the transition Jacobian is still used: the
  product of the Jacobian with the factor is generated from the symbolic
  model, and only the re-triangularisation is generic.

**How much speed to expect.** Single precision doubles the values per
vector register (four instead of two with NEON, eight instead of four
with AVX2) and halves the memory touched. For large dense blocks that
approaches a factor of two. For a filter with few states, where the
generated code is mostly scalar and already sparse, the gain can be much
smaller, and a factored time update costs somewhat more than the
conventional product it replaces. Whether the change pays for itself is
a measurement, not a given; the benchmark compares the conventional
double filter, the factored double filter and the mixed-precision
factored filter on the target.

**Hazards particular to single precision.**

- *Underflow.* Variances are squares of small numbers, and with
  subnormals flushed to zero (see "Per-thread floating-point state") a
  very small entry of D becomes exactly zero and is later divided by. A
  floor on the diagonal, and the state scaling above, prevent it.
- *Cancellation in the residual*, addressed by forming it in double.
- *Accumulated drift over long runs.* Single precision errors are larger,
  so the comparison with the reference must include long recordings and
  the worst-conditioned phases: start-up with a large initial
  covariance, very accurate sensors, long periods without a measurement.

**Acceptance.** The mixed-precision filter replaces the double one only
if, on recorded data and in simulation, its state and reconstructed
covariance stay within the agreed tolerance of the double reference and
its consistency statistics are unchanged.

### When the state is a Lie group element in machine space

Poses and orientations of machine parts are elements of groups such as
SE(3), SO(3) or, for planar motion, SE(2). For these the mixed-precision
split is not just possible but the natural formulation, and a few choices
specific to groups decide whether single precision is safe.

**The split falls on the group and tangent boundary.**

- The *nominal state* is the group element itself (a rotation and a
  translation). It is held, composed and normalised in double precision.
  Composition and re-normalisation are a few dozen scalar operations, so
  double costs nothing that matters.
- The *error state* is a small vector in the tangent space (six numbers
  for SE(3), three for SE(2)), plus any ordinary vector states such as
  biases. The covariance factor and the gains belong to this space and
  are the single-precision part.
- The *correction* is applied by the group's retraction: the nominal
  state composed with the exponential of the estimated error, in double.
- The *residual* for a pose-valued measurement is the logarithm of the
  relative transform between prediction and measurement. It is formed in
  double, where the two nearly equal poses cancel exactly as they should,
  and only the resulting small tangent vector is converted to single.

Because the error vector is small by construction, its seven digits are
spent on the error and not on the magnitude of the pose. That removes the
state-resolution problem described above for the group-valued states.

**Which error definition: this decides the conditioning.** The error can
be defined in the moving (body) frame or in the fixed (machine) frame,
and the two give covariances that differ by the adjoint of the pose. The
adjoint of an SE(3) element contains the translation, so:

- with the error expressed in the fixed frame, a small rotation
  uncertainty of a part far from the origin appears as a large
  translational uncertainty, strongly correlated with the rotation. The
  tangent covariance becomes ill conditioned in proportion to the
  distance from the origin, which is precisely what single precision
  cannot afford;
- with the error expressed in the body frame, or with the fixed frame's
  origin placed near the working volume, the lever arm is short and the
  covariance stays well conditioned.

So the frame and side of the error definition are chosen for
conditioning, stated once, and used identically in the simulator and on
the target. Each tangent-space covariance on a port carries that
convention as an annotation, and links between ports with different
conventions are rejected at build time, in the same way as mismatched
units (`compile-time.md`). Mixing conventions silently is the most
common error in this kind of code.

**Scale rotation against translation.** Tangent components are radians
and metres. If the machine's characteristic length is far from one metre,
the rotational and translational variances differ by orders of magnitude
for no physical reason. Choosing the length unit of the error state so
that the two are comparable is part of the state scaling already
recommended, and belongs in the symbolic model.

**Invariant error formulations.** For systems whose dynamics are
compatible with the group structure, defining the error through the
group operation (the invariant extended Kalman filter) makes the error
dynamics independent of the current state estimate. Where that applies:

- the transition Jacobian depends only on inputs and the time step, so
  more of it is constant or sparse and the generated propagation is
  cheaper;
- the filter's behaviour does not depend on how good the current
  estimate is, which improves consistency, especially at start-up.

Whether the machine's kinematic and sensor models qualify has to be
checked model by model; where they do, it is preferable to the plain
error-state form.

**Planar motion.** For approximately planar machines the group is SE(2)
with a three-dimensional tangent space, so the covariance factor is very
small. The same symbolic model produces the planar and the full variant
(`blocks.md`, "Generated kinematics"); the estimator follows the same
pattern, and the planar one is where the cost per step is lowest but also
where single precision has least to gain, because there is so little
arithmetic to vectorise.

**Details that matter in single precision.**

- *Exponential and logarithm near zero* use series expansions; the
  generator's small-offset handling covers this and its offset must be
  appropriate for the precision in which the function is evaluated.
- *Logarithm near a half turn* is ill conditioned. Residuals are small
  rotations, so this is avoided as long as outliers are gated before the
  logarithm's result is used.
- *Normalisation.* The nominal rotation is re-normalised in double every
  step; it is cheap and stops drift off the group.
- *Covariance after a correction.* Strictly, the covariance should be
  transported to the tangent space of the corrected state. The transport
  is close to identity for small corrections and is commonly omitted; it
  should be included if corrections can be large, for example after a
  long period without measurements.

**Types.** Group elements, tangent vectors and tangent-space covariances
are distinct types in the block package, so that adding two poses or
applying a covariance in the wrong space does not compile. The generated
functions use the generator's own geometry types internally; the boundary
types name their fields (rotation as a quaternion with named components,
translation) as described under "Eigen".

### Several poses at once: SE(3)^n with n of five or six

The state in question is a product of n poses, n usually five or six.
The product of Lie groups is again a Lie group, so everything above
applies body by body. What changes is the size: the tangent space has
6n dimensions, 30 to 36 before any bias states, and at that size the
block structure matters more than anything else.

**Size of the problem.** A 36 by 36 covariance has 666 distinct entries.
A dense propagation `F P Fᵀ` costs on the order of ninety thousand
multiply-adds, and a rank-m update touches all of P. That is small in
absolute terms but no longer negligible at kilohertz rates on the ARM
target, and it is large enough for the single-precision argument to
carry real weight: the matrices are dense blocks that vectorise well, so
halving the element size approaches the full factor of two.

**Use the six-by-six block structure.**

- If each pose's own dynamics do not depend on the other poses, the
  transition Jacobian is block diagonal with one 6 by 6 block per body.
  The propagation is then a set of small products, one per pair of
  bodies, and only for one triangle of pairs: roughly a fifth to a tenth
  of the dense cost. Coupling that exists only between neighbouring
  bodies gives a block-banded Jacobian with a similar saving.
- A sensor usually observes one body, or the relation between two. Its
  measurement Jacobian is non-zero in 6 or 12 of the 36 columns, so the
  gain and innovation covariance for that sensor involve only those
  columns of P. The per-sensor update described below fits unchanged.
- If nothing couples two groups of bodies (no shared dynamics, no
  measurement relating them, no shared process noise), they are separate
  filters. The filter is only as large as the largest coupled group.

**Generate the blocks, not the whole matrix.** Fully unrolling a
36-dimensional propagation symbolically produces thousands of
expressions; code size, compile time and instruction-cache footprint
then work against the saving. The better division of labour:

- the symbolic model generates each body's 6 by 6 transition block and
  each sensor's Jacobian block, where the structure of SE(3) (rotation
  and translation parts, zeros, shared terms) is exploited;
- the assembly over bodies is ordinary fixed-size block arithmetic in a
  short loop over block pairs, which the compiler vectorises and which
  stays compact.

A block-indexed covariance type in the block package (n by n blocks of
6 by 6, one triangle stored) expresses this directly and keeps the
per-body error convention attached to each block.

**The real degrees of freedom are few.** The n poses are not
independent: the mechanism has far fewer degrees of freedom than the 6n
coordinates used to describe it. The next subsection is about detecting
and using that automatically, which outweighs every other measure here.

**Planar variant.** With SE(2) per body the tangent space has 3n
dimensions, 15 to 18. The same block scheme applies with 3 by 3 blocks.

**Memory.** A 36 by 36 single-precision matrix is about 5 kB, double
about 10 kB. Covariance, factor and one temporary fit comfortably in a
first-level cache in either precision, held as fixed-size objects in the
node's state with no allocation.

**Factored form at this size.** The sequential structure of the UD time
update costs more as the dimension grows. A Cholesky factor with a
QR-based time update, done block-wise, keeps the arithmetic regular and
is the better match for vector code here.

### Automatic reduction when the true degrees of freedom are few

**The structure being exploited.** If the bodies can only move in k
independent ways, with k much smaller than 6n, then every admissible
small motion of all the bodies together is `δξ = B δq`: a 6n by k matrix
B maps k independent coordinates to the stacked body twists. All
uncertainty lies in the column space of B, so the full covariance is

    P = B P_q Bᵀ

a 36 by 36 matrix of rank k. This is the rank-deficient matrix that
should be usable in its full form without paying for its full size
(`compile-time.md`, "Structured matrices"): the filter works on the
k by k matrix `P_q`, and the 36 by 36 form exists only when something
asks for it.

**What reduction buys.** The cubic term goes from 36³ to k³. With k
between, say, six and twelve, that is a factor of several tens to a few
hundred, far more than block structure or single precision can give.

There are three ways to find the reduction automatically. They differ in
what they need to be told and in when the work is done.

**1. From the mechanism's structure, at generation time (preferred).**
The symbolic model describes the mechanism as bodies connected by joints,
each joint permitting motion along known screw axes. From that
description the generator derives everything:

- k, by counting joint freedoms for an open chain or tree;
- B, as the derivative of each body's pose with respect to the
  independent coordinates, taken on the tangent space. A symbolic tool
  that differentiates on Lie groups produces this directly from the
  forward kinematics. B is itself sparse: body i depends only on the
  joints between it and the base, so its 6 by k block has few non-zero
  columns;
- the reduced filter: each sensor's Jacobian becomes `H B`, with as many
  columns as that sensor's body has ancestors, and the propagation runs
  on `P_q`.

Nothing is detected at run time and nothing has variable size. The
author writes the mechanism once and never states k or B.

For mechanisms with closed loops the independent coordinates are not
simply the joint coordinates. The loop-closure constraints are known
symbolically, and B is a basis of the null space of their Jacobian.
Where that has no convenient closed form it is computed numerically,
once at configuration if it is constant, or per step by a small
rank-revealing factorisation if it depends on the configuration.

**2. A numerical check that the structure is what it is believed to be.**
Whatever the model claims, a build-time or test-time tool evaluates the
model at sampled configurations and computes numerical ranks: of the
constraint Jacobian, and of the subspace that the initial covariance and
the process noise can ever excite through the dynamics. It reports the
detected number of degrees of freedom and flags any disagreement with
the model:

- more freedoms in the model than the data supports means redundant
  states that cost time and can make the filter inconsistent;
- fewer means a motion the mechanism really has is being treated as
  impossible.

This is the "automatic detection" in the diagnostic sense. It runs
off line, uses allocation and iterative decompositions freely, and
affects no real-time code. It also catches configurations where the rank
changes, which are the mechanism's singularities.

**3. A thin-factor filter, when no structure is declared.** A
square-root filter can carry a rectangular factor, 6n by r, with
`P = S Sᵀ`, and never form the square matrix. The time update appends
the process-noise factor and re-compresses; if the noise only excites r
directions, the factor keeps r columns. Costs scale with 6n times r
squared, not with the cube of 6n.

- It needs no kinematic description, so it works for a state whose
  redundancy is not understood in advance.
- The rank is in principle data dependent, which a real-time filter
  cannot allow. The maximum rank is therefore a compile-time constant,
  and the re-compression keeps at most that many columns, so sizes and
  operation counts are fixed.
- It is an approximation if the true rank exceeds the bound, and that
  case has to be visible: the discarded part is measured and reported.

**Recommendation.** Use the first for the kinematic states, since the
mechanism's structure is known and a symbolic model of it is already
planned; keep the second as a standing test; reserve the third for
states with no declared structure.

**What stays in full form.**

- The estimator still outputs the pose of every body. Those come from
  the reduced state through the generated forward kinematics.
- A consumer that wants the uncertainty of one body gets its 6 by 6
  block `B_i P_q B_iᵀ`, computed for that body only.
- A consumer that wants the whole 36 by 36 covariance can have it
  materialised, but the port type that carries it is the pair
  `(B, P_q)`, logically a full matrix. This is the one concrete use for
  the factored matrix types that were otherwise deferred.

**Things that do not fit the rigid model.** Clearances, compliance and
calibration errors are real motions outside the ideal mechanism. They
are added as a few extra coordinates with small process noise where they
matter, each one a deliberate modelling decision. The state then grows
by those few dimensions, not back to 6n.

**Consequence for the earlier measures.** With k around ten the filter
is small again. The 6 by 6 block scheme applies to how B is assembled,
not to the covariance, and the case for single precision largely
disappears: there is little arithmetic left to vectorise, and double
precision removes the conditioning concerns at negligible cost. The
mixed-precision factored form remains relevant only if a large state
survives the reduction.

### Worked case: a gyroscope on a hinged link sees out-of-plane rotation

**The situation.** Link i is attached to its parent by a hinge, so
relative to the parent it can only rotate about the hinge axis. A
gyroscope on link i reports angular velocity with a component that is
not along that axis. The hinge cannot have produced it. In the reduced
model this is not a contradiction to be discarded; it is information
about the joints further up the chain.

**The measurement model.** The angular velocity of link i, expressed in
its own frame, is the sum of the contributions of every joint between it
and the base:

    ω_i = Σ_j (a_j seen in frame i) · q̇_j        over the ancestors j of i

where `a_j` is the axis of joint j and `q̇_j` its rate. This is exactly
the rotational part of the block `B_i` from the previous subsection, so
the gyroscope's measurement Jacobian is not derived separately: it is
the rotational rows of `B_i`, plus identity columns for the gyroscope's
bias states. The joint rates are part of the reduced state for this to
apply.

**What the update does.** The hinge's own column of that Jacobian is the
hinge axis. The out-of-plane components of the measurement have zero
entries in that column, so they cannot move the hinge's rate at all.
They have non-zero entries in the columns of ancestor joints whose axes,
seen from link i, have components in those directions. The Kalman update
therefore distributes the out-of-plane innovation among exactly those
joints, in proportion to how uncertain each one's rate currently is and
how strongly its axis projects onto the measured direction. That is the
"evidence of a true movement of another joint", obtained with no special
case in the code: it is what `H B` and the gain compute.

Two consequences follow without further work:

- The update creates correlation between the rates of the joints that
  could have caused the motion. A later measurement that pins one of
  them down (an encoder on that joint, a gyroscope on a link nearer the
  base) automatically sharpens the others.
- Gyroscopes on consecutive links, taken together, isolate each joint:
  the difference of their angular velocities, in a common frame, is the
  hinge axis times that hinge's rate. The filter does this implicitly,
  with the correct weighting, and it is the reason a gyroscope on every
  link makes every joint rate observable.

**When no joint can explain it.** If no ancestor axis has a component in
the measured out-of-plane direction, the corresponding rows of `H B` are
zero apart from the bias columns. The model then has no motion to
attribute the measurement to. That happens when the hinge axes are all
parallel, which is exactly the planar mechanism. The component is one of:

- motion of the base, if the base is not actually fixed;
- compliance or clearance in a joint or link;
- misalignment of the gyroscope's axes relative to the link;
- gyroscope bias, which is the only thing the filter can then adjust,
  and which it will adjust wrongly if the cause is one of the others;
- a fault.

The filter must not be left to absorb such a component into the bias
silently. Two measures:

- *Parity residual.* For each sensor, the part of the innovation that
  lies outside the range of that sensor's `H B` is separated out. The
  projector that does this comes from the same symbolic model. That
  residual is not used to update the mechanism's state; it is a
  monitored output with its own threshold. A persistent non-zero value
  says the model is missing a freedom, which is a precise and early
  diagnostic.
- *Add the freedom deliberately.* If the out-of-plane motion is real and
  matters, it becomes a coordinate: a base tilt, a compliance angle at a
  joint, or a mounting misalignment to be calibrated. This is the "extra
  coordinates" case above, and the parity residual is the evidence for
  which one to add.

**A run-time check of the planar approximation.** With the planar
variant of the model, every out-of-plane gyroscope component is parity
residual by construction. Its magnitude is therefore a direct, continuous
measurement of how well "approximately planar" holds on the machine as it
runs, complementing the off-line comparison of the planar and full
variants (`blocks.md`, "Generated kinematics"). A threshold on it can
raise a warning or select the full model.

**How it is computed.** A gyroscope's three axes usually have independent
noise, so the update is three scalar updates. Each uses one row of the
rotational part of `B_i`, which has as many non-zero entries as link i
has ancestors, plus one bias entry. The per-sensor, varying-covariance
scheme below applies unchanged.

**In the full SE(3)^n formulation, by contrast,** the same measurement
would update link i's own orientation, and the hinge constraints would
have to pull the other links along through additional constraint
measurements. The information reaches the same joints in the end, but
through more arithmetic, with a constraint weighting to tune, and with
no clean statement of which part of the measurement the mechanism cannot
explain.

### One joint filter versus a cascade of filters

**What a cascade is.** The chain is split into stages, typically one per
link or per group of sensors. The first stage estimates the base or root
joint from its own sensors. Its estimate is handed to the next stage as
a known input, and so on down the chain. Each stage is a small filter.

**The defining property: information flows one way.** A downstream stage
can use what upstream has estimated; an upstream stage never sees
downstream measurements. So the assumption is correct: if a gyroscope
far down the chain detects out-of-plane rotation, a cascade cannot use
it to correct the root, even when the root is the only joint able to
produce that rotation. What happens instead:

- the root's stage has no measurement of that freedom, so its estimate of
  it stays at whatever the prediction says, with growing or
  never-shrinking uncertainty;
- the downstream stage receives a root estimate that says "no
  out-of-plane rotation", sees a measurement that says otherwise, and has
  no state of its own that can explain the difference. It puts the
  discrepancy into its gyroscope bias, or rejects the measurement as an
  outlier;
- the information is lost, and the symptom looks like unmodelled motion
  or a sensor fault although the mechanism model could explain it
  completely.

In the single reduced filter the same measurement has a non-zero entry
in the root's column of `H B`, and if the root is the only joint with
that freedom, essentially the whole out-of-plane innovation goes to the
root's rate.

**The second difference: correlations.** Errors of different joints'
estimates are correlated, because the same sensors inform several of
them. The joint filter carries those correlations in `P_q`. A cascade
discards them at every stage boundary, which shows up in one of two
ways:

- if the upstream estimate is treated as exact, the downstream stage is
  overconfident: its covariance is smaller than its real error;
- if the upstream covariance is injected as extra noise at every step,
  the same upstream error is counted again each time as though it were
  fresh, independent noise, which it is not. The result can be too
  pessimistic or too optimistic depending on the time scales, and is not
  correct in either case.

**What a cascade does offer.**

- Each stage is small, so the total cost is the sum of small cubes.
- Stages can run at different rates and be tuned and tested separately.
- A fault in one stage's sensors is contained downstream of it.

**How those advantages compare after the reduction.** The cost argument
mostly disappears: with the state reduced to the true degrees of
freedom, the joint filter is itself small. Different sensor rates are
handled in a joint filter by updating with whichever sensors report in a
step. Separate development is retained by the per-sensor structure: each
sensor contributes its own generated measurement function, and the
parity residual localises a faulty sensor at least as well as stage
boundaries do. So for this mechanism the joint reduced filter is the
recommendation, and a cascade would give up accuracy for a cost saving
that is no longer needed.

**When a cascade is still reasonable.**

- The upstream states are so well measured by their own sensors that
  downstream sensors could add nothing worth having, for example a root
  joint with a good encoder and a downstream link with a noisy
  gyroscope.
- A stage lives somewhere else, such as a drive's internal observer whose
  output is all that is available.
- The stages have widely separated time scales.

It is not reasonable when a freedom is only observable from downstream,
which is the case described. If a cascade is kept for other reasons, the
state has to be allocated by observability, not by body: the root's
out-of-plane freedom belongs in the stage that owns the sensor able to
see it.

**Intermediate designs**, if stages must stay separate:

- *Consider states (Schmidt-Kalman).* The downstream stage carries the
  upstream states and their cross-covariance but does not update them.
  It becomes consistent, with correct covariances, but still passes no
  information upstream.
- *Covariance intersection* when fusing estimates whose correlation is
  unknown: safe, at the price of conservative uncertainty.
- *Feedback of downstream innovations to upstream stages.* This recovers
  the lost information, and in doing so reconstructs the joint filter in
  a more complicated form.

**One caveat that applies to both designs.** A gyroscope alone cannot
distinguish a constant rotation rate from a constant bias. In the joint
filter the out-of-plane innovation is shared between the root's rate and
the gyroscope's bias in proportion to their uncertainties, and what
separates them over time is other evidence: an encoder or a second
sensor on the root, an accelerometer's view of gravity as the angle
accumulates, known stationary periods, and the fact that bias changes
slowly while the joint moves. Whether the root's out-of-plane freedom is
observable in practice depends on which of these exist, and the off-line
rank check described above is the tool that answers it for a given
sensor layout.

### Uncertainty that grows with the distance between sensor and joint

A measurement taken far down the chain should say less about a joint
near the root than a measurement taken next to it. Part of that is
already in the joint filter; the rest has to be modelled, and the model
is what makes the uncertainty grow with distance instead of a hand-tuned
factor.

"Distance" has two meanings here, and both matter. One is the number of
joints and links between the sensor and the joint in question. The other
is metric: how long the links are and where along a link the sensor
sits. An earlier version of this section said metric distance does not
matter for a gyroscope because all points of a rigid link share one
angular velocity. That is true only of a rigid link. Real links bend and
twist, their out-of-plane vibration grows with link length, and a
gyroscope measures the local angular velocity including that vibration.
The subsection after this one deals with it.

**What the joint filter already accounts for.**

- *Uncertain rates of the joints in between.* The predicted measurement
  is a sum over all ancestors, so the innovation covariance contains
  every intermediate joint's rate uncertainty. The more uncertain joints
  lie between the sensor and the root, the larger that covariance and
  the smaller the share of an innovation that is attributed to the root.
- *Uncertain angles of the joints in between.* The root's axis is seen
  from the sensor's frame through the rotations of every joint in
  between. Provided the joint angles are states and the generated
  measurement Jacobian includes their columns, an error in an
  intermediate angle is represented, to first order, as an error in how
  the root's rate projects onto the sensor axes.

Both effects grow with the number of joints in between without any
special treatment.

**What it does not know unless told.** The mechanism is not ideal. Each
joint and link adds a small unknown rotation: a hinge axis not exactly
where the model puts it, a link that twists slightly, a sensor mounted a
fraction of a degree off. Seen from a sensor far down the chain these
add up, and their effect on a gyroscope is proportional to the rotation
rate:

    error in measured rate  ≈  (accumulated misalignment) × (angular velocity)

This matters most for exactly the case discussed. A large in-plane
rotation, seen through a small accumulated misalignment, appears as a
small out-of-plane rate. A filter that believes the ideal geometry will
attribute that leakage to the root's out-of-plane freedom as if it were
real motion, with full confidence.

**Modelling it: consider parameters.** Each joint and each sensor
mounting gets a small misalignment vector, nominally zero, with a stated
standard deviation taken from tolerances or calibration. These are not
estimated by the filter; they are carried as parameters whose
uncertainty is accounted for (the Schmidt, or consider, form of the
filter):

- the symbolic model includes the misalignments, and the generator
  produces the measurement's sensitivity to each of them. For a
  gyroscope on link i, only the misalignments between the affected joint
  and the sensor contribute, each multiplied by the angular velocity
  downstream of it;
- the innovation covariance gains a term built from those sensitivities
  and the misalignment variances. That term grows with every joint
  between sensor and root, and with the rotation rate. A distant sensor
  on a fast-moving chain therefore gets a small gain on the root; the
  same sensor on a slowly moving chain, or a sensor next to the root,
  gets a large one. This is the distance-dependent uncertainty, derived
  and not tuned;
- the filter keeps the cross-covariance between the state and the
  misalignments, because a misalignment is a constant, not fresh noise
  at every step. Simply inflating the measurement noise would treat it
  as independent from sample to sample and leave the filter overconfident
  after averaging many samples. The cross-covariance is what prevents
  that;
- the cost is modest: one extra column per misalignment component in a
  rectangular cross-covariance block, and no gain computation for them.

**When to estimate them instead.** Misalignments are constant, so they
can be identified once from recorded motion, off line, and entered as
calibration values with a much smaller residual uncertainty. That
shrinks the distance-dependent term directly and is better than
accounting for a large unknown indefinitely. Compliance that changes
with load is different: if it matters, it becomes a state with its own
dynamics, as in "Things that do not fit the rigid model".

**The parity residual scales the same way.** The threshold on the
unexplained part of a sensor's innovation is set against its expected
variance, which now includes the misalignment term. A distant sensor on
a fast chain is allowed a larger unexplained component before anything
is flagged, and a near one a smaller. Without this, the planar-validity
monitor would raise false alarms during fast in-plane motion.

**Timing adds a similar term.** If sensors along the chain are sampled at
slightly different instants, the error is the angular acceleration times
the time offset, and it accumulates in the same way when measurements
from different links are combined. Where sample times are known, the
measurement is predicted at its own time stamp; where only a bound is
known, the bound enters as a further consider parameter.

**Accelerometers.** Their reading depends on where on the link they are
mounted, through terms in the angular acceleration and the square of
the angular velocity times the lever arm. Uncertainty in the lever arm,
and the fact that angular acceleration is poorly known, make their
uncertainty grow with metric distance as well as with the number of
joints. The same consider-parameter treatment applies, with the lever
arm as the parameter.

### Flexible links: vibration that scales with link length

**The effect.** A link is a beam. Its bending and torsion modes add a
local rotation at the sensor that is not part of the rigid-body motion.
A gyroscope reads the rate of that rotation in full. For a given
excitation the slope at the end of a beam grows with its length (roughly
with the square for bending, linearly for torsion) while the frequency
of the mode falls, so longer links give larger and slower out-of-plane
vibration. The amplitude at the sensor also depends on where along the
link it is mounted, through the slope of the mode shape at that point:
small near the joint, largest towards the free end. Vibration of one
link also shakes everything attached beyond it, so the effect
accumulates along the chain as well.

**Why it must be modelled and not just tolerated.** Out-of-plane
vibration at a distal gyroscope is exactly the kind of signal that the
joint filter would otherwise attribute to an out-of-plane freedom nearer
the root. It differs from true joint motion in two ways the filter can
use: it is oscillatory with zero mean, concentrated near the modal
frequencies, and its size is predictable from link length, sensor
position and how hard the mechanism is being driven.

**Three levels of treatment**, in increasing cost.

*1. Per-sensor, per-axis noise that depends on geometry and excitation.*
The gyroscope's measurement covariance gets a vibration term in addition
to the sensor's own noise:

- anisotropic: larger on the axes in which the link is most compliant,
  normally the out-of-plane ones, so the covariance is a matrix per
  sensor and not a single number;
- scaled by a geometry factor for that sensor, a function of link
  length and mounting position, which belongs in the symbolic model as a
  parameter of the sensor;
- scaled by the current excitation: commanded acceleration or jerk, or
  the measured high-frequency content of the gyroscope signal itself.

This is the varying per-sensor covariance that the estimator is already
built around, given a physical law instead of a tuned schedule. It is
cheap and it reduces the gain from a vibrating distal sensor onto
distant joints. Its weakness is that it treats a narrow-band oscillation
as white noise. That is acceptable when the filter's bandwidth for the
joint states is well below the modal frequency, so the oscillation
averages out; it is optimistic when the two are close.

*2. Modal states.* Each significant mode of a link becomes two states, a
modal coordinate and its rate, with a natural frequency and damping,
driven by process noise and by the joint accelerations. A gyroscope on
that link measures the rigid-body rate plus the modal rate times the
mode-shape slope at its position.

- Metric distance enters through that slope, per sensor. Two gyroscopes
  on the same link see the same mode with different gains, which lets
  the filter separate vibration from rigid motion instead of merely
  discounting both.
- The out-of-plane oscillation is then explained by the modal state and
  not offered to the root joint at all.
- Cost is two states per modelled mode. With a first out-of-plane bending
  mode on each of the long links, that is a handful of states added to
  the reduced filter, not a return to 6n.
- The modal frequencies shift with payload and configuration, so they
  are parameters that may need scheduling or slow adaptation.

*3. Pre-filtering.* A notch or low-pass on the gyroscope signal before
the estimator removes the vibration but makes the remaining noise
coloured and adds delay, both of which the filter then has to be told
about. It is a fallback, not a substitute for the first two.

**Recommendation.** Start with level 1 for every gyroscope, with the
geometry factor identified from recorded data per sensor (the spectrum
of each gyroscope during representative motion gives modal frequency and
amplitude directly) and the length scaling used as a prior and a
cross-check. Add level 2 for the links where the modal frequency is
close to the bandwidth needed for the joint estimates, or where the
vibration itself must be controlled.

**Consequences elsewhere.**

- *Parity residual and the planar monitor.* Out-of-plane vibration is
  unexplained motion to a rigid planar model, so the planar-validity
  monitor would trip on it. The residual is therefore split by
  frequency: the slowly varying part indicates a real departure from
  planar motion; the part near the modal frequencies is a vibration
  measure, compared against the level expected from the excitation. The
  second is a useful output in its own right, for condition monitoring.
- *Misalignment and vibration combine.* The consider-parameter term of
  the previous subsection and the vibration term both enlarge the
  innovation covariance of a distal sensor; one grows with the number of
  joints and the rotation rate, the other with link length and
  excitation.
- *Accelerometers* are affected through the deflection itself, not only
  its slope, and through the lever arm; the same modal states serve both
  sensor types.

### Applied to the case at hand: an EKF with per-sensor varying covariance

The filter in question is an extended Kalman filter in which each
sensor's measurement covariance varies at run time. That narrows the
options above:

- **Option 1 is out.** The model is linearised about the current state
  and the measurement covariance changes, so there is no steady-state
  gain to precompute.
- **Option 2 is the main saving.** The state-transition Jacobian and each
  sensor's measurement Jacobian come from the symbolic model, so the
  propagation and the per-sensor products are generated with their
  sparsity and symmetry used. This is also where the Jacobians stop being
  hand-derived.
- **Option 3 fits the problem's shape, sensor by sensor.** Different
  sensors are independent of each other, so the joint measurement
  covariance is block diagonal with one block per sensor, and the update
  can be done one sensor at a time:
  - a sensor whose own noise is uncorrelated between its components is
    processed as scalar updates, with the current variances passed in as
    ordinary inputs. A varying variance costs nothing extra;
  - a sensor with correlated components is processed as one small block:
    a Cholesky solve of its innovation covariance, of the sensor's own
    dimension (typically up to three or six), unrolled by the generator.
    No factorisation of anything larger is needed anywhere in the filter;
  - sensors arriving at different rates, or missing in a given step, are
    handled by updating with whichever are present.
- **Option 5** remains available if robustness or single precision
  becomes a concern; the UD form takes a per-measurement variance
  directly.

Points that are specific to doing this with an EKF:

- *Linearisation point.* Updating sensor by sensor is equivalent to one
  joint update only if every sensor's Jacobian is evaluated at the same
  prior state. Re-linearising at the partially updated state between
  sensors is also legitimate, and often slightly more accurate, but the
  result then depends on the order of the sensors. Either is acceptable;
  the choice is made once, written down, and made the same way in the
  simulator.
- *Covariance as data.* Each measurement travels with its covariance and
  a validity flag as one port data type, so the estimator block's inputs
  are fixed-size and a sensor that reports nothing in a step is a flag,
  not a missing message. The covariance is carried as a diagonal or as
  one triangle, matching how the sensor's update is done.
- *Guarding the varying covariance.* A covariance supplied at run time
  can be wrong: zero, negative, non-finite, or absurdly small. The
  estimator checks it before use and treats a bad one as an invalid
  measurement. The innovation test for outliers uses the same per-sensor
  innovation covariance that the update already computes.
- *Covariance update form.* With gating and varying noise, the symmetric
  Joseph form or a square-root form is preferred to the short form,
  which can lose positive definiteness when a gain is modified or a
  measurement is strongly weighted.
- *Worst-case time.* The cost of a step depends on how many sensors
  report. The execution-time budget for the block is that of the step in
  which all of them do, and the per-sensor structure makes that bound
  easy to state.

### Suggested order of attack

For the EKF described above:

1. Generate the propagation and the per-sensor Jacobian products from the
   symbolic model, using structure and symmetry.
2. Update sensor by sensor: scalar updates where a sensor's components
   are uncorrelated, a small Cholesky solve where they are not.
3. Use a covariance update form that keeps symmetry and positive
   definiteness.
4. Consider a square-root form if robustness is a problem or single
   precision is wanted.

For a time-invariant filter elsewhere in the system, check first whether
a steady-state or scheduled gain is valid.

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
- The filter is an EKF with per-sensor varying covariance (answered).
  How many states, how many sensors and of what dimension each? Is each
  sensor's own noise uncorrelated between its components? Should sensors
  be re-linearised between updates? Is the covariance used downstream?
- The state is SE(3)^n with n of five or six, and the true degrees of
  freedom are few (answered). Roughly how many, and is the mechanism an
  open chain or does it have closed loops? Are clearances or compliance
  estimated? Which bodies does each sensor relate? Does one body's
  dynamics depend on another's? Are there bias or other vector states in
  addition?
- What is the filter's rate and its share of the cycle time today?
- Out-of-plane vibration scales with link length (stated by the owner).
  What are the link lengths, where along each link are the gyroscopes,
  and what are the lowest out-of-plane modal frequencies compared with
  the bandwidth needed for the joint estimates? Is the present varying
  covariance driven by this vibration, and how is it computed today?
- What are the mechanical tolerances or calibration accuracies of hinge
  axes and sensor mountings, and are misalignments calibrated today?
  How fast do the links rotate? Those set the distance-dependent term.
  Is link or joint compliance significant under load?
- Is the existing estimator a single filter or a cascade? If a
  cascade, is that for cost, for rates, or because stages live in
  different places (for example inside drives)?
- Where are gyroscopes and encoders mounted along the chain, and is the
  base fixed? What should happen when a sensor reports motion the
  mechanism cannot produce: warn, add a modelled freedom, or switch from
  the planar to the full model?
- How far from the machine origin do the estimated parts move, and
  which error convention (body or fixed frame) is used today? Do the
  models allow an invariant formulation?
- Mixed precision (double nominal state and residual, single-precision
  covariance factor) is the proposed way to use 32-bit lanes. What are
  the magnitudes and required resolutions of the states, and how long
  does the filter run without a reset? Those decide whether it is safe.
- Is there any cross-checking between two computers, or any checksum over
  computed values, that would need identical results?
- `std::simd` in GCC 16 is experimental and unverified here, like the
  rest of the GCC 16 feature set; revisit its exclusion from port data
  once it is stable.
