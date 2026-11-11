# rtt3 logging: plan for a lock-free logger

Status: proposal. This expands the "Logging from real-time threads" part
of `modernisation-plan.md` (section 4.9) into its own work plan. It assumes
the decisions recorded there: GCC 16, C++26 with reflection, Linux on
x86-64 and AArch64, no macros in user-facing API, no exceptions from
`rtt3`, no allocation or system calls on real-time paths.

## 1. Goals

1. A real-time thread can log at any time with bounded, small cost: no
   lock, no allocation, no system call, no formatting.
2. A filtered-out message costs one atomic load and a branch.
3. No thread that logs can be delayed by another thread that logs, or by a
   slow console, disk or network.
4. Message loss is possible under overload, but it is always counted and
   reported, never silent.
5. One logging interface for real-time and non-real-time code.
6. Format strings and arguments are checked at compile time.
7. Port data types can be logged as values without hand-written formatters.

Non-goals:

- Guaranteed delivery of every message. A logger that never drops must be
  able to block the caller, which conflicts with goal 1.
- Compatibility with the `RTT::Logger` or log4cpp interfaces.
- Capturing output of third-party libraries that write to standard output.
- Log shipping, rotation policy and retention beyond a basic file sink;
  those belong to the system the files are handed to.

## 2. Where this stands relative to what exists

| | `RTT::Logger` | OCL real-time logging (from knowledge of OCL, to be verified) | This design |
|---|---|---|---|
| Formatting | Caller, under a global mutex | Caller, into a real-time string | Drain thread |
| Allocation per message | Stream buffers | One from the TLSF pool, under the pool mutex | None |
| Shared state between logging threads | Global mutex, held during output | Pool mutex; port buffer | None: one ring per thread |
| I/O | Caller | Appender components | Drain thread |
| Cost when filtered | Lock per `<<` | Level check | One atomic load |
| Overflow behaviour | Blocks | Buffer policy of the port | Drop and count |

**Build or adopt.** Existing asynchronous loggers with per-thread queues
and deferred formatting exist (Quill, NanoLog, fmtlog, binlog). The plan
is to build in-tree, for four reasons: their front ends are macro-based,
which the fork has ruled out; they format with the `fmt` library rather
than `std::format`; logging port data through reflection needs to be part
of the encoder; and the logger has to live under the same allocation
guard, system call filter and inspection service as the rest of `rtt3`.
Their designs are the reference for the queue and record layout, and they
serve as comparison points in the benchmarks. This choice is open
question 1.

## 3. Design

### 3.1 Front end

```cpp
namespace log = rtt3::log;

log::info("configured with {} axes", n);
log::warn("axis {} over limit: {:.3f} rad", axis, position);
log::error("state at fault: {}", state);            // a port data struct
log::every(1s).warn("tracking error {:.4f}", err);  // at most once per second
log::once().info("first sample received");
```

- One function template per level. The first parameter is a
  compile-time-checked format string, so a mismatch between placeholders
  and arguments is a compile error, as with `std::format`.
- The category is implicit: each node has a category named after it, and
  calls made from a node's hooks use it. Code outside a node passes a
  category object explicitly (`log::info(cat, "...")`).
- A compile-time minimum level removes calls below it entirely.
- Because there are no macros, arguments are evaluated before the level
  check. For an argument that is costly to compute, a callable form runs
  only when the level passes:
  `log::debug([&] { return log::args("residual {}", solve()); });`

**What the call does in the calling thread:**

1. Load the category's level (relaxed atomic) and return if filtered.
2. Reserve space in the calling thread's ring; on failure, increment that
   ring's drop counter and return.
3. Write the record header and the encoded arguments.
4. Publish with one atomic store.

### 3.2 Loggable arguments

A `rtt3::log::loggable` concept decides what may be passed:

| Kind | How it is recorded |
|---|---|
| Arithmetic types, enums, `bool`, pointers | Raw bytes |
| String literals | The pointer; the text stays in read-only storage |
| `std::string_view`, C strings, bounded strings | Length prefix and bytes, truncated at a per-call-site maximum (default 128), with truncation marked in the output |
| `std::chrono` durations and time points | Raw count |
| `rtt3::trivial_data` types (port data, section 4.4 of the main plan) | Raw bytes of the object |
| `rtt3::bounded_data` types | Member-wise by reflection, bounded |
| Anything else, including `std::string` and allocating containers | Rejected at compile time, with a message suggesting a view or a bounded copy |

Port data is rendered on the drain thread by a formatter generated through
reflection (member names and values, enums by enumerator name). A type can
supply its own `std::formatter` specialisation, which then takes
precedence.

### 3.3 Call-site description

