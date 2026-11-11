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

**Is SE(3)^n the right state?** Worth asking before optimising it. If
the n poses are links of one mechanism connected by joints, a pose per
link carries far more coordinates than the mechanism has degrees of
freedom, and the constraints between them have to be enforced by
measurements or by the model. Estimating the joint coordinates (plus a
base pose if it moves) gives a state of roughly n to n + 6 dimensions
instead of 6n, with the link poses obtained from the generated forward
kinematics. That is a reduction of the cubic term by two orders of
magnitude and would dominate every other measure in this section.
Estimating full poses per link remains the right choice when the links
are not rigidly constrained (flexible structures, clearances, independent
bodies) or when the constraints are exactly what is being estimated.

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
- The state is SE(3)^n with n of five or six (answered). Are the n
  bodies links of one mechanism connected by joints, or independent or
  loosely coupled? Which bodies does each sensor relate? Does one body's
  dynamics depend on another's? Are there bias or other vector states in
  addition?
- What is the filter's rate and its share of the cycle time today?
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
