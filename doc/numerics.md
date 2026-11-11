# Numerics: SIMD, floating point and agreement with the simulator

Status: design notes for `rtt3`, with a short section on what applies to
RTT 2.x today. This was split out of `modernisation-plan.md` (section
4.9), which keeps a summary.

Decided so far: performance comes first, and the simulator and the real
deployment are expected to agree within a tolerance, not bit for bit.
Open points are in `open-questions.md` and at the end of this document.

SIMD matters to the framework in four places: how data is laid out and
copied, how threads are set up, how blocks are compiled, and how results
are compared with the simulator. The framework does not vectorise user
algorithms; it must not get in the way of code that is vectorised, and it
must make the consequences visible.

## Alignment is preserved end to end

- Channels, buffers, data objects and log rings honour the alignment of
  the value type, including over-aligned types (32 bytes for AVX, 64 for
  AVX-512, vector types from linear-algebra libraries). Static deployment
  storage is laid out with each member's alignment, and the configuration
  arena and block pools take an alignment argument.
- `rtt3::port_data` accepts over-aligned types. A compile-time check
  rejects a type whose alignment exceeds what a given channel or pool was
  built for, instead of producing a misaligned object.
- The wire encoding for the UDP bridge and the log encoder copy bytes and
  make no alignment assumption about the destination buffer.

## Layouts that vectorise

For per-axis data the recommended form is a
struct of fixed-size arrays (`std::array<double, N>` for positions, one
for velocities, and so on), not an array of per-axis structs. The
compiler can vectorise loops over it, and it is still plain port data.
Reflection treats both forms alike. Repeated ports (`modernisation-plan.md`, section 4.4) are for wiring
convenience; a block that processes all axes at once should take one
array-valued port.

## `std::simd`

GCC 16 ships the C++26 data-parallel types as an
experimental feature. They may be used inside blocks. They are not
allowed in port data types or block signatures for now: their size and
layout depend on the target's vector width, which would make a type differ
between the x86-64 desktop and the ARM target and break the shared schema.
A block converts to and from fixed-size arrays at its boundary. This is to
be revisited once the feature is stable.

## Per-thread floating-point state

Each real-time thread sets its
floating-point control state at start, from the deployment:

- Subnormal numbers are flushed to zero by default on real-time threads
  (the flush-to-zero and denormals-are-zero bits on x86-64, the flush-to
  -zero bit on AArch64). Arithmetic on subnormals can be many times slower
  on some processors, which shows up as latency spikes when a filter or
  integrator decays towards zero. This is a numerical change, so it is a
  stated default that a node can opt out of, and the simulator has to use
  the same setting for results to match.
- The rounding mode is set explicitly and floating-point exceptions are
  masked, so behaviour does not depend on what the creating thread had.
- The settings are part of the start-up report.

## Instruction sets and builds

- The target CPU is set in the toolchain file per machine class (`modernisation-plan.md`, section 4.3),
  so the compiler can use the vector instructions the deployment hardware
  has. One binary per machine class, not run-time dispatch, is the
  default; it keeps code paths identical between test and production.
- Where one binary must serve several CPUs, function multi-versioning is
  permitted for individual blocks. Its resolvers run at load time, which
  immediate binding (`modernisation-plan.md`, section 4.9) already
  moves to start-up.
- 512-bit vectors on x86-64 are off unless enabled deliberately: on some
  processors they lower the clock frequency of the core for a while and
  enlarge the register state saved at every context switch. The
  benchmark suite reports context-switch cost with and without them.
- On AArch64 the baseline is NEON. Scalable vectors are not used in port
  data, for the same reason as `std::simd`.

## Agreement between the simulator and the target: within tolerance

Decided: performance comes first, and the simulator and the real
deployment are expected to agree within a tolerance, not bit for bit. A
controller whose behaviour changes materially with last-bit differences
is fragile, and real sensors and plants perturb it far more than the
compiler does. The plan therefore does not promise or test for
bit-identical results across machines, and blocks are built for speed:
vectorisation on, fused multiply-add allowed, target-specific
instructions used.

What remains worth guarding against is not small numerical difference but
the few compiler options that change behaviour in kind:

- **Finite-math-only removes safety checks.** With `-ffast-math` (through
  `-ffinite-math-only`) the compiler may assume no value is ever NaN or
  infinite and delete tests such as `std::isnan(x)` or `x != x`. A guard
  on a sensor value or on a command about to reach an actuator can vanish
  without warning. This is the one fast-math component that is excluded.