Everything about a call that does not change between invocations is a
constant in read-only storage: level, format string, source location and a
pointer to a decode function instantiated for the argument types. The
record carries only a reference to it. The decode function knows how to
read the arguments back out of the record and format them, so the record
needs no per-argument type tags.

How the per-call-site constant is created without a macro (a unique
closure type per call site, or a string template parameter) is settled by
a spike in step L1, since it depends on what GCC 16 handles well.

### 3.4 Record layout

```
+0   call-site reference   8 bytes
+8   timestamp, ns         8 bytes   (monotonic clock)
+16  record size           4 bytes
+20  flags                 4 bytes   (truncated, callable form, ...)
+24  encoded arguments     variable, padded to 8 bytes
```

The thread identity is a property of the ring, not of each record. The
explicit size lets the drain thread and the recovery tool skip a record
without calling its decoder.

### 3.5 Per-thread ring

- A single-producer single-consumer byte ring with power-of-two capacity.
  The producer is wait-free: reserve, write, publish.
- Producer-owned and consumer-owned indices are on separate cache lines;
  each side keeps a cached copy of the other's index and refreshes it only
  when it appears to have run out.
- A record never wraps. If it does not fit before the end of the buffer, a
  padding record fills the remainder and the record starts at the
  beginning.
- **Real-time threads** get their ring during configuration, from the
  configuration arena, with a capacity set per thread in the machine file
  (default 64 kB). **Other threads** get one on first use from the heap.
  A real-time thread without a ring is a configuration error caught by
  the allocation guard in tests.
- All rings sit in one contiguous, locked memory region with a small
  directory at its start (ring count, offsets, capacities, thread names,
  build identifier). That is what makes crash recovery and external
  inspection possible.

### 3.6 Overflow and flight recorder

Two modes, selectable per ring:

- **Drop newest** (default). A record that does not fit is discarded and
  counted. The drain thread emits a line such as
  `[ctrl] 42 messages dropped` at the point in that thread's sequence
  where the gap is.
- **Flight recorder.** The ring is never drained in normal operation and
  the producer overwrites the oldest records. Its contents are written out
  on request through the inspection service, or when a node enters its
  fault state. This gives detailed history around a fault at no steady
  I/O cost. Overwriting while a reader copies is handled by the reader
  copying the region and validating it against the producer's position
  before and after.

### 3.7 Drain thread

- One non-real-time thread on a non-isolated core. It polls on its own
  timer (default 10 ms), so no logging thread ever makes a wake-up call.
- Each pass: note the current time, read every ring up to its published
  position, merge records by timestamp, format, hand to the sinks.
- **Ordering.** Within one thread, order is exact. Across threads, order
  is by timestamp; records newer than the time noted at the start of the
  pass are held back until the next pass, which bounds reordering to
  records whose producer was preempted between taking the timestamp and
  publishing.
- Formatting uses `std::format` through the call site's decode function.
  The monotonic timestamp is converted to wall-clock time using an offset
  the drain thread samples periodically.
- If a sink is slow, the drain thread falls behind, rings fill and
  producers drop. The slowness is therefore visible as drop counts and
  never as delay in a logging thread.

### 3.8 Sinks

A sink receives a decoded record (timestamp, level, category, thread,
source location, formatted text, and access to the typed fields).

| Sink | Notes |
|---|---|
| Console | Non-blocking writes; drops and counts when the terminal cannot keep up |
| File | Buffered, size-based rotation, flush interval configurable |
| Structured file | One JSON object per line, fields from the typed arguments |
| Inspection service | Live tail for the inspection tools, with level and category filters applied on the helper side |
| journald | Optional, where the target runs systemd |

Sinks, their levels and the ring sizes are set in the machine file.

### 3.9 Levels and control

- Levels: trace, debug, info, warn, error, fatal. `fatal` records the
  message and requests the node's fault transition; it does not terminate
  the process by itself.
- Per-category levels are atomics. The inspection service can read and
  change them while the system runs, and can read every ring's drop
  counter and high-water mark.
- A compile-time minimum level is a build setting.

### 3.10 Crashes and recovery

Records that the drain thread has not yet processed would be lost if the
process died. Three measures, each optional:

1. On a fatal signal, a handler makes a best-effort pass over the ring
   region and writes the raw region to a file descriptor opened at
   start-up. It does no formatting and no allocation.
2. The ring region can be a shared memory mapping, so it survives the
   process and a supervisor can collect it.
3. `app --decode-log <file>` decodes a raw region using the same
   executable, which holds the call-site constants and decode functions.
   The directory's build identifier is checked first.

### 3.11 What the library itself logs

`rtt3` uses the same logger for its own diagnostics, which makes the
rules apply to it too: the start-up report, lifecycle transitions,
connection events and drop reports go through categories under `rtt3.*`.

