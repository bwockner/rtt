# rtt3: components, ports and data types

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S6. Port data concepts are also needed by S3 and S4.

A component is a plain struct. There is no base class and no registration
call; the `rtt3::component` concept recognises it by its members, and the
lifecycle state, activity and engine live in the `rtt3::node<T>` that the
deployment wraps around it.

```cpp
struct Controller {
    rtt3::input_port<JointState>    state;
    rtt3::output_port<JointCommand> command;
    double gain = 1.0;

    std::expected<void, rtt3::error> configure() noexcept;   // optional
    void update() noexcept;                                    // required
    void stop() noexcept;                                      // optional
};
```

- Hooks are found with `requires` expressions; absent hooks cost nothing.
  All calls are direct and inlinable. Inside the library, deducing `this`
  replaces CRTP where mixins are needed.
- Ports carry only what the data path needs: no name, no description, no
  virtual functions on the read and write paths.
- A port is identified by a member pointer, so discovery is a compile-time
  expression instead of a map lookup followed by `dynamic_cast`:

  ```cpp
  rtt3::connect(ctrl.command, driver.command_in, rtt3::policy::data{});
  ```

  A mismatch of value type is a failed `rtt3::connectable<Out, In>`
  constraint, reported at the call site.
- The connection policy is a type wherever it is known statically, so the
  buffer implementation is selected at compile time.
- Metadata is not written separately. The port list, names, properties
  and operations are read from the struct by reflection (`reflection.md`), and
  anything the declaration cannot express is an annotation on the member.
  Names exist only in tables generated at compile time; a system that
  never asks for a name does not link them, and where an identifier is
  needed at run time it is a compile-time hash of the name.
- Operations are annotated member functions, called
  through a typed handle (`rtt3::caller<&Controller::reset>`), with
  own-thread and client-thread execution and an asynchronous `send` that
  returns a typed handle. Signatures are handled with variadic templates,
  `std::invoke` and `std::tuple`; engine messages are
  `std::move_only_function` in fixed-capacity storage.
- Properties are annotated data members. Reading
  and writing them from a file is an `rtt3-reflect` feature.

## Port declaration without boilerplate

In RTT 2.x one port is written in up to five places:

```cpp
class Controller : public RTT::TaskContext {
    RTT::InputPort<JointState>    state_port_;                    // 1. member
    RTT::OutputPort<JointCommand> command_port_;
    JointState state_;                                            // 2. storage for the value read
public:
    Controller(const std::string& name)
        : TaskContext(name),
          state_port_("state"), command_port_("command")          // 3. name
    {
        ports()->addEventPort(state_port_).doc("Measured state"); // 4. registration, doc, event
        ports()->addPort(command_port_).doc("Commanded torque");
    }
    bool configureHook() {
        command_port_.setDataSample(JointCommand(7));             // 5. data sample
        return state_port_.connected();                           //    connection check
    }
};
```

In `rtt3` the declaration is the only place. Each item above has one
replacement, and none of them is a statement in a constructor or hook:

| RTT 2.x | `rtt3` |
|---|---|
| Name string in the constructor | The member's identifier, by reflection; `[[=rtt3::name("...")]]` only to override |
| `addPort` / `addEventPort` call | Nothing: being a public port member is the registration |
| `.doc("...")` | `[[=rtt3::doc("...")]]` on the member |
| Event port and callback | `[[=rtt3::event]]` on the port; the handler is a member function annotated with the port it handles |
| `setDataSample` in `configureHook` | A default member initialiser: `rtt3::output_port<Path> path{rtt3::sample(Path(64))};`, or nothing at all for bounded types |
| Connection check in `configureHook` | Inputs are required by default and checked when the deployment is built; `[[=rtt3::optional]]` to relax |
| Separate member to hold the last value read | `rtt3::latched_input<T>`: keeps the latest sample, `state.get()` returns it and `state.fresh()` says whether it changed this cycle |
| Connection policy passed at connect time | Default policy as an annotation on the port, overridable on the link |
| Constructor taking a name | None; a component is an aggregate and the node name comes from the deployment member |

The same component:

```cpp
struct Controller {
    [[=rtt3::doc("Measured state")]]   rtt3::latched_input<JointState> state;
    [[=rtt3::doc("Commanded torque")]] rtt3::output_port<JointCommand> command;

    void update() noexcept { command.write(law(state.get())); }
};
```

**Event handlers** are annotated on the function, not on the port, because
the function is declared after the port it refers to:

```cpp
[[=rtt3::event]] rtt3::input_port<JointCommand> command_in;

[[=rtt3::handles(&Driver::command_in)]]
void on_command(const JointCommand& c) noexcept;
```

The signature is checked against the port's value type at compile time.

**Repeated ports.** A `std::array` of ports is reflected as indexed ports
(`axis[0]`, `axis[1]`, ...), and links can address one element or connect
two arrays of equal size element by element.

