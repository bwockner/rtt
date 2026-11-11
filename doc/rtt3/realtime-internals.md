# rtt3: lock-free and real-time internals

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S3 for the primitives. The rules in the later sections apply to
every stage that puts code on a real-time path.

- New implementations of the data object, multi-writer buffer, queues and
  memory pool on `std::atomic`, with documented memory orders, ABA handling
  and cache-line alignment, plus single-producer single-consumer variants.
- Verification: TSan in CI, stress tests on both x86-64 and ARM hardware,
  and model checking of the algorithms (for example with Relacy).
- Execution engine: a fixed-capacity queue of non-allocating callables.
- Rule for all code reachable from `update()`: no allocation, no blocking
  without priority inheritance, no exceptions, no `std::string` construction.

## Cache behaviour

A cache line is invalidated on one core whenever another core writes to
it. In RTT 2.x this happens on almost every port operation: readers
increment a counter inside the data object the writer is filling, every
hop copies a reference-counted pointer, and unrelated fields share lines.
The strategies below are ordered by how much they remove.

**1. Do not share at all.** When the deployment folds two nodes into one
thread (`deployment.md`), the link between them is a local value in the generated
step function, passed like a function argument, with no atomics and
normally no memory write (see "Same-thread links compile to function
calls").
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
aligned to the project's cache-line constant (`os-layer.md`). Across the deployment,
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
buffer reads in place. Member projection (`components-and-ports.md`) sends only the part that is
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

## Context switches

In a typical RTT 2.x system every component has its own thread, so one
control cycle through sensor, controller and driver costs three wake-ups.
Each sample written to an event port wakes the receiving thread, and an
own-thread operation call blocks the caller, wakes the callee and wakes
the caller again. The strategies below remove switches in that order of
importance.

**1. Fewer threads.** Nodes are folded into one thread when they share a
period and priority (`deployment.md`). Folding is extended to harmonic rates: a node
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
outside the library but checked by the start-up report (`os-layer.md`):
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

## Memory allocation

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
- At compile time, the port data concepts (`components-and-ports.md`) reject allocating types
  on ports that have not opted in.
- In test builds, an allocation guard replaces the global allocation
  functions and aborts with a backtrace if one is called from a thread
  marked real-time after start. The whole test suite and the example
  deployments run with the guard on.
- At run time, the start-up report states static, arena and locked sizes,
  and per-thread page-fault counts are exposed through the inspection
  service with an expected value of zero after start.

## System calls at run time

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

## Priority inversion through file I/O

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

## Logging from real-time threads

**What exists today.** `RTT::Logger` formats under a global mutex and
writes while holding it (see `../optimisation-plan.md`, D3 and D6). OCL adds a
separate real-time logging facility (`OCL::logging`): a category formats
the message in the calling thread into a real-time string, sends it as an
event through a buffered port, and appender components on non-real-time
threads write it out. That keeps I/O out of the real-time thread, but it
is not lock-free and not allocation-free: the string is allocated from the
TLSF pool on every message, and that pool is guarded by one mutex
(`rtt/os/tlsf/tlsf.c:70`, `rtt/os/tlsf/target.h:44-45`), created without
priority inheritance. Formatting cost is also paid in the real-time
thread. The description of OCL here is from knowledge of that project, not
from reading its source in this session, and should be checked against the
version in use.

The full work plan for the logger, with record layout, sinks, crash
recovery, tests and steps, is in `logging.md`. In summary:

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

## SIMD and floating point

The notes on alignment, vectorisation, per-thread floating-point state,
compiler options and comparing results with the simulator are kept in
their own document, `numerics.md`. In summary:

- Alignment of port data is preserved through channels, pools and rings,
  including over-aligned vector types.
- Blocks are built for speed. The simulator and the target agree within a
  tolerance, not bit for bit.
- `-ffinite-math-only`, and so plain `-ffast-math`, is excluded because it
  removes NaN and infinity checks.
- Outputs can be marked `rtt3::finite` for a check that no floating-point
  option can remove; it is on by default for links into actuators.
- Real-time threads set their floating-point state explicitly at start
  and flush subnormals to zero by default.
- Eigen types are supported as port data through an optional adaptor.
  The alignment of Eigen's default fixed-size types depends on compiler
  flags, so types that cross a boundary use its unaligned form.
