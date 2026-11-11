# rtt3: state estimation and calibration

Part of the `rtt3` plan; see [README.md](README.md) for the index.

Status: design notes. This material grew out of `numerics.md` and was
moved here when it outgrew that document. It concerns the estimator that
runs as blocks on the framework, not the framework itself. It is
standard estimation theory applied to what the owner has stated about
the machine; none of it has been tested on the system in question, and
statements about third-party tools (SymForce, GNSS receivers, inertial
units) are from general knowledge.

What the owner has stated, in brief: the state is a set of five or six
poses whose true degrees of freedom are few; there is one inertial unit
per link sampled at about 100 Hz, plus GNSS; sensor covariances vary at
run time; out-of-plane vibration grows with link length and disturbances
propagate at finite speed; more distant sensors are to be trusted less;
the estimator is to be a truncated factor graph whose residual curvature
indicates observability, and the same graph is to serve identification
and calibration.

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
  subnormals flushed to zero (`numerics.md`, "Per-thread floating-point state") a
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
translation) as described in `numerics.md` under "Eigen".

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

**Decided default.** With one inertial unit per link and no model of the
structure or its mode shapes, the general rule is that a sensor is
trusted less the further it is from the joint in question. The next
subsection turns that rule into a noise model. Modal states (level 2)
are not part of the default and are added only where a specific link
demands it.

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

### The default rule: trust falls with distance along the chain

**The difficulty with stating it naively.** A Kalman filter gives each
sensor one covariance. "Trust this sensor less for distant joints and
more for near ones" is a statement about pairs of sensor and joint, and
a single number per sensor cannot express it. Simply enlarging the
covariance of the outer sensors would also make each outer joint
distrust the unit on its own link, which is the best information it has.

**A disturbance that accumulates along the chain.** The rule follows
exactly from one modelling assumption, with no structural model: every
link adds its own independent disturbance to the angular velocity seen
from that link onward. The reading of the unit on link i is then

    z_i = (rigid-body rate of link i) + w_1 + w_2 + ... + w_i + n_i

where `w_j` is the disturbance contributed by link j (vibration,
compliance, small misalignment) and `n_i` is the unit's own noise. The
disturbances of different links are independent; the sum is what a
sensor further out inherits from everything before it.

What this single assumption produces:

- *Near sensors are trusted for near joints.* The difference between the
  readings on links i and i-1 contains hinge i's rate and only `w_i`
  plus the two sensor noises. Everything accumulated upstream is common
  to both and cancels. So each joint is informed precisely by the units
  on either side of it, however far out on the chain it is.
- *Distant sensors are trusted little for joints near the root.* Seen
  from link i, the root's motion is buried under `w_1 + ... + w_i`. The
  further out the sensor, the larger that sum. A distant sensor still
  contributes, but mostly by averaging down the sensor noise of the
  nearer ones, never by overruling them.
- *The weighting is per sensor and joint pair,* as the rule requires,
  and it is derived, not assigned: it is what optimal estimation gives
  under this noise model.

Formally, the noise of the stacked readings is not diagonal. The
covariance between sensors a and b is the sum of the link disturbances
up to the nearer of the two, plus the sensor noise on the diagonal. That
structure, the same as a random walk indexed by position along the
chain, is the whole rule.

**The one parameter per link.** Each link needs the variance of its own
contribution `w_j`. Without a structural model it is given by a simple
law and one constant fitted to data:

- proportional to a power of the link's length, linear or quadratic,
  chosen by which fits the recordings better;
- larger on the out-of-plane axes than in plane, so it is a diagonal
  matrix per link in the link's frame, not a scalar;
- multiplied by the current excitation level, as in level 1 above, which
  is what makes the sensor covariance vary at run time.

So the tuning burden is a constant per axis class and the choice of
exponent, not a schedule per sensor.

**How the update is computed.** Two equivalent ways, both small:

- *One block update for all gyroscopes.* With five or six units there
  are fifteen to eighteen gyroscope axes. Their innovation covariance,
  with the accumulated noise structure, is factorised once per step by
  Cholesky. Against a reduced state of about ten this is cheap, and it
  is exact. A unit that is missing or rejected has its rows left out.
- *Sequential, from the root outward, on differences.* Each unit's
  reading is taken relative to the previous unit's, rotated into its
  frame. The noise of these differences is nearly independent from link
  to link, so they can be processed one link at a time. The sensor noise
  of each unit appears in two consecutive differences; that correlation
  is neglected when vibration dominates sensor noise, or handled by
  processing two neighbouring units as one small block.

The block form is the default because it is exact and the sizes allow
it.

**Why not just inflate each sensor's own covariance with distance.**
That treats the accumulated disturbance as independent between sensors.
It gets the root right but is needlessly pessimistic for the outer
joints, because it throws away the cancellation between neighbouring
units that makes each hinge well observed. It is acceptable only as a
crude first step.

**Limits of the rule.**

- It treats link disturbances as white in time and as reaching all
  sensors at the same instant. Vibration is oscillatory and travels at a
  finite speed, so both hold only for motion slow compared with the
  vibration and with the time a wave takes to cross a link. The next
  subsection gives the corrections that do not need a structural model;
  beyond those, modal states become necessary for a link.
- It assumes the disturbance of each link is independent of the others.
  A shock that excites the whole structure at once violates that
  briefly; the excitation scaling covers it by raising all the variances
  together.
- Accelerometers need the same accumulated term plus their lever-arm
  terms.

**If the unit nearest a joint fails.** The next unit out takes over
automatically, with the additional link's disturbance included in its
uncertainty. No reconfiguration is needed, and the larger covariance of
the affected joint is visible to whatever consumes the estimate.

### Finite propagation speed: disturbances that arrive later further out

**The effect.** A mechanical disturbance does not reach every sensor at
once. It travels through the structure as an elastic wave, at the speed
of sound in the material for compression waves (several kilometres per
second in steel or aluminium) and considerably slower for the bending
waves that carry most out-of-plane vibration, whose speed also depends
on frequency. A sensor one link further out sees the same disturbance
later by the link's length divided by the wave speed. These figures are
general physical orders of magnitude, not properties of the machine in
question.

Whether it matters is a comparison of that delay with the sample period
and with the period of the vibration. For short, stiff links the delay
is a small fraction of a millisecond and can be ignored. For links of a
metre or more carrying low-frequency bending waves it can be several
milliseconds, which is several samples at a kilohertz. It is also the
same physics as link flexibility seen from the other side: a structure
behaves as a rigid body only for motion slow compared with the time a
wave takes to cross it.

**What it breaks in the default rule.** The accumulated-disturbance
model assumed that the units on links i and i-1 see the upstream
disturbance at the same instant, so that it cancels in their difference.
With a delay τ between them it cancels only for components slow compared
with τ. For a component at angular frequency ω the uncancelled fraction
is about ω τ. So:

- low-frequency shared disturbance still cancels between neighbours;
- high-frequency shared disturbance does not, and appears in the
  difference as if it were hinge motion;
- the cross-covariance between two sensors is not at zero lag. It peaks
  at the propagation delay between them.

**Three remedies, short of a structural model.**

*1. Compare sensors at matched times.* The unit on link i at time t is
compared with the unit on link i-1 at time t minus τ_i, where τ_i is
that link's propagation delay. Cancellation is restored for the band in
which a single delay is a good description.

- It needs a short history of each unit's readings in a fixed-size
  buffer, and interpolation when the delay is not a whole number of
  samples. Both are allocation-free and bounded.
- The hinge's own rate changes a little over τ_i; that error is the
  angular acceleration times the delay and joins the timing term already
  described.
- The delay per link is one more parameter. It does not have to come
  from material data: the lag at which the cross-correlation between two
  neighbouring units peaks, in recorded data, is the delay.
- Bending waves are dispersive, so one delay fits one frequency band. It
  is chosen for the band that carries most of the vibration energy; what
  remains is treated as below.

*2. Reduce the assumed correlation between sensors to what is actually
coherent.* The cross-covariance between two units in the default model
is multiplied by a coherence factor between zero and one. A factor of
one is the instantaneous model; zero treats the units' disturbances as
unrelated. The factor is measured from the same recordings, as the
height of the cross-correlation peak, and falls with the number of links
between the two units. This costs nothing at run time and stops the
filter relying on a cancellation that is only partly there. It can be
combined with matched times, in which case the coherence is that of the
delay-aligned signals and is higher.

*3. Do not count correlated samples as independent.* Propagation,
reflection at joints and ends, and ring-down make each link's
disturbance correlated from one sample to the next. A filter that treats
it as white averages it away too fast and becomes overconfident. Without
shaping-filter states, the practical measures are:

- inflate the disturbance variance by the factor that corresponds to its
  correlation time, so that many correlated samples carry the weight of
  the few independent ones they amount to;
- or apply the inertial update only about once per correlation time,
  using the average over that interval, which also reduces computation.

The correlation time is read from the autocorrelation of each unit's
signal in the recordings.

**What this asks of the data path.**