**Port groups.** A set of ports that always travels together is a struct
used as a member; its ports are named with the member as prefix
(`drive.command`, `drive.state`). Writing the group once as a template
over a role gives both sides from one definition, so the two ends cannot
drift apart:

```cpp
template <class Role>
struct DriveIo {
    typename Role::template out<JointCommand> command;
    typename Role::template in<JointState>    state;
};

struct Controller { DriveIo<rtt3::requires_> drive; ... };   // writes command, reads state
struct Drive      { DriveIo<rtt3::provides>  io;    ... };   // reads command, writes state

// one link connects every port of the group
rtt3::link(&App::ctrl, &Controller::drive, &App::left, &Drive::io)
```

A group is also what a slot (`deployment.md`) checks its alternatives against, and
what link substitution cuts as a unit.

**Derived defaults.** A port with no annotation at all is valid: its name
is the identifier, it is required if it is an input, its policy is the
latest-value policy, and its documentation is empty. Annotations are only
for departures from that.

## Data types: composition, inheritance and link adaptors

RTT 2.x requires the two ports of a connection to have exactly the same
type, and describes the structure of a type through hand-written or
generated typekits. `rtt3` keeps exact match as the default and adds a
small, explicit set of ways to relate types, all checked at compile time.

**What may travel on a port.** Requirements are concepts, so a type that
does not qualify is rejected where the port is declared:

| Concept | Requirement | What it enables |
|---|---|---|
| `rtt3::port_data` | Copyable value type, default-constructible, not polymorphic (no virtual functions), reflectable public members | May be used as a port type at all |
| `rtt3::trivial_data` | Also trivially copyable | Bitwise copy in channels, automatic wire encoding for the UDP bridge, cheapest lock-free data object |
| `rtt3::bounded_data` | Size bounded at compile time (fixed-capacity containers allowed) | No allocation on the data path, without a data sample |
| `rtt3::dynamic_data` | May allocate (`std::vector`, `std::string` members) | Allowed only with an explicit annotation on the port and a data sample given at configuration |

Polymorphic types are excluded because samples are copied by value into
buffers; a base-typed copy of a derived object would silently lose data.
Alternatives of different shape are expressed as `std::variant`.

**Composition** is the preferred way to build data types. A struct whose
members are themselves port data is port data; reflection recurses through
members, so nested structs, arrays and enums need no declaration beyond
the struct itself. The generated schema keeps the nesting, which is what
the inspection tools and the property files see.

**Inheritance** is supported in its non-polymorphic form, for extending a
record (`struct StampedPose : Header, Pose {}`). Bases are reflected and
appear in the schema as named sub-records, the same as members would. It
gives no implicit substitutability on connections; see adaptors.

**Link adaptors.** A link between ports of different types must name how
one becomes the other. The adaptor is part of the link, runs in the
writer's thread at the moment of the write, and is visible in the
exported graph:

```cpp
static constexpr auto links = rtt3::links(
    // exact type match: no adaptor
    rtt3::link(&App::sensor, &Sensor::state, &App::ctrl, &Controller::state),

    // member projection: send one member of a composite
    rtt3::link(&App::robot, &Robot::state, &App::plot, &Plotter::joints,
               rtt3::member<&RobotState::joints>),

    // base projection: send the base part of a derived record
    rtt3::link(&App::tracker, &Tracker::pose, &App::ctrl, &Controller::target,
               rtt3::base<Pose>),

    // conversion by a named function
    rtt3::link(&App::imu, &Imu::rate_deg, &App::ctrl, &Controller::rate_rad,
               rtt3::convert<&deg_to_rad>),

    // variant injection: A into variant<A, B>
    rtt3::link(&App::joystick, &Joystick::cmd, &App::mux, &Mux::command,
               rtt3::inject));
```

- *Projection* (member or base) copies only the selected part, so it is
  cheaper than sending the whole record and filtering in the reader.
- *Conversion* takes a `noexcept` function from the output type to the
  input type. Implicit conversions between port types are not applied; a
  connection from `float` to `double` or between unit types has to say so.
- *Structural match* (`rtt3::structural`) connects two distinct types
  whose members have the same names and compatible types, with a
  member-wise copy generated by reflection. It lets two component
  libraries interoperate without sharing a header, at the price of
  matching by name. It is opt-in per link.
- An adaptor is checked by the `rtt3::connectable<Out, In, Adaptor>`
  concept; the error names the link and the member or type that does not
  fit.

**Not provided: automatic joining.** Assembling one composite input from
several outputs has no single correct meaning (which arrival triggers the
reader, what happens when the parts are of different ages), so it is not a
link adaptor. It is written as a component, where those choices are
explicit. A small generic `rtt3::join<T>` component with a stated policy
can be added later if the pattern is common.

**Schema identity.** Each port data type has a schema generated by
reflection and a hash of it. The hash is part of the graph hash (`runtime-layer.md`), so
a simulator or inspection tool built against an older layout of a type is
detected rather than decoding garbage. Renaming or reordering members
changes the hash; this is deliberate.
