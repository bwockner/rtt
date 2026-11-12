# State representation and mechanism model

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A1. What the estimator estimates: poses on Lie groups, the reduction to the mechanism's true degrees of freedom, and why one joint estimator is preferred to a cascade.

## When the state is a Lie group element in machine space

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
units (`../rtt3/compile-time.md`). Mixing conventions silently is the most
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
(`kinematics.md`); the estimator follows the same
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
translation) as described in `../rtt3/numerics.md` under "Eigen".

## Several poses at once: SE(3)^n with n of five or six

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

## Automatic reduction when the true degrees of freedom are few

**The structure being exploited.** If the bodies can only move in k
independent ways, with k much smaller than 6n, then every admissible
small motion of all the bodies together is `δξ = B δq`: a 6n by k matrix
B maps k independent coordinates to the stacked body twists. All
uncertainty lies in the column space of B, so the full covariance is

    P = B P_q Bᵀ

a 36 by 36 matrix of rank k. This is the rank-deficient matrix that
should be usable in its full form without paying for its full size
(`../rtt3/compile-time.md`, "Structured matrices"): the filter works on the
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

## Worked case: a gyroscope on a hinged link sees out-of-plane rotation

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
variants (`kinematics.md`). A threshold on it can
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

## One joint filter versus a cascade of filters

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