- Time stamps on inertial samples have to be accurate to a fraction of
  the propagation delay, otherwise alignment makes things worse. Where
  the delay is well under a sample period, hardware time stamps or a
  common sampling trigger are needed for remedy 1 to be worth doing;
  remedies 2 and 3 do not depend on it.
- The per-unit history buffers are part of the estimator block's state,
  fixed in size from the largest delay and the sample rate.

**A further limit of the accumulation model that this exposes.** The
model has each link's disturbance appearing only on the sensors beyond
it. That is right for slow effects: compliance at a joint rotates
everything outboard and nothing inboard. Waves travel both ways. An
impact at the tip reaches the outermost unit first and moves inward, the
reverse of the assumed order. The coherence factor absorbs this in a
statistical sense, since it only records how related two units are; the
sign of the measured lag between neighbours shows which way disturbances
actually travel, and is a useful diagnostic of where they originate.

**At the actual sample rate: a few hundred hertz.** The inertial units
are sampled at hundreds of hertz, so one sample period is roughly two to
ten milliseconds. That changes which of the effects above matter.

- *Propagation delay is mostly below one sample.* Aligning units at
  matched times (remedy 1) would mean interpolating by a fraction of a
  sample in signals that have little bandwidth to spare. It is dropped
  from the default. The coherence factor (remedy 2) and the treatment of
  correlated samples (remedy 3) remain, and are sufficient: they record
  how much cancellation is really available without needing to resolve
  the delay.
- *Sampling skew between units is now the larger timing error.* Units
  that sample on their own clocks can differ by up to a whole sample
  period, several milliseconds, and drift relative to each other. That
  is larger than any propagation delay and has the same effect on the
  cancellation between neighbours. The measures, in order of
  preference: a common sampling trigger for all units; otherwise
  hardware time stamps and predicting each measurement at its own time;
  otherwise the known bound enters as a timing uncertainty.
- *Aliasing becomes the first thing to check.* Structural vibration
  above half the sample rate does not disappear. Unless it is removed
  before sampling, it folds down and appears as slow apparent motion,
  which is exactly what the filter would attribute to a joint, and it is
  incoherent between units. Inertial sensors generally sample internally
  at a much higher rate and offer a configurable low-pass before
  decimation; that filter has to be set with its cut-off safely below
  half the output rate on every unit, identically. Its delay is then a
  known, common latency of a few milliseconds that the estimator
  accounts for as a measurement delay. This is a configuration check on
  hardware not examined here.
- *Only the low modes are seen.* Vibration below the cut-off is in band
  and is what the accumulated-disturbance model and the correlation
  remedies deal with. Low modes of long links ring for many samples, so
  successive samples are still correlated and remedy 3 still applies.
- *Use integrated outputs where the unit provides them.* If orientation
  is integrated from angular rate, a unit's internally integrated angle
  increments are preferable to sampled rates at this output rate,
  because they do not lose the motion between samples.

**What this means for effort.** At a few hundred hertz, with a reduced
state of about ten and fifteen to eighteen gyroscope axes, the estimator
is computationally trivial on the target. Nothing in this document about
vector width, single precision or factorisation cost is needed to make
it fast enough. The value is in the structure and the noise model: the
reduction to the true degrees of freedom, the accumulated disturbance
with a measured coherence, anti-aliasing and time alignment. The
performance material remains relevant to the control path that runs at
the fast rate, not to this filter.

**Order of work.**

1. Confirm the anti-alias setting and output rate of every inertial
   unit, and how their samples are time-stamped or triggered.
2. From recordings with all units logged together, compute the
   autocorrelation of each unit and the cross-correlation between
   neighbours.
3. Apply the correlated-sample treatment (remedy 3) using the measured
   correlation times.
4. Set the coherence factors (remedy 2) from the measured
   cross-correlation.
5. Consider matched-time alignment only if the recordings show a lag of
   more than a sample between neighbours that is not explained by
   sampling skew.

### The sensor set: one 100 Hz inertial unit per link, plus GNSS

The owner has stated the sensors: an inertial unit on every link, each
sampled at about 100 Hz, and GNSS. This subsection draws the
consequences. Where the GNSS antenna is mounted, what kind of receiver
it is and whether the machine's base moves are not yet known, so several
points are conditional.

**Sampling at 100 Hz.**

- One sample is 10 ms and the usable band ends at 50 Hz. The units'
  internal low-pass must cut off below that, around 40 Hz or lower, the
  same on every unit. Anything the structure does above it must not
  reach the output, or it returns as false slow motion.
- Propagation delay along the links is far below one sample and is
  dropped from consideration entirely. The coherence factor between
  neighbouring units is still measured and used.
- Sampling skew between units of up to 10 ms is significant. The error
  it causes in the difference between two units is the angular
  acceleration times the skew; at one radian per second squared that is
  0.01 rad/s, which is typically several times a gyroscope's noise. If
  the units report over a field bus, the time of reception also jitters
  by an amount of this order. A common trigger, or time stamps taken at
  the sensor, is therefore worth more here than any refinement of the
  noise model.
- The filter runs at the inertial rate. If the controller runs faster,
  it uses the filter's prediction extrapolated to its own time, not a
  faster filter.

**The state is larger than the kinematic freedoms alone.** Earlier
subsections spoke of a reduced state of about ten. That counts the
mechanism's coordinates and rates. Every inertial unit also needs its
gyroscope bias estimated, three states each, and possibly accelerometer
bias. With five or six units that is fifteen to thirty-six further
states, and a moving base adds its pose and velocity. A realistic total
is thirty to fifty states. At 100 Hz this is still computationally
trivial on the target, so the conclusion about effort stands, but:

- the bias states are almost uncoupled from each other and enter each
  measurement through an identity block, so the block structure
  described under "Several poses at once" applies to them and is worth
  using;
- observability of each bias depends on the motion and on the other
  sensors, which the off-line rank check should confirm for the actual
  layout.

**Accelerometers as inclinometers.** With hinge axes that are roughly
horizontal, the direction of gravity seen by each unit gives that link's
inclination, and differences between neighbouring links give the hinge
angles without drift. This is likely the main source of absolute joint
angle. Its errors are the link's own acceleration and vibration, which
the measurement model should include and not merely tolerate:

- the acceleration at the unit's mounting point due to the motion of all
  joints upstream (centripetal and tangential terms through the lever
  arms) is computed from the state by the generated kinematics and
  subtracted in the prediction;
- what remains is covered by the same accumulated-disturbance noise as
  the gyroscopes, with the lever-arm terms noted earlier;
- during strong acceleration the accelerometer covariance is raised, so
  that the gyroscopes carry the estimate through the transient.

**GNSS.**

- *Frames and precision.* Receiver output is converted to a local
  level frame about a fixed origin near the work site. Earth-referenced
  coordinates are large numbers and must be handled in double precision
  before the conversion; this is the nominal-state side of the
  mixed-precision split, not the covariance side.
- *Lever arm.* The antenna is not at the origin of the body it is
  mounted on. Its position is predicted through the generated kinematics
  from the state, which is also what lets a single antenna inform
  orientation when the machine or the link carrying it rotates.
- *Varying covariance.* The receiver reports its own accuracy and fix
  type, and these change abruptly (between a fixed and a float
  carrier-phase solution, or to a code-only solution). The measurement
  covariance follows the report, with a floor, and the fix type is part
  of the measurement's port data.
- *Errors are not white.* Multipath and a converging carrier-phase
  solution give errors that persist for seconds. Treated as white, a
  slowly wandering position error is averaged into false confidence. A
  small slowly varying offset state per axis, or an inflated covariance
  matched to the correlation time, prevents that.
- *Rate and latency.* GNSS arrives much less often than the inertial
  data and tens to hundreds of milliseconds late. The filter keeps a
  short fixed-size history so that a late measurement is applied against
  the state at the time it was valid and its effect carried forward.
  The receiver's time pulse gives the time stamp. At 100 Hz the history
  is a few tens of entries.
- *What it makes observable.* Position, and through the lever arm and
  motion some orientation. Heading from a single antenna needs
  translation or rotation of the antenna; a two-antenna receiver gives
  heading directly. Which case applies is an open point.

**Two antennas on some configurations, one on others.** The owner has
stated that some machine configurations carry dual GNSS and others do
not. Heading is the quantity this changes, and it has to be handled as a
property of the configuration, not as a design choice made once.

- *What the second antenna adds.* The vector between the two antennas is
  observed directly, which gives heading at all times, including at
  rest, and one tilt angle as well. Its accuracy improves in proportion
  to the distance between the antennas. It suffers the same outages and
  multipath as position, and on a machine the working equipment itself
  can block the sky for one antenna in some poses, so its covariance
  varies and it is sometimes absent.
- *One estimator, two sensor sets.* The baseline observation is one more
  factor. On a configuration without the second antenna the factor is
  permanently absent, which the fixed-structure formulation already
  treats as zero weight. Nothing else in the estimator differs. Which
  sensors a machine has is part of its machine file, and in the
  deployment it is an optional alternative (`deployment.md`).
