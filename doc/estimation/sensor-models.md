# Sensor models and noise

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A0 and A2. How each sensor is modelled, how much it is trusted, and how its errors are kept out of the states.

## Uncertainty that grows with the distance between sensor and joint

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

## Flexible links: vibration that scales with link length

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

## The default rule: trust falls with distance along the chain

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

## Finite propagation speed: disturbances that arrive later further out

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

## The sensor set: one 100 Hz inertial unit per link, plus GNSS

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
  deployment it is an optional alternative (`../rtt3/deployment.md`).
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

## Deciding when accelerometer inclination can be trusted

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

## Coloured noise in GNSS positions

GNSS position errors persist: the error at one epoch is a good
predictor of the error at the next. An estimator that treats each fix as
an independent sample averages the error away far too quickly, reports a
position uncertainty that is too small, and lets the slowly wandering
error pull the other states with it. The following describes where the
correlation comes from and the ways of correcting for it. It is from
general knowledge of GNSS and has not been checked against the receiver
in use.

**Where the correlation comes from.**

- *Multipath.* Reflections change only as the satellite geometry and the
  antenna's surroundings change, so for a stationary antenna the error
  varies over tens of seconds to minutes. On this machine the working
  equipment is itself a moving reflector and obstruction, so the error
  also depends on the machine's pose.
- *The state of the carrier-phase solution.* A float solution converges
  over minutes; a fixed solution has small, slowly varying residuals
  from the atmosphere over the baseline to the reference station; a
  code-only solution has metre-level errors correlated over many
  minutes.
- *Steps.* A change of fix type, or of the set of satellites used, moves
  the position abruptly. This is not coloured noise but a jump, and
  needs separate handling.
- *The receiver's own filtering.* Many receivers output a position that
  has already been through an internal filter with a motion model. Such
  output is correlated by construction, and the correlation depends on
  the receiver's dynamic-model setting. Feeding it to a second estimator
  as independent measurements counts the same information repeatedly.
  Where the receiver offers an unfiltered or per-epoch solution, that is
  the one to use.
- Vertical errors are typically larger and more strongly correlated than
  horizontal ones, so the model is per axis in the local level frame.

**Ways of correcting, compatible with the windowed estimator.**

*1. An explicit slowly varying error state.* Each axis gets an offset
that follows a first-order model with a correlation time and a
stationary variance; the measurement is position plus offset plus white
noise. In the graph the offsets at consecutive grid states are linked by
the corresponding factor, or the offset is given sparse values in time
like any slow quantity.

- It makes the reported uncertainty correct even where the offset cannot
  be separated from true position.
- It can actually be estimated, and so removed, wherever something else
  pins the position down: the machine standing still, the inertial
  dynamics over short intervals, the kinematic constraints of the
  mechanism.
- Cost is three states per antenna.

*2. Differencing consecutive measurements.* For an error that follows a
first-order model with coefficient φ from one epoch to the next, the
combination of the current fix minus φ times the previous one has white
noise. That combination is used as the measurement, as a factor on the
two states concerned. No extra states are needed, and the factor
connects only adjacent states, so the structure of the problem is
unchanged. It is equivalent to the first method with the offset
eliminated, and is the cheaper choice when the offset itself is of no
interest.

*3. Split GNSS into what it does well at each time scale.* Changes in
position over short intervals are far more accurate than position
itself, because the slowly varying errors cancel. Where the receiver
provides velocity from Doppler or carrier phase, that is used as a
separate, nearly white measurement of motion, and the position only as a
slow anchor with a long correlation time. This usually gives more than
any refinement of the position noise model alone.

*4. Inflation or decimation.* The simplest: enlarge the covariance by
the factor that corresponds to the correlation time, or use one fix per
correlation time. It keeps the uncertainty roughly honest and gives up
the short-term information. Acceptable as a first step, not as the end
state.

**Handling the steps.** At a change of fix type the error process is
restarted: the link between the offset before and after the change is
given a large variance, so the jump is taken up by the offset and not
smeared into the machine's position. The correlation time and variance
then follow a table by fix type. A robust loss covers isolated bad
fixes; it is not a substitute for the restart, because a step is not an
outlier that goes away.

**Using what this machine offers.**

