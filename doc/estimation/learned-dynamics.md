# Evaluation: learned models for dynamic information the kinematic model cannot observe

Part of the state estimation and calibration plan; see
[README.md](README.md) for the index and the work streams.

Streams: X3 (exploratory). Status: an evaluation of a proposed research
direction, not a design. The description of the two techniques is from
general knowledge of the literature and was not checked against current
papers. No established result is known to me for combining them on this
kind of machine, so this is research with the corresponding risk.

## The proposal

The estimator in this plan is built on a kinematic model: geometry,
joint coordinates, rates. It deliberately knows little about forces,
masses, stiffness or the material being worked. Some of that information
is nevertheless present in the sensor streams, in how the machine
responds to commands and to the ground. The proposal is to extract it
with two learned components:

- a **joint-embedding predictive architecture** (JEPA): a
  self-supervised model trained to predict, from a window of sensor and
  command data, the *representation* of a later or masked window, not
  the raw signal. Because it predicts in a learned latent space, it is
  not forced to model noise and unpredictable detail, and the latent
  state it learns tends to hold whatever is persistent and predictive;
- networks constrained by the **Buckingham Pi theorem**: the inputs are
  combined into dimensionless groups before any learning, either fixed
  in advance or found as the first layer under the constraint that every
  group is dimensionless. Any physical relation can be written in such
  groups, which reduces the number of inputs and makes one relation
  valid across machines of different size.

## What "otherwise unobservable" could mean here

Quantities the kinematic estimator does not have, that matter to the
uses already discussed:

| Quantity | Why it is wanted | Where the information would come from |
|---|---|---|
| Load carried on the tool | Shifts the vibration baseline; changes actuation response | Response of the joints to commands; low-frequency vibration |
| Resistance of the material | Stored energy before a breakthrough; blade load | Motion achieved for a given command; base reaction |
| Contact state and stiffness | `contact-detection.md` | Vibration behaviour, transients, actuation residual |
| State of the hydraulics | Compliance, lag, deadband varying with temperature and pressure | Command-to-motion relation over time |
| Structural vibration state | Disturbance on the outer sensors | Inertial signals in the low band |
| Track slip and ground properties | Dozer terrain preview | Commanded versus achieved motion; pitch response |

## The first question is observability, not architecture

A learned model cannot extract what the signals do not contain. Before
any network is chosen, each target quantity needs an answer to: is it
determined, even in principle, by commands, inertial data and GNSS at
100 Hz, with whatever else is measured?

- Where a rough physical relation can be written down, the same
  curvature test used elsewhere in this plan answers it.
- Where it cannot, a data test does: fit the simplest possible model to
  predict the quantity from the signals on a test machine that has a
  temporary reference sensor for it. If a simple model cannot do better
  than chance, a large one will not conjure the information.

Several of the quantities above are strongly confounded with each other
in the available signals (load against material resistance against
hydraulic state). A latent representation will encode some mixture of
them. Whether the mixture can be separated depends on excitation and on
what else is measured, exactly as for the physical model.

## What each technique offers

**JEPA.**

- *Uses the data that exists.* Operational recordings are plentiful and
  unlabelled. Self-supervised pre-training makes use of them; labels are
  needed only afterwards, and far fewer of them.
- *Suited to noisy signals.* Predicting a representation instead of the
  signal avoids spending capacity on vibration and sensor noise that
  cannot be predicted anyway.
- *A latent state with memory.* What persists and helps prediction, such
  as a load or a ground condition, is what such a model is pushed to
  keep.
- *A built-in novelty measure.* Prediction error in the latent space
  rises when the machine behaves unlike anything in training, which is
  useful as a monitor in its own right.

Its costs: the latent state is not a physical quantity and has to be
read out by a second, supervised stage; training has known failure modes
(representations collapsing to a constant) that need the standard
counter-measures and experience; and results depend on choices of
masking, horizon and architecture that have no established recipe for
inertial and command data on machines.

**Dimensionless groups.**

- *One model across machine variants.* This plan already has to cope
  with different link lengths, sensor stacks and kinematic variants.
  Expressed in dimensionless groups (accelerations against gravity,
  lengths against a link length, time against a natural period, forces
  against a weight), a relation learned on one size can transfer to
  another. This is the strongest argument for it here.
- *Fewer inputs, less data.*
- *Physical consistency for free.* A network that can only form
  dimensionless combinations cannot learn a relation that depends on the
  choice of units. The unit annotations already planned for ports and
  block parameters are exactly the information needed to build the
  groups.
- *More interpretable.* The learned groups can be inspected and compared
  with known ones.

Its limits: the theorem only helps if the list of governing quantities
is complete. A relevant quantity that is not among the inputs (oil
temperature, material density) leaves the similarity incomplete, and the
model then varies with something it cannot see. And with data from a
single machine size, the benefit is little more than good normalisation;
the gain appears when several sizes are in the training set.

**Together.** The combination is natural: dimensionless inputs to the
encoder, prediction in the latent space. I know of no demonstrated
result for it on comparable systems. It should be approached as an
experiment with a defined stopping rule, not as a planned component.

## Where a learned component would attach to the estimator

In increasing order of how much is entrusted to it:

