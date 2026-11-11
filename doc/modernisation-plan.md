# rtt3: plan for a C++26 successor to Orocos RTT

Status: proposal for a new fork. Scope is the C++ API only. The CORBA
transport and the Python and Lua bindings are not carried forward.

## 1. Decisions taken

| Topic | Decision |
|---|---|
| Language and compiler | GCC 16 with `-std=c++26 -freflection` is the one supported configuration. Reflection is required: it is how components, data types and deployments are described. There is no C++23 mode and no hand-written description path. |
| API | A new, clean API. The classic RTT API (`TaskContext`, `InputPort<T>`, `OperationCaller`, ...) is not provided, not even as a wrapper. Existing components are ported by hand with a migration guide. |
| Namespace and install | Everything lives in namespace `rtt3` with its own header directory, library names and CMake package, so it installs next to RTT 2.x and can be loaded into the same process. |
| Introspection | The typed core carries no names and no runtime type registry. Names, by-name lookup, configuration files, the inspection service and runtime loading are a separate library on top. It is optional for the core but is part of every deployment that replaces today's deployer use (4.6). |
| Deployment | Static first, with install-time variation as a first-class requirement: sensor and actuator stacks, kinematic variants and hardware simulation are selected per machine from a configuration file, without rebuilding (4.5). Runtime loading of component libraries remains for stacks not known at build time. |
| Macros | Replaced by templates wherever the language allows. The few that remain are listed in 4.8. |
| OS targets | Linux only, standard and PREEMPT_RT kernels, on x86-64 (desktop testing) and ARM (deployment). |
| Errors | The API reports failure through `std::expected` and never throws. Exceptions are tolerated only at the boundary to user code (4.2). |

## 2. Where the code is today

Measured on this tree, excluding `rtt/transports/corba`:

- About 105k lines under `rtt/`. `scripting` (111 files), `internal` (100),
  `os` (117), `types` (59) and `marsh` (33) are the bulk.
- C++98 with Boost standing in for later standard features:
  `boost::function` 495 uses, `boost::bind` 205, `boost::shared_ptr` 140,
  `boost::intrusive_ptr` 55, `boost::serialization` 56, `boost::mpl` 44,
  `boost::fusion` 23, plus `function_types`, `lambda`, Spirit Classic and
  Graph (both in scripting).
- 742 `const std::string&` parameters. Ports, operations, properties,
  services and peers are found by name in `std::map`/`std::vector`
  containers and recovered with `dynamic_cast` (81 uses), for example
  `DataFlowInterface::getPortType<T>(name)` and `ConnFactory::createConnection`.
- Every `PortInterface` stores three `std::string`s and is polymorphic over
  about 20 virtuals.
- `rtt/os` carries gnulinux, Xenomai 2, RTAI/LXRT, eCos, macOS and Win32
  targets, hand-written atomics per architecture and a vendored TLSF.
- Components register through `ORO_CREATE_COMPONENT` macros into a global
  `std::map<std::string, factory>` behind unversioned `extern "C"` symbols.
- Roughly 375 `#define`s besides include guards.
- Build: vendored `FindBoost.cmake`, catkin and rosbuild support, a
  `configure` wrapper, `.travis.yml`, optional Xerces with a vendored
  TinyXML fallback, 96 test files on Boost.Test.

## 3. Approach and target architecture

### Build new, do not convert

With the classic API gone and C++26 with reflection as the floor, very little of the
existing code survives in recognisable form: the DataSource and typekit
machinery, the `fusion`-based operation plumbing, the string-keyed
interfaces and the `fosi` layer are all replaced, not modernised. A
mechanical pass that swaps Boost for `std` in code that is about to be
deleted would be wasted work.

The plan is therefore to write `rtt3` as a new source tree in this
repository, port the parts that hold real value, and delete the old `rtt/`
tree at the end. The parts worth porting, as algorithms and behaviour
rather than as code:

- the lock-free data object, buffers, queues and memory pool;
- activity semantics (periodic, triggered, slave, file-descriptor, simulation);
- the component lifecycle state machine and its error states;
- connection policies (data, buffer, circular; pull and push; shared);
- operation call semantics (own thread and client thread, send and collect);
- the behavioural expectations encoded in the existing tests.

The old tree stays buildable until removal, as the benchmark baseline and
as a reference for behaviour.

### Libraries

```
rtt3-os        Linux threads, clocks, mutexes, memory (x86-64 and ARM)
rtt3-core      typed ports, channels, lock-free primitives, activities,
               execution engine, lifecycle, static deployment.
               No names, no runtime type registry.
rtt3-reflect   names, by-name lookup, type descriptions, property files,
               runtime component loading, introspection for tooling
```

A statically deployed real-time system links `rtt3-os` and `rtt3-core`
only. No library depends on Boost.

### Naming and concurrent installation

| Item | RTT 2.x | rtt3 |
|---|---|---|
| Namespace | `RTT` | `rtt3`, with an inline ABI namespace (`rtt3::v0`) |
| Headers | `<rtt/...>` | `<rtt3/...>` |
| Libraries | `liborocos-rtt-<target>` | `librtt3-os`, `librtt3-core`, `librtt3-reflect` |
| CMake package and targets | `orocos-rtt` | `rtt3`, targets `rtt3::core` and so on |
| pkg-config | `orocos-rtt-<target>.pc` | `rtt3-*.pc` |
| Component search path | `RTT_COMPONENT_PATH` | `RTT3_COMPONENT_PATH` |
| Plugin entry points | unversioned `extern "C"` symbols | none (4.8) |

Because every symbol is mangled under `rtt3`, both versions can be loaded
into one process, so a system can be migrated one component at a time, with
a small hand-written bridge component where data has to cross.

### What is not carried forward

- CORBA transport, Python and Lua bindings (out of scope by request).
- The classic API and with it the `DataSource`, `TypeInfo`/typekit and
  `OperationInterfacePart` machinery.
- All non-Linux OS targets and assembler atomics.
- catkin, rosbuild, the `configure` script, Xerces.
- **Proposed, to be confirmed (open question 1):** the scripting engine
  (program scripts and state machines). It is 111 files written entirely
  against the DataSource machinery and Spirit Classic, so keeping it means
  writing a new interpreter, not porting one.

## 4. Workstreams

### 4.1 Build and toolchain

- CMake 3.28 or later, target-based, exported targets, `CMakePresets.json`
  including an `aarch64-linux-gnu` cross preset.
- Compiler: GCC 16 with libstdc++ 16, on both x86-64 and AArch64. No older
  GCC is supported. GCC 16 defaults to `-std=gnu++20`, so the build always
  sets the standard explicitly.
- One build mode: `-std=c++26 -freflection`, extensions off. The flags are
  exported as usage requirements of the `rtt3::core` CMake target, so
  component projects get them by linking it.
- The C++23 library and the C++26 additions in libstdc++ 16 can be used
  throughout, including in the libraries themselves: `std::expected`,
  `std::move_only_function`, `std::print`, `std::flat_map`, deducing
  `this`, pack indexing, expansion statements, and (once confirmed on the
  real toolchain) `std::inplace_vector` for fixed-capacity storage.
