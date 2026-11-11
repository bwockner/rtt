# rtt3: OS layer for Linux on x86-64 and ARM

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S2.

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

- Supported architectures: x86-64 and AArch64. 32-bit ARMv7 is question
  M5, because it changes the atomics design.
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
