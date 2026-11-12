# Calibration observations as data

Part of the state estimation and calibration plan; see
[README.md](README.md) for the index. Companion to
[calibration.md](calibration.md), which describes how these observations
are used as factors. Streams: A5.

Status: a design sketch for discussion. No code exists. The C++ below
shows the intended shape of the types, not a final interface, and uses
C++26 features that have not been tried on GCC 16 here. These types
belong to an off-line calibration tool, so they may allocate and use
`std::string` and `std::vector`; they are not port data and never appear
on a real-time path.

## What the record has to capture

1. **What was measured**: a typed payload per kind of observation.
2. **Between what**: the frames, reference points, axes, targets and
   instrument set-ups involved, by identifier.
3. **How well**: an uncertainty model, which is often more than one
   number.
4. **Under which assumptions**: most importantly that several
   observations were taken in the same machine pose, from the same
   instrument set-up, to the same mark.
5. **Whether it still counts**: a status that can change after the fact,
   with the reason, and that propagates to everything depending on it.
6. **Where it came from**: who, when, with which instrument, as entered.

## Identifiers

Every entity is referred to by a typed identifier, so that a point
cannot be passed where a frame is expected, and so that the dependencies
of an observation can be read off its members.

```cpp
namespace cal {

template <class Tag> struct id {
    std::uint32_t value{};
    friend auto operator<=>(id, id) = default;
};

using frame_id       = id<struct frame_tag>;        // a link, the base, the site frame
using point_id       = id<struct point_tag>;        // a named reference point on a frame
using axis_id        = id<struct axis_tag>;         // a named direction on a frame
using target_id      = id<struct target_tag>;       // a prism or antenna mounted at a point
using mark_id        = id<struct mark_tag>;         // a fixed mark in the world
using instrument_id  = id<struct instrument_tag>;   // a particular tape, total station, rover
using setup_id       = id<struct setup_tag>;        // one placement of an instrument
using pose_group_id  = id<struct pose_group_tag>;   // "the machine did not move"
using component_id   = id<struct component_tag>;    // a physical part: sensor unit, tool, pin
using observation_id = id<struct observation_tag>;
using session_id     = id<struct session_tag>;

}
```

In files the entities are named by strings; identifiers are assigned on
loading.

## Reference entities

```cpp
namespace cal {

enum class source { drawing, surveyed, defines_frame, estimated };

struct reference_point {
    point_id    id;
    frame_id    frame;
    std::string name;                 // "boom.pin_a"
    vec3        nominal;              // coordinates in the frame, metres
    vec3        sigma;                // prior standard deviation; zero if it defines the frame
    source      origin;
};

struct reference_axis {
    axis_id     id;
    frame_id    frame;
    std::string name;
    std::variant<vec3, std::pair<point_id, point_id>> definition;   // direction, or through two points
};

struct target {                       // something an instrument aims at
    target_id    id;
    point_id     at;                  // the reference point it marks
    target_offset offset;             // from that point to the optical or phase centre; may be
                                      // partly unknown, see "Points measured on a surface"
    double       prism_constant;      // metres, zero for an antenna
    component_id mounting;            // changes if the target is re-mounted
};

struct mark { mark_id id; std::string name; std::optional<vec3> known_position; };

struct instrument {
    instrument_id id;
    std::string   kind, serial;
    std::chrono::sys_days calibrated_on;
};

struct setup {                        // an instrument placed somewhere, not moved since
    setup_id      id;
    instrument_id instrument;
    bool          levelled;           // if true its vertical is gravity: four unknowns, not six
    time_range    valid;
};

}
```

## Uncertainty

A single "expected precision" is not enough for the instruments
involved.

```cpp
namespace cal {

struct noise_model {
    // Per-component standard deviation, in the units of the payload.
    std::vector<double> sigma;

    // Part that grows with the measured quantity, e.g. 1 mm + 2 ppm of distance.
    std::vector<double> proportional;

    // Full covariance where components are correlated (converted coordinates).
    std::optional<matrix> covariance;

    // Errors shared with other observations: the scale error of one tape,
    // the centring error of one set-up. Named so the solver creates one
    // unknown for all observations that share it.
    std::vector<shared_error_id> shared;

    robust_loss loss = robust_loss::none;
};

}
```

