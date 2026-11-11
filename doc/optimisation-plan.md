# RTT 2.x: low-risk optimisations with no API change

Status: proposal. This is separate from `modernisation-plan.md` and makes
no assumption that the `rtt3` work happens. Everything here applies to the
existing code base, as used on Linux (standard and PREEMPT_RT kernels) on
x86-64 and ARM.

## 1. Ground rules

- **No source API change.** Component code compiles without edits. No
  public class, function or macro is renamed, removed or given a different
  signature, and default behaviour stays the same unless an item says
  otherwise and puts the change behind an option.
- **ABI is treated separately.** Many hot paths are templates in headers,
  so fixing them means components must be recompiled, though not edited.
  Each item states its ABI effect:
  - *none*: only `.cpp` files or build flags change;
  - *recompile*: header templates or inline functions change;
  - *layout*: the size or layout of an exported class changes, so RTT and
    all components must be rebuilt together.
- **Nothing here has been measured.** The findings come from reading the
  code, and the expected effect of each item is an estimate. Step 0 builds
  the benchmarks; an item that shows no measurable gain is dropped.

File and line references are against `master` at `18a6e04c7`, that is,
after `toolchain-2.10` was merged into it.

## 2. Step 0: measure first

A small benchmark suite, kept in `tests/`, run on the desktop and on the
ARM target:

| Benchmark | What it shows |
|---|---|
| Port write then read, one connection, data and buffer policies, small (8 byte) and large (4 kB) samples | Per-sample cost of the data-flow path |
| Port write with 1, 2 and 4 connections | Fan-out cost |
| Event port write to callback execution | Signalling and engine cost |
| `OperationCaller` call and send, own-thread and client-thread | Engine message cost |
| `log(Debug)` with output level at Info, and `Logger::In` scope entry | Cost of logging that produces no output |
| Periodic activity wake-up latency under load, with and without a low-priority thread contending on an RTT mutex | Determinism, priority inversion |

Use `perf` for the cycle counts and a `cyclictest`-style histogram for the
latency case. Record the baseline before changing anything.

## 3. Findings and proposed changes

### 3.1 Determinism (highest value on PREEMPT_RT)

**D1. Mutexes are created without priority inheritance.**
`rtos_mutex_init` calls `pthread_mutex_init(m, 0)` and
`rtos_mutex_rec_init` sets only the recursive type
(`rtt/os/gnulinux/fosi.h:206` and `:216`). Every `os::Mutex` and
`os::MutexRecursive` in RTT and in components is therefore open to
unbounded priority inversion: a low-priority thread holding the lock can be
preempted by a medium-priority thread while a high-priority thread waits.

- Change: set `PTHREAD_PRIO_INHERIT` on both mutex kinds.
- This alters scheduling behaviour, so it goes behind a CMake option,
  proposed default on for the gnulinux target. Uncontended lock and unlock
  stay in user space; contended ones cost somewhat more.
- ABI: none.

**D2. Timed mutex locks use the wall clock.**
`rtos_mutex_trylock_for` and `rtos_mutex_lock_until` convert to
`CLOCK_REALTIME` because `pthread_mutex_timedlock` requires it
(`fosi.h:247-285`, as the comments there note). A clock step from NTP or
an administrator shortens or lengthens the timeout.

- Change: use `pthread_mutex_clocklock` with `CLOCK_MONOTONIC` where glibc
  provides it (2.30 and later), keeping the current code as the fallback.
- ABI: none.

**D3. The logger takes a global lock even when nothing is logged.**
Three separate problems, all on paths that real-time code commonly touches:

- `Logger::operator<<` checks only whether the logger is started, then
  locks the global `inpguard` mutex, and only then checks the level
  (`rtt/Logger.inl:53-60`). A statement such as
  `log(Debug) << "x=" << x << endlog();` with the level at Info takes the
  global mutex once per `<<`.
- `Logger::In` takes the same mutex three times per scope and copies two
  `std::string`s: `getLogModule()` in the constructor, then `in()`, then
  `out()` in the destructor (`rtt/Logger.cpp:403-438`).