- The exact GCC 16 point release is pinned in the toolchain image, and
  moving to a newer one is a reviewed change gated by the reflection
  conformance tests (4.7).
- **ARM toolchain.** Distribution and vendor cross compilers will lag
  behind GCC 16, so the project provides its own: a container image with an
  `aarch64-linux-gnu` GCC 16 cross compiler and a sysroot taken from the
  target. Deployments link libstdc++ statically, so the board needs no new
  runtime library; only the sysroot's glibc version has to match the target.
- **Clang** cannot build `rtt3` at all until it implements reflection. No
  CI job depends on it, and clangd and clang-tidy will not parse `rtt3`
  headers or component code in the meantime.
- CI on GitHub Actions, Linux only: GCC 16 on x86-64 and AArch64, with
  ASan, UBSan and TSan jobs.
- New test suite in a current framework (open question 9). The existing
  Boost.Test files are not ported, since they exercise an API that no
  longer exists; they are read as a specification when writing new tests.
- A benchmark harness from the start: wake-up latency, port write and read
  latency, operation call cost, start-up time, binary size, compile time.
  RTT 2.x numbers are recorded first for comparison.

### 4.2 Error handling

Rules:

- Every fallible `rtt3` function returns `std::expected<T, rtt3::error>`
  and is `noexcept` and `[[nodiscard]]`. `rtt3::error` is a small trivially
  copyable value (an error code plus a pointer to static context), so
  returning it never allocates.
- Outcomes that are part of normal operation are not errors. Reading a port
  returns a flow status (`no_data`, `old_data`, `new_data`); a full buffer
  on write returns a status according to the connection policy.
- Constructors do not fail. Components and library objects are cheap to
  construct; anything that can fail happens in `configure()` or in factory
  functions that return `expected`.
- No `rtt3` function throws, at configuration time or on the update path.

**Why not allow exceptions at configuration time as well?** With
`std::expected` available there is little left in their favour:

| Argument for exceptions | Assessment |
|---|---|
| Constructors cannot return a value | Solved by the configure step RTT already has, and by factory functions. |
| Less boilerplate when propagating errors through several layers | True, C++ has no propagation operator. The monadic operations (`and_then`, `transform`, `or_else`) cover most cases, and configuration call chains here are shallow. |
| Errors cannot be silently ignored | `[[nodiscard]]` with `-Werror=unused-result` gives the same guarantee at compile time. |
| The standard library throws anyway | True, and unavoidable; see below. It is a reason to tolerate exceptions, not to add more. |
| Familiar to users | Weak; a mixed model (throwing at configuration, status on the update path) is harder to teach than one rule. |

Against them: a throw has unbounded and allocation-dependent cost, so it
must never happen on a real-time path anyway; a second error channel makes
every function's contract two-part; and `expected` puts the failure modes
in the signature where both the reader and the compiler can see them.

**Where exceptions are still tolerated.** The libraries are compiled with
exceptions enabled, because two sources remain that `rtt3` does not control:

- The standard library (`std::bad_alloc`, `std::format` errors, container
  operations) during configuration. `rtt3` code is written so that these
  cannot occur for reasons other than memory exhaustion, which is treated
  as fatal.
- User code inside component hooks. By default the `rtt3::component`
  concept requires hooks to be `noexcept`, which makes the contract
  checkable at compile time; a throw then terminates the process
  deterministically. A deployment can instead opt a node into
  `rtt3::catch_exceptions`, which wraps its hooks in one `try`/`catch` and
  moves the component to its fault state, as RTT 2.x does today. This is
  for components that wrap third-party code that throws.

Reflection queries may signal errors with exceptions
during constant evaluation; that happens in the compiler and has no
run-time effect.

### 4.3 OS layer: Linux on x86-64 and ARM

The use case is one OS on two architectures: desktop x86-64 for testing,
usually on a standard kernel without real-time privileges, and ARM running
PREEMPT_RT for deployment.

- `rtt3::os` is a small set of classes written directly against POSIX and
  Linux calls. It is an ordinary internal interface, not a trait or
  template parameter. Keeping OS calls in this one library is the only
  provision for other systems.
- Threads: `pthread` with `SCHED_FIFO`/`SCHED_RR`/`SCHED_OTHER`, optional
  `SCHED_DEADLINE`, CPU affinity, thread names, explicit stack size with
  prefaulting.
- Timing: `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME)` for periodic
  activities; `<chrono>` types in the API.
- Locks: mutexes created with `PTHREAD_PRIO_INHERIT`; condition variables
  on `CLOCK_MONOTONIC`. `std::mutex` is not used on real-time paths, as it
  gives no priority inheritance guarantee.
- Memory: `mlockall`, heap prefault and trim control in one start-up call.
  A TLSF-backed `std::pmr::memory_resource` is available for bounded-time
  allocation; `std::pmr` containers replace `rt_string` and `oro_allocator`.
- **Same binary on both kinds of machine.** When real-time scheduling or
  memory locking is refused (no `CAP_SYS_NICE`, as on a typical desktop or
  CI container), the request is downgraded and reported once. A strict mode
  for deployment turns the downgrade into an error.
- Start-up report: kernel preemption model, effective policy and priority
  per thread, locked memory, CPU isolation.
- A simulated clock and simulation activity for deterministic tests.
- Logging: a lock-free, non-formatting front end usable from real-time
  threads, with `std::format`-based sinks on a non-real-time thread.

**ARM specifics.**

- Supported architectures: x86-64 and AArch64. 32-bit ARMv7 is open
  question 5, because it changes the atomics design.
- x86 has a strong memory model that hides most memory-ordering mistakes;
  ARM does not. Lock-free tests therefore run on ARM on every change.
- Every atomic on a real-time path is checked with
  `static_assert(std::atomic<T>::is_always_lock_free)`. No algorithm
  depends on double-width compare-and-swap; ABA protection packs an index
  and a generation counter in one 64-bit word.
- Cache-line padding uses a fixed project constant per architecture, not
  `std::hardware_destructive_interference_size`, whose value can vary with
  compiler flags and so must not appear in installed headers.
- Static linking plus LTO is the default for static deployments, so the
  deployed artefact is one executable.
- CI: AArch64 on ARM runners where available, `qemu-user` otherwise
  (functional tests only; emulation says nothing about timing or ordering).
- Latency benchmarks run on the desktop and on a reference PREEMPT_RT ARM
  board and are tracked separately.

### 4.4 Components and typed ports

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
  and operations are read from the struct by reflection (4.7), and
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

#### Data types: composition, inheritance and link adaptors

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
reflection and a hash of it. The hash is part of the graph hash (4.6), so
a simulator or inspection tool built against an older layout of a type is
detected rather than decoding garbage. Renaming or reordering members
changes the hash; this is deliberate.

### 4.5 Compile-time deployment

A deployment is a struct. Its data members are the nodes, annotations give
each node its activity, and a constant member lists the links:

```cpp
struct App {
    [[=rtt3::periodic(1ms), =rtt3::priority(80)]] Sensor     sensor;
    [[=rtt3::periodic(1ms), =rtt3::priority(79)]] Controller ctrl;
    [[=rtt3::triggered_by(&Driver::command_in)]]  Driver     left, right;

    static constexpr auto links = rtt3::links(
        rtt3::link(&App::sensor, &Sensor::state,       &App::ctrl,  &Controller::state),
        rtt3::link(&App::ctrl,   &Controller::command, &App::left,  &Driver::command_in, rtt3::policy::buffer<8>{}),
        rtt3::link(&App::ctrl,   &Controller::command, &App::right, &Driver::command_in, rtt3::policy::buffer<8>{}));
};

int main() { return rtt3::run<App>(); }
```

Instances are identified by their data member, so several nodes of one
component type need no tags. What the compiler does with it:

- **Validation**: value types match, every required input is connected, no
  input has two writers unless its policy allows it, triggering ports
  belong to their node. Errors name the member, for example
  "input `App::right.command_in` has no writer".
- **Storage**: components, channels and buffers have sizes known at
  compile time. Nothing is allocated at start-up and the deployment can sit
  in static storage.
- **Scheduling**: nodes that share a period and priority can be folded into
  one thread with a fixed execution order, computed from the link graph
  (topological order; cycles are errors unless broken with a delayed link).
- **Channel selection**: where the graph proves one writer and one reader,
  a single-producer single-consumer channel is used.
- **Start-up**: the configure, start and stop order is a constant sequence.
  `rtt3::run` returns the first `expected` error, with the node it came from.
- **Dead code**: no factories, name tables or type descriptions are linked
  unless `rtt3-reflect` is used.

#### Install-time variation: slots

The current system uses runtime components and the deployer to choose,
per installed machine, which sensor and actuator stacks, which kinematic
variant and whether simulated hardware are used. A purely fixed deployment
struct cannot express that, so the deployment gets a variation mechanism
that keeps compile-time checking:

```cpp
struct App {
    [[=rtt3::periodic(1ms), =rtt3::priority(79)]] Controller ctrl;

    [[=rtt3::triggered_by<&rtt3::slot_port::command_in>]]
    rtt3::slot<EthercatDrive, CanDrive> drive;                  // exactly one is active

    rtt3::slot<ScaraKinematics, GantryKinematics> kinematics;

    rtt3::optional_slot<ForceSensorA, ForceSensorB> force;     // may be absent

    static constexpr auto links = rtt3::links(
        rtt3::link(&App::ctrl, &Controller::command, &App::drive, rtt3::port<"command_in">),
        ...);
};

int main(int argc, char** argv) { return rtt3::run<App>(rtt3::machine_file(argv[1])); }
```

- A slot lists the alternatives that may occupy one position in the
  system. Which one is active is read from the machine configuration file
  at start-up and does not change afterwards.
- The compiler checks every alternative against every link that touches
  the slot, so each selectable combination is validated at build time. A
  machine file can select a combination but cannot create an invalid one.
  Links to a slot name the port by identifier; every alternative must have
  a port of that name and value type, which reflection verifies.
- Storage is sized for the largest alternative, so there is still no
  allocation. The cost is one indirect call per slot per cycle, instead of
  an inlined call; links between fixed nodes are unaffected.
- Hardware simulation is selected by the same file, at the level of
  connections, as the deployer extension does today: the real source or
  sink of a port is disconnected and a UDP-backed port is connected in its
  place. In `rtt3` this is *link substitution*:
  - a link, or all links of a node, is declared substitutable in the
    deployment (`rtt3::substitutable`), so the set of places where
    simulation may cut in is fixed and visible at compile time;
  - the machine file may then replace either end of such a link with a UDP
    endpoint (4.6). The component on the kept side is unchanged and
    unaware of it;
  - a node whose links have all been substituted can be marked disabled in
    the machine file, so a hardware driver is neither configured nor
    started when its hardware is absent;
  - substitution is resolved once at start-up. A substitutable link costs
    one indirection per port access; links not declared substitutable keep
    the fully static path and cannot be redirected by any file or tool.

  No mock components are needed, generated or hand-written.
- Kinematic variation that is only a difference in parameters is a
  property set in the machine file, not a slot; a slot is for variants
  with different code.
- The machine file also carries property values and, per slot, activity
  overrides (period, priority, CPU). `app --check machine.toml` validates
  a file against the binary and prints the resolved deployment without
  touching hardware, so an installation can be verified before first start.

The approach assumes the set of stacks is known when the software is
built, even though the choice among them is made at install time. Where
that does not hold, a slot can include `rtt3::loaded<Interface>`, an
alternative supplied by a component library loaded at start-up (4.6 and
4.8) and checked against the slot's port interface at load time instead of
compile time.

Optional `rtt3::autowire<App>` connects outputs to inputs whose name and
value type both match, with ambiguity a compile error. It is off by
default; explicit links are the norm.

`rtt3::reflect::expose(app)` makes a static deployment visible to runtime
tooling and lets dynamically loaded components attach to it.

### 4.6 Runtime layer (`rtt3-reflect`): the deployer's replacement

Today the deployer does three jobs: it assembles the system for a given
machine, it applies configuration, and it is the access point for
inspection tools. Assembly moves into the deployment struct and its slots
(4.5). The other two jobs, and loading of components not known at build
time, are this library. It is a new design, not a port of the classic
interfaces.

- **Machine configuration files**: slot selection, property values and
  activity overrides, with validation and clear errors that name the
  offending entry. Format is open question 10.
- **Name tables** generated by reflection: look up a node, port,
  property or operation by name and obtain a type-erased handle.
- **Deployment graph as data.** The simulator and the inspection programs
  both work by mapping the deployment graph into their own graphs, with
  ports as nodes. The graph is therefore a defined, versioned data model,
  not something tools reconstruct through individual queries:
  - components and slots (with all alternatives and the active one);
  - ports, each with a stable identifier, direction, value type and the
    annotations from its declaration (documentation, unit, event);
  - connections with their policy;
  - activities (period, priority, thread grouping) and lifecycle state;
  - the schema of every value type that appears on a port.

  The whole model is known at compile time, so it is available in two
  ways: `app --graph` writes it as a file without starting anything (all
  alternatives, or resolved for a given machine file), and the running
  process serves the resolved graph. Tools can build their graphs offline
  from the file and attach later. A content hash identifies the graph, so a
  tool can tell whether a running system matches the description it loaded.
  Port identifiers are derived from the member path (`ctrl.command`) and
  stay stable across builds unless the member is renamed.
- **Port access for external programs.**
  - *Observe*: receive samples written to an output port, through an
    optional observer on any exposed output port, checked with one atomic
    load per write and copying into a lock-free buffer. Read-only; for
    inspection.
  - *Substitute*: stand in for the source or sink of a substitutable link
    (4.5), as selected by the machine file at start-up. For simulation.