## The payloads

One struct per kind of observation. Each names the entities it relates
by identifier and holds the values as entered.

```cpp
namespace cal {

struct tape_distance {
    point_id from, to;
    double   length;                          // metres, as read
    std::optional<double> temperature;        // for the tape's expansion
    bool     supported;                       // false if the tape hung free (sag)
};

struct plumb_alignment {                      // an axis of a link is vertical
    axis_id  axis;
    bool     pointing_down;
};

struct level_reading {                        // digital level or inclinometer on an axis
    axis_id  axis;
    double   angle;                           // radians from horizontal
};

enum class face { one, two };

struct ts_polar {                             // raw total-station observation
    setup_id  from;
    target_id to;
    double    horizontal_angle;               // radians, instrument convention recorded in the set-up
    double    zenith_angle;                   // radians from the upward vertical
    double    slope_distance;                 // metres, before prism constant
    face      telescope_face;
    double    atmospheric_ppm;
    bool      tracking;                       // true if taken while tracking, not as a static shot
    bool      lock_settled;                   // false while the instrument was still acquiring
    std::vector<target_id> candidates;        // other prisms that were in view
};

struct ts_position {                          // only if the instrument gives coordinates
    setup_id  from;
    target_id to;
    vec3      position;                       // in the instrument's frame
};

struct gnss_position {                        // a rover on a point, or the machine's own antenna
    target_id antenna;
    vec3      position;                       // in the site frame
    fix_type  fix;
};

struct touch {                                // a reference point placed on a fixed mark
    point_id  point;
    mark_id   on;
};

struct frame_tie {                            // a designed or assumed relation between frames
    frame_id  a, b;
    enum class kind { coincident_points, parallel_axes, known_transform } relation;
    std::variant<std::pair<point_id, point_id>,
                 std::pair<axis_id, axis_id>,
                 transform> detail;
    bool      exact;                          // a constraint, not a noisy observation
};

struct onboard_snapshot {                     // what the machine's own sensors read in a static pose
    recording_ref recording;                  // which log, which time interval
    time_range    interval;
};

using payload = std::variant<tape_distance, plumb_alignment, level_reading,
                             ts_polar, ts_position, gnss_position,
                             touch, frame_tie, onboard_snapshot,
                             offset_measurement, width_measurement, epoch_tie>;   // the last two: see
                                                                       // "Points measured on a surface"

}
```

A stream from a tracking instrument is not stored as thousands of
`ts_polar` records. It is one series object: the set-up, the target, a
reference to the sample file, per-sample lock flags, and the identifier
of the time-offset parameter it shares.

## The observation record

```cpp
namespace cal {

enum class role   { calibration, validation, check };
enum class status { valid, suspect, invalid, superseded };

struct provenance {
    session_id    session;
    std::string   taken_by;
    time_point    taken_at;
    instrument_id instrument;
    std::string   as_entered;         // the raw text or file line, before parsing and unit conversion
    entry_method  method;             // typed by hand, imported, automatic
};

struct observation {
    observation_id id;
    payload        what;
    noise_model    noise;
    role           use = role::calibration;

    // Assumptions this observation shares with others.
    std::optional<pose_group_id> pose;        // absent for observations that do not depend on machine pose

    // Physical parts whose state the observation depends on, with the
    // version of each at the time (see "Physical changes").
    std::vector<component_version> components;

    provenance     origin;
    std::string    note;
};

}
```

Status is deliberately not a member. It changes over time and is derived
from the event log described below.

## Linked observations

A link states an assumption that several observations share. Each kind
of link corresponds to something the solver treats as one unknown.

