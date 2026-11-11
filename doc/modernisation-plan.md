# rtt3: plan for a C++23 successor to Orocos RTT

Status: proposal for a new fork. Scope is the C++ API only. The CORBA
transport and the Python and Lua bindings are not carried forward.

## 1. Decisions taken

| Topic | Decision |
|---|---|
| Language and compiler | GCC 16 is the supported compiler and the floor. C++23 is the required language level. C++26 reflection (`-std=c++26 -freflection`) is on in the default build and is the normal way to describe a component; a hand-written description remains as a fallback because the feature is still experimental. |
| API | A new, clean API. The classic RTT API (`TaskContext`, `InputPort<T>`, `OperationCaller`, ...) is not provided, not even as a wrapper. Existing components are ported by hand with a migration guide. |
| Namespace and install | Everything lives in namespace `rtt3` with its own header directory, library names and CMake package, so it installs next to RTT 2.x and can be loaded into the same process. |
| Introspection | The typed core carries no names and no runtime type registry. Names, by-name lookup, property files and runtime loading are an opt-in library on top. |
| Deployment | Static first: a deployment is described in C++ and checked and wired by the compiler. Runtime loading is optional. |
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

With the classic API gone and C++23 as the floor, very little of the
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
- Two build modes, both tested: the default, `-std=c++26 -freflection`, and
  a fallback, `-std=c++23`, in which the reflection headers are not
  included. The built libraries are identical in both.
- With GCC 16 as the floor, the whole C++23 library can be used in
  installed headers without version checks (`std::expected`,
  `std::move_only_function`, `std::print`, `std::generator`,
  `std::flat_map`, deducing `this`). C++26 library additions reported for
  libstdc++ 16, such as `std::inplace_vector`, are candidates for the
  fixed-capacity containers the core needs; each is confirmed on the real
  toolchain before use and only used in the C++26 build mode.
- **ARM toolchain.** Distribution and vendor cross compilers will lag
  behind GCC 16, so the project provides its own: a container image with an
  `aarch64-linux-gnu` GCC 16 cross compiler and a sysroot taken from the
  target. Deployments link libstdc++ statically, so the board needs no new
  runtime library; only the sysroot's glibc version has to match the target.
- **Clang** is not a supported compiler. It cannot build the reflection
  tier, and no CI job depends on it. Clang-based tools (clangd,
  clang-tidy) will not parse headers that use reflection or annotations;
  the `-std=c++23` fallback mode exists partly so that such tools can still
  be run on the core.
- CI on GitHub Actions, Linux only: GCC 16 in both build modes, on x86-64
  and AArch64, with ASan, UBSan and TSan jobs.
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

Reflection queries in the C++26 tier may signal errors with exceptions
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
- Metadata (port list, names, documentation, properties, operations) is
  one constant, found through a single customisation point:

  ```cpp
  template <> inline constexpr auto rtt3::describe<Controller> = rtt3::component_info{
      rtt3::name{"Controller"},
      rtt3::port{&Controller::state,   "state",   rtt3::doc{"Measured joint state"}},
      rtt3::port{&Controller::command, "command"},
      rtt3::property{&Controller::gain, "gain"},
      rtt3::operation{&Controller::reset, "reset", rtt3::own_thread},
  };
  ```

  Reflection (4.7) generates this by default; it is written by hand only
  to override what reflection produces or in the `-std=c++23` fallback mode. Names exist only in this table. A system that never
  asks for a name does not link them; where an identifier is needed at run
  time it is a compile-time hash of the name.
- Operations are member functions listed in the description and called
  through a typed handle (`rtt3::caller<&Controller::reset>`), with
  own-thread and client-thread execution and an asynchronous `send` that
  returns a typed handle. Signatures are handled with variadic templates,
  `std::invoke` and `std::tuple`; engine messages are
  `std::move_only_function` in fixed-capacity storage.
- Properties are ordinary data members listed in the description. Reading
  and writing them from a file is an `rtt3-reflect` feature.

### 4.5 Compile-time deployment

A deployment is a type listing components, activities and connections:

```cpp
using App = rtt3::deployment<
    rtt3::node<Sensor,     rtt3::periodic<1ms, rtt3::priority<80>>>,
    rtt3::node<Controller, rtt3::periodic<1ms, rtt3::priority<79>>>,
    rtt3::node<Driver,     rtt3::triggered<&Driver::command_in>, rtt3::tag<"left">>,
    rtt3::node<Driver,     rtt3::triggered<&Driver::command_in>, rtt3::tag<"right">>,
    rtt3::link<&Sensor::state,       &Controller::state,                    rtt3::policy::data>,
    rtt3::link<&Controller::command, rtt3::at<"left",  &Driver::command_in>, rtt3::policy::buffer<8>>,
    rtt3::link<&Controller::command, rtt3::at<"right", &Driver::command_in>, rtt3::policy::buffer<8>>
>;

int main() { return rtt3::run<App>(); }
```

