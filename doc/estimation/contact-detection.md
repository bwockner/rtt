# Evaluation: detecting tool contact from changing vibration behaviour

Part of the state estimation and calibration plan; see
[README.md](README.md) for the index and the work streams.

Streams: X1 (exploratory). Status: an evaluation of an idea, not a
design. Nothing here has been tested on data. The structural behaviour
described is textbook beam dynamics and the orders of magnitude are
general, not properties of the machine.

## The idea

A link chain is a structure whose vibration depends on how its end is
held. A tool hanging free and a tool resting on or pressed into a work
surface are different boundary conditions, so the structure's natural
frequencies, damping and mode shapes differ. If the inertial units
already on the links can see that difference, contact can be inferred
without any additional sensor.

## Is the effect there?

In principle, yes, and it can be large.

- *Frequency.* Constraining the free end of a beam raises its lowest
  bending frequency substantially. For an ideal uniform beam clamped at
  one end, pinning the other end raises the first frequency by a factor
  of about four. A real chain with compliant joints and a compliant
  surface will show less, but the direction is always upward.
- *Mode shape.* A free end is where the vibration is largest. A held end
  becomes a point of little motion. With a unit on every link, the
  relative vibration amplitude along the chain is a coarse picture of
  the mode shape.
- *Damping.* Contact with soil or another dissipative surface adds
  damping; ring-down after a disturbance becomes shorter.
- *Direction.* A surface constrains the tool mainly along its normal, so
  the change differs between in-plane and out-of-plane vibration.

How large the change is depends on the stiffness of the contact compared
with the stiffness of the structure at the tool. A hard surface gives a
clear change; loose material or a light touch gives a small one. This is
the main limit of the method: it detects constraint, not touch.

## Can the present sensors see it?

Partly.

- The inertial units sample at about 100 Hz, so only vibration below
  roughly 40 Hz is visible after anti-alias filtering. The lowest modes
  of long links are typically within that band when the tool is free.
- When contact raises a mode's frequency by a large factor it may move
  towards or beyond the top of the band. A mode that disappears from
  the band is still evidence, but a weaker and more ambiguous one than
  a mode that is tracked to its new frequency.
- Resolving a low frequency needs several cycles of it. A mode at a few
  hertz takes of the order of half a second to a second to characterise.
  Amplitude-ratio features between units need less time than frequency
  estimates.
- There must be something exciting the structure. A machine standing
  still and quiet gives no vibration to analyse.