- **UDP port bridge.** Substituted link ends are served over UDP, as
  today, so the simulator can run on the device or on another host:
  - each substituted port maps to a UDP endpoint; the mapping comes from
    the machine file, with a deterministic default derived from the graph,
    and is included in the exported graph so the simulator can read it
    instead of having it configured twice;
  - socket I/O runs on a bridge thread outside the real-time threads and
    hands samples over through lock-free buffers of the link's declared
    policy, so control threads make no system calls for simulation traffic
    and the kept component sees the same buffering behaviour as with the
    real peer;
  - datagrams carry a sequence number and the graph hash, so loss,
    reordering and a simulator built against a different deployment are
    detected instead of silently producing wrong data;
  - the encoding is generated from the reflected value-type schema. Whether
    it must match what the current deployer extension puts on the wire, so
    that the existing simulator works unchanged, is open question 11;
  - value types larger than one datagram need a stated rule (reject at
    compile time, or fragment); the default is to reject.

  The exported graph marks which links are substitutable, and the served
  graph shows which are currently substituted and which nodes are
  disabled, so the simulator and inspection programs see the system as it
  actually runs. A machine file that substitutes any link makes the
  process report at start-up that it is running against simulated I/O, and
  strict deployment mode (4.3) refuses to start in that case, so a
  simulation configuration cannot end up driving a real machine unnoticed.
  The same bridge can carry observed ports to the inspection programs.
  Deployments that do not call `rtt3::reflect::expose` contain none of this.
- **Control interface** on a local or TCP socket: lifecycle states, reading and
  writing properties, calling operations, and, where time is simulated,
  stepping the simulated clock so that the simulator and the controller
  advance in lock step.
- **Data-type descriptions** (member names and types, enum value names),
  generated by reflection, used by configuration files, taps and logging.
- **Runtime component loading** from shared libraries (4.8), for
  `rtt3::loaded` slot alternatives and for attaching extra components,
  such as loggers or test stimuli, to a running system. Connections made
  this way are type-erased and checked by comparing compile-time type
  identifiers; this is the one place a runtime type check remains.
- **A client library and a command-line client** as the reference users
  of the graph model and port access. Adapting the existing simulator and
  inspection programs to them is outside this plan, but the graph format
  and protocol are specified for that purpose, and that specification is
  written early (phase 4) so the tools can be adapted in parallel.

A migration aid converts existing deployer XML and `.cpf` property files
into machine files where the mapping is mechanical; deployer scripts
(`.ops`) are not converted (open question 1).

### 4.7 Reflection (GCC 16)

Reflection is the description mechanism of `rtt3`. There is no other.

**What GCC 16 provides.** Reflection is experimental and off by default; it
needs `-std=c++26 -freflection` and the `<meta>` header. Status below is
from the GCC 16 release notes and the cppreference support table; it has
not been compile-tested here, as this plan was written on a machine with
GCC 13 only. Confirming it on the real toolchain is the first task of
phase 0, because the whole design now rests on it.

| Paper | Feature | Use here |
|---|---|---|
| P2996R13 | `^^` operator, splicers, `std::meta::info`, `<meta>` queries | Everything below |
| P3394R4 | Annotations, `[[=value]]` | Port, operation, property and activity metadata |
| P3096R12 | Function parameter reflection | Operation argument names and types |
| P3293R3 | Splicing a base class subobject | Descriptions of derived data types |
| P3491R3 | `define_static_string`, `define_static_object`, `define_static_array` | Placing generated tables in static storage |
| P3795R2 | Reflection clean-ups | - |
| P1306 | Expansion statements, `template for` (experimental) | Iterating members without recursive templates |

To confirm on a real GCC 16: P3560 (error handling in `std::meta`
functions) and P3068 (constexpr exceptions) were not confirmed by the
sources available.

**How it is used.**

- *Port discovery*: walk `nonstatic_data_members_of(^^T, ctx)` and keep
  members whose type is a specialisation of a port template.
- *Names*: `identifier_of(member)`, placed in read-only storage with
  `define_static_string`.
- *Annotations* carry everything the declaration cannot express:

  ```cpp
  struct Driver {
      [[=rtt3::doc("Joint command"), =rtt3::event]]
      rtt3::input_port<JointCommand> command_in;

      [[=rtt3::name("status")]]                 // override the identifier
      rtt3::output_port<Status> status_out;

      [[=rtt3::property, =rtt3::unit("rad/s")]]
      double max_velocity = 1.0;

      [[=rtt3::operation(rtt3::own_thread), =rtt3::doc("Clear a latched fault")]]
      std::expected<void, rtt3::error> reset(int axis) noexcept;

      [[=rtt3::hidden]]                         // not part of the interface
      rtt3::output_port<Debug> trace;

      void update() noexcept;
  };
  ```

  Operation argument names come from `parameters_of`.
- *Data types*: member names and types, bases (`bases_of`) and enum value
  names (`enumerators_of`) are read from the type alone, so plain structs
  and enums need no typekit and no annotation. A type from another library
  is reflected the same way; if its public members are not the right view
  of it, it is wrapped in a struct that is.
- *Deployments*: nodes and their activities, as in 4.5.
- Only public members are reflected. Private ports and properties are not
  part of a component's interface.

**Containing the dependency.** Requiring an experimental feature with no
fallback is the largest technical risk in the plan. Three measures limit it:

- All uses of `<meta>` are confined to one internal module, `rtt3/meta/`,
  whose only output is a small set of ordinary constexpr tables
  (`component_info`, `type_info`, `deployment_info`). The rest of the
  library, including validation, scheduling and `rtt3-reflect`, consumes
  those tables and never touches a reflection value. A change in the
  feature is then a change in one module.
- A reflection conformance test set exercises exactly the queries that
  module relies on, independent of the rest of `rtt3`. It runs first in CI
  and is the gate for moving to a new GCC release.
- The tables have a documented plain-C++ form. This is not a supported
  user API, but it means that if reflection had to be abandoned, a
  hand-written or generated description could be reintroduced without
  redesigning the layers above.

`<meta>` is costly to include; keeping it in one module also keeps that
cost out of most translation units. Compile time is tracked from phase 0.

**Not planned.** `define_aggregate` (generating new class types) until
there is a concrete need. Contracts, also experimental in GCC 16, are a
separate topic.

### 4.8 Macro policy

A macro is allowed only where the language has no other mechanism, and
never in the API a component author writes against.

| RTT 2.x | rtt3 |
|---|---|
| `ORO_CREATE_COMPONENT`, `ORO_LIST_COMPONENT_TYPE`, `ORO_CREATE_COMPONENT_LIBRARY` | `rtt3::plugin::registrar<T>` (below) |
| `ORO_SERVICE_PLUGIN` family | The same registrar |
| `ORO_main` | Plain `main` calling `rtt3::run<App>()` |
| `OROBLD_*`, `OROPKG_*`, `OROCFG_*` configuration macros | Generated `rtt3/config.hpp` with `inline constexpr` values, used with `if constexpr` |
| `OROCOS_TARGET`, `fosi` function macros, `oro_atomic_*` | Deleted; `rtt3::os` and `std::atomic` |
| Token pasting and stringising helpers | Deleted with their users |
| Preprocessor repetition for signatures up to N arguments | Variadic templates |
| Deprecation and unused-argument macros | Standard attributes |
| Logger file and line macros | `std::source_location` default arguments |