- *What differs in behaviour, and must be visible.*

  | | Dual antenna | Single antenna |
  |---|---|---|
  | Heading at rest | Observed | Not observed; uncertainty grows |
  | Heading in motion | Observed directly and through motion | Through horizontal acceleration and lever-arm motion only |
  | Start-up | Usable immediately | Needs motion before heading-dependent outputs are valid |
  | Frozen linearisation in the marginal prior | Benign | The case that needs first-estimate Jacobians and the spurious-curvature monitor |
  | Weak directions to watch | Few | Heading, and its combinations with gyroscope bias |

  The observability summary that the estimator outputs is what tells a
  consumer which situation it is in. Consumers must be written against
  that summary, not against an assumption about the configuration.
- *Calibration differences.*
  - With two antennas, the direction of the antenna baseline relative to
    the mechanism's plane of motion is a parameter to calibrate. A sweep
    of one hinge with the total station gives the plane's heading in the
    survey frame, the baseline gives it in the GNSS frame, and their
    difference is that parameter. Driving in a straight line gives the
    same without the instrument.
  - With two antennas, heading ties between slew epochs come for free,
    so recapture after a slew is rarely needed
    (`calibration-data.md`). Without, the tie has to come from a slew
    sensor or the integrated yaw rate, or the heading is re-established
    with the instrument.
  - Instrument-free field calibration is therefore considerably easier
    on dual-antenna configurations; the single-antenna ones are the case
    the field procedure has to be designed and proven for.
- *Testing.* Both sensor sets are tested, in simulation and on recorded
  data. The dual-antenna machine can supply the single-antenna test for
  free: the same recording is processed with the baseline factor
  switched off, and the result compared with the full solution, which
  then serves as the reference for heading.

**A moving base changes the picture of the root.** If the machine's base
is not fixed to the ground, the root of the chain is a free body with
six freedoms, located by GNSS and the nearest inertial unit. Its roll
and heading are then the out-of-plane freedoms that only the base has,
and every inertial unit down the chain sees them. That is the earlier
worked case in its most useful form: five or six units all carry
evidence of the base's out-of-plane rotation, each weighted by the
disturbance accumulated between it and the base, and a cascade that
estimated the base from its own sensors alone would ignore all but one
of them.

### Deciding when accelerometer inclination can be trusted

An accelerometer gives a link's inclination only to the extent that what
it measures is gravity. A fixed threshold on how far the reading's
magnitude is from one g, or on the gyroscope's rate, is the simplest
test and has well-known weaknesses: it chatters near the threshold, it
is expressed in raw units that depend on the sensor and on vibration, a
gyroscope bias shifts it, and an acceleration perpendicular to gravity
can leave the magnitude almost unchanged. The alternatives below are
ordered from the most to the least preferable for this system; the
detectors named are from the inertial-navigation literature and are
described from general knowledge.

**1. Ask the right question.** What matters is not whether the link is
motionless but whether the accelerometer's reading is explained. Two
consequences:

- *Model the motion instead of detecting its absence.* The acceleration
  at each unit caused by the motion of the joints upstream is predicted
  from the state through the generated kinematics (the lever-arm terms
  already described) and removed in the measurement model. Inclination
  then remains usable during smooth motion, and only the unmodelled
  remainder has to be dealt with.
- *Let the residual decide.* How well an accelerometer sample agrees
  with the model, given everything else in the window, is its whitened
  residual. That is a properly scaled, dimensionless test statistic
  which already accounts for gyroscope bias, for the predicted motion
  and for the other sensors. A fixed threshold on it has a meaning (a
  false-rejection probability), which a threshold on raw acceleration
  does not.

**2. Weight, do not gate.** Replace the yes-or-no decision by a
covariance that grows smoothly with the evidence of disturbance. The
accelerometer's measurement covariance is raised as a function of the
deviation of the magnitude from one g, the angular rate and angular
acceleration, and the commanded motion. Nothing switches, so nothing
chatters, and a slightly disturbed sample still contributes a little.
This is the varying per-sensor covariance once more. A robust loss on
the gravity factors does the same thing from the residual side.

**3. If a decision is needed, use a windowed likelihood-ratio test.**
For the cases that do need a discrete answer (starting a static
calibration pose, applying a zero-rate update), the standard improvement
on a threshold is a test over a short window that combines both sensors:
the mean squared deviation of the accelerometer samples from a gravity
vector of magnitude one g in their own average direction, divided by the
accelerometer noise variance, plus the mean squared angular rate divided
by the gyroscope noise variance. The sum is compared with a threshold.

- It is a likelihood-ratio test of "stationary" against "moving", so the
  threshold is chosen from a false-alarm probability and is in noise
  units, not raw units.
- Averaging over the window makes it far less sensitive to single noisy
  samples than a per-sample threshold.
- Simpler members of the same family (accelerometer variance alone,
  magnitude alone, rate energy alone) are special cases and are weaker.
- The angular rate used is the bias-corrected one from the estimator.

**4. Make the decision sticky.** Stationarity persists: a machine that
was still a moment ago is probably still. A two-state model with small
transition probabilities, updated by the test statistic, gives
hysteresis and a minimum dwell time in a principled way, and outputs a
probability instead of a flag. The commands are a strong prior for it:
with no motion commanded, the prior for "stationary" is high.

**5. Use the chain, not one sensor.** With a unit on every link, a link
is stationary only if everything upstream of it is. A joint test over
all units (all rates near their biases, and all gravity directions
consistent with one static configuration of the mechanism) is more
powerful than five separate tests, and it also localises motion: the
first unit along the chain that fails the test marks the innermost joint
that is moving.

**6. In the factor graph, let the optimiser choose.** Instead of deciding
before the solve, each interval can carry a factor that is a mixture of
two hypotheses: a tight one (the link did not move: zero rate, constant
gravity direction) and a broad one (it may have). The solve picks the
component that fits, and because it is a window it can revise the choice
for an earlier interval in the light of later data. When the static
hypothesis holds, the zero-rate factor makes the gyroscope biases
directly observable, which is the main benefit of detecting stillness at
all. Mixture factors make the cost non-convex; with a good initial value
from the previous solve and one of the detectors above as the starting
guess this is manageable, and the iteration count stays fixed.

**Vibration.** A machine with its engine running is never still in the
accelerometer's sense, so tests based on variance fail exactly when they
are needed. Two measures: the test is applied to the mean over a window
long compared with the vibration period, so that zero-mean vibration
cancels; and the noise variances in the test are the current ones,
tracked from the high-frequency content of the signals, so that the
threshold rises and falls with the vibration level instead of being
tuned for one engine state.

**Learned detectors.** Classifiers trained on labelled data exist for
this task. They are not proposed here: the methods above are sufficient,
their behaviour can be stated and tested, and a learned detector would
need data and verification of its own.

**Recommendation.** Model the lever-arm accelerations and weight the
accelerometer factors by covariance and robust loss as the default; this
removes the need for a hard decision in normal operation. Use the
windowed test with the two-state model, across the whole chain, where a
discrete answer is needed, in particular for static calibration poses
and zero-rate updates. Add mixture factors if bias estimation during
operation needs the benefit of stillness without a prior decision.

### A truncated factor graph in place of a pure EKF

**Direction from the owner:** the estimator is to be a truncated factor
graph, that is, a fixed-lag smoother, not a pure extended Kalman filter.
The statements below about how such estimators and the tools for them
behave are from general knowledge and were not checked for this plan.

**What it is.** The estimator keeps the states of the last N time steps
(the window) as unknowns. Every measurement and every piece of the
motion model is a factor that ties together the states it involves. Each
cycle, the newest state and its factors are added, the whole window is
re-solved as a non-linear least-squares problem, and the oldest state is
removed by marginalisation, which leaves behind a prior on the oldest
state that remains. "Truncated" refers to that removal: the history
beyond the window survives only as that prior.

A window of one state solved with one iteration is the EKF. The
factor-graph form is therefore a generalisation, not a different
estimator, and the EKF remains available as the smallest configuration.

**What it gains for this problem.**

- *Late measurements need no special machinery.* A GNSS fix that arrives
  a hundred milliseconds late is attached to the state at the time it
  was valid, which is still in the window. The history buffer and
  re-propagation described under "GNSS" are replaced by the window
  itself, provided the window is longer than the largest latency.
- *Re-linearisation.* States on Lie groups, an inclination measurement
  through gravity, and a lever arm through several joints are all
  strongly non-linear. Within the window every factor is re-linearised
  at the current best estimate on each solve. The question of whether to
  re-linearise between sensors, and the order dependence that came with
  it, disappears.
- *Outliers.* A robust loss on GNSS factors down-weights multipath
  without a hard accept-or-reject gate, and a measurement that looked
  wrong when it arrived can regain weight once later data supports it.
- *Slowly varying errors.* Sensor biases and the GNSS offset are
  estimated over the whole window instead of from one step to the next,
  which separates them better from real motion.
- *A natural consistency measure.* Each factor's weighted residual after
  the solve is the generalisation of the parity residual: it says how
  well that sensor agrees with everything else in the window.

**What carries over unchanged.** Everything about the model:

- the reduction to the true degrees of freedom. The unknowns per time
  step are the reduced coordinates, rates and biases, not a pose per
  link;
- the generated measurement functions and their Jacobians. A factor is a
  residual function and its derivative, which is what the symbolic
  generator produces;