- `logit` builds the time, level and module prefix through string streams
  before it checks whether the line will be written (`Logger.cpp:183-187`).

Combined with D1 this lock is a priority-inversion point shared by every
thread in the process.

- Change: test the level before taking the lock in `operator<<`; build the
  prefix in `logit` only after the level test; keep the current module name
  per thread (thread-local pointer) so that `Logger::In` needs no lock. The
  module name is currently one global shared by all threads, so per-thread
  storage also fixes module names being attributed to the wrong thread.
- ABI: recompile (`operator<<` is a template in `Logger.inl`). `Logger`
  uses a private implementation pointer, so its layout does not change.

**D4. Memory is not locked or prepared on the Linux target.**
`mlockall` is called for the Xenomai and RTAI targets but nowhere for
gnulinux (the only matches are `rtt/os/xenomai/fosi_internal.cpp:82` and
`rtt/os/lxrt/fosi_internal.cpp:75`), and the allocator is left at its
defaults. Real-time threads can therefore take page faults on first touch
of heap or stack, and any thread freeing a large block can cause memory
to be unmapped, which interrupts all cores running the process.

- Change: at main-thread start-up on gnulinux, behind an option and an
  environment switch, call `mlockall(MCL_CURRENT | MCL_FUTURE)`, set
  `M_TRIM_THRESHOLD` and `M_MMAP_MAX` so the heap neither shrinks nor maps
  separate regions, and prefault new thread stacks. Failure for lack of
  privilege is logged and ignored, so desktop testing is unaffected.
- ABI: none.

**D5. The real-time memory pool can grow at run time.**
When TLSF is enabled, the options that let the pool grow through `sbrk`
and `mmap` are on by default (`rtt/CMakeLists.txt:117-118`), so an
allocation from a real-time thread can end in a system call and page
faults. `OperationCaller::send` clones the caller through the real-time
allocator on every call (`rtt/internal/LocalOperationCaller.hpp:706-710`).
That allocator is this pool when TLSF is enabled and plain `malloc` when
it is not (`rtt/os/oro_malloc.h:54`).

- Change: no code change; size the pool for the application at start-up,
  build with pool growth off for deployment, and use the existing pool
  statistics to find the right size. Document this.
- ABI: none.

**D6. The logger holds its global lock while writing to the console and
to the log file.** `logit` takes `inpguard` and, still holding it, writes
the line to standard output and to the file sink
(`rtt/Logger.cpp:183-200`). Every `log() << ...` from any thread takes the
same lock (D3). If a low-priority thread is in the middle of that write
and the console or disk is slow, a real-time thread that logs anything, or
enters a `Logger::In` scope, waits for the device. Priority inheritance
(D1) does not help, because the holder is waiting for I/O, not for the
CPU.

- Change: format the line under the lock into a local buffer, release the
  lock, then write. Per-line ordering between threads can then differ
  slightly from lock order; lines stay whole. Together with the level
  check before locking (D3), a real-time thread that logs nothing never
  touches the lock at all.
- A further step, still without API change, is to hand finished lines to a
  writer thread through the lock-free buffer the logger already has for
  remote logging (`Logger.cpp:223`), so that no caller ever performs the
  write. This changes when output appears and what is lost on a crash, so
  it goes behind an option.
- ABI: none.

**D7. Lazy symbol binding.** Nothing in the build requests immediate
binding, so the first call to each library function from a real-time
thread resolves the symbol under the dynamic loader's lock and may fault
in pages from disk.

- Change: link RTT and, through `UseOROCOS-RTT.cmake`, components and
  executables with `-Wl,-z,now`. Start-up becomes slightly slower.
- ABI: none.

**D8. The real-time allocator is guarded by one mutex.** TLSF is built
with locking on (`rtt/os/tlsf/tlsf.c:70`) and the lock is an ordinary RTT
mutex (`rtt/os/tlsf/target.h:44-45`), so it has no priority inheritance
until D1 is applied. Everything that uses the real-time allocator from
several threads contends on it: `rt_string`, `OperationCaller::send` (D5),
and OCL's real-time logging, which allocates a string per message.