**Registration without macros.** Needed only for runtime loading. A
component library names its exported types with one line per type at
namespace scope:

```cpp
inline const rtt3::plugin::registrar<Sensor> sensor_registration;
```

The registrar adds a typed factory, named from the reflected type, to a
registry owned by `librtt3-reflect`; the loader opens the library and reads
what was added. There are no `extern "C"` entry points. The exact spelling
is settled with a test that registration survives static archives and
`--gc-sections`.

**Macros that remain**, all internal to library headers: include guards or
`#pragma once`; one symbol-visibility macro from a generated export header;
the test framework's assertion macros in tests. CI fails on any new
`#define` outside an allow-list.

### 4.9 Lock-free and real-time internals

- New implementations of the data object, multi-writer buffer, queues and
  memory pool on `std::atomic`, with documented memory orders, ABA handling
  and cache-line alignment, plus single-producer single-consumer variants.
- Verification: TSan in CI, stress tests on both x86-64 and ARM hardware,
  and model checking of the algorithms (for example with Relacy).
- Execution engine: a fixed-capacity queue of non-allocating callables.
- Rule for all code reachable from `update()`: no allocation, no blocking
  without priority inheritance, no exceptions, no `std::string` construction.

#### Cache behaviour

A cache line is invalidated on one core whenever another core writes to
it. In RTT 2.x this happens on almost every port operation: readers
increment a counter inside the data object the writer is filling, every
hop copies a reference-counted pointer, and unrelated fields share lines.
The strategies below are ordered by how much they remove.

**1. Do not share at all.** When the deployment folds two nodes into one
thread (4.5), the link between them is a plain variable with no atomics.
The data is written and read on the same core, in execution order, so it
is still in the first-level cache when the consumer runs. This is the
largest single gain and it follows from the scheduling fold, not from any
tuning.

**2. Readers that do not write.** For links that do cross threads, choose
channel algorithms in which the reading side stores nothing into memory
the writer uses:

| Link shape | Channel | Cross-core writes per operation |
|---|---|---|
| One writer, one reader, latest value | Triple buffer: each side owns a slot, one atomic exchange swaps in the spare | One shared word |
| One writer, many readers, latest value, `trivial_data` | Sequence lock: writer bumps a counter, readers only load and retry | None from readers |
| One writer, one reader, queue | Ring with separate head and tail, each side caching the other's index | One shared word, and only when the cached index runs out |
| Several writers | Multi-writer queue as today | Inherent contention; used only where the graph requires it |

The sequence lock is not wait-free for readers, so it is selected only
where the reader's retry is bounded by the writer's period; the triple
buffer is wait-free on both sides. The deployment picks the channel from
the link graph, so the common shapes never pay for the general one.

**3. No reference counting or shared bookkeeping on the data path.**
Channels in a static deployment live as long as the process, so there are
no smart-pointer copies per operation. Per-node statistics (cycle and
trigger counts) are written only by the owning thread and read by the
inspection service without synchronisation beyond an atomic load.

**4. Layout by owner.** Within a channel, fields written by the producer
and fields written by the consumer are on different cache lines, each
aligned to the project's cache-line constant (4.3). Across the deployment,
the compiler knows which thread owns each node and each channel end, so
the deployment's storage is arranged per thread: everything one thread
writes is contiguous, with a line boundary before the next thread's data.
Components are not interleaved in declaration order as they would be in a
hand-written struct.

**5. Separate read-mostly from write-often.** Flags that are read on every
operation but almost never change (observer enabled, link substituted,
lifecycle state) are grouped on lines that hold no frequently written
data, so checking them never misses because of a neighbour.

**6. Copy less.** Samples are copied once per link, not twice: there is no
"last written value" copy unless a link asks for it. For large samples a
loan interface lets the writer fill a channel slot in place and publish it
(`auto s = port.loan(); ...; s.publish();`), and the reader of a triple
buffer reads in place. Member projection (4.4) sends only the part that is
needed. Draining a queue reads all available samples with one index update.

**7. Keep threads where their data is.** Thread affinity is part of the
machine file. Recommended practice, checked by the start-up report: pin
real-time threads, isolate their cores, and place threads that exchange
data at a high rate on the same core or on cores that share a cache level.
A migrated thread starts with a cold cache, so unpinned real-time threads
are reported as a warning.

**8. Hot and cold code and data.** Lifecycle handling, error paths and
configuration are kept out of line and marked cold, so the update path of
a folded thread is one compact region of code. Names and descriptions are
in read-only storage away from the data path. Deployment storage is
locked, prefaulted and can optionally be backed by huge pages to reduce
address-translation misses.

**9. Inspection and simulation stay off the hot lines.** Port observers
and the UDP bridge copy into their own single-producer rings laid out as
in point 4, and their threads run on a non-isolated core.

**Checking it.** The layout rules are enforced at compile time: library
types annotate which side owns each field, and a reflection-based check
fails the build if fields of different owners share a cache line. At run
time, the benchmark suite records cache-miss counts alongside latency, and
`perf c2c` is used on x86-64 to find remaining shared lines (on ARM the
equivalent needs the statistical profiling extension, so that analysis is
done on the desktop and confirmed by timing on the target).

#### Context switches

In a typical RTT 2.x system every component has its own thread, so one
control cycle through sensor, controller and driver costs three wake-ups.
Each sample written to an event port wakes the receiving thread, and an
own-thread operation call blocks the caller, wakes the callee and wakes
the caller again. The strategies below remove switches in that order of
importance.

**1. Fewer threads.** Nodes are folded into one thread when they share a
period and priority (4.5). Folding is extended to harmonic rates: a node
with a 10 ms period can run every tenth cycle of a 1 ms thread at the
same priority. A whole processing chain then costs one wake-up per period.
The fold is reported at build time and at start-up (which nodes share
which thread), and a node can opt out with an annotation when it must be
preemptible separately.

**2. Triggered nodes run inline.** A node triggered by a port whose writer
is in the same thread runs directly after the writer in the same cycle;
there is no wake-up and no queueing through the scheduler.

**3. Cross-thread triggers are coalesced and deferred.**
- A trigger is one wake-up per writer cycle, however many samples were
  written.
- The writer checks an atomic "already pending" flag first, so a target
  that is running or already signalled costs no system call.
- Wake-ups are issued at the end of the writer's cycle, not in the middle
  of `update()`. Waking a higher-priority thread mid-cycle would preempt
  the writer immediately and cost two extra switches for the same work.

**4. Operation calls that do not switch.**
- A call between nodes in the same thread is a direct function call.
- Client-thread operations are direct calls from any thread; the
  annotation states that the operation is safe for that.
- An own-thread operation called from another thread is asynchronous by
  default: `send` enqueues the request and returns a handle, and the
  result is collected on a later cycle. A blocking cross-thread call from
  a real-time node is a compile-time error unless the call site marks it
  as intended, because it costs two switches and ties the caller's timing
  to the callee's.