So with the present sensors this can classify a sustained state ("the
tool is resting on something stiff") with a latency of some tenths of a
second, when there is excitation. It cannot detect the moment of touch
quickly, and it cannot say anything when nothing excites the structure.

## Methods, in increasing cost

**1. Amplitude ratios along the chain.** For each pair of neighbouring
units, the ratio of vibration energy in a low band, separately in plane
and out of plane. A tool that becomes constrained shows a drop in the
outermost unit's share. This needs no frequency resolution, is cheap,
and responds fastest. It is sensitive to where the excitation comes
from, so the ratios have to be compared with what is normal for the
current excitation.

**2. Tracking a mode's frequency and damping.** A low-order
autoregressive model per axis, updated recursively, gives the dominant
frequency and damping in the band continuously. Equivalent formulations
are an adaptive notch or a small filter with a modal state whose
frequency is a slowly varying parameter. This is the direct estimate of
the boundary condition.

**3. A two-hypothesis test.** With the free and constrained behaviour
each described by a model, two predictors run side by side and their
likelihoods are compared over a window, with a two-state model for
persistence, exactly as for stillness detection (`sensor-models.md`).
The output is a probability of contact. This is the proper statistical
form, and it requires both models to be known.

**4. Active probing.** A small deliberate disturbance, a brief dither on
one actuator, and observation of the ring-down. It removes the
dependence on incidental excitation and gives the cleanest reading, at
the cost of moving the machine on purpose. Whether that is acceptable
during work is a question for the owner.

## What confounds it

The free-vibration behaviour is not a constant, so "changed" has to be
judged against a baseline that itself moves.

- *Configuration.* Modal frequencies depend strongly on how the chain is
  extended. The baseline is a function of the joint coordinates, which
  the estimator knows.
- *Load.* Material carried in or on the tool lowers the frequencies.
  That is the opposite direction to contact, which helps separate them,
  but an unknown load shifts the baseline.
- *Hydraulic stiffness.* Much of the compliance of such a mechanism is
  in the oil columns of the cylinders, and varies with cylinder
  position, pressure and oil temperature.
- *Excitation source.* Engine and pump harmonics, travel over rough
  ground, and the motion itself each excite different modes, and a
  change of excitation can look like a change of structure to the
  simpler features.
- *Other contacts.* Anything else touching the structure, or the base
  settling, changes the boundary conditions too.

The baseline therefore has to be learned per machine as a map over
configuration and, ideally, load, or tracked adaptively during periods
known to be free. That is a calibration effort of its own.

## Cues that are probably better primary detectors

Vibration is not the obvious first choice for contact detection. The
following are more direct, and most use data already planned for.

| Cue | Basis | Speed | Needs |
|---|---|---|---|
| Actuator force | Cylinder pressures or motor currents give joint force; force beyond what motion and gravity explain is external | Fast | Pressure or current measurement |
| Actuation residual | Commanded motion that does not happen, or happens against resistance. The residual of the actuation factors (`calibration.md`) is an estimate of external force | Moderate | Recorded commands and an identified actuation model |
| Impact transient | A spike in acceleration at the moment of touch | Fast, but only at the instant | Higher sample rate than 100 Hz to be reliable |
| Base reaction | The base tilts or lifts when the tool pushes on the ground | Moderate | The base inertial unit |
| Geometry | The tool's estimated position against a known or mapped surface | Immediate, as a prior | A surface model and calibrated kinematics |

Vibration-based detection is best seen as complementary to these: it
needs no additional sensor, and it responds to the stiffness of the
constraint, which the force-based cues do not directly give.

## Why it is worth having at all

Contact is useful to the estimator, not only to the application.

- *It switches the noise model.* The link disturbance levels and the
  coherence between units, which the sensor models depend on, differ
  between free motion and work in contact. Knowing the state lets the
  right parameters be used instead of a compromise.
- *It provides a constraint.* A tool point held by a surface is, while
  it does not slip, a fixed point in the world or a point on a known
  surface. That is a strong kinematic measurement: it anchors the far
  end of the chain in the way a still base anchors the near end, and
  makes heading and base motion more observable. The same idea is
  standard in legged-robot estimation, where foot contact is detected
  and used as a kinematic constraint.
- *It explains residuals.* Forces at the tool deflect the structure.
  Without a contact state, that deflection appears as unexplained
  motion.

Because a wrongly assumed constraint is damaging, the contact factor is
used with a weight that follows the contact probability, or as a mixture
factor that the solver can decline, and never as a hard switch.

## The stated use: sudden loss of contact, and compensating the overshoot

The owner's example of what this is for: the tool suddenly loses contact
with the work surface (it breaks through, slips off, or the material
gives way), and the machine should compensate for the overshoot that
follows. This is a much more specific and more demanding case than
classifying a sustained state, and it changes the assessment.

**What happens physically.** While the tool pushes on the surface, the
structure is bent and the oil in the cylinders is compressed; energy is
stored. When the resistance vanishes, that energy is released: the tool
springs in the direction it was pushing, the actuators surge because
they are no longer opposed, and the structure rings at its free
frequency. The overshoot is that first excursion.

**The time available.** The first excursion peaks within a fraction of
the period of the lowest free mode: of the order of a quarter to a half
period. For a first mode of a few hertz that is roughly 50 to 250 ms.
Everything between the event and an effective counter-action has to fit
inside it: sensing, detection, the control cycle, and the response of
the actuators, which for hydraulic valves is itself tens of
milliseconds.

**Consequence: estimating the changed boundary condition is too slow to
be the detector.** Characterising a mode of a few hertz takes several
tenths of a second, which is the whole budget. Vibration-based
boundary estimation cannot trigger the compensation. Its place in this
use case is different and described below.

**What can detect the release fast enough.**

| Cue | Why it is fast | Limit |
|---|---|---|
| Collapse of actuator force (pressure or current) | The load disappears from the actuator at once | Needs that measurement |
| Sudden acceleration of the tool link | The released structure accelerates immediately; visible in the outermost unit's accelerometer and gyroscope within a sample or two | At 100 Hz a sample is 10 ms, so detection takes 10 to 30 ms of the budget |
| Jump in the estimator's innovation | The "in contact" model stops predicting the measurements | As fast as the sensor feeding it |

Detection should be a sequential change test on these signals (a
cumulative-sum test on the innovation or on the force), which is
designed to minimise delay for a given false-alarm rate, and not a
windowed average, which trades away exactly the delay that matters here.

**Arming the compensation in advance.** Most of the response can be
prepared before the event, so that detection only has to release it.

- While in contact, the stored energy is known approximately: from the
  actuator force where it is measured, or from the structure's
  deflection, which appears as a discrepancy between the tool link's
  measured inclination and what the joint coordinates predict.
- The free behaviour for the current configuration and load is known
  from the baseline map of free modal frequency and damping.
- From those two, the expected free response to an instantaneous
  release is computed continuously: its direction, its size and its
  frequency. The counter-action is prepared from it: an immediate
  reduction of the actuator command so that the actuators do not surge,
  followed by damping of the ring-down at the known free frequency,
  shaped so that the correction does not itself excite the mode.
- On detection, the prepared action is applied in the same control
  cycle, with no computation left to do.

**So the roles divide as follows.**

- *Fast transient cues* detect the release.
- *Boundary-condition and modal knowledge* does not detect it, but
  supplies what the compensation needs: the free frequency and damping
  for the present configuration, which is what makes active damping and
  command shaping effective, and, afterwards, confirmation from the
  ring-down that the tool really is free.
- *The contact state before the event* supplies the stored energy and
  so the size of the response.

**False alarms.** A compensation applied when nothing happened is a
disturbance of the machine's own making. The two parts of the response
differ in how much harm they do if wrong. Added damping is harmless.
Cutting or reversing the command is not. The response is therefore
graded by confidence: damping engages on weak evidence, the command
reduction only on strong evidence, and its size follows the estimated
stored energy, which is zero when the tool was not loaded.

**Measures that do not depend on detection.** Because the budget is so
tight, part of the answer should be to make the event less violent in
the first place: limiting the stored energy when a breakthrough is
likely (pressure or force limiting as resistance rises), limiting
actuator speed while the force is high, and choosing how the valves
meter flow so that an unloaded actuator does not run away. These work
however late detection is.

**Is it achievable with the present sensors?** Marginally, and only for
the lowest modes.

- With actuator force measured, detection is fast and the limit is the
  actuators' response.
- With inertial units at 100 Hz alone, detection takes one to three
  samples, which with the control cycle and valve response consumes much
  of the budget for a mode of a few hertz and all of it for anything
  faster.
- This is the clearest case for a higher inertial sample rate, at least
  on the outermost link, and for running the detection at that rate. It
  is also the case that would reopen the deferred work in
  `filter-numerics.md`, since a detector and predictor at several
  hundred hertz or more is no longer computationally free.

**What to establish first.** From recordings of real loss-of-contact
events: how long the first excursion takes, how large it is, how early
each candidate signal shows it, and how well the excursion is predicted
from the force or deflection just before. Those four numbers decide
whether compensation is possible with the present sensors, possible
with a faster inertial unit or pressure sensing, or better addressed by
the detection-independent measures alone.

## A second use: a dozer blade passing over a hole, and pre-empting the body's dip

The owner's second example: a dozer's blade passes over a hole in the
work surface, and the aim is to act before the feedback that arrives
when the machine's body later dips into the same hole. This is a
different problem from the sudden release above, and a much more
favourable one for the ideas in this document.

**The situation.** The blade is ahead of the tracks. It reaches the hole
first: the material it was cutting or riding on is briefly absent, so
its load and its support change. Some time later the front of the
tracks reaches the same place and the body pitches into it. Because the
blade is carried at the end of a long lever from the body's pitch axis,
a small pitch of the body moves the cutting edge by a larger amount, and
the blade gouges. Ordinary feedback corrects the blade from the body's
measured pitch, which is after the body has moved and before the
actuators have caught up.

**Why this case is easier: there is time.** The delay between the blade
crossing a point and the tracks reaching it is the distance between
them divided by the speed: typically from a few tenths of a second to
several seconds. That is one to two orders of magnitude more than the
budget for a sudden release. Slow cues are therefore usable here,
including the estimate of changed vibration behaviour, which was too
slow to trigger anything in the first use.

**The underlying idea: the machine drives on what its blade has just
passed over.** The blade's own path is a preview of the surface the
tracks are about to meet. Pre-emption is then not a special reaction to
a hole but a general mechanism:

1. *Remember the surface along the path.* A short profile of the
   surface, indexed by distance travelled, covering the stretch between
   the blade and the rear of the tracks. Its entries come from where the
   cutting edge has been (the surface left behind) and from events
   detected at the blade (a hole: where it started, where it ended, and
   an estimate of how deep it remains).
2. *Predict the body's motion over that profile.* The tracks rest on the
   profile over their whole contact length, so the body's pitch and
   height follow a smoothed version of it; a hole short compared with
   the tracks is bridged, a long one is not. With the travel speed, this
   gives the body's pitch and heave a short time ahead.
3. *Correct the blade ahead of time.* The predicted effect of that pitch
   and heave on the cutting edge is cancelled by a blade command issued
   early enough for the actuators to follow.
4. *Keep the feedback.* The prediction is imperfect, so the correction
   from the body's measured pitch stays in place and handles what the
   preview got wrong.

**Detecting the hole at the blade.** In order of directness:

- a drop in blade load (lift and tilt cylinder pressures, or draft force
  where measured), and its return at the far edge;
- a change in the estimator's actuation residual for the blade: the
  blade moves more easily than the model in contact predicts;
- the change in vibration behaviour of the blade and its arms when the
  cutting edge is no longer supported, as evaluated in the first part of
  this document. Here its latency is acceptable;
- a small drop of the blade itself, if it was resting on the surface.

The start and end of the event, placed along the path, give the hole's
position and length. Position along the path should come from the
locally consistent position output (`sensor-models.md`, GNSS), which
does not jump when corrections change, with distance travelled taken
from that and not from track speed, since tracks slip.

**What makes the prediction uncertain.**

- *Material falling in.* A blade carrying material partly fills the hole
  as it passes. How much is unknown, so the remaining depth is
  uncertain. The amount by which the blade's load falls and does not
  recover is some indication of how much material was lost, and so of
  how much the hole was filled. The pre-emptive correction is applied
  with a gain below one in proportion to this uncertainty.
- *Where across the machine.* The blade is wider than the track gauge
  and its load is one number, or at best a left and right pair from the
  tilt circuit. A hole may lie under one track, both, or neither. Pitch
  from a hole under both tracks is predictable; roll from a hole under
  one is much less so and is left mainly to feedback.
- *Turning.* The tracks do not follow the blade's path exactly in a
  turn. The profile is kept per track, and the kinematics place each
  track on it.
- *The blade not being loaded.* If the blade is carried above the
  surface, it senses nothing. The earliest cue is then the body itself:
  the onset of pitch as the front of the track enters the hole. The dip
  develops over the time it takes the track to advance into it, so even
  that onset, with the speed, allows the remainder of the motion to be
  predicted from a model and partly anticipated.
- *Ground behaviour.* Edges collapse and soft ground compacts under the
  tracks, so the surface the tracks meet is not exactly the one the
  blade left.

**What it needs from the estimator.**

- The base is a moving body with pitch, roll and height as states, with
  the blade's position relative to it from the linkage kinematics.
- A terrain profile as a fixed-size buffer along the path, one per
  track, holding height and a confidence for each sample. It lives with
  the estimator's state and involves no allocation.
- The predicted terrain input also improves the estimator itself: a
  pitch change that the motion model expects is tracked with less lag
  than one that arrives as a surprise.
- A model of the body on its undercarriage: at least the geometric
  bridging over the contact length, and where it matters the pitch
  dynamics of the machine on its tracks.

**How it relates to the first use.** Both start from detecting that a
tool has lost its support. In the first, the loss is the disturbance and
the response must come within tens of milliseconds. In the second, the
loss is information about a disturbance that will arrive later by a
different route, and the response is planned over the intervening time.
The detection can share its implementation; what follows it is
different.

**Assessment for this use.** Promising, and achievable with slow cues.
The value comes mostly from the terrain memory and the prediction of
body motion, which help with every irregularity the blade leaves or
passes over, not only with holes. Blade load is the preferred detector
where it is measured; the vibration-based cue is a usable alternative
here, which it was not for the sudden release.

**What to establish first.** From recordings of passes over known holes
or steps: the delay between the event at the blade and the body's
response, how well the body's pitch is predicted from a simple bridging
model of the tracks on the recorded profile, and which blade-side signal
marks the hole most reliably.

## Assessment

- **Feasible as a supporting cue** for sustained, reasonably stiff
  contact, with a latency of tenths of a second, when the structure is
  excited.
- **Not sufficient alone**: silent without excitation, weak for soft or
  light contact, slow for the moment of touch, and dependent on a
  baseline that varies with configuration, load and oil temperature.
- **Limited by the sample rate.** At 100 Hz only the lowest modes are
  visible and impact transients are poorly captured. A higher rate on
  the outer links would shorten the latency, keep shifted modes in band
  and make impacts usable. This, together with any correction for
  vibration in the estimator, is the case in which a faster inertial
  rate would be justified, and with it the deferred work in
  `filter-numerics.md`.
- **For sudden loss of contact**, the stated use, it is too slow to be
  the trigger. It contributes the free modal frequency and damping that
  the compensation needs, and confirmation afterwards. Detection has to
  come from actuator force, the tool link's acceleration, or the
  estimator's innovation, with a sequential change test.
- **For anticipating the body's motion over a hole the blade has
  passed**, the second stated use, the time available is long enough
  for slow cues, and the vibration-based estimate is usable alongside
  blade load. The larger gain there is from remembering the surface
  along the path and predicting the body's motion over it.
- **Cheap to find out.** Whether the effect is visible on this machine
  is answered by existing or easily made recordings, before anything is
  built.

## Suggested steps

1. **Look at the data.** From recordings with known free and contact
   periods (labelled from pressures, video or the operator), compute
   spectrograms per unit and axis and the amplitude ratios between
   neighbours. If the two conditions are not visibly different there,
   stop.
2. **Quantify the confounders.** Repeat across configurations, loads and
   surfaces. This decides whether a baseline map is practical.
3. **Prototype the features off line**: amplitude ratios and a
   recursive frequency estimate, combined in a two-hypothesis test with
   persistence, scored against the labels for latency, false alarms and
   misses.
4. **Fuse** with the actuation residual, the base reaction and, where
   available, actuator force. The output is a contact probability with
   a flag saying whether there was enough excitation to judge.
5. **Only then** consider a higher sample rate or active probing, on
   the evidence of what the off-line prototype lacks.

## Open points

- Actuator pressure or force is rarely available, and commands
  (joystick or pilot valve rate) only on some machines (stated by the
  owner). So the fast cues are mainly inertial, with the actuation
  residual where commands exist. On which machine types are commands
  available?
- Contact detection is wanted, at least, for detecting sudden loss of
  contact and compensating the overshoot (stated by the owner). How
  long does the first excursion take on the machine, and how large is
  it? How fast can the actuators respond? Do recordings of such events
  exist? Is the estimator's use of contact (noise model, kinematic
  constraint) also wanted?
- Which surfaces matter (hard ground, loose material, structures)?
- Do recordings exist with periods of known contact?
- Can the inertial sample rate be raised on the outer links, and is a
  small deliberate dither acceptable?
- For the dozer case: are lift and tilt cylinder pressures measured?
  How far ahead of the tracks is the cutting edge, and at what speeds
  does the machine work? Is the body's pitch already fed back to the
  blade, and from which sensor? Is a design or as-built surface model
  available on the machine?
- Which machine types are in scope? The examples so far imply both an
  articulated chain of five or six links and a dozer.
- Is the load on the tool known or estimated?