- the noise model: per-sensor varying covariance, the disturbance that
  accumulates along the chain with its measured coherence (one joint
  factor per time step for all inertial units, whitened by a factor of
  its covariance), the consider parameters for imperfect geometry, and
  the treatment of correlated samples.

**What it costs and how to keep it real-time safe.**

- *Arithmetic.* Factors connect only states that are adjacent in time,
  so the normal equations are block tridiagonal. Solving them costs the
  window length times the cube of the per-step state size, not the cube
  of the whole. With about fifty states per step and a window of twenty,
  one iteration is a few million operations; at 100 Hz that is well
  within the target's capacity.
- *Fixed structure.* The set of factors per step is fixed by the sensor
  layout. The sparsity pattern is then known at build time and the
  solver is a block-tridiagonal Cholesky over fixed-size blocks in
  fixed-size arrays, with no allocation and a fixed operation count. A
  missing measurement is a factor with zero weight, not a change of
  structure.
- *Fixed iteration count.* One or two Gauss-Newton iterations per cycle,
  warm-started from the previous solution. Stopping on a convergence
  test would make the time data dependent. An iteration budget that is
  not enough is reported, and the partial result is still a valid
  improvement.
- *General-purpose smoothing libraries are not suitable as they are.*
  Incremental solvers re-order and re-eliminate variable amounts of the
  problem per step and allocate as they go, which gives variable timing.
  The planned symbolic generator ships its own optimiser aimed at
  fixed-structure problems, which may fit; whether it is allocation-free
  and bounded in the sense required here has to be checked. The
  fallback, a hand-written block-tridiagonal solver, is small.
- *Marginalisation.* The prior left by a removed state is a linearised
  factor. The variables it involves must keep the linearisation point
  they had at that moment, or the estimator becomes overconfident. This
  is a known subtlety and has a standard remedy; it needs a test of its
  own.
- *Covariance of the current state* comes out of the factorisation of
  the last block at no extra cost, so consumers that want uncertainty
  still get it.

**Choosing the window.** Long enough to contain the largest measurement
latency and a few correlation times of the slow errors; no longer,
because cost is proportional to it. Not every 100 Hz sample needs its
own state: inertial data between two states can be combined into one
factor, so that states are kept at, say, 10 to 20 Hz and a window of
twenty covers one to two seconds.

**Where it runs.** Two arrangements are compatible with the framework:

- in the real-time thread at the inertial rate, if the worst-case solve
  time fits the cycle with margin. The bounded formulation above is what
  makes that a question of measurement rather than hope;
- or as a separate node at lower priority, with the real-time thread
  running a light predictor that propagates the most recent smoothed
  state to the present. The link between them is a latest-value channel.
  This decouples the controller's timing from the solver completely, at
  the cost of the correction arriving one solve later.

The second is the safer starting point; the first can follow once the
solve time has been measured on the target.

**Testing.** With the window set to one and a single iteration the
estimator must reproduce a reference EKF within tolerance, which gives a
fixed point to test against. Larger windows are then judged on recorded
data and in simulation by estimation error and consistency, and on the
target by worst-case solve time.

### Putting measurements onto a time grid

**Aim.** States exist only at grid times, for example every 50 or
100 ms, while inertial samples arrive at 100 Hz from several units with
their own sampling instants and GNSS arrives late at its own times. The
number of states sets the cost of the solve; the number of raw
measurement factors sets the cost of re-linearising. Both are reduced by
turning many raw samples into few factors attached to grid states,
without pretending a sample was taken at a time it was not.

Three mechanisms, each suited to a different kind of measurement.

**1. Integrate rates between grid states (pre-integration).** For each
inertial unit, the gyroscope samples between two consecutive grid times
are integrated into one rotation increment, and that single quantity
becomes a factor between the two states.

- Each unit is integrated on its own time stamps. The first and last
  samples of an interval are split in proportion to the part that falls
  inside it, so sampling skew between units is handled exactly and every
  raw sample contributes to exactly one interval.
- The factor's covariance is accumulated along with the increment.
- The increment depends on the gyroscope bias. Its sensitivity to the
  bias is accumulated too, so that when the solve changes the bias
  estimate the increment is corrected to first order instead of being
  re-integrated from raw samples. Only a large change of bias triggers a
  re-integration.
- Where a unit supplies integrated angle increments itself, those are
  summed directly.

This replaces ten or so rate factors per unit per interval by one, and
it is the main reduction.

**2. Combine repeated observations of a slowly varying quantity.**
Accelerometer inclination is observed a hundred times a second but
changes slowly. Within one interval the samples are brought to the grid
time using the motion the gyroscopes measured over the same interval,
and combined with weights from their covariances into one factor at
that state.

- The combined factor's covariance must not shrink as if the samples
  were independent. With disturbances correlated over a time T, an
  interval of length Δ holds about Δ / T independent samples, not Δ
  times the sample rate. This is the correlated-sample treatment
  described earlier, applied at the point where samples are merged.
- Resampling helps the noise model here: once the interval is longer
  than the disturbance's correlation time, the combined factors of
  successive intervals are close to independent, which is what the
  estimator assumes. Choosing the grid spacing above the measured
  correlation time is therefore a design rule, not only a cost saving.

**3. Attach a measurement at its true time by interpolating between the
two neighbouring states.** A GNSS fix valid at a time between two grid
states becomes a factor on both of them, with the state at the
measurement time obtained by interpolation.

- Because the state includes rates, the interpolation uses position and
  rate at both ends (a constant-acceleration-noise model between grid
  states), not a straight line between poses. On the group-valued parts
  it is done in the tangent space.
- The factor involves only two adjacent states, so the block-tridiagonal
  structure is untouched.
- Latency is handled by the same mechanism: the fix attaches to whichever
  pair of states brackets its time stamp.

**What stays as it was.** The disturbance that accumulates along the
chain keeps its structure under integration and averaging: the combined
quantities of the units along the chain are correlated in the same
pattern, with smaller magnitude. The single joint factor per interval
for all inertial units, whitened by a factor of that covariance, remains
the form.

**Choosing the grid.**

- Spacing above the disturbance correlation time and well below the time
  scale of the motion to be estimated; a few intervals per period of the
  fastest motion of interest.
- A fixed, uniform grid. Grid times are multiples of a base period, so
  the structure of the problem never changes.
- The newest grid state lags the present by up to one interval. The
  real-time predictor carries the estimate from there to the current
  instant using the latest inertial samples.

### Removing the oldest state

When a new grid state is added, the oldest one leaves the window. How it
leaves decides whether the estimator stays consistent.

**The method: marginalisation, in square-root form.** The oldest state
is eliminated from the linearised problem, and what the eliminated
factors implied about the next state is kept as a prior on that state.

- Only three kinds of factor involve the oldest state: the prior it
  inherited, its own measurements, and the factors linking it to the
  next state. Because the problem is a chain, eliminating it produces a
  prior on the next state alone, a dense block of the per-step state
  size. There is no spread of fill-in to other states.
- This step is the information-filter recursion. The window is a
  smoother running in front of a Kalman filter that summarises
  everything older.
- It is carried out by an orthogonal factorisation of the stacked,
  whitened Jacobians of those factors, not by forming and subtracting
  normal-equation blocks. The prior is stored as a triangular
  square-root information matrix with its right-hand side. This keeps it
  positive semi-definite by construction and is the well-conditioned
  form.
- Cost is the cube of the per-step state size, once per grid step, with
  a fixed operation count.

**The linearisation point must be frozen.** The prior is a linearised
summary, computed about the values the states had when the oldest one
was removed. It is stored together with that reference value of the next
state, and its residual is evaluated as the difference, on the manifold,
between the current value and that reference.

The subtlety: other factors on the same state are still re-linearised
at its current value. If the prior was linearised at one value and the
neighbouring factors at another, the estimator acquires information in
directions that the measurements cannot actually observe, and becomes
overconfident. Here that concerns heading in particular, which a single
GNSS antenna does not observe while the machine is stationary. The
standard remedy is to evaluate the Jacobians, with respect to a state
that carries a marginalisation prior, at the frozen reference value for
every factor touching it (first-estimate Jacobians). It costs nothing at
run time and is the default.

**Decisions that become final at the window's edge.**

- A measurement folded into the prior can no longer be re-weighted or
  rejected. The window is the last opportunity to decide that a GNSS fix
  was an outlier, using everything that arrived after it. Robust-loss
  weights are frozen at the value they have when the factor is
  marginalised.
- For the same reason the window should be at least as long as the time
  it takes for a bad measurement to become recognisable.

**States that outlive the window.** Biases and other slowly varying
quantities can be modelled in two ways.

- *One copy per grid state, linked by a slow random walk.* They are then
  ordinary members of the per-step state and are marginalised with it.
  The structure stays block tridiagonal.
- *One shared copy for the whole window.* Fewer unknowns, but the shared
  variables connect to every state, which adds a border to the
  block-tridiagonal system. The solver handles a border by one extra
  elimination step; marginalising a time state then leaves a prior that
  couples the next state with the shared variables.