**5. Nothing on the update path can block.** Channels are lock-free, so
there are no switches caused by lock contention. Mutexes exist only on
configuration and inspection paths and use priority inheritance. Memory is
locked and prefaulted, so there are no page faults, each of which is a
kernel entry and a possible reschedule.

**6. Real-time threads never wake helper threads.** The log drain, the
port observers and the UDP bridge poll their rings on their own timers at
a modest rate. A real-time thread therefore never makes a wake-up system
call on behalf of logging, inspection or simulation. Where lower latency
is wanted for a helper, it uses the same pending-flag scheme as point 3.

**7. One sleep per thread per period.** Each periodic thread makes a
single absolute-time sleep call per cycle. Clock reads go through the
vDSO and do not enter the kernel. I/O-driven chains (a fieldbus cycle, a
file descriptor) run in the thread that waits for the I/O, so the chain
is not handed to a second thread when data arrives.

**8. System configuration that removes involuntary switches.** These are
outside the library but checked by the start-up report (4.3):
- real-time threads pinned to isolated cores (`isolcpus`, `nohz_full`,
  `rcu_nocbs`), with device interrupts steered elsewhere;
- distinct priorities for real-time threads sharing a core, so they do
  not time-slice against each other;
- interrupt threads for the fieldbus or network device at a priority
  consistent with the control thread that depends on them.

**Checking it.** Each thread's voluntary and involuntary switch counts are
read from the kernel and exposed through the inspection service, with the
expected value (one voluntary switch per period) alongside. A benchmark in
CI runs the example deployment and fails if the number of wake-ups per
cycle rises. `perf sched` and the `sched_switch` tracepoint are the tools
for finding the cause when it does.

#### Memory allocation

Memory is obtained in three stages, each stricter than the one before.

| Stage | When | What is allowed |
|---|---|---|
| Static | Build time | Everything whose size follows from types and link policies: components, channels, buffers, engine queues. It is one object, in static storage or one allocation. |
| Configuration | Between process start and the first cycle | A bounded arena for sizes that come from the machine file: data samples for `dynamic_data` ports, buffer capacity overrides, loaded component libraries. |
| Run | After start | No allocation and no release, by any real-time thread. |

**Static stage.** Buffer capacities are part of the link policy type, so
storage is an array member, not a pointer to the heap. Engine message
queues and their payload pools are sized from the operations each node
declares.

**Configuration stage.**
- One monotonic arena (`std::pmr::monotonic_buffer_resource`) serves all
  configuration-time requests and is never freed. Its upper bound is
  stated in the machine file; exceeding it is a start-up error, so memory
  use is known before the machine runs.
- Ports that carry `dynamic_data` get their per-slot capacity from a data
  sample here. Writing a larger value at run time returns a status; it
  does not grow the slot.
- All memory is then locked (`mlockall` with current and future pages)
  and touched, including thread stacks, which have explicit sizes.

**Run stage.**
- Fixed-size block pools with lock-free free lists serve the few things
  that are created and destroyed per operation, mainly asynchronous
  operation requests. An empty pool returns an error to the caller.
- Bounded types are the default for variable-length data
  (`std::inplace_vector`, fixed-capacity strings). Where variable-size
  allocation in a real-time thread is unavoidable, typically inside
  third-party code, a per-thread TLSF resource gives constant-time
  allocation without a lock. It is opt-in per node.
- Library code on the update path avoids the standard facilities that
  allocate behind the scenes: `std::function` (replaced by fixed-storage
  callables), `std::string` construction, `std::shared_ptr`, stream and
  format output, and exceptions.

**The general-purpose heap** is still used by helper threads. It is
configured at start so that it never returns memory to the kernel or maps
new regions for large blocks (`M_TRIM_THRESHOLD`, `M_MMAP_MAX`), and it is
pre-grown. This matters to the real-time threads even though they do not
allocate: unmapping memory anywhere in the process interrupts every core
running one of its threads. Transparent huge pages are left to explicit
request only, to avoid background compaction stalls.

**Enforcement.**
- At compile time, the port data concepts (4.4) reject allocating types
  on ports that have not opted in.
- In test builds, an allocation guard replaces the global allocation
  functions and aborts with a backtrace if one is called from a thread
  marked real-time after start. The whole test suite and the example
  deployments run with the guard on.
- At run time, the start-up report states static, arena and locked sizes,
  and per-thread page-fault counts are exposed through the inspection
  service with an expected value of zero after start.

#### System calls at run time

The target for a real-time thread is **one system call per cycle**: the
call that puts it to sleep until its next activation. Everything else is
either done in user space or moved to a helper thread.

| Need | How it is met without a system call per cycle |
|---|---|
| Sleep until next period | `clock_nanosleep` with an absolute time; this is the one call |
| Read the clock | vDSO `clock_gettime`. The start-up report verifies that the kernel's clock source allows it; on a fallback clock source every read becomes a real system call |
| Wake another thread | Pending-flag check first, coalesced and deferred to the end of the cycle; a system call only when the target is actually asleep |
| Locks | None on the update path; elsewhere uncontended locks stay in user space |
| Memory | No `mmap`, `brk`, `munmap` or `madvise` after start (see above), and no page faults |
| Logging | Lock-free ring, drained and written by a helper thread |
| Inspection and simulation | Rings to helper threads; the helper batches datagrams (`recvmmsg`, `sendmmsg`) so that socket traffic costs a few calls per cycle, not one per port |
| Scheduling attributes, affinity, thread names | Set once at start, never per cycle |
| Timers and signals | Not used; no POSIX timers, no signal handlers, no timer file descriptors that need a read per expiry |
| Waiting on file descriptors | One `epoll_wait` per activation for all descriptors of that thread, then drain until empty |

Device I/O inside a driver component is the driver's own business, but
the same rule is recommended and documented: batch per cycle, and prefer
memory-mapped or polled interfaces to a call per item.

Kernel entries that are not system calls are reduced by the same means as
in the two sections above: no page faults, interrupts steered off the
real-time cores, the periodic tick disabled on isolated cores, and no
unmapping elsewhere in the process.

**Enforcement.**
- A strict test mode installs a per-thread system call filter (seccomp)
  on each real-time thread after start, allowing only the calls in the
  table. Any other call raises a signal that the test reports with a
  backtrace. This turns "no unexpected system calls" from a guideline
  into a failing test.
- The benchmark suite records system calls per cycle per thread and fails
  when the count rises.

#### Priority inversion through file I/O

Priority inheritance bounds the wait for a lock only if the holder can
make progress when boosted. A thread that is waiting for a disk, a full
pipe or a slow console cannot, so any lock it holds meanwhile blocks a
real-time waiter for as long as the device takes. File I/O reaches
real-time threads by four routes, and each is closed separately.

**1. Direct I/O from a real-time thread.** A `write` or `read` on a file
can block inside the kernel on journal commits, writeback throttling or a
lock on the file held by another thread.

- Real-time threads perform no file operations after start: no `open`,
  `close`, `read`, `write`, `fsync` or `stat`, and nothing on standard
  output or standard error. The only descriptors they may touch are device
  descriptors a driver node has registered.
- This is enforced by the per-thread system call filter in strict test
  mode (see above).

