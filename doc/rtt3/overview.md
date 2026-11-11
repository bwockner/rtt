# rtt3: overview, decisions and architecture

Status: proposal. Scope is the C++ API only. The CORBA transport and the
Python and Lua bindings are not carried forward.

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

## Decisions taken

| Topic | Decision |
|---|---|
| Language and compiler | GCC 16 with `-std=c++26 -freflection` is the one supported configuration. Reflection is required: it is how components, data types and deployments are described. There is no C++23 mode and no hand-written description path. |
| API | A new, clean API. The classic RTT API (`TaskContext`, `InputPort<T>`, `OperationCaller`, ...) is not provided, not even as a wrapper. Existing components are ported by hand with a migration guide. |
| Namespace and install | Everything lives in namespace `rtt3` with its own header directory, library names and CMake package, so it installs next to RTT 2.x and can be loaded into the same process. |
| Introspection | The typed core carries no names and no runtime type registry. Names, by-name lookup, configuration files, the inspection service and runtime loading are a separate library on top. It is optional for the core but is part of every deployment that replaces today's deployer use (`runtime-layer.md`). |
| Same-thread data flow | Design goal, not a hard requirement: no framework overhead where none is required. A link between two nodes in the same thread should cost what an inlined function call costs (`deployment.md`). |
| Deployment | Static first, with install-time variation as a first-class requirement: sensor and actuator stacks, kinematic variants and hardware simulation are selected per machine from a configuration file, without rebuilding (`deployment.md`). Runtime loading of component libraries remains for stacks not known at build time. |
| Macros | Replaced by templates wherever the language allows. The few that remain are listed in `conventions.md`. |
| OS targets | Linux only, standard and PREEMPT_RT kernels, on x86-64 (desktop testing) and ARM (deployment). |
| Errors | The API reports failure through `std::expected` and never throws. Exceptions are tolerated only at the boundary to user code (`conventions.md`). |

## Where the code is today

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

## Approach and target architecture

## Build new, do not convert

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

## Libraries

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

## Naming and concurrent installation

| Item | RTT 2.x | rtt3 |
|---|---|---|
| Namespace | `RTT` | `rtt3`, with an inline ABI namespace (`rtt3::v0`) |
| Headers | `<rtt/...>` | `<rtt3/...>` |
| Libraries | `liborocos-rtt-<target>` | `librtt3-os`, `librtt3-core`, `librtt3-reflect` |
| CMake package and targets | `orocos-rtt` | `rtt3`, targets `rtt3::core` and so on |
| pkg-config | `orocos-rtt-<target>.pc` | `rtt3-*.pc` |
| Component search path | `RTT_COMPONENT_PATH` | `RTT3_COMPONENT_PATH` |
| Plugin entry points | unversioned `extern "C"` symbols | none (`conventions.md`) |

Because every symbol is mangled under `rtt3`, both versions can be loaded
into one process, so a system can be migrated one component at a time, with
a small hand-written bridge component where data has to cross.

## What is not carried forward

- CORBA transport, Python and Lua bindings (out of scope by request).
- The classic API and with it the `DataSource`, `TypeInfo`/typekit and
  `OperationInterfacePart` machinery.
- All non-Linux OS targets and assembler atomics.
- catkin, rosbuild, the `configure` script, Xerces.
- **Proposed, to be confirmed (question M1):** the scripting engine
  (program scripts and state machines). It is 111 files written entirely
  against the DataSource machinery and Spirit Classic, so keeping it means
  writing a new interpreter, not porting one.

## Risks

- **No automatic migration.** Every existing component has to be ported by
  hand. Mitigation: side-by-side loading, a migration guide written against
  a real component, and a mapping table from each classic call to its
  `rtt3` equivalent.
- **No inherited regression net.** The old tests do not compile against the
  new API. Mitigation: treat them as a checklist of behaviours, and port
  the lock-free and activity tests first.
- **Lock-free rewrites** can introduce rare ordering bugs that x86 hides.
  Mitigation: `realtime-internals.md` verification on ARM.
- **Combinations of slot alternatives** multiply the checks the compiler
  performs. Mitigation: each alternative is checked against the slot's
  links independently, so cost grows with the number of alternatives, not
  with the number of combinations.
- **Compile time and error quality** on large systems. Mitigation:
  benchmark from stage S0; keep checks in constexpr functions over the
  generated tables.
- **Dependence on an experimental compiler feature, with no fallback.**
  If GCC changes or withdraws part of reflection, `rtt3` does not build
  until it is adapted, and no other compiler can stand in. Mitigation: the
  containment measures in `reflection.md`, a pinned toolchain image, and the stage S1
  spike before anything else is built on it.
- **Tooling.** clangd and clang-tidy cannot parse `rtt3` code at all until
  Clang implements reflection, so editor support and static analysis are
  limited to what GCC provides.
- **Ecosystem.** OCL (deployer, taskbrowser, reporting), typegen/orogen and
  rtt_ros integration do not work with `rtt3` and are outside this plan.
- **Licence.** The existing code is GPL version 2 only, with a linking
  exception, and has many copyright holders, so its licence cannot in
  practice be changed. Code derived from it must stay under the same
  terms; only code written new, without copying or adapting RTT 2.x
  sources, can carry a different licence. The choice for `rtt3` has to be
  made before code is ported. See `../open-questions.md`, section 4.