The first is simpler and is the default; the second is worth it only if
the number of slow states dominates the per-step state, which with three
to six bias states per inertial unit it may.

**Degenerate directions.** At start-up, and whenever something is
unobservable for a while, the block being eliminated is singular. The
square-root form tolerates this, and a weak initial prior on every state
keeps the factorisation well defined without materially affecting the
estimate.

**What not to do.**

- *Dropping the oldest factors without a prior* discards information;
  the estimate then depends only on the window, and drifts in whatever
  the window alone cannot observe.
- *Fixing the oldest state at its estimate* treats it as perfectly
  known, which makes everything after it overconfident. It is acceptable
  only for quantities that are so well observed that their uncertainty
  is negligible.

**Tests specific to this step.** With linear models, the windowed
estimator with marginalisation must reproduce the Kalman filter exactly,
for any window length. With the real models, the consistency statistics
in simulation, with and without first-estimate Jacobians, show whether
spurious information is being gained, particularly in heading during
stationary periods.

### Higher-order derivatives for the poorly observed dimensions

Some dimensions of the state are observed weakly: heading, the base's
roll, an out-of-plane freedom near the root, and the biases tied up with
them. "Higher-order derivatives" can help with these in three distinct
ways, recorded here because they do not conflict. The owner has since
clarified the intent: the second derivative of the cost, its curvature,
as the indicator of which dimensions are poorly observed. That is the
subject of the next subsection; the three readings below remain as
complementary measures.

**1. Higher time derivatives in the state and the motion prior.**

So far the state holds coordinates and rates, with the prior between
grid states saying that acceleration is unpredictable noise. Adding the
accelerations as states, with the prior on jerk instead, changes what
the accelerometers can do:

- an accelerometer away from a joint axis measures, besides gravity, the
  tangential term (angular acceleration times lever arm) and the
  centripetal term (rate squared times lever arm). Without acceleration
  in the state, the tangential term can only be approximated or treated
  as disturbance. With it, the term is a direct measurement of a state;
- the weak dimensions are then informed through two channels with
  different error characteristics: gyroscopes through the rate,
  accelerometers through the acceleration and the square of the rate.
  The window ties the two together by integration;
- the interpolation used for resampling becomes higher order, which
  makes measurements placed between grid states more accurate.

Costs and cautions: one more state per coordinate that gets this
treatment, and an acceleration state that no sensor observes only adds
noise. So it is applied selectively, to the coordinates for which a
lever arm makes the acceleration observable, not to all of them.

**2. Sensors of higher derivatives, where the lever arm works in favour.**

The distance rule so far was about noise: disturbance accumulates
outward. For accelerometers there is an opposite effect on the signal.
For rotation about an axis near the root, the centripetal and tangential
accelerations seen by a unit grow in proportion to its distance from
that axis. A unit far out on the chain sees a large acceleration from a
small out-of-plane or slewing rotation of the root, where its gyroscope
sees only the same rate as every other unit, buried under more
vibration.

- Signal and disturbance both grow with distance, so there is a best
  distance, not a monotonic rule. The accumulated-disturbance model
  supplies the noise side and the generated kinematics the signal side;
  the estimator then weights each unit correctly without a special case.
- For the base, pre-integrating the full inertial measurement (velocity
  and position increments, not only rotation) makes heading observable
  whenever the machine accelerates horizontally: the direction of the
  acceleration in the unit's frame is compared, across the window, with
  the direction implied by successive GNSS positions. With a single
  antenna this is the main source of heading while the machine moves,
  and the estimator correctly reports growing heading uncertainty when
  it does not.
- Accelerometers respond to vibration far more strongly than gyroscopes
  do, since acceleration grows with the square of frequency for a given
  amplitude. The low-pass before sampling matters even more for them,
  and their vibration term in the noise model is larger.

**3. Higher-order terms of the measurement model, where uncertainty is
large.**

Linearisation is accurate when the uncertainty is small compared with
the curvature of the model. For a dimension with a large uncertainty,
such as heading before the first horizontal acceleration, it is not: the
first-order model gives a biased prediction and an innovation covariance
that is too small.

- Inside the window this is mostly handled already, because every solve
  re-linearises at the current estimate.
- The weak point is the marginalisation prior, which is linearised once
  and frozen. A dimension that is still very uncertain when its state
  leaves the window is summarised badly. Two remedies: keep such a
  dimension out of the per-step state and carry it as a variable shared
  across the window, so that it is never frozen until it has been
  observed; or include the second-order term for that dimension when
  forming the prior.
- Second derivatives of the measurement functions are available from the
  symbolic generator. Restricted to the few weakly observed dimensions
  they are cheap, and they give both the bias correction to the
  predicted measurement and the extra term in its covariance.
- For the very first fix on a dimension with no prior information at
  all, a small set of hypotheses or sigma points over that dimension is
  simpler and more reliable than any Taylor expansion.

**Suggested use.** Apply the second reading first, since it uses data
already being collected: pre-integrate the full inertial measurement of
the base unit, and include the lever-arm accelerations in the model of
the outer units. Add acceleration states (the first reading) for the
freedoms that those terms make observable. Keep the third in reserve for
start-up and for any dimension that the consistency statistics show to
be overconfident.

### Low curvature of the residual as the indicator of poor observability

**The idea.** The estimator minimises a sum of squared, whitened
residuals. The second derivative of that cost with respect to the state,
its curvature, says how sharply the data determine each direction of the
state. A direction along which the cost is nearly flat is one the
measurements in the window barely constrain. Curvature is therefore a
direct, continuously available measure of observability, obtained from
quantities the solver already computes.

**What is computed.**

- The Gauss-Newton approximation of the curvature is the information
  matrix, the product of the whitened Jacobian with its transpose. The
  solver has it, in factored form, at every solve.
- Observability belongs to directions, not to individual coordinates.
  Heading may be well determined and a gyroscope bias well determined
  while one combination of the two is not. So the object of interest is
  the set of directions of smallest curvature, not the diagonal.
- Curvature has to be judged against a scale to mean anything, because
  the state mixes angles, lengths, rates and biases. Two usable scales:
  - against the prior: the measurement information expressed in
    coordinates in which the prior is the identity. A value much less
    than one in some direction says this window's measurements add
    little to what was already believed there. It is dimensionless and
    can carry a fixed threshold;
  - against the state scaling already recommended, when the prior itself
    is weak, as at start-up.

**Computing it without unbounded work.** A full eigen-decomposition is
an iterative algorithm whose running time depends on the data, which
does not belong in a real-time thread. Three bounded alternatives, in
order of preference:

- *Test known candidate directions.* The off-line observability
  analysis of the model (the rank check described earlier) lists the
  directions that can become weak for this mechanism and sensor layout:
  heading, base roll, particular combinations with biases. The curvature
  along each is one small matrix-vector product per solve. This is exact
  for those directions, deterministic and cheap.
- *Read it from the factorisation.* With a pivoted orthogonal
  factorisation the trailing diagonal entries of the triangular factor
  bound the smallest curvatures and identify the directions involved.
- *A fixed number of iterations* of a symmetric eigenvalue method,
  warm-started from the previous solve, when unknown weak directions
  have to be found. If the estimator runs beside the real-time thread,
  this is where it belongs.

**What the estimator does with a weak direction.**

- *Reports it.* The covariance of the output is large in that direction,
  and a compact observability summary (the smallest scaled curvature and
  which named direction it belongs to) is an output of the block, so
  that a consumer can refuse to act on, say, heading while it is not
  determined.
- *Does not move along it on the strength of noise.* A Gauss-Newton step
  in a nearly flat direction is the ratio of two small numbers and can
  be large and arbitrary. The step is restricted to the well-determined
  directions, and in the weak ones the estimate stays where the prior
  puts it. A sound prior already has much of this effect; the explicit
  restriction matters at start-up and through long unobservable periods,
  when the prior in that direction is weak too.
- *Does not freeze it prematurely.* A direction that is still flat when
  its state reaches the end of the window would be summarised by a
  marginalisation prior linearised at an essentially arbitrary value.
  Such a direction is carried as a variable shared across the window, or
  its marginalisation is deferred, until its curvature has risen above
  the threshold.
- *Can ask for excitation.* The summary says what kind of motion would
  make the direction observable (horizontal acceleration for heading,
  rotation about a particular joint for a bias), which an operator
  prompt or a calibration routine can use.

**Curvature that should not be there.** The same quantity works as a
check in the other direction. Some directions are known, from the
model, to be unobservable in certain conditions: heading with one
antenna while the machine stands still. If the curvature along such a
direction grows anyway, the estimator is manufacturing information,
which is the signature of inconsistent linearisation. Monitoring this is
the run-time counterpart of the first-estimate-Jacobian test, and it is
cheap because the direction is known in advance.

**Where genuinely higher-order information comes in.**

- *Observability through motion.* Many directions are unobservable from
  one instant's measurements and become observable only through how the
  outputs change over time; formally, through higher-order derivatives
  of the outputs along the motion. In a windowed estimator this needs
  nothing special: once the window spans enough motion, the direction
  acquires ordinary first-order curvature. This is the main reason a
  window observes more than a filter step does, and it is an argument
  for window length in the weak directions.