**2. A lock shared with a thread that does I/O.** This is the common case
in practice: a logger or recorder holds a mutex while writing, and a
real-time thread waits for that mutex.

- Design invariant: **a real-time thread shares no lock with any thread
  that performs I/O.** All hand-over between the two goes through bounded
  lock-free rings: log records, observed port samples, property snapshots
  to be saved, recorded data.
- A full ring never blocks the real-time side. The record is dropped and a
  per-ring drop counter is incremented; the counters are visible through
  the inspection service, so loss is reported, not hidden.
- The library has two mutex types, one for configuration and inspection
  code and none usable on the update path. In debug builds a helper thread
  entering an I/O call while holding any library mutex is reported, and so
  is any lock taken by a thread marked real-time.
- Standard I/O streams have their own internal locks, so they fall under
  the same rule: only helper threads use them.

**3. I/O hidden behind memory access.** A page fault on memory backed by a
file is a disk read, taken while holding the process's memory-map lock,
which every other faulting or mapping thread also needs.

- All memory, including program text and shared libraries, is locked and
  resident before the first cycle.
- Executables are linked with immediate binding (`-z now`) so that no
  symbol is resolved lazily on first call, which would take the dynamic
  loader's lock and may touch disk. Static linking, the default for
  deployments, removes most of this.
- No library is loaded after start. Component libraries for loaded slot
  alternatives are opened during configuration, and their thread-local
  storage is touched by each thread at thread start.
- Helper threads do not map or unmap memory after start either, so they
  never hold the memory-map lock for long.
- Shared memory, if used, is backed by memory and prefaulted, never by a
  file on disk.

**4. I/O that is the component's job.** Drivers must talk to devices.

- A driver that performs blocking device I/O runs as its own node with its
  own thread at a priority chosen for that device, and exchanges data with
  control nodes through links. The control thread then shares neither a
  lock nor a call path with the device.
- The start-up report lists, for each registered device descriptor, the
  thread that uses it, so a control thread that ends up doing device I/O
  is visible.

**Services that replace I/O in components.** Because components may not
write files themselves, the runtime layer provides the usual needs as
helper-thread services fed by rings: logging, recording of observed ports
to disk, and saving property snapshots. Configuration and property files
are read before start. Saving properties at run time takes a consistent
snapshot on the real-time side (a copy into a ring) and writes it from the
helper.

**Helper thread hygiene.** Helpers run at normal priority on non-isolated
cores. Console output is written without blocking and dropped when the
console cannot keep up, since a serial console can stall a writer for
milliseconds. Log files are best placed on a device that the control
process does not otherwise depend on.

#### Logging from real-time threads

**What exists today.** `RTT::Logger` formats under a global mutex and
writes while holding it (see `optimisation-plan.md`, D3 and D6). OCL adds a
separate real-time logging facility (`OCL::logging`): a category formats
the message in the calling thread into a real-time string, sends it as an
event through a buffered port, and appender components on non-real-time
threads write it out. That keeps I/O out of the real-time thread, but it
is not lock-free and not allocation-free: the string is allocated from the
TLSF pool on every message, and that pool is guarded by one mutex
(`rtt/os/tlsf/tlsf.c:65`, `rtt/os/tlsf/target.h:44-45`), created without
priority inheritance. Formatting cost is also paid in the real-time
thread. The description of OCL here is from knowledge of that project, not
from reading its source in this session, and should be checked against the
version in use.

The full work plan for the logger, with record layout, sinks, crash
recovery, tests and steps, is in `rtt3-logging-plan.md`. In summary:

**Design for `rtt3`: record now, format later.**

```cpp
rtt3::log::warn("axis {} over limit: {:.3f} rad", axis, position);
```

In the calling thread this does four things and nothing else:

1. Loads the category's level (one relaxed atomic load) and returns if the
   message is filtered.
2. Reserves space in **this thread's own** ring.
3. Writes a fixed header (pointer to the call site's static description,
   timestamp) followed by the raw bytes of the arguments.
4. Publishes the record with one atomic store.

There is no formatting, no allocation, no lock and no system call. The
text is produced later by a drain thread.

- **One ring per thread.** Each real-time thread is the single producer of
  its ring and the drain thread the single consumer, so the producer is
  wait-free and never contends with another logging thread. Rings are
  sized at configuration and laid out as in "Cache behaviour". The drain
  thread merges records from all rings by timestamp.
- **Call-site description.** The format string, level, source location and
  the argument types are a constant in read-only storage, one per call
  site; the record carries only its address. The format string is checked
  against the argument types at compile time, as `std::format` does.
- **Arguments** must satisfy a `loggable` concept: trivially copyable
  values, string literals (stored as a pointer), and bounded strings
  (copied up to a stated maximum). Port data types are loggable as values:
  the sample is copied in binary and rendered on the drain thread by a
  formatter generated through reflection, so logging a whole state struct
  costs one small copy.
- **Timestamps** come from the monotonic clock through the vDSO, or from
  the cycle start time the node already has, when per-message resolution
  is not needed.
- **Overflow.** A full ring drops the new record and increments a counter;
  the drain thread later emits a "n messages dropped" line in that
  thread's sequence. The real-time thread never waits. A flight-recorder
  mode, selectable per ring, overwrites the oldest records instead and is
  only written out on request or on a fault.
- **Repetition control** for messages inside control loops:
  `log::every(1s, ...)` and `log::once(...)`, with per-site state owned by
  the calling thread.
- **Levels and categories.** Each node has a category named after it;
  levels are atomics that the inspection service can change while running.
- **Drain thread.** Non-real-time, on a non-isolated core, polling on its
  own timer so that no real-time thread ever wakes it. It formats with
  `std::format` and writes to the sinks: console (non-blocking), file, and
  the inspection service. Because each record has typed fields, a
  structured sink (one JSON object per line) costs nothing extra.
- **Crashes.** Records still in the rings are lost if the process dies
  before the drain thread runs. To limit that, the rings live in one
  memory region that a fault handler flushes on a best-effort basis, and
  that can optionally be a shared mapping so a separate process can
  recover the last records after a crash.
- **Non-real-time threads** use the same interface with their own rings,
  so there is one logging path, not two.

**The cost of having no macros.** A logging function's arguments are
evaluated before the level check inside it, whereas a macro can skip them.
For cheap values this does not matter. For an argument that is costly to
compute, the call takes a callable that runs only if the level passes:
`log::debug([&] { return rtt3::log::args("residual {}", expensive()); })`.

**Checking it.** Logging at an enabled level from a real-time thread runs
under the allocation guard and the system call filter in the test suite,
and a benchmark records the cost per message in both the filtered and the
recorded case.

## 5. Phasing

Each phase ends with tests green on x86-64 and AArch64.

