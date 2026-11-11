# rtt3: calibration observations as data

Part of the `rtt3` plan; see [README.md](README.md) for the index.
Companion to [estimation.md](estimation.md), which describes how these
observations are used as factors.

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
    vec3         offset;              // from that point to the optical or phase centre
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
                             touch, frame_tie, onboard_snapshot>;

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
| Pose group | The machine did not move between these observations | One static pose | The stillness statistic over the group's time span, from the on-board sensors |
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
    double        stillness_statistic;        // the windowed test of estimation.md
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

## How the records become factors

`std::visit` over the payload selects the residual function for that
kind, generated from the symbolic model. The identifiers say which
unknowns it touches: the parameters of the frames and points involved,
the pose of its pose group, the pose of its set-up, the position of its
mark, the versioned parameters of its components, and its shared-error
terms. The role decides whether it enters the fit or only the
validation. The status decides whether it is used at all.

Reading and writing the records, and the hash of their schema, come from
reflection over the structs, like port data types, so the file format
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
- Is a `stale` state for results acceptable in addition to the four
  observation states?