What the compiler does with it:

- **Validation**: value types match, every required input is connected, no
  input has two writers unless its policy allows it, triggering ports
  belong to their node. Failures are concept or `static_assert` errors with
  constexpr-built messages.
- **Storage**: components, channels and buffers are members of `App` with
  sizes known at compile time. Nothing is allocated at start-up and the
  deployment can sit in static storage.
- **Scheduling**: nodes that share a period and priority can be folded into
  one thread with a fixed execution order, computed from the link graph
  (topological order; cycles are errors unless broken with a delayed link).
- **Channel selection**: where the graph proves one writer and one reader,
  a single-producer single-consumer channel is used.
- **Start-up**: the configure, start and stop order is a constant sequence.
  `rtt3::run` returns the first `expected` error, with the node it came from.
- **Dead code**: no factories, name tables or type descriptions are linked
  unless `rtt3-reflect` is used.

`rtt3::reflect::expose(app)` makes a static deployment visible to runtime
tooling and lets dynamically loaded components attach to it.

### 4.6 Runtime layer (`rtt3-reflect`)

A new design, not a port of the classic interfaces. Scope:

- Name tables built from `rtt3::describe<T>`: look up a node, port,
  property or operation by name and obtain a type-erased handle.
- Type-erased connection of two ports by name, with the type check done by
  comparing compile-time type identifiers. This is the one place a runtime
  type check remains.
- Type descriptions for data types (member names and types, enum value
  names), supplied through `describe<T>` in C++23 and generated in C++26.
  They drive property files, logging of port data and any transport.
- Property files: load and save the described properties of a component.
  Format is open question 10.
- Runtime component loading from shared libraries (4.8).
- An introspection interface sufficient for a task browser or reporter to
  be written against it. Those tools themselves are outside this plan.

### 4.7 C++26 reflection (GCC 16)

With GCC 16 as the only supported compiler, reflection is available in
every supported build and is the default way a component is described:
the `describe<T>` tables are generated from the declarations themselves.
The hand-written form of 4.4 stays as an explicit override and as the
fallback if the experimental feature changes or has to be switched off.

**What GCC 16 provides.** Reflection is experimental and off by default; it
needs `-std=c++26 -freflection` and the `<meta>` header. Status below is
from the GCC 16 release notes and the cppreference support table; it has
not been compile-tested here, as this plan was written on a machine with
GCC 13 only.

| Paper | Feature | Use here |
|---|---|---|
| P2996R13 | `^^` operator, splicers, `std::meta::info`, `<meta>` queries | Everything below |
| P3394R4 | Annotations, `[[=value]]` | Port, operation and property metadata |
| P3096R12 | Function parameter reflection | Operation argument names and types |
| P3293R3 | Splicing a base class subobject | Type descriptions for derived structs |
| P3491R3 | `define_static_string`, `define_static_object`, `define_static_array` | Placing generated tables in static storage |
| P3795R2 | Reflection clean-ups | - |
| P1306 | Expansion statements, `template for` (experimental) | Iterating members without recursive templates |

To confirm on a real GCC 16 before design freeze: P3560 (error handling in
`std::meta` functions) and P3068 (constexpr exceptions) were not confirmed
by the sources available. Mainline Clang and MSVC are reported to have no
P2996 support yet.

**Rules.**

- On by default; compiled out when `__cpp_impl_reflection` and
  `__cpp_lib_reflection` are not defined (the `-std=c++23` mode). Header-only, under `rtt3/meta/`; the built libraries never
  depend on it.
- Reflection fills the primary template of `rtt3::describe<T>`. An explicit
  specialisation always wins. It adds no runtime data structure of its own,
  and CI checks that a reflected and a hand-written description of the same
  component are identical.
- `<meta>` is included only from `rtt3/meta/` because of its compile-time cost.

**Uses.**

- *Port discovery*: walk `nonstatic_data_members_of(^^T, ctx)` and keep
  members whose type is a specialisation of a port template.
- *Names*: `identifier_of(member)`, placed in read-only storage with
  `define_static_string`.
- *Annotations* replace the remaining entries of the description:

  ```cpp
  struct Driver {
      [[=rtt3::doc("Joint command"), =rtt3::event]]
      rtt3::input_port<JointCommand> command_in;

      [[=rtt3::property, =rtt3::unit("rad/s")]]
      double max_velocity = 1.0;

      [[=rtt3::operation(rtt3::own_thread), =rtt3::doc("Clear a latched fault")]]
      std::expected<void, rtt3::error> reset(int axis) noexcept;

      void update() noexcept;
  };
  ```

  Argument names come from `parameters_of`. A header using `[[=...]]` is a
  C++26 header; there is no wrapper macro.