| Phase | Content | Exit criterion |
|---|---|---|
| 0. Baseline | Fork, GCC 16 toolchain image (native and AArch64 cross), reflection conformance tests and a spike of `rtt3/meta/` on the real compiler, new CMake and CI beside the old tree, benchmark harness, RTT 2.x numbers recorded on desktop and ARM board | Reproducible baseline; every reflection feature in 4.7 confirmed working or the plan revised; empty `rtt3` libraries build and install next to RTT 2.x |
| 1. OS layer | `rtt3::os`, memory locking, graceful downgrade, per-thread log rings and drain thread, simulated clock | Thread, timing and lock tests pass; wake-up latency measured on the ARM board |
| 2. Lock-free primitives | Data object, buffers, queues, pool | Stress and model-checking tests pass on both architectures |
| 3. Components and ports | `rtt3/meta/` component tables, `component` concept, ports, channels, policies, `connect`, activities, lifecycle, error type | Hand-wired example runs; port latency no worse than baseline |
| 4. Static deployment | Deployment struct, slots, substitutable links and machine-file selection (minimal reader), graph model specification and `--graph` export, validation, scheduling fold, channel selection, `run` | Example system with a simulated and a real hardware alternative runs from one binary, with zero allocations after start-up and no string handling after configuration |
| 5. Operations and properties | Annotated operations and properties, typed callers, engine messages | Operation call cost no worse than baseline |
| 6. Runtime layer | `rtt3-reflect`: full machine files and `--check`, names, served graph, link substitution with the UDP port bridge, port observation, control interface, client library, reflected data-type descriptions, registrar and loader, `expose`, converter for deployer XML and `.cpf` | A simulator stand-in drives substituted links over UDP with the hardware node disabled, and an inspection client maps the graph and observes ports, both as separate processes; an RTT 2.x and an rtt3 component load into one process |
| 7. Release | Delete the old `rtt/` tree, migration guide with a worked port of a real component, API and ABI policy | Tagged 3.0 |

Phases 1 and 2 do not depend on any open question. Phases 3 to 5 depend
on questions 2 and 3; phase 6 on questions 1, 4 and 10.

## 6. Risks

- **No automatic migration.** Every existing component has to be ported by
  hand. Mitigation: side-by-side loading, a migration guide written against
  a real component, and a mapping table from each classic call to its
  `rtt3` equivalent.
- **No inherited regression net.** The old tests do not compile against the
  new API. Mitigation: treat them as a checklist of behaviours, and port
  the lock-free and activity tests first.
- **Lock-free rewrites** can introduce rare ordering bugs that x86 hides.
  Mitigation: 4.9 verification on ARM.
- **Combinations of slot alternatives** multiply the checks the compiler
  performs. Mitigation: each alternative is checked against the slot's
  links independently, so cost grows with the number of alternatives, not
  with the number of combinations.
- **Compile time and error quality** on large systems. Mitigation:
  benchmark from phase 0; keep checks in constexpr functions over the
  generated tables.
- **Dependence on an experimental compiler feature, with no fallback.**
  If GCC changes or withdraws part of reflection, `rtt3` does not build
  until it is adapted, and no other compiler can stand in. Mitigation: the
  containment measures in 4.7, a pinned toolchain image, and the phase 0
  spike before anything else is built on it.
- **Tooling.** clangd and clang-tidy cannot parse `rtt3` code at all until
  Clang implements reflection, so editor support and static analysis are
  limited to what GCC provides.
- **Ecosystem.** OCL (deployer, taskbrowser, reporting), typegen/orogen and
  rtt_ros integration do not work with `rtt3` and are outside this plan.
- **Licence.** The tree is GPL v2 with a linking exception. Code written
  new for `rtt3` could be licensed differently, but anything derived from
  the existing sources cannot; this needs a decision before work starts.

## 7. Open design questions

1. **Scripting**: confirm that program scripts and state machines are
   dropped. If state machines are needed, a small typed state-machine
   library in C++ is a far smaller job than a new script interpreter.
2. **Lifecycle**: keep the RTT state model (pre-operational, stopped,
   running, run-time error, exception, fatal) and hook set as it is, or
   simplify it?
3. **Operations threading model**: keep both own-thread and client-thread
   execution with `send` and `collect`, or reduce the typed API to one model?
4. **Inter-process transport**: with CORBA gone, is any transport in scope
   (POSIX message queues as today, shared memory, Zenoh, DDS), or is `rtt3`
   single-process with bridges written as ordinary components?
5. **ARM variants**: AArch64 only, or 32-bit ARMv7 as well? Which board and
   kernel should be the reference for latency numbers?
6. **Compiler**: decided, GCC 16. Remaining point: is a project-built
   AArch64 cross toolchain acceptable, or must the build use a vendor SDK?
7. **Links in static deployments**: fixed for the life of the process, or
   changeable at run time (disconnect, reconnect, change policy)? Fixed
   links allow the channel selection and inlining in 4.5.
8. **ABI stability**: is a stable shared-library ABI needed for separately
   built component libraries, or is rebuilding everything acceptable?
9. **Test framework**: Catch2, GoogleTest or doctest?
10. **Property files**: keep the `.cpf` XML format for existing
    configuration files, or move to another format (TOML, YAML, JSON)?
11. **Runtime tooling and variation**: partly answered (deployer-driven
    install-time variation, inspection tools, hardware simulation).
    Remaining points:
    - Is the set of sensor and actuator stacks closed when the software is
      built, or are stacks added in the field without a rebuild? This
      decides whether slots are enough or `rtt3::loaded` is essential.
    - The simulator works through a deployer extension that disconnects a
      port's real source or sink and connects a UDP-backed port instead
      (answered). What happens to the disconnected hardware component
      today: is it not loaded, loaded but not started, or left running?
    - Is it acceptable to declare in the deployment which links may be
      substituted, or must any link be substitutable from the file alone?
      Declaring them keeps all other links fully static.
    - What encoding does the extension use on the wire, and must `rtt3`
      stay compatible with it so the simulator runs unchanged?
    - How are UDP port numbers assigned today: fixed per port in a file,
      or computed?
    - Is simulated time stepped in lock with the controller, or does the
      simulator run in real time against the controller's own clock?
    - Do the inspection programs use the same UDP mechanism, or another
      route (task browser, CORBA)? Do they only read, or also write
      properties and call operations?
    - Is the deployer extension in a repository I can read? Its behaviour
      would be the specification for substitution and the bridge.
    - Roughly how many machine variants exist, and do deployer scripts
      (`.ops`) contain logic beyond loading, connecting and configuring?
12. **Fallback mode**: decided. Reflection is required; there is no
    C++23 mode and no hand-written description path.
13. **Exceptions from user hooks**: is `noexcept` hooks with termination
    the right default, with `catch_exceptions` as opt-in, or the reverse?
14. **Project name and licence**: is the package published as `rtt3`, does
    it keep the Orocos name, and is the existing licence kept?
15. **Portability beyond Linux**: adopt the low-cost guardrails that keep
    an RTOS or microcontroller port possible (core builds with
    `-fno-exceptions -fno-rtti`, no POSIX headers in the core, lock-free
    primitives parameterised on word size), or leave them out?
16. **Data types**: do existing data types use inheritance, and is any of
    it polymorphic? Are typekits generated by typegen or written by hand?
    Is connecting a member of a composite to a port (projection), or
    connecting types from different libraries by structure, something you
    need, or is exact type match enough?
