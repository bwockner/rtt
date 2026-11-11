# Open questions, licence position and unverified assumptions

This file collects, in one place, what is still undecided across the three
plans, what the licence allows, and what the plans assume without having
checked. The plans themselves remain the reference for the design; when a
question here is answered, update the plan it belongs to as well.

Numbers in brackets refer to the open-question lists in the plans: M for
`rtt3/questions.md`, L for `rtt3/logging.md`.

## 1. Questions that block design work

These decide the shape of `rtt3` stages S4 and S6 to S11 and should be answered
first.

**The existing simulator and deployer extension**

- Can the simulator's reflection-based block wrapper, or a few example
  blocks, be read? Its conventions for recognising inputs, outputs, state,
  parameters and the time step are meant to become the block mapping in
  `rtt3` unchanged. [M19]
- What generates the blocks, and does it also generate the wiring between
  them? [M19]
- What are the simulator's execution rules: fixed step, multi-rate, how
  loops are broken? The in-thread execution rules of `rtt3` are meant to
  match them. [M19]
- Can the deployer extension that substitutes ports with UDP be read? Its
  behaviour is the specification for link substitution and the UDP
  bridge. [M11]
- What encoding does that extension put on the wire, and must `rtt3` stay
  compatible with it so the simulator runs unchanged? [M11]
- How are UDP port numbers assigned today? [M11]
- What happens to the disconnected hardware component today: not loaded,
  loaded but not started, or left running? [M11]
- Is simulated time stepped in lock with the controller, or does the
  simulator run in real time? [M11]

**Variation and tooling**

- Is the set of sensor and actuator stacks closed when the software is
  built, or are stacks added in the field without a rebuild? This decides
  whether slots are enough or runtime loading is essential. [M11]
- May the deployment declare which links can be substituted, or must any
  link be substitutable from the machine file alone? [M11]
- How do the inspection programs connect today, and do they only read, or
  also write properties and call operations? [M11]
- Do deployer scripts (`.ops`) contain logic beyond loading, connecting
  and configuring? If so, dropping scripting needs revisiting. [M1, M11]

**Toolchain and targets**

- Is deployment AArch64 only, or is 32-bit ARMv7 needed? It changes the
  design of the lock-free primitives. Which board and kernel are the
  reference for latency numbers? [M5]
- Is a project-built GCC 16 AArch64 cross toolchain acceptable, or must
  the build use a vendor SDK? [M6]
- Which GCC version and flags does the simulator build with today? [M19]

## 2. Questions that shape the API but do not block starting

- **Scripting**: confirm that program scripts and state machines are
  dropped. [M1]
- **Lifecycle**: keep the RTT state model and hook set, or simplify? [M2]
- **Operations**: keep both own-thread and client-thread execution with
  send and collect, or one model? Are operations mostly called between
  components, or from the deployer and tools? [M3, M18]
- **Blocking cross-thread calls** from real-time nodes: compile-time error
  (as planned) or warning?
- **Exceptions from user hooks**: `noexcept` hooks with termination by
  default and catching as opt-in, or the reverse? [M13]
- **Links in static deployments**: fixed for the life of the process, or
  changeable at run time? [M7]
- **Properties**: are they changed while components run? Are attributes
  and constants used separately from properties? Keep the `.cpf` format
  or move to another? [M10, M18]
- **Data types**: is any existing port data type polymorphic? Are typekits
  generated or hand-written? Are projection and structural matching
  needed, or is exact type match enough? [M16]
- **Ports**: which parts of port set-up cost the most today? Latched
  inputs and port groups are the first candidates to cut if scope is
  tight. [M17]
- **SIMD and numerics**: agreement within tolerance between simulator and
  target, with performance first, is decided, and Eigen is the
  linear-algebra library. Still open: is flushing subnormals to zero on
  real-time threads acceptable as the default? Which Eigen version, and
  are Eigen types on ports fixed-size, dynamic, or both? What were the
  circumstances of the crash seen with 16-byte alignment on x86? Is using
  unaligned Eigen types at boundaries acceptable? Is any
  Eigen code built with OpenMP or an external BLAS? Is there any
  cross-checking between two computers, or checksum over computed values,
  that would need identical results? See `rtt3/numerics.md`. [M20]
- **Component style**: can algorithmic components be written as functions
  of inputs returning outputs, so same-thread links are register-passed,
  and which components need port members? Should inlining be bounded for
  very large single-thread graphs? [M21]
- **Compile-time evaluation**: which parameters are the same on every
  machine and could be fixed at build time? Is unit checking on links
  wanted? Are execution-time budgets known for existing components? See
  `rtt3/compile-time.md`. [M22]
- **Filters**: the estimator is an EKF with per-sensor varying
  covariance. Still open: number of states, number and dimension of
  sensors, whether each sensor's noise is uncorrelated between its
  components, whether to re-linearise between sensor updates, whether the
  covariance is used downstream, how many true degrees of freedom the
  five or six SE(3) bodies have and whether the mechanism has closed
  loops, which error convention is
  used, and the state magnitudes, required
  resolutions and run length that decide whether a mixed-precision
  factored form is safe. See `rtt3/estimation.md`.
