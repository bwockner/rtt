# RTT modernisation plan (C++17 core, C++23 opt-in)

Status: proposal for a new fork. Scope is the C++ API only; the CORBA transport
and the Python and Lua bindings are out of scope and are not carried forward.

## 1. Decisions taken

| Topic | Decision |
|---|---|
| Language baseline | The library builds as C++17. C++20/23 features live in an additive header layer that switches on when the consumer compiles with a newer standard. |
| Introspection | The typed core carries no names and no `TypeInfo`. Names, by-name lookup, properties, marshalling and scripting become an opt-in library layered on the core. |
| Deployment | Static first: a deployment is described in C++ and checked and wired by the compiler. Runtime loading (`ComponentLoader`, `PluginLoader`) stays as an optional module. |
| OS targets | Keep a pluggable OS layer, expressed as a C++ trait instead of the `fosi` C macros, so RTOS and bare-metal backends can be added. Linux is the reference backend. |

## 2. Where the code is today

Measured on this tree, excluding `rtt/transports/corba`:

- About 105k lines under `rtt/`. `scripting` (111 files), `internal` (100),
  `os` (117), `types` (59) and `marsh` (33) are the bulk.
- No C++ standard is set in CMake; the code is C++98 with Boost standing in
  for later standard features.
- Boost usage: `boost::function` 495 uses, `boost::bind` 205,
  `boost::shared_ptr` 140, `boost::intrusive_ptr` 55, `boost::serialization` 56,
  `boost::mpl` 44, `boost::fusion` 23, plus `function_types`, `type_traits`,
  `lambda`, `call_traits`, `lexical_cast`, Spirit Classic (scripting parsers)
  and Graph (scripting).
- Operation signatures are taken apart with `function_types` + `fusion`
  (`OperationInterfacePartFused.hpp`, `FusedFunctorDataSource.hpp`,
  `CreateSequence.hpp`, `UnMember.hpp`, `BindStorage.hpp`).
- 742 `const std::string&` parameters. Ports, operations, properties,
  services and peers are all found by name in `std::map`/`std::vector`
  containers and then recovered with `dynamic_cast` (81 uses), for example
  `DataFlowInterface::getPortType<T>(name)` and `ConnFactory::createConnection`.
- Every `PortInterface` stores three `std::string`s (name, full name,
  description) and is polymorphic over 20 or so virtuals.
- `rtt/os` carries gnulinux, Xenomai 2, RTAI/LXRT, eCos, macOS and Win32
  targets, hand-written atomics per architecture (`oro_i386`, `oro_x86_64`,
  `oro_powerpc`, `oro_gcc`, `oro_msvc`, `oro_noasm`) and a vendored TLSF.
- Components register through `ORO_CREATE_COMPONENT` macros into a global
  `std::map<std::string, factory>`; there is a static-link variant, but it
  still resolves by string at start-up.
- Build: vendored `FindBoost.cmake`, catkin and rosbuild (`manifest.xml`)
  support, a `configure` wrapper, `.travis.yml`, optional Xerces with a
  vendored TinyXML fallback, tests on Boost.Test.

## 3. Target architecture

Four libraries, each depending only on the ones above it:

```
rtt-os          OS trait + Linux backend; threads, clocks, mutexes, memory
rtt-core        typed ports, channels, buffers, activities, execution engine,
                TaskContext lifecycle. No names, no TypeInfo, no Boost.
rtt-reflect     names, TypeInfo/typekits, properties, operations by name,
                marshalling, runtime component and plugin loading
rtt-scripting   program and state machine scripts (depends on rtt-reflect)
```

`rtt-core` is the part a statically deployed real-time system links.
`rtt-reflect` gives back the classic API: `TaskContext::getPort("name")`,
`addOperation("name", ...)`, `.cpf` files, and so on.

### Compatibility model

- **Classic API** (`RTT::TaskContext`, `InputPort<T>`, `OutputPort<T>`,
  `Property<T>`, `OperationCaller<Sig>`) keeps its spelling. It is
  implemented in `rtt-reflect` as thin wrappers over the typed core, so a
  C++17 component that includes `<rtt/RTT.hpp>` and links `rtt-reflect`
  keeps compiling. Breaks are limited to `boost::` types in public
  signatures (see 4.2).