| Link | Assumption | Shared unknown it creates | Evidence that should be kept |
|---|---|---|---|
| Pose group | No joint moved between these observations | The joint positions | The stillness statistic over the group's time span, from the on-board sensors |
| Slew epoch | The upper structure was not slewed | The plane's heading | Slew sensor, integrated yaw rate; see "Plane heading" |
| Base placement | The base was not moved on the ground | Base position and tilt | Track motion, GNSS, base inertial unit |
| Set-up | The instrument did not move | The instrument's pose (four unknowns if levelled) | A repeated shot to a fixed target at the start and end |
| Mark | These touches were on the same physical mark | The mark's position | - |
| Target mounting | The prism or antenna was not re-mounted | Its offset from the reference point | The component version |
| Shared error | These readings used the same tape, or the same centring | One scale or offset error | The instrument identifier |
| Face pair | Two shots of the same target on both faces | Cancels the instrument's axis errors | - |
| Repeat | These are repeated measurements of the same quantity | Nothing; used for blunder checks | - |
| Supersedes | A re-measurement replaces an earlier one | Nothing; the earlier one becomes `superseded` | - |

```cpp
namespace cal {

struct pose_group {
    pose_group_id id;
    time_range    span;
    double        stillness_statistic;        // the windowed test of sensor-models.md
    std::optional<onboard_snapshot> sensors;  // inertial and GNSS data for this pose
};

struct link {
    enum class kind { face_pair, repeat, supersedes } what;
    std::vector<observation_id> members;
};

}
```

Pose groups, set-ups, marks and targets are entities in their own right,
referred to by identifier from the observations, and not lists of
observations. That is what lets one of them be invalidated as a unit.

## Dependency graph and invalidation

**Nodes.** Every entity above, every observation, every link, and every
derived result (a calibration, a validation report).

**Edges.** "Depends on". They are not written by hand: an observation
depends on every entity whose identifier appears in its payload and
header, which reflection over the struct members enumerates. A result
depends on every observation that was used in the solve that produced
it.

```
reference_point ─┐
target ──────────┼─▶ observation ─┐
setup ───────────┤                ├─▶ calibration result ─▶ validation report
pose_group ──────┤                │
instrument ──────┘   link ────────┘
```

**Events.** Nothing is edited or deleted. Changes are appended events:

```cpp
namespace cal {

struct event {
    time_point  at;
    std::string by;
    node_ref    subject;                      // any entity, observation, link or result
    status      new_status;
    std::string reason;                       // "tripod bumped", "wrong pin", "moved during group"
};

}
```

The current status of a node is the worst of its own latest event and
the current status of everything it depends on. So:

- marking a **set-up** invalid, because the tripod was knocked,
  invalidates every observation taken from it and every result that
  used any of them;
- marking a **pose group** invalid, because the stillness test failed
  when the recording was examined, invalidates the group's observations
  as a group, though each may still be usable individually if it does
  not need the pose to be shared;
- correcting a **reference point's** coordinates makes results that used
  it stale, not wrong: a third state, `stale`, is used for results whose
  inputs have changed and which need solving again;
- marking an **observation** suspect removes it from the next solve or
  down-weights it, without discarding it.

Every result records exactly which observations it used and their
status at the time, as a hash, so a stale result is detected
mechanically and a past result can be reproduced.

**Suspect is not invalid.** Leave-one-out analysis and normalised
residuals flag observations automatically as suspect. A person confirms
or clears them. Automatic checks never mark anything invalid on their
own.

### Physical changes are versions, not invalidations

Replacing a sensor unit, re-mounting a prism or changing the tool does
not make earlier observations wrong. They are still true statements
about the machine as it was. Treating them as invalid would throw away
what they say about everything that did not change.

```cpp
namespace cal {

struct component_version {
    component_id  part;                       // "imu.stick", "prism.tool", "tool"
    std::uint32_t version;                    // incremented by a physical-change event
};

}
```