- Change: none beyond D1, which gives this lock priority inheritance.
  Hold times are short and bounded, so with D1 the remaining cost is
  contention, not inversion. Per-thread pools would remove the contention
  but change how memory is sized and are not an easy win.
- ABI: none.

### 3.2 Data-flow path (highest value for throughput)

A write followed by a read passes through a chain of channel elements.
Each hop currently pays for the following.

**F1. Neighbour lookup is an out-of-line, locked, reference-counted call.**
`ChannelElementBase::getOutput()` and `getInput()` are defined in the
library, take a shared lock and return a smart pointer by value
(`rtt/base/ChannelInterface.cpp:57-67`). The reference count functions are
also out-of-line exported functions (`ChannelInterface.cpp:472-476`). Per
hop that is three non-inlined calls and four atomic read-modify-write
operations (lock, add reference, unlock, release), on both the write and
the read side (`rtt/base/ChannelElement.hpp:91-105`).

- Change, step 1: define `getOutput`, `getInput`, `intrusive_ptr_add_ref`
  and `intrusive_ptr_release` inline in the header, keeping the exported
  definitions for existing binaries. No semantic change.
- Change, step 2 (measure first): for elements that can only ever have one
  input and one output, replace the shared lock and reference count on the
  hot path with a single atomic pointer load, keeping the lock for
  connect and disconnect. This needs a careful argument about element
  lifetime during disconnection and is the one item in this section that
  is not mechanical.
- ABI: recompile for step 1; layout for step 2.

**F2. A `dynamic_cast` on every hop.**
`ChannelElement<T>::getOutput()` and `getInput()` call `narrow<T>`, which
is a `dynamic_cast` (`ChannelElement.hpp:66-74`,
`ChannelElementBase.hpp:101-112`). The fan-out and fan-in elements do the
same for every connection on every write and read
(`ChannelElement.hpp:271`, `:307`, `:230`). A `static_cast` is not possible
because `ChannelElementBase` is a virtual base
(`ChannelElement.hpp:58`). The type was already checked when the
connection was made.

- Change: cache the typed pointer when a neighbour is attached and use the
  cached pointer on the read and write paths. For the fan-out element this
  is a typed vector kept in step with the existing output list.
- ABI: recompile (new members in class templates only).

**F3. A tracing hook is called on every write even when tracing is compiled out.**
`OutputPort<T>::write` calls `traceWrite()` (`rtt/OutputPort.hpp:255`).
It is an out-of-line library function whose body is empty unless LTTng
support is built in (`rtt/base/OutputPortInterface.cpp:117-120`,
`rtt/os/traces.h:9-10`). The typed `InputPort<T>::read` has no such call.

- Change: make it an inline no-op when LTTng is not enabled.
- ABI: recompile.

**F4. `connected()` is checked through two virtual calls before each write.**
`OutputPort<T>::write` calls `connected()`, which is out of line and
forwards to `getEndpoint()->connected()`, and then calls
`getEndpoint()->getWriteEndpoint()->write()`, which reports
`NotConnected` itself (`OutputPort.hpp:250-258`,
`OutputPortInterface.cpp:61-64`).

- Change: call `write` on the endpoint directly and use its result; fetch
  the endpoint once.
- ABI: recompile.

**F5. False sharing in the lock-free data object.**
`DataObjectLockFree<T>::DataBuf` holds the sample, the status and the two
counters with no alignment, in one array (`rtt/base/DataObjectLockFree.hpp:108-120`,
`:147`). For small samples, a reader updating its buffer's `read_counter`
invalidates the cache line the writer is filling in the neighbouring
buffer. The lock-free buffer and its pool deserve the same check.

- Change: align `DataBuf` to the cache line size. Memory use per
  connection rises for small sample types.
- ABI: recompile.