- *Stillness.* While the base is known not to move, the antenna's true
  position is constant, so all variation in the fixes is error. The
  offset is then observed directly. Averaging helps only as far as the
  correlation allows: a still period shorter than the correlation time
  yields roughly one independent sample, however many fixes it contains.
- *Two antennas.* Errors common to both cancel in the baseline, so
  heading from the baseline is much less affected by slowly varying
  errors than position is. The remaining error is differential
  multipath, with a shorter correlation time. The baseline therefore
  gets its own, different noise model.
- *Pose-dependent error.* When the working equipment is raised into an
  antenna's view of the sky, multipath and blockage increase. The
  covariance is raised as a function of the equipment's pose relative to
  the antenna, which the kinematics already provide. A map of the effect
  can be measured once per machine type.

**Finding the parameters.** The correlation time and variance per axis
and per fix type come from stationary recordings, by the autocorrelation
of the position series, or against the total station as truth. As for
the other noise parameters, they are fitted by likelihood, not by
minimising residuals.

**Checking that it worked.** If the model is right, the GNSS innovations
are uncorrelated in time. Their autocorrelation at a few lags is a cheap
running monitor: a clearly non-zero value says the remaining colour is
being absorbed by some other state. The position uncertainty reported
during a long still period should also level off at about the stationary
size of the offset and not keep shrinking.

**The actual situation: velocity is available, and carrier-phase
corrections often but not always.** That settles the default and adds
one design point.

*Velocity carries the motion.*

- GNSS velocity is a separate factor with nearly white noise. Its
  accuracy depends little on whether corrections are being received, so
  it is the measurement that stays good through a corrections outage.
- The antenna's velocity is the base's velocity plus the rotation of the
  structure acting on the antenna's lever arm. Predicted through the
  kinematics, it also informs angular rate and, while the machine
  translates, heading. For a single-antenna machine this is a second
  source of heading in motion, beside horizontal acceleration. Direction
  of travel alone does not say which way the machine faces: the sign
  comes from the commanded direction or the inertial data.
- Velocity and position reported by one receiver are not strictly
  independent, since both come from the same signals and possibly the
  same internal filter. Treating them as independent is the usual
  approximation; the innovation monitor shows whether it is acceptable
  for this receiver.
- Velocity has its own latency, which is estimated like the other time
  offsets.

*Position is the slow anchor, with a quality that changes.*

- The offset model's variance and correlation time follow the solution
  type: small and short for a fixed carrier-phase solution, larger and
  longer for float, largest for a solution without corrections. The age
  of the corrections, where the receiver reports it, grades the
  transition instead of making it a switch.
- When corrections are lost, the estimate is carried by velocity and the
  inertial sensors, and absolute position uncertainty grows towards the
  uncorrected level. When they return, the offset process is restarted
  as at any change of fix type.
- A change of reference station is also a step and is handled the same
  way.

*What consumers see when corrections come back.* The absolute position
estimate will move, possibly by decimetres, as the anchor regains its
accuracy. For anything that controls motion relative to the machine or
to recently visited points, such a jump is harmful and carries no
information. The estimator therefore offers two position outputs:

- a *locally consistent* one, driven by velocity and inertial data,
  continuous by construction and accurate over short distances, which
  drifts slowly in absolute terms;
- a *globally referenced* one, which is the best absolute estimate and
  may step when the anchor changes quality.

The offset state is exactly the difference between the two, so this
costs nothing extra. Which output a consumer uses is a statement about
what it needs, and it is a property of the port it connects to.

*Calibration and validation with a rover* need the fixed solution to be
useful at the centimetre level. The fix type is already part of each
observation record, and the session plan checks that corrections are
available before scheduling rover observations.

**Suggested default.** Velocity as a separate, nearly white measurement
of the antenna's motion through the lever arm. Position with an offset
state per axis whose variance and correlation time follow the fix type
and the age of corrections, restarted at fix and reference-station
changes, and anchored during stillness. Covariance raised by equipment
pose. Two position outputs, local and global. The innovation
autocorrelation as a monitor. Measurement differencing replaces the
offset state if the extra states prove unwelcome, at the price of the
two-output scheme.

## Keeping other errors out of the slowly varying biases