Each parameter that belongs to a component (a unit's mounting
orientation, a target's offset) is a separate unknown per version. An
observation taken before the change constrains the old version's
parameter and all the parameters that were unaffected, such as link
lengths. An observation taken after constrains the new one. Nothing has
to be invalidated, and the old data keeps contributing where it is still
relevant.

## Points measured on a surface, not where the model has them

**The problem.** The model's reference points are usually on a link's
centre plane: the centre of a pin, a point on the centreline. A prism, a
tape end or a rover pole cannot be put there. It sits on an accessible
surface, a side plate or a pin end, displaced from the modelled point
along the normal of a reference plane by an offset that is often not
known. That offset may be measured separately, typically with a tape to
the centreline, or it may have to be estimated or ignored.

The record therefore has to say three things that the earlier `target`
with a fixed offset vector could not: which plane the measurement is
referred to, that the displacement from it is a quantity in its own
right, and whether that quantity is known, measured or unknown.

**Entities.**

```cpp
namespace cal {

using plane_id  = id<struct plane_tag>;
using offset_id = id<struct offset_tag>;

struct reference_plane {              // e.g. a link's centre plane, the mechanism's plane of motion
    plane_id    id;
    frame_id    frame;
    std::string name;
    vec3        normal;               // nominal, in the frame
    point_id    through;              // a reference point lying in the plane
    double      tilt_sigma;           // how well the nominal normal is trusted, radians
};

enum class side { positive, negative };          // which way along the normal

struct surface_offset {               // distance from the plane to an accessible surface
    offset_id    id;
    plane_id     from;
    side         towards;
    std::optional<double> nominal;    // from drawings, if known
    double       sigma;               // prior; large if unknown
    component_id surface;             // the part carrying the surface (a plate, a pin end)
};

// How a target relates to the reference point it stands for.
struct fixed_offset   { vec3 vector; };                      // fully known, e.g. a pole of known height
struct normal_offset  { offset_id along; vec3 in_plane; };   // unknown or measured distance along a plane normal,
                                                             // plus a known displacement within the plane
using target_offset = std::variant<fixed_offset, normal_offset>;

}
```

`target::offset` becomes a `target_offset`. A surface offset is an
entity with an identifier, so every observation made on the same surface
refers to the same one, and it is one unknown in the solve however many
observations use it.

**Observing the offset itself.** The tape to the centreline is an
observation of that unknown, with its own payload:

```cpp
namespace cal {

struct offset_measurement {           // direct measurement of a surface offset
    offset_id of;
    double    distance;               // metres, perpendicular to the plane
    bool      to_centreline;          // measured to a marked centreline, as opposed to half a measured width
};

struct width_measurement {            // both faces accessible: full width across the plane
    offset_id positive_side, negative_side;
    double    width;
};

}
```

A width measurement constrains the sum of two offsets. With a symmetry
assumption, stated as a link between the two offsets, it gives each.

**Three ways to treat an offset**, chosen per offset by its prior and
the observations available, not by separate code:

- *Known*: a nominal value with a small sigma, from drawings or a
  measurement already made.
- *Measured in the session*: a wide prior and one or more
  `offset_measurement` records. The offset is estimated, and the tape
  reading is checked against the other observations like any other.
- *Unknown and unmeasured*: a wide prior and nothing else. The component
  of each observation along the plane normal then carries no information
  about the mechanism, and only the components within the plane
  constrain it. This is what "projecting onto the reference plane" means
  in the fit; it happens automatically and needs no separate projected
  observation type.

**Mirror pairs.** Where the same feature is accessible on both sides,
observing both and linking them as a mirror pair locates the centre
plane without knowing either offset, provided the two are equal or their
sum is known from a width measurement. It is the most robust way to
find a centre plane with a total station and is worth planning for.

```cpp
struct mirror_pair {                  // added to the link kinds
    observation_id positive_side, negative_side;
    bool           assume_symmetric;
};
```

**The plane's direction is uncertain too.** A side plate is not exactly
parallel to the plane of motion. An error in the normal turns an offset
of size d into an in-plane error of about d times the tilt. With several
points observed on one surface, the tilt (two angles) can be estimated;
otherwise it is carried as a considered uncertainty from `tilt_sigma`.

**How much it matters.**

- For a planar model of the mechanism, offsets along the normal of the
  plane of motion do not enter the in-plane kinematics. They matter only
  through the tilt term above, which is small, so treating them as
  unknown is usually acceptable.
- For the full three-dimensional model they matter directly.
- A tape distance between two surface points on different links is the
  hypotenuse of the in-plane distance and the difference of the two
  offsets. If both points are on the same side with similar offsets the
  effect is second order and often negligible; if they are on opposite
  sides it is not. The residual is computed exactly from the offsets in
  either case, so nothing has to be judged by hand, but the session plan
  should prefer same-side pairs.
- An offset along the normal is strongly correlated with the lateral
  position and the roll of whatever carries the plane. A total station
  alone separates them poorly; a tape to the centreline, a mirror pair
  or a width measurement breaks the correlation. The observability
  report shows which offsets were actually determined.

**Invalidation.** A surface offset belongs to a component. If the plate
is replaced or a wear pad is added, that is a new version of the
component and a new unknown; the old observations keep constraining
everything else.

## Plane heading, and the levels of "nothing moved"

**Heading is an unknown of the session.** External instruments observe
points in their own frame. To use them, the direction in which the
mechanism's plane of motion faces, its heading in the instrument's
frame, has to be known, and it generally is not: it is inferred during
the calibration from the plane-offset points or from explicit centreline
points. It is also the quantity most easily disturbed. Any slew or
movement of the machine changes it, and every observation that relied on
the old value then belongs to a situation that no longer exists. This is
expected to be the commonest reason for having to recapture a pose, so
the data model treats it explicitly.

**How heading is determined, in order of robustness.**

- *One target swept through a joint's range.* A point on a moving link,
  observed at several joint positions without slewing, traces an arc
  that lies in a plane parallel to the plane of motion. Fitting that
  plane gives the heading and the sideways tilt directly. The target's
  offset from the centre plane does not matter, since it is the same at
  every position. This needs one prism and no knowledge of any offset.
- *Explicit centreline points*, far apart. Two points known to lie in
  the centre plane give the heading; the uncertainty is roughly the
  point accuracy divided by the horizontal distance between them, so the
  baseline should be as long as the machine allows.
- *Points on one surface.* Several points on the same plate share one
  offset and so lie in a plane parallel to the centre plane; they give
  the heading without the offset being known.
- *Mirror pairs*, which give centre-plane points from surface points.
- *Points on different surfaces with unknown offsets* are the weak case:
  a difference between the two offsets is indistinguishable from a
  heading error of that difference divided by the baseline. The
  observability report flags this.
- *On-board sources.* A two-antenna GNSS heading, or a slew sensor, tie
  heading between situations even when the instrument does not. Only
  some machine configurations have two antennas, so which ties are
  available is read from the machine's configuration record, and the
  session plan differs accordingly: single-antenna machines need a slew
  sensor tie or fresh heading observations after every slew.

Heading error is multiplied by reach: at the tool point it produces a
sideways error of the reach times the heading error. The session plan
uses that to say how long a baseline or how wide a sweep is needed for
the accuracy wanted.

**Not one assumption but a nesting of them.** "Same pose" lumps together
things that change at very different rates. Split, each level is an
entity with its own identifier, time span and evidence, and each owns
the unknowns that are constant while it lasts:

| Level | Holds while | Unknowns it owns | Typical evidence |
|---|---|---|---|
| Instrument set-up | The instrument is not moved | Instrument pose | Check shot to a fixed target |
| Base placement | The machine's base is not moved on the ground | Base position and tilt | Track or wheel motion, GNSS, base inertial unit |
| Slew epoch | The upper structure is not slewed | **Plane heading** | Slew sensor, integrated yaw rate, two-antenna heading |
| Pose group | No joint is moved | Joint positions | The stillness test over all units |

```cpp
namespace cal {

using base_placement_id = id<struct base_placement_tag>;
using slew_epoch_id     = id<struct slew_epoch_tag>;

struct base_placement {
    base_placement_id id;
    time_range        span;
    evidence          stationary;     // what showed the base did not move
};

struct slew_epoch {
    slew_epoch_id     id;
    base_placement_id within;
    time_range        span;
    evidence          no_slew;        // slew sensor reading, integrated yaw, with their uncertainty
};

struct pose_group {                   // replaces the earlier definition
    pose_group_id id;
    slew_epoch_id within;
    time_range    span;
    double        stillness_statistic;
    std::optional<onboard_snapshot> sensors;
};

}
```

An observation still names only its pose group; the levels above follow
from it. The point of the nesting is that moving one joint ends a pose
group but leaves the heading intact, so the heading is determined by all
the observations of a slew epoch together, across many joint poses, and
the arc sweep above becomes an ordinary set of observations instead of a
special procedure.

**What happens when something changes.**

- *A known change ends a level; it invalidates nothing.* If the operator
  slews, the current slew epoch is closed at that time and a new one
  begins. Observations in the old epoch remain valid statements about
  the machine as it then stood. They still constrain everything internal
  to the mechanism (lengths, offsets, mountings) through their own
  epoch's heading unknown.
- *The new epoch starts without a heading.* It needs its own heading
  observations before anything in it is useful for quantities that
  depend on heading. The tool reports this as the re-establishment set:
  the smallest set of observations that makes the new epoch's heading
  determined, typically one sweep or two centreline points.
- *An unnoticed change is the real invalidation.* If the evidence shows
  a slew happened part-way through an epoch, the epoch is suspect.
  - If the time of the change can be located, from the slew sensor or
    the yaw rate, the epoch is **split** at that time: observations are
    re-assigned to the part they fall in. Nothing is lost except that
    each part now needs enough heading observations of its own.
  - If it cannot, the epoch's observations that depend on heading are
    invalid for that purpose and have to be recaptured.
- *The same applies one level up.* Moving the base ends the base
  placement and every slew epoch in it.

```cpp
namespace cal {

struct split {                        // an event: one level instance becomes two
    node_ref   subject;               // a slew epoch, base placement or pose group
    time_point at;
    std::string reason;
};

}
```

**Tying epochs together instead of starting from nothing.** A slew does
not make the new heading entirely unknown if something measured the
slew. The change reported by a slew sensor, or the integrated yaw rate
over the move, is an observation relating the headings of two
consecutive epochs, with its own uncertainty:

```cpp
namespace cal {

struct epoch_tie {                    // added to the payloads
    slew_epoch_id from, to;
    double        heading_change;     // radians
    tie_source    source;             // slew sensor, integrated gyroscope, two-antenna GNSS
};

}
```

With a tie, a new epoch inherits heading from the previous one to within
the tie's accuracy, and needs fewer external observations, or none, to
be usable. Without one, each epoch stands alone. Whether the on-board
tie is accurate enough to avoid recapture is again a question the
observability report answers for the actual numbers.

**Consequences for planning a session.**

- Do everything that needs one heading before slewing.
- Sweep a target through each joint's range early in each slew epoch:
  it establishes the heading and exercises the joint.
- After any slew, take the re-establishment observations first.
- Record the slew sensor and the base inertial unit throughout, so that
  unnoticed changes can be found and located afterwards.

## Single-joint sweeps

A sweep (one joint moved alone, with a tracked prism and the inertial
units recording) is described in `calibration.md`. As data it is one more
"nothing else moved" assumption, between the slew epoch and the pose
group, together with two series.

```cpp
namespace cal {

using sweep_id = id<struct sweep_tag>;
using joint_id = id<struct joint_tag>;

struct single_axis_motion {           // "only this joint moved, about one fixed axis"
    sweep_id      id;
    slew_epoch_id within;
    joint_id      moving;
    time_range    span;
    bool          both_directions;    // needed to separate play and instrument timing from geometry

    // Evidence, filled in when the recordings are examined.
    double rate_direction_scatter;    // spread of the gyroscope rate direction, radians
    double held_joint_motion;         // largest relative rotation seen at joints meant to be held
    double base_tilt_change;          // from the base inertial unit
    double out_of_plane_residual;     // of the tracked points about their fitted plane, metres
};

struct ts_track_series {              // a tracking stream, not thousands of single records
    sweep_id   during;
    setup_id   from;
    target_id  to;
    sample_ref samples;               // file and range: angles, distance, time stamp, lock flag per sample
    time_offset_id clock;             // the instrument's time offset, shared by all its series
};

struct inertial_series {              // the on-board recording covering the sweep
    sweep_id      during;
    recording_ref recording;
    time_range    still_before, still_after;   // for the bias
};

}
```

The sweep owns one unknown, the joint's axis as a line, shared by every
sample of both series. Its evidence fields are what decide whether the
single-axis assumption held. A sweep that fails them is marked suspect
as a unit; if the failure can be localised in time (a held joint started
to creep half-way), the sweep is split like any other level.

## How the records become factors

`std::visit` over the payload selects the residual function for that
kind, generated from the symbolic model. The identifiers say which
unknowns it touches: the parameters of the frames and points involved,
the pose of its pose group, the pose of its set-up, the position of its
mark, the versioned parameters of its components, and its shared-error
terms. The role decides whether it enters the fit or only the
validation. The status decides whether it is used at all.

Reading and writing the records, and the hash of their schema, come from
reflection over the structs, like port data types in `rtt3`, so the file format
needs no separate description.

## What else is needed

The request listed relationships, links, type, frame reference and
expected precision. The following are not in that list and matter.

**For correctness of the fit**

- *Role.* Whether an observation is for calibration or held out for
  validation. Without it, validation data leaks into the fit.
- *Conventions made explicit.* Angle units; whether the vertical angle is
  from the zenith or the horizon; the direction in which horizontal
  angles increase; telescope face; slope versus horizontal distance;
  prism constant; target height. Each is a classic source of systematic
  error and belongs in the instrument or set-up record, not in
  someone's memory.
- *Shared errors.* Readings from one tape share its scale error;
  observations from one set-up share its centring error. Modelled as
  independent, they are over-trusted.
- *Datum.* Which frame defines the origin and orientation, and which
  parameters are fixed by convention. This is part of the data set,
  because it decides what the results mean.
- *Exact constraints versus observations.* Two points that coincide by
  design are a constraint; a tape reading between them is an
  observation. The record must say which.
- *Association ambiguity.* With several prisms in view, which one was
  measured is itself uncertain. The candidates are recorded so the
  assignment can be checked and redone.

**For linking to the machine's own data**

- *The on-board sensor data for each static pose.* The inertial and GNSS
  readings during a pose group are factors on that pose. The record
  needs a reference to the recording and the time interval.
- *Evidence for the same-pose assumption.* The stillness statistic and
  the duration, so the assumption can be re-examined.
- *Time stamps and their clock.* For anything taken in motion, and for
  matching manual observations to recordings.

**For conditions that change the answer**

- *Environment.* Temperature and pressure for distance measurement;
  temperature and tension for a tape.
- *Machine condition.* Engine running or not, tool fitted, load carried,
  hydraulic temperature, how the machine is supported. Deflection and
  vibration depend on all of them, and observations taken in different
  conditions should not be assumed comparable.
- *Machine identity and configuration.* Serial number, component
  versions with sensor serial numbers, and the version and schema hash
  of the model the data was collected for.

**For trust and repeatability**

- *Raw values as entered*, kept beside the parsed ones, with unit and
  correction steps recomputable.
- *Instrument identity and calibration date.*
- *Who entered it and how*, and a place for a photograph reference.
- *An append-only history.* Supersede, never overwrite.
- *Results stored separately from observations.* Post-fit residual,
  normalised residual and leave-one-out influence belong to a particular
  solve, not to the observation.
- *The plan.* The proposed observations with their predicted information
  gain, linked to what was actually taken, and what was skipped and why.

## Open points

- Should the calibration tool share these types with the simulator, as
  the block package does, so that simulated calibration sessions use the
  same records?
- Which file format: one file per session, or one growing record per
  machine?
- How are manual observations entered in the field: a form on a tablet,
  a file from the instrument, typed afterwards?
- Does the total station export raw angles and distances with lock
  status, or only coordinates?
- Who is allowed to mark an observation invalid, and does that need a
  second person?
- Which reference planes are used (each link's centre plane, one plane
  of motion for the whole mechanism), on which surfaces are points
  actually measured, and are both sides accessible for mirror pairs?
  Are centrelines marked on the links?
- Does the machine have a slew sensor, and how accurate is it? Is the
  base moved during a calibration session, or only slewed? Is sweeping
  one prism through each joint's range practical?
- Is a `stale` state for results acceptable in addition to the four
  observation states?