- **Reassociation breaks compensated arithmetic.** Code that depends on
  the exact order of operations (compensated summation, error-free
  transformations, some angle-wrapping tricks) can be simplified into
  something wrong, not something slightly different. Such functions are
  marked to be compiled without reassociation.
- **Process-wide side effects.** Linking with fast-math can switch the
  floating-point mode of the whole process at start-up on some toolchain
  versions. The mode is set explicitly per thread (above), so this neither
  surprises nor is relied on.

Recommended build for blocks, as the default of the block package:
optimise fully, allow contraction, drop `errno` and trapping-math
semantics (`-fno-math-errno -fno-trapping-math -ffp-contract=fast`), and
optionally allow reciprocal and signed-zero relaxations. Not
`-ffinite-math-only`, and reassociation only where the author of a block
turns it on.

## Finite checks that cannot be optimised away

Because non-finite
values are the failure that matters on a real machine, the framework
offers the check itself: a port or a block output annotated
`rtt3::finite` is tested on every write, by inspecting the bit pattern
with integer operations, so the test survives any floating-point option.
A non-finite value puts the node into its error state and is not
delivered. It is on by default for links into nodes marked as actuators
and off elsewhere.

## Comparing against the simulator, done so that tolerance works

- *Per-step replay.* A block is fed recorded inputs step by step and its
  outputs compared within a bound. Inputs are taken from the recording at
  every step, so differences do not feed back and accumulate through the
  rest of the system.
- *State drift.* A block with internal state (an integrator, a filter)
  still accumulates its own difference. Stable filters stay within a
  bound; a pure integrator drifts without limit, so the comparison uses a
  bound that grows with time or resynchronises the state periodically.
- *Discrete decisions.* A threshold, a mode switch or a rounding to an
  integer can fall on the other side for a last-bit difference, and the
  two runs then differ by a whole event for a short time. Comparisons
  around such events allow a shift of a sample or two in time, not just a
  bound in value, or compare the event sequence separately from the
  signals.
- *Closed loop.* End-to-end comparison of simulation and real machine is
  about behaviour (tracking error, settling, limits respected), not about
  matching traces sample by sample; open-loop traces of a system with
  contact diverge whatever the arithmetic.

## Where bit-exact results would still be needed

None of these is
known to apply to the current system; they are listed so the decision can
be revisited if one appears:

- two computers computing the same value and cross-checking each other
  for safety, or voting, with an equality comparison;
- checksums, hashes or signatures taken over floating-point results;
- regulatory or certification evidence that requires reproducing a
  recorded run exactly on different hardware.

Results on the same binary and the same machine are repeatable in any
case, which is what regression tests on one platform need.

## Checking it

Replay tests compare within stated tolerances on x86-64
and AArch64. The benchmark suite includes a block over array data built
with and without vectorisation, so a lost vectorisation shows as a
regression. A test confirms that the `rtt3::finite` check still fires in
a build with every fast-math option on.

## What applies to RTT 2.x today

- **Over-aligned types and the real-time allocator.** TLSF aligns blocks
  to twice the pointer size, 16 bytes on 64-bit targets
  (`rtt/os/tlsf/tlsf.c:138`). A type that needs 32-byte alignment is not
  guaranteed it when allocated through `rt_allocator`. Check any such
  types used with real-time strings, containers or `send`.
- **Subnormal numbers.** Flush-to-zero can be set at the start of each
  real-time thread, in the component's `configureHook` or `startHook`,
  since it is per-thread state.
- **Build flags** are chosen by each component, not by RTT. Avoid plain
  `-ffast-math` for the reason given above; prefer
  `-fno-math-errno -fno-trapping-math -ffp-contract=fast`, or add
  `-fno-finite-math-only` after it.

## Open points

- Is flushing subnormals to zero on real-time threads acceptable as the
  default?
- Do port data types today contain vector types from a linear-algebra
  library, and which one? That decides how much over-alignment support is
  needed first.
- Is there any cross-checking between two computers, or any checksum over
  computed values, that would need identical results?
- `std::simd` in GCC 16 is experimental and unverified here, like the
  rest of the GCC 16 feature set; revisit its exclusion from port data
  once it is stable.