1. *As a monitor.* Latent prediction error as a novelty or fault
   indicator, alongside the parity residual. Nothing depends on it.
2. *As a mode estimate.* A probability of contact, or of a load class,
   that selects between noise models or weights a contact factor. The
   estimator stays physical; the learned part only tells it which
   regime applies.
3. *As a virtual sensor.* An estimate of a physical quantity (load,
   blade force) with a stated uncertainty, entering the factor graph as
   a measurement. This requires the uncertainty to be calibrated, which
   has to be tested, not assumed.
4. *As a residual model.* The physical model predicts; the learned model
   predicts what the physical model gets wrong. For the actuation model
   in particular (deadband, backlash, lag varying with temperature),
   learned actuator models are an established practice in legged
   robotics, and a learned residual on top of the identified smooth
   model is the conservative way to adopt it.
5. *As the process model itself.* Not proposed. It removes the structure
   that makes the estimator checkable.

A learned output is never used as evidence for calibration parameters
without a guard, since a model trained on a miscalibrated machine will
have absorbed the miscalibration.

## Constraints from the target and from safety

- *Inference is a block.* The deployed model is a pure function with
  fixed-size state and a fixed operation count, generated as plain C++
  with its weights as constants, like the generated kinematics. No
  learning framework runs on the target. At 100 Hz a small network is
  well within budget; the size limit comes from the worst-case time of
  the thread it runs in.
- *Bounded influence.* Whatever it feeds is clamped and weighted by its
  own confidence, and the system must behave acceptably with it switched
  off. That is also the test configuration that shows what it adds.
- *Out-of-distribution behaviour.* The novelty measure gates the
  model's own outputs: when the input is unlike the training data, its
  estimates are down-weighted to nothing.
- *The simulator is of limited help.* A rigid-body simulator does not
  represent soil or hydraulics faithfully, which are the very things
  this is meant to capture. It is useful for quantities it does
  represent (load, contact with a rigid surface) and for testing the
  integration, not as the main source of training data.

## Simpler alternatives that must be beaten

A learned latent model is only worth its complexity if it outperforms
these, and each is a useful step in its own right.

- *The physical identification already planned*: actuation factors with
  smooth deadband and backlash, and their residuals as an external-force
  estimate (`calibration.md`).
- *A virtual sensor trained with a temporary reference.* Fit pressure
  sensors or a load cell to one test machine, record, and train a small
  supervised model to reproduce them from the permanent sensors. Deploy
  the model, not the sensor. This is the most direct route to the
  quantities in the table and needs no self-supervised stage.
- *Hand-built features with a small classifier*, as in
  `contact-detection.md`.
- *Sparse regression in dimensionless variables*, which yields an
  explicit formula instead of a network when the relation is simple.
- *A physical model with a learned residual*, item 4 above.

The case for JEPA specifically is label efficiency: if pre-training on
unlabelled fleet data lets the read-out stage reach a given accuracy
with far fewer reference-sensor recordings, or transfer to a machine
variant with none, it has earned its place. That is a measurable claim.

## Suggested steps and stopping rules

1. **Choose two or three target quantities** and establish, per
   quantity, whether it is observable at all (the section above). Stop
   for any that is not.
2. **Collect** synchronised recordings (stream A0), and instrument one
   machine with temporary reference sensors for the targets.
3. **Baselines**: physical identification and a small supervised model
   on dimensionless inputs. Record their accuracy and how it degrades on
   a different machine variant.
4. **Dimensionless groups**: compare fixed, hand-chosen groups with
   learned ones; test transfer across variants. Stop here if transfer
   does not improve.
5. **JEPA pre-training** on unlabelled data, then the same read-out
   stage. Compare with step 3 at equal numbers of labels. Stop if it
   does not reduce the labels needed or improve transfer.
6. **Integration** at the lowest level that delivers the benefit,
   starting as a monitor or mode estimate.
7. **Deployment** as a generated block, with the off-switch test.

Steps 1 to 4 are worth doing regardless of the outcome of step 5.

## Assessment

- **Dimensionless formulation: recommended.** It is cheap, fits the
  unit information the plan already carries, and directly addresses the
  variation between machine configurations. It is worth doing for any
  learned or identified model, with or without JEPA.
- **A virtual sensor trained against temporary reference sensors: the
  practical route** to the dynamic quantities, and the baseline for
  everything else.
- **JEPA: a reasonable research bet, not a dependency.** Its potential
  is in using unlabelled data and in transfer. It carries real training
  risk, has no established recipe for this kind of data, and its latent
  state still has to be tied to physical quantities by a supervised
  stage. Nothing else in the plan should wait for it.
- **Nothing learned replaces the observability question.** The same
  confounding that limits the physical estimator limits the learned one.

## Open points

- Which dynamic quantities are actually wanted, and for which of the
  uses (contact, overshoot compensation, terrain preview, load)?
- Can one machine be fitted temporarily with pressure sensors or load
  cells as a reference?
- How much unlabelled recorded data exists, from how many machines and
  variants?
- How many machine sizes are in scope? The case for dimensionless
  groups grows with that number.
- Is a learned component acceptable in the control path at all, and if
  so at which of the five levels above?
- Is there existing work or a preferred toolchain for training and for
  generating inference code?