## 4. Enforcement and verification

- **Compile time**: format checking; the `loggable` concept; a
  reflection-based check that producer-owned and consumer-owned fields of
  the ring do not share a cache line.
- **Tests**:
  - ring unit tests for wrap, padding, exact-fit, full and drop cases;
  - round-trip tests: every loggable kind encodes and decodes to the same
    text as direct `std::format`;
  - stress tests with many producer threads under TSan, on x86-64 and on
    ARM hardware;
  - model checking of the reserve and publish protocol and of the
    flight-recorder read;
  - logging from a real-time thread under the allocation guard and the
    system call filter, at enabled and at filtered levels;
  - a slow-sink test showing that producers are not delayed and that
    drops are reported;
  - a crash test: kill the process with a fatal signal, decode the dumped
    region, find the last records.
- **Benchmarks**, on the desktop and on the ARM target, tracked in CI:
  cost of a filtered call; cost of a recorded call with zero, four
  integer, and one 64-byte struct argument; worst case over a long run
  under load; drain throughput. `RTT::Logger`, and one existing
  asynchronous logger as an outside reference, are measured alongside.

Targets, to be confirmed or revised by the first measurements: a filtered
call within a few nanoseconds, a recorded call with a handful of scalar
arguments within a few tens of nanoseconds on the desktop, and no
allocation or system call in either.

## 5. Work plan

| Step | Content | Exit criterion |
|---|---|---|
| L0 | Benchmarks of `RTT::Logger` and one outside logger on both architectures | Baseline recorded |
| L1 | Spike on GCC 16: call-site constant without a macro, compile-time format checking, decode function per call site, reflection formatter for a struct | The four techniques work, or the design in 3.1 to 3.3 is revised |
| L2 | Ring, record layout, encoder and decoder, drain thread with console sink; level filtering | Round-trip, ring and stress tests pass; alloc guard and syscall filter pass |
| L3 | Per-thread ring registration tied to node threads, categories per node, machine-file configuration, drop reporting, `every` and `once` | Example deployment logs from its real-time threads with zero allocations and system calls |
| L4 | File, structured and journald sinks; wall-clock conversion; merge ordering with hold-back | Slow-sink test passes; ordering test passes |
| L5 | Flight-recorder mode; fatal-signal dump; shared mapping; `--decode-log` | Crash test passes |
| L6 | Inspection service integration: live tail, level control, counters | Levels changed and logs followed from a separate process |

L0 to L2 belong to phase 1 of the main plan, since every later phase uses
the logger. L3 needs nodes and threads (phase 3) and the machine file
(phase 4). L6 needs the runtime layer (phase 6). L4 and L5 can be done at
any point after L2.

## 6. Risks

- **Call-site constants without macros** may hit compiler limits or
  produce poor diagnostics. Mitigation: the L1 spike; the fallback is a
  string template parameter form, `log::warn<"...">(args)`, which is less
  pleasant to write but uses only well-established features.
- **Argument evaluation when filtered** is a real cost for careless call
  sites. Mitigation: the callable form, documentation, and a benchmark
  that makes the difference visible.
- **Compile time**: each call site instantiates a decode function.
  Mitigation: measure in L1; share decode functions between call sites
  with identical argument types.
- **Timestamp cost** depends on the clock source. Mitigation: the
  start-up check on the clock source in the main plan; optional use of
  the node's cycle start time.
- **Loss under overload** will surprise users of a blocking logger.
  Mitigation: drop lines in the output, counters in the inspection
  service, and ring sizes that are easy to raise per thread.
- **Recovery depends on the exact executable.** A dumped region cannot be
  decoded with a different build. Mitigation: the build identifier check,
  and keeping executables of released versions.

## 7. Open questions

1. **Build or adopt**: is building in-tree acceptable, or would you prefer
   to wrap an existing asynchronous logger and accept its macros and its
   formatting library?
2. **Existing logging**: do your components log through `RTT::Logger`,
   through OCL's real-time logging, or both? Is any tooling built on the
   current log format or on log4cpp configuration files?
3. **Drop or block for non-real-time threads**: should threads that are
   not real-time also drop when their ring is full (as planned, one rule
   for all), or wait for the drain thread?
4. **Flight recorder**: is history around a fault valuable enough on your
   machines to make it part of the first version?
5. **Crash recovery**: is the fatal-signal dump enough, or is the shared
   mapping with an external collector needed?
6. **Sinks**: which destinations matter on the target (file, journald,
   network), and is structured output wanted?
7. **Call syntax**: if the natural form `log::warn("...", args)` proves
   unreliable on GCC 16, is `log::warn<"...">(args)` acceptable?