**The mechanism.** A bias state is an integrator of the residual. Any
error that the model does not explain and that does not average to zero
(a misalignment, an unmodelled lever-arm term, coloured vibration, a
time offset) pushes the bias a little at every update, and the bias
keeps what it is given. Since the true bias is known to move slowly, the
estimator should not be able to move its estimate quickly. The measures
below limit that rate. They are from general knowledge of inertial
estimation and have not been tried on the system in question.

**1. Give the bias only as many unknowns as its real rate of change
warrants.** In the windowed estimator the most direct control is
structural. A bias does not need a value at every grid state. It is
represented by a few values spaced at the time scale on which it can
really change, tens of seconds to minutes, with every grid state in
between using the interpolated value. The estimator then cannot make the
bias follow anything faster than that spacing, whatever the residuals
do, and the number of unknowns falls as well. A slow random walk links
consecutive bias values.

**2. Take the random-walk strength from the sensor, not from tuning.**
The noise that drives the bias is set from the unit's measured bias
instability and rate random walk (from a long stationary recording, by
Allan variance), not chosen to make the filter look good. A value larger
than the sensor's real behaviour is an open invitation for other errors
to be absorbed.

**3. Tie the bias to what is known about it.** A pure random walk can
wander without limit. The physical bias cannot: it stays within a known
range of its calibrated value. A weak restoring term towards the
calibrated value (a first-order Gauss-Markov model with a long time
constant) bounds the variance and means that an error pushed into the
bias decays instead of persisting. In graph terms it is a prior factor
on each bias value centred on the calibration.

**4. Remove the largest slow component by model.** Most slow bias
movement is thermal. With the bias calibrated as a function of
temperature, the state only has to represent the small remainder, whose
range and random walk are then much tighter. The process noise can also
follow the rate of temperature change, so the bias is free to move
during warm-up and nearly fixed in steady conditions.

**5. Update the bias only when it is observable.** Using the curvature
indicator: while the current motion does not separate a bias from the
quantities it is confused with, the bias is held (treated as a
considered parameter: its uncertainty is accounted for but its value is
not changed), and it is updated only when the curvature along it is
above the threshold. Stillness is the clearest such condition, since a
gyroscope then reads its bias directly; those periods act as anchors
between which the bias is interpolated.

**6. Apply only part of the update.** A graded version of the previous
point: for selected states the estimator applies a chosen fraction of
the computed correction, with the covariance updated consistently for
that fraction. Zero is the considered-parameter case, one the ordinary
update. It gives a direct, per-state control over how fast residuals can
flow into a bias while keeping the reported uncertainty honest, and is
simple to add to either a filter or the windowed solver.

**7. Give the errors somewhere else to go.** Leakage is a symptom of a
missing model term. Everything described earlier reduces it at the
source: lever-arm accelerations in the accelerometer model, the
accumulated disturbance with its coherence, the treatment of correlated
samples, estimated time offsets, calibrated mounting, and a robust loss.
Where a disturbance is coloured, a short-memory state for it absorbs
what would otherwise go into the long-memory bias.

**8. Do not let the bias uncertainty collapse or run away.** If the
bias covariance shrinks to nothing, a real change can no longer be
followed; if it grows without bound, the bias absorbs everything. The
restoring term gives a ceiling. A floor consistent with the sensor's
instability gives the other limit.

**Detecting leakage that still occurs.**

- The estimated bias moving faster than the sensor's instability allows
  is evidence that something else is being absorbed. The estimator
  compares the two and flags it.
- A bias estimate that correlates with the motion, the load or the
  heading, and not with temperature, is being used as a dumping ground.
- The bias found during stillness is close to the truth. A consistent
  difference between it and the value the estimator holds just before
  each still period measures the leakage directly.

**The cost of being too strict.** A bias that is held too rigidly
pushes its real changes, above all thermal transients after power-on,
into the states it is confused with, typically heading or an
inclination. That is why the limits are tied to the sensor's measured
behaviour and to temperature, and not simply made as tight as possible.

**Suggested default.** Sparse bias values in time with a
sensor-derived random walk and a weak pull towards a
temperature-compensated calibration; updates gated by observability,
with stillness as the anchor; the leakage checks running as monitors.
Partial updates are kept as the adjustment to reach for if a particular
bias still drifts.

## Very stable gyroscopes: account for the bias, do not estimate it

