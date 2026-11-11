# rtt3: operations and properties

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S10.

**RTT 2.x today.** Providing one operation and one property, and calling
the operation from another component:

```cpp
// provider
Drive::Drive(const std::string& name) : TaskContext(name), max_velocity_(1.0) {
    addOperation("reset", &Drive::reset, this, RTT::OwnThread)
        .doc("Clear a latched fault").arg("axis", "Axis index");
    addProperty("max_velocity", max_velocity_).doc("Velocity limit [rad/s]");
}
bool Drive::configureHook() {
    if (max_velocity_ <= 0.0 || max_velocity_ > 10.0) { /* log, */ return false; }
    return true;
}

// caller
RTT::OperationCaller<bool(int)> reset_;                    // signature repeated
Controller::Controller(const std::string& name) : TaskContext(name), reset_("reset") {
    requires("drive")->addOperationCaller(reset_);         // name repeated
}
bool Controller::configureHook() { return reset_.ready(); }   // bound at run time, by name
// ... and the deployment connects the services and loads a .cpf file per component.
```

**`rtt3`, provider side.** The declaration is the registration:

```cpp
struct Drive {
    [[=rtt3::property, =rtt3::doc("Velocity limit"), =rtt3::unit("rad/s"), =rtt3::range(0.0, 10.0)]]
    double max_velocity = 1.0;

    [[=rtt3::operation, =rtt3::doc("Clear a latched fault")]]
    std::expected<void, rtt3::error> reset(int axis) noexcept;

    void update() noexcept;
};
```

**`rtt3`, caller side.** One member gives access to every operation of a
provider type, with the signature taken from the function itself:

```cpp
struct Controller {
    rtt3::uses<Drive> drive;

    void update() noexcept {
        if (fault) drive.send(&Drive::reset, axis);      // asynchronous
    }
    [[=rtt3::completes(&Drive::reset)]]
    void on_reset(std::expected<void, rtt3::error> result) noexcept;   // runs in this node's thread
};

// deployment: bound and checked at compile time
rtt3::link(&App::ctrl, &Controller::drive, &App::left)
```

What disappears, item by item:

| RTT 2.x | `rtt3` |
|---|---|
| `addOperation("name", &C::f, this, thread)` | `[[=rtt3::operation]]` on the member function; name is the identifier |
| `.doc(...)`, `.arg("axis", "...")` | `rtt3::doc` annotation; argument names and types by reflection |
| Execution thread argument | Own-thread by default; `rtt3::operation(rtt3::client_thread)` to change |
| `OperationCaller<Sig>` member with the signature and name repeated | `rtt3::uses<Provider>`; the call names the member function, so signature and name cannot drift |
| `requires(...)->addOperationCaller`, `connectServices`, `ready()` checks | A link in the deployment, checked at compile time; an unbound `uses` is a build error unless marked optional |
| `SendHandle`, `collect`, status polling | An annotated completion handler, called in the caller's thread on a later cycle; or `auto h = drive.send(...)` and `h.poll()` when a handle is preferred |
| `addProperty("name", member)` | `[[=rtt3::property]]` on the member; default is its initialiser |
| Range and sanity checks in `configureHook` | `rtt3::range`, `rtt3::one_of`, or a named check function, applied when the machine file is loaded and when a value is changed at run time |
| `loadProperties("file.cpf")` per component | A section per node in the machine file; unknown and missing keys are reported by `--check` |
| `addAttribute`, `addConstant` | Dropped as separate kinds; a value to be watched is `[[=rtt3::readout]]`, visible to inspection, never settable |

Further points:

- **Property groups.** Annotating a struct member makes all its public
  members properties, nested under its name in the machine file:
  `[[=rtt3::property]] Gains gains;`. Bounded arrays and enums (by
  enumerator name) work without extra code.
- **Required properties.** A property with no sensible default is marked
  `rtt3::required`, and a machine file that omits it is rejected.
- **Changing properties at run time.** By default a property can be set
  only while the node is stopped. One marked `rtt3::tunable` can be
  changed while running; the new value is applied by the node's own thread
  between two cycles, so user code needs no lock or atomic. A handler
  annotated `[[=rtt3::handles(&Drive::max_velocity)]]` is called after a
  change, for values that need derived state recomputed.
- **Units, ranges and documentation** are part of the exported graph, so
  an inspection tool can build an editor for a node's properties without
  per-component code.
- **Calls within one thread** are plain function calls; the same source
  line becomes a queued request only when the deployment places the two
  nodes in different threads.
- **Decoupled callers.** `rtt3::uses<Drive>` needs the provider's header.
  Where that coupling is unwanted, the operations are declared in an
  interface struct that both sides include, and any provider that has
  matching annotated functions can be linked to it (checked by reflection
  on names and signatures).

Two limits of the language as it stands:

- C++26 reflection cannot generate member functions, so a caller cannot be
  given a synthesised `drive.reset(axis)`. The call names the function
  through its member pointer instead, which is what keeps it checked.
- Whether annotations can be placed on individual function parameters (for
  per-argument documentation) on GCC 16 is to be confirmed in the stage S1
  spike. The fallback is a `rtt3::arg("axis", "Axis index")` annotation on
  the function.