- *The neglected second-order term.* The Gauss-Newton curvature omits
  the term made of the residuals times the second derivatives of the
  measurement functions. Where the first-order curvature vanishes, that
  term is all that is left, but at the solution the residuals are at the
  noise level, so its sign and size are essentially noise. It should not
  be relied on to make a direction observable.
- *The shape of the cost along a weak direction.* What second and higher
  derivatives do legitimately provide is a better description of the
  uncertainty where the quadratic approximation is poor. Along a single
  weak direction the cost can simply be evaluated at a handful of points:
  a one-dimensional scan costs a few extra residual evaluations and
  reveals a flat valley, a skewed one, or two separate minima, such as
  heading being ambiguous by half a turn. That is more informative and
  more robust than any Taylor term, and it is affordable precisely
  because the weak directions are few.

**Relation to the parity residual.** The two diagnostics are
complementary views of the same whitened Jacobian. Directions in
measurement space that no state can produce give the parity residual:
the model cannot explain the data. Directions in state space that no
measurement responds to give low curvature: the data cannot determine
the model. One flags a missing freedom, the other a redundant or
unexcited one.

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

## Identification and calibration with the same graph

**Direction from the owner:** the factor graph used for estimation is
also to be used for system identification and calibration: mounting
offsets, sensor intrinsics, and non-linear models such as deadband and
backlash.

**The principle: one model, and each parameter has a role.** Every
quantity the residual functions depend on, other than the time-varying
state, is a parameter. The same generated residual functions are used in
all cases; what changes is how a parameter is treated:

| Role | Treatment | Typical use |
|---|---|---|
| Fixed | A constant | Values known exactly or irrelevant to accuracy |
| Considered | Not estimated; its stated uncertainty is carried into the estimate | Calibrated values with residual uncertainty, during normal operation |
| Calibrated off line | An unknown shared by the whole of a recording, solved in a batch | Mounting offsets, intrinsics, actuator models |
| Tracked on line | A slowly varying unknown shared across the window | Biases, and a few parameters that drift or wear |

The role is configuration, not code. The residuals, their Jacobians and
the noise model are identical in the calibration tool and in the running
estimator, so the two cannot drift apart, and a calibration is by
construction a calibration of the model the estimator actually uses.

**Batch calibration.** The same graph over a whole recording, with the
states at every grid time and the parameters as shared unknowns.

- It runs off line or in a non-real-time node. Allocation and iteration
  to convergence are allowed there, and robust losses can be used
  freely.
- The structure is the block-tridiagonal chain with a border for the
  shared parameters. Eliminating the time states leaves a small dense
  system in the parameters alone, so a long recording costs time
  proportional to its length, not to its cube.
- Its result is a parameter value with a covariance. The value goes into
  the machine file; the covariance becomes that parameter's uncertainty
  in the considered role. The record of which recording produced it, and
  the fit statistics, are kept with it.

**Identifiability is the curvature test again.** Whether a parameter can
be determined from a recording is the curvature of the batch cost along
it, and the off-diagonal terms say which parameters are confused with
which. This is the same indicator as for weak state directions, and it
serves three purposes:

- before calibrating, to design the motion: candidate trajectories are
  scored by the smallest curvature they produce, so that a calibration
  routine exercises what needs exercising;
- after calibrating, to report honestly which parameters were determined
  and which merely kept their prior;
- on line, to update a tracked parameter only while the current motion
  makes it observable, and hold it otherwise.

Some combinations can never be separated from data and need a
convention. The rotation of an inertial unit about its link's hinge axis
is indistinguishable from an offset in that joint's zero position; an
overall rotation of every frame changes nothing. These show up as
exactly flat directions and are removed by fixing one of the parameters
involved.

**Mounting offsets and geometry.** Orientation of each inertial unit on
its link, its position along the link, the GNSS antenna's lever arm,
joint zero offsets, hinge axis directions and link lengths. Orientations
are group elements and are updated in their tangent space, like the
state. These are the consider parameters of "Uncertainty that grows with
the distance between sensor and joint", now estimated: after
calibration their uncertainty is small, and the distance-dependent term
shrinks with it.

**External observations of how frames relate.** During calibration the
owner intends to use manual and survey measurements: a tape measure
between known points in different frames, a plumb bob to align a link
with gravity, and three-dimensional positions from a total station.
These fit the graph as further factor types. They matter because they
supply what the on-board sensors cannot: inertial units do not observe
lengths or where a joint axis is, and GNSS observes one point. The
external observations fix scale, tie the geometric frames of the links
to the sensor frames, and remove the conventions that would otherwise
have to be imposed.

*Reference points.* Each link's description in the symbolic model names
its reference points (pin centres, marked targets, the tool tip) with
coordinates in that link's frame. A coordinate taken from drawings is a
parameter with a prior from the manufacturing tolerance, not an exact
value, unless it defines the frame.

*Static poses.* These observations are taken with the machine at rest.
Each is attached to a pose node, a state with no dynamics, shared by all
observations made in that pose. While the machine is still the inertial
units contribute strong factors of their own: the averaged accelerometer
reading is the direction of gravity in each unit's frame, and the
averaged gyroscope reading is its bias. A calibration data set is a
number of static poses together with the dynamic recordings, solved as
one problem.

*The factor types.*

| Observation | Residual | What it mainly determines | Things to model |
|---|---|---|---|
| Tape measure between a point on one frame and a point on another | Difference between the measured length and the distance between the two points computed through the kinematics. One scalar | Link lengths, pin positions, joint zero offsets (through the change of a distance with joint angle) | Accuracy of a millimetre or a few; sag, tension and temperature on long spans; which points were actually used |
| Plumb bob alignment of a link with gravity | The angle between a named axis of the link and the vertical. Two components | The orientation of the link's geometric axis relative to gravity in that pose, and so the mounting orientation of that link's inertial unit about two axes | Accuracy of the order of a milliradian; which edge or pair of points defines the axis; that it says nothing about rotation about the vertical |
| Total station position of a target on a link | Difference between the observed and predicted position of the target in the instrument's frame. Three components | Positions of points on links, antenna lever arm, kinematic parameters, base pose in the survey frame | The prism's offset from the named point; the pose of the instrument |

Further observations of the same kind cost nothing to add, since each is
one more residual function over the same kinematics: a digital level or
protractor on a link; a reference point touched to a fixed mark from
several different configurations, where the mark's position is unknown
but the same each time; a point constrained to a known plane or line.
The repeated-touch observation is especially useful because it needs no
instrument and constrains the whole chain at once.

*Total station specifics.*

- The instrument measures two angles and a distance. Using those raw
  quantities with their own accuracies is better than using converted
  coordinates with an assumed equal accuracy in all directions; the
  distance and the angles have very different error characteristics, and
  the difference grows with range.
- The instrument's frame is an unknown in the problem. A levelled
  instrument has four unknowns, its position and its azimuth, because
  its vertical is gravity. That makes each observation also a tie to
  gravity, complementing the plumb bob.
- Observing the GNSS antenna, or surveyed control points, with the same
  instrument connects the survey frame to the GNSS frame.
- If the instrument is moved during a session, each set-up is its own
  frame, tied to the others by targets observed from more than one
  set-up.

*Human error.* Manual observations contain blunders: a wrong point, a
misread digit, a unit slip. They are few and each carries much weight,
so a mistake does real damage.

- Every observation is entered with its stated accuracy, the names of
  the points or axis involved, and the pose it belongs to.
- The procedure asks for more observations than the minimum, so that
  each is checked by the others.
- After the solve, each observation's normalised residual is reported,
  and the solve is repeated with each manual observation left out in
  turn; one whose removal changes the result markedly is flagged.
- A robust loss limits what a single bad value can do before it is
  found.

*Planning the session.* Because observability is curvature, the set of
observations can be assessed before anyone goes to the machine: given a
list of proposed poses and measurements, the tool reports which
parameters would be determined and how well. It can also propose the
next most informative observation, which keeps a session short. The same
analysis shows which conventions still have to be imposed, if any,
after the external observations are included.

*Data handling.* The data model for these records is in
`calibration-data.md`. The observations, the static pose recordings, the
instrument set-ups and the result are stored together as one calibration
record per machine, so that a calibration can be reproduced, extended
with later observations, or re-solved when the model changes.

**Tracking total station: lock and latency.** The owner's instrument can
track a prism, single or multiple prisms can be fitted, and the time it
takes to lock on is a problem. Consequences for how it is used:

- *Two different delays.* Acquisition time, the seconds it takes to find
  and lock a prism after pointing at a new one or after losing it,
  limits how many observations fit in a session and rules out switching
  prisms during motion. Measurement latency, the delay between the
  instant a tracked position is valid and the time stamp it arrives
  with, corrupts dynamic data: the error is the target's speed times the
  delay, several centimetres at a metre per second for a delay of some
  tens of milliseconds. Tracking instruments also commonly take the
  angles and the distance at slightly different instants, which adds an
  error of the same kind.