**The owner's experience.** With gyroscopes whose bias changes extremely
slowly, of the order of a degree over hours, results improved when the
bias was left out of the state estimation altogether. The question is
whether it would be better to put it back with a severely limited rate
of change.

**Assessment: neither. Keep it out of the solve, but keep it in the
model.** There are three options, not two.

| | Bias estimated | Bias omitted | Bias considered |
|---|---|---|---|
| Value used | Estimated continuously | A fixed correction, or none | A fixed correction, re-measured at stillness |
| Path for other errors to enter it | Yes, however slow | None | None |
| Reported uncertainty | Includes bias | Ignores bias: slightly overconfident | Includes bias |
| Extra unknowns | Three per unit | None | None |

The third is what the earlier subsection calls a considered parameter.
The bias is not an unknown, so nothing can leak into it, which is what
made omission work. Its uncertainty is still carried into the estimate,
so the covariance stays honest, which omission gives up. Putting the
bias back as a state with a very small rate limit approaches the same
thing from the other side, but any non-zero path still integrates
residuals over hours, and it adds unknowns to gain nothing.

**Why estimating it is unlikely to help here.** The argument is about
observability, not about tuning.

- On the axes that have an absolute reference (inclination from gravity,
  joint angles from neighbouring units, heading with two antennas) a
  bias this small produces an angle error that the reference removes
  long before it matters. The bias is observable there, and not needed.
- On an axis with no absolute reference, heading at rest with a single
  antenna, the bias would matter, since it accumulates uncorrected. But
  there it is not observable from motion either: an unobserved heading
  and a yaw bias are the same thing to the estimator.
- The one situation in which the bias is both useful and observable is
  stillness, when the gyroscope reads it directly. That is a
  measurement of a constant, not a job for the motion estimator.

So an estimated bias state is either redundant or undetermined almost
all of the time, and in both cases it is available to absorb whatever
the model does not explain. That matches what was observed.

**Proposed handling.**

- *The value.* Each unit's bias is a correction applied to its readings.
  It is measured by averaging during stillness (with the stillness
  decision of "Deciding when accelerometer inclination can be trusted"),
  starting from the calibrated, temperature-compensated value.
- *Re-anchoring.* When a still period is long enough, the correction is
  replaced by the new average. A new value that differs from the old by
  more than the sensor's instability allows for the time elapsed is not
  applied silently; it is flagged, because it more likely means the
  machine was not really still, or something else has changed.
- *The uncertainty.* The bias has a standard deviation that starts at
  the accuracy of the last anchor and grows slowly with the time since
  it, at the sensor's measured instability. That uncertainty enters the
  estimate as a considered parameter. On a single-antenna machine at
  rest, the reported heading uncertainty then grows at the right rate,
  and a consumer can see when heading has become too uncertain to use.
- *No bias unknowns in the factor graph* for these units, which also
  removes three unknowns per unit from every solve.

**A check worth making: the Earth's rotation.** The Earth turns at
fifteen degrees per hour. A gyroscope stable to a fraction of a degree
per hour sees that plainly: at rest it reads its bias plus the
component of the Earth's rate along each of its axes, and that
component changes whenever the unit's orientation changes. If the Earth
rate is not modelled, then:

- a "bias" measured at stillness contains an Earth-rate term that is
  only right for that orientation;
- an estimated bias state is forced to follow the orientation of the
  link, which looks exactly like other errors leaking into it, and
  could be part of why estimating the bias made things worse;
- heading integrated from the yaw rate drifts by up to several degrees
  per hour from this cause alone, far more than the sensor's own bias.

The Earth-rate vector is known from latitude, and its components in
each sensor's frame follow from the estimated orientation, so
subtracting it is a small addition to the gyroscope model. Whether it
matters depends on what "a degree over hours" refers to: if the sensors
really are that stable, Earth rate is the dominant uncorrected term and
must be included; if the figure describes the angle error after all
corrections, it may already be hidden in the bias. This needs
confirming against the sensor's data sheet and a stationary recording
in two orientations.

**When to reconsider.** An estimated bias state, sparse in time and
tightly limited as in the previous subsection, becomes worthwhile only
if all of these hold: the bias changes by an amount that matters between
still periods, some axis lacks an absolute reference for long stretches,
and the motion in those stretches makes the bias observable. Thermal
transients after power-on are the likeliest case, and a temperature
model of the bias handles them better than a free state does.
