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

- Are cylinder pressures or motor currents measured? If so, vibration
  is a secondary cue from the start.
- What is contact detection for: switching the estimator's model, a
  kinematic constraint, control of the tool, or monitoring? The required
  latency differs by an order of magnitude between these.
- Which surfaces matter (hard ground, loose material, structures)?
- Do recordings exist with periods of known contact?
- Can the inertial sample rate be raised on the outer links, and is a
  small deliberate dither acceptable?
- Is the load on the tool known or estimated?