**F6. Fan-in read takes a shared lock even for a single input.**
`MultipleInputsChannelElement<T>::read` locks `inputs_lock` and goes
through the channel selection logic on every read
(`ChannelElement.hpp:161-169`).

- Change: fast path when there is exactly one input. Lower priority; do it
  only if the benchmarks show it after F1 and F2.
- ABI: recompile.

### 3.3 Execution engine

**E1. A `dynamic_cast` and a map lookup per port event.**
`processPortCallbacks` casts `taskc` to `TaskContext*` every time the
queue is non-empty (`rtt/ExecutionEngine.cpp:238`), and
`TaskContext::dataOnPortCallback` looks the port up in a
`std::map<PortInterface*, SlotFunction>` for each event
(`rtt/TaskContext.cpp:450-454`).

- Change: cache the cast result when the engine is created. Small gain;
  replacing the map is not worth doing on its own.
- ABI: layout (a new member in an exported class), so do it only together
  with another change that already forces a full rebuild.

**E2. Legacy atomic builtins.**
On GCC the atomic operations map to the `__sync_*` builtins
(`rtt/os/oro_gcc/oro_arch.h:63-152`), which are always sequentially
consistent. On x86-64 this costs nothing extra. On AArch64 they are full
barriers, and without ARMv8.1 atomics they become load/store-exclusive
loops.

- Change, build only: compile for the actual target CPU on ARM
  (`-mcpu=` for the board) so the compiler emits the single-instruction
  atomics where the CPU has them.
- Change, code: move reference counts to `__atomic_*` with relaxed
  increment and acquire-release decrement. Leave the ordering of the
  lock-free algorithms alone; weakening those needs proof, not a quick win.
- ABI: none for the build flag; recompile for the code change.

### 3.4 Build configuration (no code change)

**B1. Stop routing internal calls through the dynamic symbol table.**
The library is built with default symbol visibility and no
`-fvisibility` flag anywhere in the build, so calls between functions
inside `liborocos-rtt` go through the PLT and cannot be inlined across
translation units. Add `-fno-semantic-interposition` and link with
`-Wl,-Bsymbolic-functions`. This keeps every symbol exported, so nothing
that components link against disappears. It does change behaviour for
anyone who overrides RTT functions with `LD_PRELOAD`.

**B2. Link-time optimisation for the RTT library itself**
(`CMAKE_INTERPROCEDURAL_OPTIMIZATION`), behind an option. Lets the
out-of-line helpers in 3.2 be inlined within the library even before the
header changes land.

**B3. Target CPU flags on ARM**, as in E2, set in the cross toolchain file
rather than in RTT's CMake files.

**B4. Confirm the build type in use.** The default is `RelWithDebInfo`
(`orocos-rtt.default.cmake:40`), which is appropriate. Check that the
deployed build is not `Debug`, and that LTTng tracing is not enabled in
production builds unless it is wanted (see F3).

Switching to `-fvisibility=hidden` is deliberately **not** proposed: RTT
relies on `dynamic_cast` and type information across shared library
boundaries, and getting the export annotations complete is not a quick or
safe change.

### 3.5 Usage-level wins in component code

These need no change to RTT at all, only a review of the components:

- **Output ports copy every sample twice by default.** `OutputPort`'s
  `keep_last_written_value` constructor argument defaults to true
  (`OutputPort.hpp:141`), so each `write` stores the sample in the port and
  then again in each connection (`OutputPort.hpp:245-256`). For large
  samples on ports where late-connecting readers do not need the last
  value, pass `false`.
- **Avoid `Logger::In` and `log()` in `updateHook`** until D3 is in, and
  prefer checking the level once outside loops afterwards.
- **Call `setDataSample`** before connecting ports that carry dynamically
  sized types, so no allocation happens on first write.
- **Review connection policies**: a lock-free data connection suits a
  single latest value; unbuffered where possible avoids pool traffic.

Context switches, in particular:

- **One thread per component is the default cost.** Tightly coupled
  components can share a thread today by giving the followers a
  `SlaveActivity` and executing them from the leader's `updateHook`, in
  data-flow order. A chain of three components then costs one wake-up per
  cycle instead of three.
- **Event ports wake the receiver for every sample.** Where the receiving
  component is periodic anyway, a plain port read in `updateHook` avoids
  the extra wake-ups.
- **Own-thread operations cost two switches per call** (caller blocks,
  callee runs, caller resumes). Operations that are safe to run in the
  caller's thread can be declared `ClientThread`; otherwise prefer `send`
  and collect the result on a later cycle to a blocking `call` from a
  real-time thread.
- **Thread placement**: pin real-time activities to isolated cores using
  the existing CPU affinity argument of `Activity`, and avoid equal
  priorities on one core.

System calls and allocation, in particular:

- **Find them first.** Run the deployed system under `strace -f -c` or
  `perf trace -s` for a minute and look at the per-thread counts for the
  real-time threads. Anything beyond the periodic sleep and the wake-ups
  between components is a candidate. `perf stat -e page-faults` per thread
  should read zero after start-up.
- **Reserve capacity in `configureHook`**: size vectors and strings there
  and call `setDataSample`, so that `updateHook` never grows a container.
- **Keep I/O out of `updateHook`**: file and console output, including
  logging at a level that is actually written, belongs in a non-real-time
  component fed through a buffered port.
- **`send` allocates per call** (see D5). In a hot loop, keep and reuse
  the `SendHandle` pattern sparingly, or use a port instead of an
  operation for high-rate data.

File I/O and priority inversion, in particular:

- **No file or console output from a real-time component.** Until D6 is
  in, that includes logging at any level that is written. Route data to be
  stored through a buffered port to a non-real-time component that does
  the writing.
- **Reporting and marshalling components** write files in their own
  thread; give them a non-real-time activity and connect them with
  buffered, lock-free connections, never with a locked policy.
- **Do not share a mutex** between a real-time component and anything
  that performs I/O, including through an operation that the I/O
  component calls while holding its own lock.
- **Avoid a serial console** as the destination of standard output on the
  target; redirect it to a file or to memory.

Logging from real-time components, in particular:

- **Prefer OCL's real-time logging to `RTT::Logger` in real-time code.**
  As far as I know from the OCL project (not verified against its source
  here), `OCL::logging` formats into a real-time string in the caller and
  passes it through a buffered port to appender components that write on
  non-real-time threads. That removes the global logger lock and the I/O
  from the real-time thread (D3, D6).
- It is not lock-free or allocation-free: each message allocates from the
  TLSF pool under the pool's mutex (D8), and formatting runs in the
  real-time thread. So: apply D1, size the pool for the worst logging
  burst with growth off (D5), look up categories once in `configureHook`,
  check the level before building a message, and make sure the port to
  the appenders uses a lock-free buffered connection of adequate size.

- **spdlog in real-time components** (used downstream): use asynchronous
  loggers only, with the overflow policy set to overwrite or discard
  rather than the default, which blocks the caller when the queue is
  full. Look loggers up once in `configureHook`. Note that the
  asynchronous queue is guarded by a `std::mutex` without priority
  inheritance and that the message text is formatted in the caller, so
  this is suitable for soft real-time use, not a lock-free path. See
  `rtt3-logging-plan.md` for the comparison.

### 3.6 Existing branches in this repository that cover some items

Several of the findings above already have work on unmerged branches,
inherited from upstream pull requests that were never merged. None of
them is on `master`. They are starting points to review, not finished
fixes.