- *Latency is a calibration parameter.* As for the inertial units, the
  instrument's time offset is estimated from motion, with separate
  offsets for angles and distance if the data shows the need. Until it
  is known, dynamic validation uses slow motion or stop-and-go.
- *Lock status is data.* Samples taken while the instrument is searching
  or has just re-acquired are marked invalid from the instrument's own
  status and not used. A static observation requires the pose to be held
  longer than the acquisition time.
- *Order the observations to avoid re-acquisition.* The session plan
  sequences poses and prisms so that the instrument turns as little as
  possible between observations and line of sight is never lost in
  between.
- *Several prisms.* Three prisms on one link give that link's full pose,
  but one instrument observes them one after another, so this is a
  static-pose technique. With several prisms in view the instrument may
  lock onto the wrong one. Each observation is therefore checked against
  the position predicted for every candidate prism and assigned to the
  one it fits; an observation that fits none, or two equally, is
  rejected.

**Calibration without a total station.** A stated goal is field
calibration that does not need the instrument, to make it cheaper. The
approach is two tiers that share one model.

*Reference calibration, with the instrument.* Done once per machine or
per machine type, at the factory or depot. It determines everything,
including the parameters that field methods cannot see, and yields their
uncertainties.

*Field calibration, without it.* Re-determines only the parameters that
change in the field (a replaced sensor, a re-mounted unit, a changed
tool, wear) and treats the rest as considered parameters carrying the
uncertainty from the reference calibration. Its observations:

- *Self-calibration from motion, needing no equipment.*
  - Moving one joint at a time: the axis about which a link's gyroscope
    sees rotation is that hinge's axis expressed in the sensor's frame,
    which gives the unit's mounting orientation about two axes.
  - Static poses: the accelerometers give gravity in each sensor frame,
    which with the hinge axis completes the mounting orientation, and
    the gyroscopes give their biases.
  - Slewing the base with GNSS running: the antenna traces a circle,
    which locates the slew axis and the antenna's lever arm from it.
  - Time offsets from any sufficiently lively motion.
- *Scale, which motion cannot supply.* Inertial data does not determine
  lengths, so at least one external length is needed: the tape between
  reference points, or the repeated touch of the tool point on a fixed
  mark, or on a mark whose position is known.
- *Low-cost absolute references.* A GNSS rover placed on reference
  points or at the tool point is a position observation like the total
  station's with a larger uncertainty, in a frame that is already the
  machine's. A laser level or a string line constrains points to a known
  plane or line. The plumb bob and a digital level tie links to gravity.

*Whether it is good enough is computed before it is tried.* For a
proposed field procedure, the curvature analysis gives the covariance of
every parameter it leaves, and the error prediction of the previous
paragraphs turns that into expected tool-point accuracy over the
workspace. That number, compared with the specification, says whether
the instrument can be dispensed with, and which additional cheap
observation would help most if not.

*Validating without the instrument.* Independent checks that remain
available: touching marks that were not used in the fit; tape
measurements between points that were not used; a GNSS rover at the tool
point; tracing a surface known to be flat or level. These are weaker
than an independent three-dimensional reference, and that should be
stated with the result. The fit's own residual statistics are a
necessary check but not a validation.

*Establishing trust in the field procedure.* The instrument is the means
of proving that it is not needed. On a few machines, both calibrations
are done and compared: the field result should agree with the reference
within the uncertainty the field procedure reports. Once that holds
across machines and repeated trials, the reported uncertainty of the
field procedure can be relied on without the instrument present.

**Single-joint motion: a tracked prism's point cloud and the gyroscope
axis.** The owner's intended procedure, as understood here: move one
joint alone while the total station tracks a prism outboard of it and
the inertial units record, and assume that the motion is a rotation
about one fixed axis. In robot calibration this is known as circle-point
analysis, and it suits this machine and this instrument particularly
well.

*What each data stream gives under the single-axis assumption.*

- The prism's positions lie on a circle. The circle's plane gives the
  direction of the joint axis in the instrument's frame, its centre
  gives a point on the axis, and its radius gives the prism's distance
  from the axis.
- Every inertial unit outboard of the moving joint sees an angular
  velocity along that same axis. The direction of the measured rate,
  averaged with weight on the faster samples and after removing the
  bias found from the still periods before and after, is the joint axis
  in that unit's own frame. One sweep therefore gives two of the three
  mounting angles of every unit beyond the joint.
- Together they express one physical axis in the instrument's frame and
  in each sensor's frame, which ties the two.

*Why it suits a tracking instrument with latency.* A delay in the
tracked positions moves each point along its path. On a circle that is
a shift along the circle: the plane, the centre and the radius are
unaffected to first order. So the geometry is recovered correctly even
before the instrument's time offset is known, which is exactly the
situation in the field. What latency does affect is the pairing of
position with time; see the time-offset point below. The exception is a
difference in timing between the instrument's angle and distance
readings, which moves points off the circle in a direction that depends
on the direction of travel. Sweeping the joint both ways makes that
error reverse, so it can be recognised, averaged out or estimated.

*What a sweep in each direction also provides.*

- *The instrument's time offset.* The angle travelled along the arc,
  against time, is compared with the angle obtained by integrating the
  gyroscope. The shift that aligns them is the offset.
- *Gyroscope scale factor.* The total arc angle from the instrument is
  an independent angle reference; its ratio to the integrated rate is
  the scale factor about that axis.
- *Backlash and compliance.* A consistent gap between the forward and
  the return circle is play in the joint. A radius or plane that varies
  smoothly with the joint angle is deflection under the changing
  gravity load.
- *The plane heading* of the current slew epoch, since for the hinge
  joints the axis is the normal of the plane of motion
  (`calibration-data.md`).

*From axes to the mechanism.* Doing this for each joint in turn gives
every joint axis as a line in the instrument's frame, in one slew epoch.
The distances and angles between consecutive axes are the kinematic
parameters. For a planar mechanism the hinge axes are parallel, and the
perpendicular distances between them are the link lengths.

*Limits to plan around.*

- *Arc length.* The centre and radius of a circle are poorly determined
  from a short arc, and badly correlated with each other. The axis
  direction needs the points to spread widely in the plane. Use as much
  of the joint's range as possible and mount the prism far from the
  axis. The observability report gives the resulting uncertainty before
  the sweep is made.
- *Parallel axes add no orientation.* On a planar mechanism every hinge
  gives the same axis direction, so sweeping more hinges does not supply
  the third mounting angle of a unit. That comes from a second,
  non-parallel axis, which the slew provides, or from gravity in the
  static poses. Slewing is therefore part of the inertial calibration
  even though it ends the slew epoch.
- *An axis parallel to gravity* leaves the accelerometers uninformative
  about rotation around it, so for the slew axis the complementary
  information has to come from a hinge sweep or the instrument.
- *The single-axis assumption has to be checked, not trusted.* The usual
  violations:
  - another joint creeps while it is supposed to be held, which
    hydraulic cylinders do under load;
  - the base rocks as the centre of gravity moves with the link,
    especially on soft ground or tyres;
  - the joint has play, so the axis is not fixed.

  The evidence is in the data: the scatter of the gyroscope's rate
  direction, relative rotation between units on links that should be
  rigidly related, the base unit's tilt during the sweep, and the
  out-of-plane residual of the circle. Where the base does rock, its
  measured tilt is used to correct the prism positions instead of
  discarding the sweep.

*One step, not two.* The classical method fits a circle per joint and
then derives parameters from the fitted axes. Here the raw tracked
positions and inertial samples go into the graph directly, with the
single-axis assumption as the factor that makes all samples of a sweep
share one axis. That uses the data once, with correct weights, and
carries the uncertainty of each axis into everything derived from it.
The separate circle fit is kept as the starting value for the solve and
as an immediate check in the field that the sweep was good.

**Choosing validation poses: where the predicted error is largest.** A
calibration is validated by an independent measurement, typically the
total station on the tool point, in poses that were not used for
fitting. Which poses to use should not be left to convenience: the
calibration itself predicts where it is weakest, and those are the
poses that test it hardest.

*The prediction.* For a quantity that matters, usually the position of
the tool point, and for any configuration of the machine, the expected
error is obtained by pushing the uncertainties through the generated
kinematics:

- the calibrated parameters' covariance, through the sensitivity of the
  tool point to each parameter in that configuration. This term grows
  with reach, because angular errors near the root are multiplied by the
  distance to the tool;
- the estimator's own uncertainty of the joint coordinates in that
  configuration, given the sensors, through the kinematic Jacobian. This
  depends on the pose too: inclination from gravity is poor for a link
  that is near vertical about the relevant axis, for example;
- both together give a covariance of the tool point per configuration.
  Its largest principal value, or its size along the direction that
  matters for the task, is the predicted error there.

*The search.* The predicted error is a smooth function of the joint
coordinates, with derivatives available from the symbolic model. The
workspace is sampled within joint limits, the worst samples are refined
by a few steps of local optimisation, and poses that cannot be reached
safely or measured (no line of sight to the instrument) are excluded. A
handful of poses is then chosen greedily, each new one being the worst
remaining that is not close to those already chosen, so that the set
covers different weaknesses instead of five variations of one.

