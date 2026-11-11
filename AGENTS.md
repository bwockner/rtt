# AGENTS.md

Guidance for automated agents and new contributors working in this
repository. Read `README.md` first for the intent of the fork.

## What this repository is

A fork of [orocos-toolchain/rtt](https://github.com/orocos-toolchain/rtt),
the Orocos Real-Time Toolkit, with two lines of work:

1. Maintaining the existing **RTT 2.x** code for Linux (standard and
   PREEMPT_RT kernels) on x86-64 and ARM, with optimisations that need no
   change to component source code.
2. Designing and later building **`rtt3`**, a successor with a new API.
   No `rtt3` code exists yet. Do not create an `rtt3/` tree, new CMake
   packages or new libraries unless asked; the design is still being
   settled.

Out of scope for this fork: the CORBA transport, Python and Lua bindings.
Do not spend effort on them, but do not break them either while they
remain in the tree.

## Documents to read before changing anything

| File | Use it for |
|---|---|
| `doc/rtt3/README.md` | Index of the `rtt3` plan: one file per topic, the stages, what each depends on and what can run in parallel |
| `doc/rtt3/overview.md` | Decisions taken, architecture, risks |
| `doc/rtt3/logging.md` | The lock-free logger for `rtt3` |
| `doc/rtt3/numerics.md` | SIMD, alignment, floating-point settings, comparing results with the simulator |
| `doc/optimisation-plan.md` | Findings in the RTT 2.x code and the order to fix them |
| `doc/open-questions.md` | Everything not yet decided, the licence position, and what was assumed without verification |

The plans are proposals. Each records which points the owner has decided
and which are still open. When a decision is made, update the plan it
belongs to and `doc/open-questions.md` together.

## Repository layout

```
rtt/                 RTT 2.x sources (about 105k lines without CORBA)
  base/              port and channel interfaces, buffers, data objects
  internal/          connection factory, operation plumbing, lock-free queues
  os/                OS abstraction ("fosi"), threads, mutexes, TLSF allocator
    gnulinux/        the only target this fork cares about
    xenomai/ lxrt/ ecos/ macosx/ win32/   other targets, unused here
    oro_*/           per-architecture atomic operations
  extras/            activities (periodic, slave, file descriptor, simulation)
  types/ typekit/    runtime type system
  scripting/         program and state machine scripts (Boost.Spirit Classic)
  marsh/             property files (.cpf), vendored TinyXML
  plugin/ deployment/  plugin and component loading
  transports/        corba, mqueue
tests/               Boost.Test suite, 41 CTest tests
config/              CMake modules, including a vendored FindBoost.cmake
doc/                 upstream documentation sources, plus the plans above
UseOROCOS-RTT*.cmake CMake helpers used by component packages
```

Where the hot paths are, for anyone touching performance:

| Path | Files |
|---|---|
| Port write and read | `rtt/OutputPort.hpp`, `rtt/InputPort.hpp`, `rtt/base/ChannelElement.hpp`, `rtt/base/ChannelInterface.cpp` |
| Lock-free data and buffers | `rtt/base/DataObjectLockFree.hpp`, `rtt/base/BufferLockFree.hpp`, `rtt/internal/TsPool.hpp`, `rtt/internal/AtomicMWSRQueue.hpp` |
| Component cycle | `rtt/ExecutionEngine.cpp`, `rtt/os/Thread.cpp`, `rtt/Activity.cpp` |
| Mutexes, clocks, periodic wait | `rtt/os/gnulinux/fosi.h`, `rtt/os/gnulinux/fosi_internal.cpp`, `rtt/os/Mutex.hpp` |
| Logger | `rtt/Logger.cpp`, `rtt/Logger.inl` |
| Real-time allocator | `rtt/os/tlsf/`, `rtt/os/oro_malloc.h`, `rtt/os/oro_allocator.hpp` |
| Component registration macros | `rtt/Component.hpp`, `rtt/plugin/ServicePlugin.hpp` |

## Building and testing RTT 2.x

Verified in this configuration: Ubuntu 24.04, GCC 13.3, CMake 3.28,
Boost 1.83, x86-64, CORBA off. Nothing has been built on ARM or with any
other compiler.

```sh
apt-get install -y libboost-all-dev        # Boost is the only hard dependency
mkdir build && cd build
cmake .. -DENABLE_TESTS=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j4                                    # around 20 minutes on 4 cores
ctest --timeout 180 -j2                     # about 2 minutes
```

Things to expect:

- The target defaults to `gnulinux` when `OROCOS_TARGET` is not set.
- CMake prints deprecation warnings about the minimum version and old
  policies. The build prints over a thousand compiler warnings, most of
  them Boost.Bind placeholder deprecation notes. Neither is a failure.
- `buffers_test` takes close to two minutes. It is a stress test, not a
  hang.
- `scripting_test` hangs intermittently when run in parallel with other
  tests (seen with `ctest -j3` and `-j4`; 43 consecutive passes when run
  alone). Re-run it alone before treating a timeout as a regression. The
  cause has not been investigated.
- `.travis.yml` is the only CI configuration and is obsolete. There is no
  working CI.

## Branches

- `master`: upstream `master` plus `toolchain-2.9` and `toolchain-2.10`,
  two small fixes and the plans. This is the branch to work from.
- `toolchain-2.x`: upstream release branches. `toolchain-2.9` and
  `toolchain-2.10` are fully contained in `master`.
- Many other branches are unmerged upstream pull requests. Some overlap
  with items in `doc/optimisation-plan.md` (section 3.6 lists them, with
  their state). Treat them as starting points to review. Several are
  marked work in progress or proof of concept by their authors, and
  several conflict with `master`.
- `toolchain-2.9.1` and `fix/2.9.1-abi-compatibility` carry
  ABI-compatibility workarounds for a released version. Do not merge them
  into `master`.

## Rules for changes to RTT 2.x

- **No source API change.** Component code must keep compiling without
  edits. State the ABI effect of every change in the commit message or
  pull request: none, recompile (header templates and inline functions),
  or layout (size or layout of an exported class).
- **Measure before optimising.** The optimisation plan's findings come
  from reading the code. Build the benchmark first (step 0 of that plan)
  and keep the numbers with the change.
- **Lock-free code**: x86 hides most memory-ordering errors. A change to
  the atomics, data objects, buffers or queues is not verified until it
  has run on ARM hardware. Do not weaken memory ordering as a quick win.
- **Behaviour changes go behind an option**, with the default argued in
  the change (for example priority inheritance on mutexes).
- **Licence headers**: every file under `rtt/` carries the GPL v2 plus
  exception header. Keep it on files you modify, and put the same header
  on new files that are part of RTT 2.x.
- **Commit messages** follow the upstream style, a lower-case area prefix
  and a short summary: `os: ...`, `extras: ...`, `cmake: ...`,
  `logger: ...`, `tlsf: ...`.

## Rules for `rtt3` design work

These are decisions already taken; do not reopen them without being asked.

- GCC 16, `-std=c++26 -freflection`. No older language mode, no fallback
  without reflection, Clang not supported.
- Namespace `rtt3`, separate header path, library and package names, so it
  installs next to RTT 2.x. No classic `TaskContext` API or wrapper.
- No macros in the API a component author writes against. No Boost.
- No exceptions thrown by `rtt3`; fallible calls return `std::expected`.
- On real-time paths: no allocation, no lock, no formatting, no file I/O,
  and no system call other than the thread's periodic sleep.
- Linux only, x86-64 and AArch64.
- No framework overhead where none is required. As a design goal, not a
  hard requirement, a link between two nodes in the same thread should
  cost what an inlined function call costs. Framework glue on that path
  has no type erasure, virtual calls or function pointers.
- Numerics: performance first. The simulator and the target agree within
  a tolerance, not bit for bit. Do not build blocks with
  `-ffinite-math-only` (and so not with plain `-ffast-math`): it removes
  NaN and infinity checks.
- Components, port data types and algorithm blocks are plain structs and
  plain functions with no framework dependency in their signatures, so
  they can be shared with the simulator.

## What could not be verified, and should not be taken as fact

- **GCC 16 and C++26 reflection.** No GCC 16 was available where the
  plans were written. Statements about what it supports come from release
  notes and secondary sources. Stage S1 of the `rtt3` plan is a spike
  on the real compiler.
- **OCL** (deployer, task browser, real-time logging), the downstream
  components, the simulator and its block wrapper, and the deployer
  extension for UDP port substitution are not in this repository.
  Descriptions of them in the plans come from the owner's statements or
  from general knowledge.
- **spdlog**: the queue implementation was checked in its source; the
  statement that message text is formatted in the calling thread was not.
- **MuJoCo**: described from general knowledge only.
- **Performance**: no benchmark has been run. Every expected gain and
  every latency target in the plans is an estimate.

## Practical notes

- Use British spelling in the plan documents, to match what is there.
- Keep file and line references in `doc/optimisation-plan.md` valid: it
  states the commit they refer to. If you change a cited file, update the
  reference or the stated commit.
- The upstream manuals under `doc/` describe RTT 2.x as upstream ships it
  and are not maintained here.
