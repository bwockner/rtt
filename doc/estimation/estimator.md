# The estimator: a truncated factor graph

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A3 and A4. The windowed solver, the time grid, removal of old states, and observability from the curvature of the residual.

## A truncated factor graph in place of a pure EKF

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

## Putting measurements onto a time grid

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

## Removing the oldest state

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

## Higher-order derivatives for the poorly observed dimensions

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

## Low curvature of the residual as the indicator of poor observability

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