- **Typed API** (new namespace, proposed `rtt::`) is C++17. It uses
  `template<auto>` member-pointer handles, `std::string_view`,
  `std::optional`, `std::variant` and `if constexpr`.
- **C++23 layer** is guarded by feature-test macros (`__cpp_concepts`,
  `__cpp_lib_expected`, `__cpp_explicit_this_parameter`, ...) and only adds
  overloads and diagnostics. No ABI-affecting type changes shape depending
  on the consumer's standard, so C++17 and C++23 translation units can link
  against the same library binary.

## 4. Workstreams

### 4.1 Build and repository

- Minimum CMake 3.22; target-based, exported targets
  (`rtt::os`, `rtt::core`, `rtt::reflect`, `rtt::scripting`), `CMakePresets.json`.
- `CMAKE_CXX_STANDARD 17` for the library; CI also builds and tests as C++20
  and C++23.
- Delete: vendored `FindBoost.cmake` and the other `config/Find*.cmake`
  modules no longer needed, `configure`, `manifest.xml`, catkin and
  `env-hooks`, `.travis.yml`, `UseOROCOS-RTT*.cmake` (replace with a small
  config package and an `rtt_add_component()` helper).
- CI on GitHub Actions: GCC 9 (C++17 floor) and GCC 13+/Clang 17+ (C++23),
  with ASan, UBSan and TSan jobs. TSan matters because the lock-free code is
  being rewritten.
- Tests: move off Boost.Test to Catch2 or GoogleTest (see open questions).
  Port the tests before touching the code they cover; the 96-file suite is
  the regression net for the whole effort.

### 4.2 Removing deprecated dependencies

Out of the fork entirely: `rtt/transports/corba`, omniORB/TAO detection,
`ORO_REMOTING`, RTAI/LXRT, Xenomai 2, eCos, `oro_arch` assembler, Xerces,
`std::auto_ptr`, dynamic exception specifications (34 `throw(...)`).

Boost replacements in `rtt-os` and `rtt-core` (goal: zero Boost):

| Today | Replacement |
|---|---|
| `boost::shared_ptr`, `weak_ptr`, `make_shared`, `scoped_ptr`, `enable_shared_from_this` | `std::` equivalents |
| `boost::function`, `bind`, `mem_fn`, `lambda`, `ref` | `std::function` and lambdas; a fixed-capacity, non-allocating callable for real-time paths |
| `boost::type_traits`, `mpl`, `static_assert`, `call_traits`, `utility` | `<type_traits>`, variadic templates, `static_assert`, `if constexpr` |
| `boost::function_types` + `fusion` | variadic signature traits, `std::tuple`, `std::apply`, `std::invoke` |
| `boost::intrusive_ptr` | a small in-tree intrusive pointer on `std::atomic` (kept because channel elements must not allocate a control block) |
| `boost::array`, `tuple`, `cstdint` | `std::array`, `std::tuple`, `<cstdint>` |
| `boost::thread`, `date_time` | `std::thread`/OS trait, `<chrono>` |
| `boost::filesystem` | `std::filesystem` (only in `rtt-reflect` loaders) |
| `boost::lexical_cast` | `std::from_chars`/`to_chars` |
| `oro_atomic_*`, `CAS`, `AtomicInt`, `volatile` flags | `std::atomic` with explicit memory orders |

The `fusion` rewrite is the largest single item: it replaces the five
headers listed in section 2 with one variadic implementation and removes a
large amount of template instantiation cost.

Boost that survives, only in the optional layers, until separately replaced:
Serialization (type decomposition in `rtt/types` and the mqueue transport),
Spirit Classic and Graph (scripting). See open questions.

Source-compatibility consequence: any classic-API signature that names a
`boost::` type changes, notably `boost::shared_ptr` handles and
`boost::function` callbacks such as `DataFlowInterface::SlotFunction`.
Lambdas and function pointers keep working; code passing `boost::bind`
results into `std::function` also keeps working. Code that stores the
`boost::` types by name needs editing. A `RTT_LEGACY_BOOST_ALIASES` option
could ease this but is not recommended.

### 4.3 OS abstraction as a trait

Replace `fosi.h` and the per-target macro headers with a backend type that
the core is compiled against:

```cpp
struct linux_backend {
    using thread = ...;  using mutex = ...;  using condition = ...;
    using clock  = ...;  // monotonic, nanosecond
    static result set_scheduling(thread&, sched_params);
    static result set_affinity(thread&, cpu_set);
    static void*  rt_allocate(std::size_t) noexcept;
    ...
};
```

- Selected at configure time (`RTT_OS_BACKEND`), checked by a `static_assert`
  trait set in C++17 and by a `concept os_backend` in C++20 and later, which
  gives a porter a readable list of what is missing.
- Linux backend: pthreads for priority, policy, affinity and
  `clock_nanosleep`; `std::atomic`; `mlockall` and stack prefault helpers.
- Requirements a bare-metal backend imposes on the core, to be honoured
  from the start: no exceptions or RTTI required in `rtt-core`
  (`-fno-exceptions -fno-rtti` build in CI), no heap use after
  configuration, no `<iostream>`, no `<filesystem>`, no static-init order
  dependencies (`StartStopManager` goes).
- TLSF becomes one optional `std::pmr::memory_resource`; `rt_string` and
  `oro_allocator` become pmr aliases.
- `Logger` splits: a lock-free, non-formatting real-time front end in
  `rtt-core` and sinks in `rtt-reflect`.

### 4.4 Typed core: ports and connections without strings

Core port types carry only what the data path needs:

```cpp
namespace rtt {
template <class T> class output_port;   // no name, no virtuals on the write path
template <class T> class input_port;
template <class T, class Policy> class channel;  // data / buffer / circular
}
```

A component declares ports as plain members. A port is identified by a
member pointer, so discovery is a compile-time expression instead of a map
lookup followed by `dynamic_cast`:

```cpp
struct Controller : rtt::component<Controller> {
    rtt::input_port<JointState>    state;
    rtt::output_port<JointCommand> command;
    void update();
};

rtt::connect(ctrl.command, driver.command_in, rtt::policy::data{});   // T mismatch is a compile error
```

- Type agreement is checked by overload resolution. The
  `getTypeInfo()` pointer comparison in `ConnFactory.cpp` and the
  `dynamic_cast<InputPort<T>*>` calls disappear from the core.
- The connection policy becomes a type where it is known statically, so the
  buffer implementation is chosen at compile time and the write path can be
  inlined. `ConnPolicy` as a runtime struct remains in `rtt-reflect`.
- Port enumeration without macros: C++17 uses an explicit
  `static constexpr auto ports = rtt::ports(&Controller::state, &Controller::command);`
  list. This is the one piece of boilerplate; C++26 reflection can remove it
  later without an API change.
- Names are attached outside the object, in `rtt-reflect`, as a constexpr
  table of `{string_view, member pointer}` per component type. The classic
  `getPort("state")` is a lookup in that table, and a statically deployed
  system that never asks for a name links none of it.
- Where identifiers must exist at run time (remote transports, logging),
  use compile-time FNV-1a hashes of the name with the string kept in a
  separate section, not `std::string` members.
- Classic `RTT::InputPort<T>`/`OutputPort<T>` become wrappers that own a
  core port plus a name and implement `PortInterface`.

The same treatment applies, in this order, to operations
(`rtt::operation<Sig>` as a member, called through a typed handle, with
`OperationCaller<Sig>` by name layered on top), then properties and
attributes, then services and peers.

String overhead reductions that apply even to the classic API:
`std::string_view` parameters for lookups, heterogeneous lookup in sorted
flat containers instead of `std::map<std::string, ...>`, no `std::string`
construction on any path reachable from `updateHook`.

### 4.5 Compile-time deployment

A deployment is a type listing components, activities and connections:

```cpp
using App = rtt::deployment<
    rtt::node<Sensor,     rtt::periodic<1'000'000 /*ns*/, rtt::priority<80>>>,
    rtt::node<Controller, rtt::periodic<1'000'000,        rtt::priority<79>>>,
    rtt::node<Driver,     rtt::triggered<&Driver::command_in>>,
    rtt::link<&Sensor::state,       &Controller::state,   rtt::policy::data>,
    rtt::link<&Controller::command, &Driver::command_in,  rtt::policy::buffer<8>>
>;

int main() { App app; return app.run(); }
```

This works in C++17 through `template<auto>` parameters. What the compiler
does with it:

- **Validation**: port value types match, every required input is
  connected, no input has two writers unless its policy allows it, the
  triggering port belongs to the node. Failures are `static_assert`s; under
  C++20 they are concept failures with clearer messages.
- **Storage**: all components, channels and buffers are members of `App`
  with sizes known at compile time, so nothing is allocated at start-up and
  the whole deployment can be placed in static storage.
- **Dispatch**: `rtt::component<Derived>` is CRTP, so `updateHook` and the
  port read and write calls are direct, inlinable calls. With the C++23
  layer, deducing `this` removes the CRTP parameter from user code.
- **Scheduling**: nodes that share a period and priority can be folded into
  one thread with a fixed execution order, computed from the link graph at
  compile time (topological order, cycles reported as errors unless
  explicitly broken with a delayed link).
- **Start-up**: the configure, start and stop order is a constexpr sequence,
  replacing `StartStopManager` and string-keyed factories.
- **Dead code**: no component factories, name tables or type registries are
  linked unless `rtt-reflect` is used. LTO is on by default for static
  deployments.

Mixed deployments are supported: `App` can expose its nodes to
`rtt-reflect` (`rtt::reflect::expose(app)`), which registers name tables so
runtime tooling can inspect or attach additional dynamically loaded
components. `ORO_CREATE_COMPONENT` continues to work in `rtt-reflect` for
dynamic loading.

Systems with more than one instance of a component type need
instance tags (`rtt::node<Driver, ..., rtt::tag<left>>` and
`rtt::link<rtt::at<left, &Driver::command_in>, ...>`); this is part of the
first design, not a follow-up.

### 4.6 C++23 opt-in layer

Added when the consumer's standard allows it, never required:

- Concepts: `rtt::port_data`, `rtt::component_type`, `rtt::os_backend`,
  `rtt::connectable<Out, In>`; used to constrain the same templates that
  C++17 guards with `static_assert`.
- `std::expected<T, rtt::errc>` returning variants of configuration-time
  calls (`connect`, `configure`, loader functions) next to the `bool`
  returning ones.
- Deducing `this` for components, as above.
- String-literal template parameters for naming
  (`rtt::named<"state", &Controller::state>`), feeding the same tables as the
  C++17 form.
- `std::span` for buffer and bulk port access, `std::move_only_function` for
  one-shot engine messages, `constexpr` containers for larger deployment
  graph checks, `std::format`/`std::print` sinks in the non-real-time logger,
  `[[likely]]`, `std::unreachable`, `std::to_underlying`, `std::source_location`.
- Possible later: a `std::generator`/coroutine based alternative to
  `updateHook`, and modules (`import rtt;`) once toolchain support is settled.

### 4.7 Lock-free and real-time internals

- Reimplement `DataObjectLockFree`, `BufferLockFree`, `AtomicQueue`,
  `AtomicMWSRQueue`, `ListLockFree` and `TsPool` on `std::atomic`, with
  documented memory orders, ABA handling and cache-line alignment
  (`std::hardware_destructive_interference_size`).
- Add single-producer/single-consumer variants and let the static
  deployment pick them when the link graph proves one writer and one reader.
- Verification: TSan in CI, stress tests, and model checking of the queue
  and data-object algorithms (for example with Relacy or a similar tool).
- `ExecutionEngine`: fixed-capacity message queue of non-allocating
  callables; `SendHandle` kept in the classic layer.
- `Signal`/`Handle` reduce to what event ports need.

## 5. Phasing

Each phase ends with the full test suite green on the C++17 and C++23 jobs.

