# Open points

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Questions for the owner that the plans in this directory depend on.
The data-model questions are at the end of `calibration-data.md`. A
prioritised list covering all plans is in
[../open-questions.md](../open-questions.md).

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
- GNSS velocity is available and carrier-phase corrections often are
  (stated by the owner). Which receiver, and does it output a filtered
  position or a per-epoch solution? How long and how frequent are the
  corrections outages, and does the receiver report the age of
  corrections? Do consumers need a continuous local position, an
  absolute one, or both? Does the working equipment visibly degrade the
  fix in some poses?
- Gyroscope bias was previously left out of the estimation with good
  results (stated by the owner). Does "a degree over hours" describe
  the sensor's bias stability or the angle error after corrections?
  Which gyroscopes are these? Is the Earth's rotation compensated
  today? Was a fixed bias correction applied when the state was
  omitted, and how was it obtained?
- For the biases: do long stationary recordings exist from which each
  unit's bias instability can be measured? Do the units report
  temperature? How long is warm-up, and does the machine stand still
  often enough in normal work to anchor the biases?
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
