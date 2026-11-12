# Identification and calibration

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A5 and A6. Using the estimator's factor graph for calibration: parameter roles, external observations, field procedures and validation. The record types are in `calibration-data.md`.

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
`../rtt3/runtime-layer.md` and `../rtt3/logging.md`); calibration values and
uncertainties as per-machine properties in the machine file
(`../rtt3/operations-and-properties.md`); and the parameter roles expressed
through the same fixed, tunable and configured distinctions that
properties already have (`../rtt3/compile-time.md`).