| Phase | Content | Exit criterion |
|---|---|---|
| 0. Baseline | Fork, new CMake, CI, port tests off Boost.Test, benchmark harness (port write/read latency, operation call cost, start-up time, binary size, compile time) | Existing behaviour measured and reproducible |
| 1. Strip | Remove CORBA, dead OS targets, rosbuild/catkin, Xerces, assembler atomics; mechanical Boost to `std` replacement | `rtt/os`, `base`, `internal` free of Boost except `fusion`/`function_types` |
| 2. Signatures | Variadic rewrite of the operation machinery | No `fusion`, `mpl` or `function_types`; compile time down measurably |
| 3. OS trait | Backend trait, Linux backend, pmr memory, logger split | Core builds with `-fno-exceptions -fno-rtti`; a stub second backend compiles |
| 4. Library split | Separate `rtt-core` from `rtt-reflect` and `rtt-scripting`; classic API still implemented the old way | Core links with no name tables or `TypeInfo` |
| 5. Typed ports | New core ports, channels and lock-free primitives; classic ports become wrappers | Classic tests pass on wrappers; write/read latency no worse than baseline |
| 6. Static deployment | `rtt::deployment`, validation, scheduling fold, `expose` bridge | Example system runs with zero allocations after `main` starts and no string lookups |
| 7. Typed operations, properties, services | Same pattern as ports | Classic layer fully implemented on the typed core |
| 8. C++23 layer | Concepts, `expected`, deducing `this`, named NTTPs | Same examples written both ways produce equivalent binaries |
| 9. Release | Migration guide, deprecation attributes on classic calls that have typed equivalents, API and ABI policy | Tagged 3.0 |

Phases 1 to 3 are low-risk and independent of the design questions below.
Phases 4 to 7 depend on them.

## 6. Risks

- **Operation machinery rewrite**: scripting and `OperationCaller` both sit
  on it. Mitigation: do it before the library split, with the existing tests.
- **Lock-free rewrites** can introduce rare ordering bugs. Mitigation: 4.7
  verification, and keep the old algorithms' structure where it is sound.
- **Compile time and error quality** of the deployment templates on large
  systems. Mitigation: benchmark from phase 0, keep the checks in constexpr
  functions over index arrays instead of deep recursive templates.
- **Two APIs** double the documentation and support surface until the
  classic layer is frozen.
- **Ecosystem**: OCL (deployer, taskbrowser, reporting), typegen/orogen and
  rtt_ros integration all depend on the parts being moved into `rtt-reflect`.
  They are out of this plan but will need matching forks or will stop working.
- **Licence**: the tree is GPL v2 with a linking exception; new code in the
  fork needs a decision before the first external contribution.

## 7. Open design questions

1. **Fork identity**: keep the `RTT::` namespace, `orocos-rtt` package name
   and header paths for the classic layer, or rename everything and ship the
   old names as a compatibility package? This decides whether the fork can be
   installed next to RTT 2.x.
2. **Scripting**: port the Spirit Classic parsers to a maintained parser
   (Spirit X3, PEGTL, hand-written), freeze the module as legacy, or drop it?
   It is 111 files and the last user of Spirit Classic and Boost.Graph.
3. **Typekits and Boost.Serialization**: keep Serialization for structure
   decomposition in `rtt-reflect`, or replace it with an aggregate
   reflection approach (Boost.PFR style now, C++26 reflection later)? This
   also decides whether existing generated typekits remain usable.
4. **Inter-process transport**: with CORBA gone, is the POSIX mqueue
   transport enough, should it be dropped as well, or is a new transport
   (shared memory, Zenoh, DDS) part of the fork's scope?
5. **Dynamic reconfiguration in static deployments**: must connections in a
   compile-time deployment be changeable at run time (disconnect, reconnect,
   change policy), or are they fixed for the life of the process? Fixed
   links allow the SPSC selection and inlining described in 4.5.
6. **Error handling in the core**: `bool` returns as today, an in-tree
   `expected`-like type usable from C++17, or exceptions at configuration
   time only? The `-fno-exceptions` goal for bare-metal constrains this.
7. **Operations threading model**: keep both `OwnThread` and `ClientThread`
   with `send`/`collect`, or simplify the typed API to one model?
8. **Which RTOS first**: naming one concrete second backend (Zephyr,
   FreeRTOS, Xenomai 4/EVL, QNX) early will shape the trait far better than
   a stub. Which one matters to you?
9. **Minimum compilers**: is GCC 9/Clang 10 an acceptable C++17 floor, and
   GCC 13/Clang 17 for the C++23 layer? Is MSVC needed at all?
10. **ABI stability**: is a stable shared-library ABI for `rtt-reflect`
    required for separately built component libraries, or is rebuild-the-world
    acceptable?
11. **Test framework**: Catch2, GoogleTest, or keep Boost.Test as a
    test-only dependency to avoid porting 96 files?
12. **XML properties**: keep `.cpf` with the vendored TinyXML, move to
    TinyXML-2/pugixml, or change format?