- **Estimator architecture**: a truncated factor graph (fixed-lag
  smoother) over the reduced state, not a pure EKF and not a cascade.
  Still open: window length and what sets it, whether the solve runs in
  the real-time thread or beside it, which solver, whether a delayed
  smoothed output is wanted, the grid spacing, and whether biases are
  per grid state or shared across the window. See `rtt3/estimation.md`.
- **Observability indicator**: the curvature of the residual along a
  direction is the measure of how well it is observed. Still open: which
  directions are weak in practice, what consumers should do while one is
  flagged, and whether the threshold is fixed or adaptive. See
  `rtt3/estimation.md`.
- **Unexplained sensor motion**: when a gyroscope reports rotation the
  mechanism model cannot produce, should the system warn, model an extra
  freedom, or switch from the planar to the full model? Where are the
  sensors mounted and is the base fixed? See `rtt3/estimation.md`.
- **Identification and calibration**: the estimator's factor graph is
  also to be used for calibration (mounting offsets, intrinsics, time
  offsets, deadband and backlash). Still open: whether commands are
  recorded on the sensors' time base, what is measured on the actuator
  side of each joint, which parameters should be tracked on line, and
  whether a bench procedure for inertial intrinsics exists, and for
  the tape, plumb-bob and total-station observations: which reference
  points exist, what the instrument outputs, and how many static poses.
  For validation: which quantity defines accuracy and what the
  specification is. See
  `rtt3/estimation.md`. [M24]
- **Generated kinematics**: SymForce is planned for closed-form
  kinematics. Which version, is generated code checked in or regenerated,
  which calculations stay on KDL, and what tolerance defines
  "approximately planar" per machine? [M23]
- **Inter-process transport**: any transport in scope beyond the UDP
  bridge, or single-process with bridges as components? [M4]
- **ABI stability** for separately built component libraries. [M8]
- **Test framework** for the new test suite. [M9]
- **Subsystems and single-process closed-loop tests** with the MuJoCo
  plant block: wanted? [M19]
- **Portability beyond Linux**: adopt the low-cost guardrails that keep an
  RTOS or microcontroller port possible, or leave them out? If better
  determinism is wanted later, is the likely route a co-kernel on the same
  processor or a microcontroller beside the Linux board? [M15]
- **Project name**: is the package published as `rtt3`, and does it keep
  the Orocos name? [M14]

**Logging**

- Build the logger in-tree, or wrap an existing asynchronous logger? [L1]
- Are the downstream spdlog loggers synchronous or asynchronous, with
  which overflow policy and sinks? Do real-time components call spdlog
  directly today? [L2]
- Should non-real-time threads also drop when their ring is full? [L3]
- Flight recorder and crash recovery in the first version? [L4, L5]
- Is routing `rtt3` records into the existing spdlog sinks the right
  default? Is structured output wanted? [L6]
- Is `log::warn<"...">(args)` acceptable if the natural call syntax
  proves unreliable on GCC 16? [L7]

## 3. Decisions waiting on RTT 2.x

- **Which unmerged fix branches to take.** `master` has the toolchain
  branches and two small fixes. The priority-inheritance branch is the
  obvious next one; the others are listed with their state in
  `optimisation-plan.md`, section 3.6.
- **Priority inheritance default.** The plan proposes on by default behind
  a build option. It changes scheduling under contention.
- **`-Bsymbolic-functions`** breaks overriding RTT functions with
  `LD_PRELOAD`. Acceptable?
- **The intermittent `scripting_test` hang** under parallel test runs has
  not been investigated, and it is not known whether it predates the
  `toolchain-2.10` merge.

## 4. Licence position

This is a reading of the licence text in the repository, not legal advice.
Decisions with commercial consequences should be checked with someone
qualified.

**What the code is under.** `COPYING.md` and the header of every file
under `rtt/` state the GNU General Public License **version 2 only** (the
headers say "version 2 of the License", with no "or any later version"),
plus a special exception: instantiating templates, using macros or inline
functions from a file, or linking with it, does not by itself make the
resulting executable subject to the GPL. In practice that lets components
and applications that merely use RTT be licensed as their authors wish,
while RTT itself stays under the GPL.

**Can the licence be changed?** The GPL does not forbid relicensing, but
only the copyright holders can do it, and all of them have to agree.

- The file headers name many holders: Peter Soetens (the large majority
  of files), The SourceWorks, FMTC, Sylvain Joyeux, Intermodalics and
  several individuals, and the history contains contributions from many
  more. There is no sign of a contributor agreement assigning rights to
  one party. Getting agreement from all of them is not realistic, so for
  practical purposes **the licence of the existing code cannot be
  changed**.
- A fork has no more rights than any other recipient. It must distribute
  the existing code, and anything derived from it, under the same terms:
  GPL v2 with the exception.