- *Type descriptions* for aggregate data types and enums, generated from
  the type alone (`nonstatic_data_members_of`, `bases_of`,
  `enumerators_of`). This removes the need for typekits for plain structs.
- *Deployment as a struct*: nodes are data members with activity
  annotations, so instances are identified by member instead of by tag.
  The same validation as 4.5 runs over the reflected members.
- *Diagnostics* that name the offending member, for example
  "input `App::right.command_in` has no writer".

**Not planned.** `define_aggregate` (generating new class types) until
there is a concrete need. Contracts, also experimental in GCC 16, are a
separate topic.

**Risk.** The feature is behind an experimental flag in one compiler and
may change in GCC 16 point releases and in GCC 17, and the project now
relies on it by default. Two things contain that: it only produces tables
that can also be written by hand, and the `-std=c++23` mode is built and
tested in CI, so a regression in the compiler costs convenience, not
function. The exact GCC 16 point release is pinned in the toolchain image. Only public members are reflected by default.

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

The registrar adds a typed factory, named from `describe<Sensor>`, to a
registry owned by `librtt3-reflect`; the loader opens the library and reads
what was added. There are no `extern "C"` entry points. The exact spelling
is settled with a test that registration survives static archives and
`--gc-sections`.

**Macros that remain**, all internal to library headers: include guards or
`#pragma once`; one symbol-visibility macro from a generated export header;
`#if` on the two reflection feature-test macros to include the C++26 tier;
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
| 0. Baseline | Fork, GCC 16 toolchain image (native and AArch64 cross), new CMake and CI beside the old tree, benchmark harness, RTT 2.x numbers recorded on desktop and ARM board | Reproducible baseline; empty `rtt3` libraries build and install next to RTT 2.x |
| 1. OS layer | `rtt3::os`, memory locking, graceful downgrade, logger front end, simulated clock | Thread, timing and lock tests pass; wake-up latency measured on the ARM board |
| 2. Lock-free primitives | Data object, buffers, queues, pool | Stress and model-checking tests pass on both architectures |
| 3. Components and ports | `component` concept, ports, channels, policies, `connect`, activities, lifecycle, error type | Hand-wired example runs; port latency no worse than baseline |
| 4. Static deployment | `rtt3::deployment`, validation, scheduling fold, channel selection, `run` | Example system runs with zero allocations after start-up and no string handling |
| 5. Operations, properties, description | `describe<T>` written by hand first, then generated by reflection (4.7); typed callers, engine messages | Operation call cost no worse than baseline; reflected and hand-written descriptions identical in CI |
| 6. Runtime layer | `rtt3-reflect`: names, type-erased connect, type descriptions, property files, registrar and loader, `expose` | A static deployment can be inspected and extended at run time; an RTT 2.x and an rtt3 component load into one process |
| 7. Reflection extras | Reflected type descriptions for data types, deployment as a struct, member-naming diagnostics | Example system written with no hand-written descriptions at all |
| 8. Release | Delete the old `rtt/` tree, migration guide with a worked port of a real component, API and ABI policy | Tagged 3.0 |

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
- **Compile time and error quality** of deployment templates on large
  systems. Mitigation: benchmark from phase 0; keep checks in constexpr
  functions over index arrays instead of deep recursive templates.
- **Single, very recent compiler.** GCC 16 only, with an experimental
  feature on by default. Mitigation: pinned toolchain image, the tested
  `-std=c++23` fallback mode, static linking of libstdc++ on the target.
- **Tooling.** clangd and clang-tidy cannot parse reflection code; IDE
  support for component headers will be limited until Clang catches up.
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
11. **Runtime tooling**: how much by-name runtime access do you actually
    use (task browser, reporter, deployer XML)? This sets how large
    `rtt3-reflect` needs to be, and whether phase 6 can shrink.
12. **Fallback mode**: keep the hand-written `describe<T>` path and the
    `-std=c++23` build mode as insurance (as planned), or require
    reflection outright? Requiring it removes a build mode and some code,
    but ties every component to an experimental compiler flag.
13. **Exceptions from user hooks**: is `noexcept` hooks with termination
    the right default, with `catch_exceptions` as opt-in, or the reverse?
14. **Project name and licence**: is the package published as `rtt3`, does
    it keep the Orocos name, and is the existing licence kept?
15. **Portability beyond Linux**: adopt the low-cost guardrails that keep
    an RTOS or microcontroller port possible (core builds with
    `-fno-exceptions -fno-rtti`, no POSIX headers in the core, lock-free
    primitives parameterised on word size), or leave them out?