*What else to include.* Predicted error only knows about the
uncertainties in the model. Three further kinds of pose test what the
model may be missing:

- *Furthest from the calibration data.* Poses unlike any used in fitting
  test extrapolation, where a model that has absorbed an unmodelled
  effect fails first.
- *Where unmodelled effects are largest.* Full reach and heaviest load
  for deflection under gravity; both directions of approach to the same
  pose for backlash; poses near a kinematic singularity.
- *Where two models disagree most.* The planar and the full variant of
  the kinematics, or the model with and without a compliance term,
  differ by a computable amount in every pose; the poses where they
  differ most show which one the machine follows.

A few ordinary working poses are added so that the validation also says
something about typical accuracy, not only about the worst case.

*Reading the result.* In each validation pose there is a measured error
and a predicted covariance.

- The measured error against the specification is the accuracy test.
- The measured error scaled by the predicted covariance is the honesty
  test. Values consistently larger than expected mean the uncertainties
  are understated or an effect is missing; consistently smaller means
  they are overstated and the machine is better than it claims.
- The pattern of the errors across poses is diagnostic. Error growing
  with reach points to deflection or an angular offset near the root; a
  jump between the two directions of approach is backlash; an offset
  that turns with the base is a lever-arm or heading error.

*Keeping validation independent.* The poses of largest predicted error
are also, by the same computation, the poses where a new calibration
observation would help most. Measurements taken for validation must not
then be fed back into the fit and still counted as validation. If they
are used to improve the calibration, a fresh set of validation poses is
computed from the improved result and measured anew.

*Motion as well as poses.* The same idea extends to trajectories. Fast
reversals and high angular acceleration are where time offsets,
vibration and the actuation models contribute most error, so a dynamic
validation run is composed of the motions with the largest predicted
error, checked against an independent reference such as the total
station tracking a prism.

**Sensor intrinsics.** Scale factors, axis non-orthogonality, bias as a
function of temperature (which needs the temperature as an input), and
the response of gyroscopes to acceleration.

- *Time offsets belong here.* The sampling skew of each inertial unit
  and the latency of GNSS are parameters like any other. Because a
  measurement is attached at its own time by interpolation between grid
  states, the residual is a smooth function of that time, and an offset
  can be estimated from motion. This turns the skew from an uncertainty
  that has to be bounded into a number that is known.
- Some intrinsics are better determined on a bench, with the unit held
  still in several orientations, than on the machine. Those enter the
  on-machine calibration as considered parameters.

**Noise parameters are a different kind of unknown.** The per-link
disturbance constants, coherence factors, correlation times and
measurement covariances cannot be found by minimising the same squared
residuals: making a covariance larger always reduces the weighted
residual. They are fitted by likelihood, which includes the
normalisation term, or by matching the statistics of the residuals to
what the model predicts. This is done off line, after the geometric
calibration, on the same recordings.

**Actuation models: deadband, backlash, friction.**

- *They need a model of how commands produce motion.* So far the motion
  prior between grid states has been generic. Identification adds an
  actuation factor per joint and interval: the commanded input, passed
  through the non-linear model, predicts the joint's motion, and the
  difference from the estimated motion is the residual. The commands
  therefore have to be recorded on the same time base as the sensors.
  Once identified, the same factor improves estimation in normal
  operation, because the estimator then knows what motion to expect.
- *They need to be smooth enough to optimise.* Deadband and backlash have
  kinks, and a least-squares solver needs derivatives. Each is written as
  a smooth approximation with a sharpness parameter, fitted first in a
  soft form and then sharpened. The symbolic generator differentiates
  these like any other expression.
- *Backlash has memory.* Where the mechanism sits within its gap is a
  state, not a parameter: a bounded extra coordinate per joint with
  backlash, estimated along with the rest. Its width is the parameter.
- *They need the right excitation.* Deadband is identified by slow
  passes of the command through zero; backlash by direction reversals;
  friction by a range of speeds. Without those in the recording the
  curvature test reports the parameters as undetermined, which is the
  correct answer.
- *They need measurements on both sides.* Backlash and compliance are
  the difference between what the actuator does and what the link does.
  The link side is seen by the inertial units; the actuator side needs
  its own measurement (a motor encoder, a cylinder position or pressure)
  or the effect cannot be separated from the command-to-motion gain.

**On-line tracking.** Only a few parameters are tracked while the
machine works: sensor biases, and optionally a small number that
indicate wear, such as a backlash width. Each has a slow random walk and
is updated only while observable. A tracked value that drifts away from
its calibrated value is a maintenance signal in its own right. A change
of calibration while the estimator is running is applied between solves
and is accompanied by an inflation of the affected prior, since the
prior was formed under the old values.

**Order of calibration.** Estimating everything at once invites
parameters to absorb each other's errors. A staged procedure is more
robust: sensor intrinsics first (bench, or stationary periods), then
time offsets and mounting geometry with designed motions, then noise
parameters, then actuation models with command sweeps. Each stage
treats the earlier results as considered parameters.

**Guarding against absorption.** A flexible model will fit effects it
does not represent: unmodelled vibration or link compliance can be
absorbed into a mounting angle or a scale factor. Three safeguards:
priors from mechanical tolerances on every geometric parameter;
validation on a recording that was not used for fitting; and the
parameter correlation report, which shows when two parameters were only
determined in combination.

**Testing with the simulator.** The simulator can inject known mounting
errors, intrinsics, time offsets and actuator non-linearities into the
plant model. Running the calibration on simulated recordings and
recovering the injected values, within their reported uncertainty, is an
end-to-end test of the whole procedure and can run in continuous
integration. It also tests that reported uncertainties are honest.

**What this asks of the framework.** Commands and all sensor data
recorded with consistent time stamps (port observation and the logger,
`runtime-layer.md` and `logging.md`); calibration values and
uncertainties as per-machine properties in the machine file
(`operations-and-properties.md`); and the parameter roles expressed
through the same fixed, tunable and configured distinctions that
properties already have (`compile-time.md`).

## Open points

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
- The sensors are one inertial unit per link at about 100 Hz, plus GNSS
  (stated by the owner). Still open:
  - Is the machine's base fixed, or does it move over the ground?
  - Some configurations have dual GNSS and some do not (answered).
    Where are the antennas mounted, what is the distance between the
    two, is the solution carrier-phase based, and what are its rate and
    latency? Which configurations are single-antenna, and do they have
    a slew sensor?
  - How do the inertial units report (which bus), with what low-pass
    setting, and are their samples triggered together or time-stamped at
    the sensor?
  - Are accelerometers used for inclination today, and are sensor biases
    estimated?
  - At what rate does the control loop that consumes the estimate run?
  - Do recordings with all units and GNSS logged together exist?
- Curvature of the residual is the intended observability indicator
  (clarified by the owner). Which directions are weak in practice, and
  what should a consumer do while one is flagged: hold the last value,
  stop, or ask for motion? Is a fixed threshold on scaled curvature
  acceptable, or should it adapt?
- For the time grid and truncation: what grid spacing is intended, and
  do the inertial units provide integrated angle increments? Is heading
  observed directly (two antennas) or only through motion? Are sensor
  biases to be one copy per grid state or shared across the window?
- The estimator is to be a truncated factor graph (stated by the
  owner). How long a window is intended, and is it set by GNSS latency
  or by something else? Should the solve run in the real-time thread or
  beside it? Is the symbolic generator's own optimiser intended as the
  solver? Is a smoothed, delayed estimate also wanted, or only the
  current one?
- There is one inertial unit per link, and the default rule is that
  trust falls with distance along the chain, without modelling mode
  shapes (decided). What are the link lengths and the lowest
  out-of-plane vibration frequencies compared with the bandwidth needed
  for the joint estimates? How is the varying covariance computed
  today? Are recordings available to fit the per-link disturbance
  constant?
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
- How is stillness detected today for accelerometer inclination, and
  what goes wrong with it (chatter, engine vibration, slow motion
  missed)? Are commands available to the estimator as a prior?
- For external calibration observations: which reference points exist
  on each link and how well are their coordinates known? Does the total
  station give raw angles and distance or only coordinates, and is it
  tied to the GNSS frame by control points? Is the machine held static
  for each observation, and in how many poses? Is the repeated-touch
  observation (a tool point on a fixed mark) practical on this machine?
- For single-joint sweeps: can each joint be moved alone over most of
  its range while the others hold, and how much do held cylinders creep?
  How firmly is the base supported during calibration? Is "singular
  motion" meant as one joint at a time, as assumed here?
- The total station can track, single and multiple prisms are
  possible, lock-on time is a problem, and calibration without the
  instrument is a goal (stated by the owner). Which quantity defines
  accuracy (tool point position, an angle, a depth) and what is the
  specification? What equipment can be assumed in the field (tape, GNSS
  rover, laser level)? Which parameters are expected to change in the
  field? Does the instrument report lock status and time stamps for its
  measurements?
- For identification and calibration: are commands recorded on the same
  time base as the sensors? What is measured on the actuator side of
  each joint (encoder, cylinder position, pressure)? Which parameters
  are expected to drift and should be tracked on line? Is there a bench
  procedure for inertial intrinsics today?
