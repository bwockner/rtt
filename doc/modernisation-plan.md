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

## 5. Phasing

Each phase ends with tests green on x86-64 and AArch64.

| Phase | Content | Exit criterion |
|---|---|---|
| 0. Baseline | Fork, GCC 16 toolchain image (native and AArch64 cross), reflection conformance tests and a spike of `rtt3/meta/` on the real compiler, new CMake and CI beside the old tree, benchmark harness, RTT 2.x numbers recorded on desktop and ARM board | Reproducible baseline; every reflection feature in 4.7 confirmed working or the plan revised; empty `rtt3` libraries build and install next to RTT 2.x |
| 1. OS layer | `rtt3::os`, memory locking, graceful downgrade, logger front end, simulated clock | Thread, timing and lock tests pass; wake-up latency measured on the ARM board |
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