- Because the grant is version 2 only, the existing code cannot be moved
  to GPL v3 either without the same agreement.
- The fork may add its own copyright notices for its own changes, but
  those changes, being modifications of GPL files, are under the same
  licence.

**What this means for `rtt3`.**

- Code that is genuinely new and not derived from RTT 2.x sources can be
  placed under any licence its author chooses. Sharing a repository with
  GPL code does not change that.
- "Derived" is the test that matters. The modernisation plan says the
  lock-free algorithms, activity semantics and lifecycle are carried over
  "as behaviour, not as code". If any `rtt3` file is produced by copying,
  translating or adapting an RTT 2.x file, it is a derivative work and
  must stay under GPL v2 with the exception. Re-implementing an algorithm
  from its published description or from a written specification is not
  copying the code; working from the source file open in the next window
  is at least arguable.
- So the choice has to be made **before** `rtt3` code is written:
  - keep GPL v2 with the exception for `rtt3` as well, and port code
    freely; or
  - choose another licence for `rtt3`, and then write it without
    consulting RTT 2.x sources for the parts concerned, with a record of
    what each part was based on.
- A mixed outcome is possible: a permissively licensed neutral block and
  wire-encoding package (which has no RTT ancestry at all and is meant to
  be shared with the simulator), and a GPL-with-exception framework.
- Header-only `rtt3` code under the existing licence remains usable from
  proprietary components, because the exception explicitly covers
  templates and inline functions.

**Third-party code in the tree.** TLSF (`rtt/os/tlsf/`) is dual licensed
GPL v2 / LGPL v2.1 by its authors. TinyXML (`rtt/marsh/tinyxml*`) is
vendored; check its own licence text before reusing it elsewhere. Boost is
a build dependency, not part of the tree.

**Open points for the owner.**

- Which licence should `rtt3` carry? [M14]
- Should the neutral block and wire packages have a different, more
  permissive licence so the simulator and other tools can use them freely?
- Is anything in the fork intended for distribution outside the
  organisation? The GPL's obligations attach to distribution, not to
  internal use.

## 5. Assumptions made without verification

| Assumption | Where it matters | How to settle it |
|---|---|---|
| GCC 16 implements the reflection features the design relies on (annotations, parameter reflection, static-storage helpers, expansion statements) | The whole `rtt3` design | Stage S1 spike on a real GCC 16 |
| Error handling of reflection queries and constexpr exceptions in GCC 16 | Quality of compile-time diagnostics | Same spike |
| Annotations can be placed on function parameters | Per-argument documentation of operations and blocks | Same spike; fallback is an annotation on the function |
| A per-call-site constant can be created without a macro | Natural logging call syntax | Logging step L1; fallback is a string template parameter |
| Reflection works on declarations from C headers | Wrapping C functions | Same spike |
| "ARM" means AArch64 | Lock-free design, toolchain | Owner |
| The simulator and inspection programs are separate processes | UDP bridge and inspection service design | Owner |
| OCL's real-time logging formats in the caller and allocates from TLSF | Advice in the optimisation plan | Read the OCL source in use |
| spdlog formats the message text in the calling thread | Assessment in the logging plan | Read the spdlog version in use |
| MuJoCo's stepping interface is arrays inside a state object | Plant block design | Read MuJoCo's headers |
| The expected gains in the optimisation plan | Whether each item is worth doing | Step 0 benchmarks |
| Latency targets for the logger | Acceptance criteria | Logging step L0 and L2 benchmarks |

## 6. Facts established about the code during this work

Recorded here because they are easy to lose and several plans depend on
them. File references are against `master` after the `toolchain-2.10`
merge.

- RTT mutexes on Linux are created without priority inheritance
  (`rtt/os/gnulinux/fosi.h`).
- Memory is never locked on the Linux target; `mlockall` is only called
  for Xenomai and RTAI.
- `RTT::Logger` takes one global mutex per `<<`, checks the level only
  after locking, and holds the mutex while writing to the console and file.
- Each hop in a port connection costs an out-of-line call, a shared lock,
  a reference-count increment and decrement, and a `dynamic_cast`.
- `ChannelElementBase` is a virtual base, which is why the cast cannot be
  a `static_cast`.
- Output ports keep a copy of the last written value by default, so each
  write copies the sample twice.
- The TLSF pool is guarded by one RTT mutex and is allowed to grow through
  system calls by default; `OperationCaller::send` allocates from it on
  every call.
- Nothing in the build requests hidden visibility, immediate binding or
  link-time optimisation.
- On GCC the atomic operations use the legacy `__sync` builtins.
- Components and plugins register through macros that export unversioned
  `extern "C"` symbols, which would collide between RTT 2.x and any
  successor that kept them.
- `master` was an ancestor of `toolchain-2.10`, so merging the toolchain
  branches was a fast-forward and rewrote nothing.
- The test suite passes on x86-64 with GCC 13 and Boost 1.83, apart from
  the intermittent `scripting_test` hang noted above.