| Item | Branch | State |
|---|---|---|
| D1 | `enable-priority-interitance-for-mutexes-with-gnulinux` | One commit setting `PTHREAD_PRIO_INHERIT` on both mutex kinds, unconditionally. Conflicts with `master` in `rtt/os/gnulinux/fosi.h` only because neighbouring functions moved; the change itself is small |
| D3, D6 | `fix-thread-safe-logger` | One commit, described by its author as a proof of concept, moving logger state to thread-local storage. Merges cleanly; needs review against D3 and D6 |
| D8 | `master-drop-boost-thread-usage` | Includes a commit that gives TLSF a plain pthread mutex instead of the RTT timed mutex; the rest concerns the macOS target. Conflicts in `fosi.h` |
| D5, D8 | `tlsf-valgrind-support` | Valgrind annotations for TLSF plus a fix to the pool area size calculation. Merges cleanly |
| F1 | `meyerj/fix_deadlock_in_concurrent_connection_disconnection` | Finer-grained locking in `ConnectionManager` and the multiple-outputs channel element, with a test; the last commit is marked work in progress. Merges cleanly, touches 21 files |
| not listed | `master-file-descriptor-activity-locking-and-robustness` | Removes a mutex from `FileDescriptorActivity::trigger()`. Conflicts with the later rework of that class already on `master` |

SIMD and floating point, in particular:

- **Over-aligned types and the real-time allocator.** TLSF aligns blocks
  to twice the pointer size, 16 bytes on 64-bit targets
  (`rtt/os/tlsf/tlsf.c:138`). A type that needs 32-byte alignment, such as
  a fixed-size vector type built for AVX, is not guaranteed that alignment
  when allocated through `rt_allocator`. Check any such types used with
  real-time strings, containers or `send`.
- **Subnormal numbers** can make a control loop's execution time jump
  when a filter state decays towards zero. Setting flush-to-zero at the
  start of each real-time thread (in the component's `startHook` or
  `configureHook`, since it is per-thread state) removes that; it changes
  results slightly and must match the simulator.
- **Build flags** are not set by RTT; components choose their own.
  Building for speed is reasonable, with one exception: `-ffast-math`
  includes `-ffinite-math-only`, which lets the compiler delete NaN and
  infinity checks, including guards on sensor values and actuator
  commands. Prefer `-fno-math-errno -fno-trapping-math -ffp-contract=fast`
  to the blanket option, or add `-fno-finite-math-only` after it.

## 4. Order of work

| Step | Items | ABI effect | Why this order |
|---|---|---|---|
| 0 | Benchmarks and baseline | none | Everything else is judged against it |
| 1 | B1, B2, B3, B4 | none | No code change; immediately reversible |
| 2 | D1, D2, D4, D5, D6, D7 (D8 is covered by D1) | none | Largest determinism gain; library-only or configuration-only changes |
| 3 | D3 | recompile | Removes a process-wide lock from real-time paths |
| 4 | F3, F4, F1 step 1, F2 | recompile | Mechanical; together they remove most per-hop overhead |
| 5 | F5, E2 (code part) | recompile | Depends on measurements from step 4, mainly on ARM |
| 6 | F1 step 2, F6, E1 | layout | Only if steps 4 and 5 leave a measurable cost; needs concurrency review |

Steps 1 to 3 can be released as a patch release. Steps 4 and 5 require
components to be recompiled, so they belong in one release, announced as
such. Step 6 is optional.

## 5. Verification

- The existing test suite passes after every step, with TSan runs for
  steps 3 to 6.
- Steps 4 to 6 are also tested on ARM hardware, since x86 hides most
  memory-ordering errors.
- For D1, a test that reproduces priority inversion on an `os::Mutex`
  before the change and shows bounded latency after it.
- Benchmarks are re-run after each step and the numbers recorded next to
  the baseline.

## 6. Not proposed, and why

- **Replacing Boost types, removing virtual inheritance from channel
  elements, typed port lookup**: these change the API or the object model
  and belong to the `rtt3` plan.
- **Rewriting the lock-free algorithms or weakening their memory
  ordering**: possible gains on ARM, but not an easy win and easy to get
  subtly wrong.
- **Hidden symbol visibility**: see 3.4.
- **Thread loop and periodic wait**: the per-cycle cost there is an
  uncontended mutex and two clock reads through the vDSO
  (`rtt/os/Thread.cpp:137`, `rtt/os/gnulinux/fosi_internal.cpp:284-305`);
  not worth changing.
