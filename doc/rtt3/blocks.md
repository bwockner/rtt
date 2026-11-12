# rtt3: portable functions as components (blocks)

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S4 (the neutral block package) and S7 (blocks as nodes).

**Aim.** Algorithms and their data types are written once as plain C++,
with no dependency on the framework, so the same function can run inside
the simulator or as a node in the real-time system and can be moved from
one to the other without being rewritten. The framework's job is to turn
such a function into a component from its declaration alone.

**What a portable function looks like.**

```cpp
// control/pid.hpp: includes nothing from rtt3 except the annotation header
struct PidParams { double kp = 1, ki = 0, kd = 0; double limit = 10; };
struct PidState  { double integral = 0, previous_error = 0; };

[[=blk::block]]
Torque pid_step(PidState& state, const PidParams& params,
                Position setpoint, Position measured,
                std::chrono::nanoseconds dt) noexcept;
```

Rules that keep it portable:

- **Plain types only** in the signature: value types that satisfy the port
  data requirements (`components-and-ports.md`). No ports, no framework error type, no handles.
- **No hidden inputs.** Time comes in as a parameter, never from a clock;
  configuration comes in as a parameter struct, never from a file; state
  is passed in, never kept in statics. The function is then a pure
  function of its arguments, which is what makes it give the same result
  in the simulator, in the real-time system and in a unit test.
- **No side effects** beyond its outputs and its state argument: no I/O,
  no allocation, no logging through a framework.
- Failure is reported through the return value (`std::expected` with the
  library's own error enum, or a status member of the result).

A stateful block may also be written as a class whose member function is
the step (`struct Pid { PidParams params; Torque step(...); };`); the
object is then the state.

**Mapping to a node.** Because the declaration is C++, the roles of the
parameters follow from their types and need no hints in the common case:

| In the declaration | Becomes |
|---|---|
| Parameter by value or by `const&` | Input port named after the parameter, holding the latest value |
| Return value | Output port; a returned struct can be unpacked into one port per member |
| First non-const reference parameter | The node's state, kept inside the node and value-initialised at configuration |
| `const&` parameter whose type is marked `blk::parameters`, or which is annotated so | Properties, loaded from the machine file |
| `std::chrono` duration parameter annotated or named `dt` | Supplied by the node: its period, or the measured or simulated time since the last run |
| Further non-const reference parameters | Output ports (annotate `blk::inout` to make one both) |
| `std::expected<T, E>` return | `T` is the output; an error puts the node in its error state |

Anything that departs from these defaults is said with an annotation on
the parameter or the function. Used in a deployment:

```cpp
[[=rtt3::periodic(1ms), =rtt3::priority(79)]]
rtt3::function_node<^^pid_step> pid;

rtt3::link(&App::sensor, &Sensor::position, &App::pid, rtt3::port<"measured">)
```

The node is an ordinary node: it appears in the graph, is folded into a
thread with its neighbours, can occupy a slot, and its links can be
substituted. The call is direct and inlinable.

**Start from the simulator's block wrapper.** The simulator already
defines its blocks with a C++26 reflection wrapper over simple functions,
so that blocks are easy to generate. That settles two things: the
simulator is on a reflection-capable GCC, and a working convention for
"which parameter is what" already exists. `rtt3` should adopt that
convention, not introduce a second one:

- The wrapper, or the part of it that turns a function into a block
  description, is extracted into a small header-only package with no
  dependencies (called `blk` in this document), used by both the simulator
  and `rtt3`. If it cannot be extracted as it stands, `blk` is written to
  accept the same function shapes.
- The mapping table above is therefore provisional. Where it differs from
  what the simulator's wrapper does today, the wrapper wins, since blocks
  and generators already depend on it.
- `rtt3::function_node` adds only what a real-time node needs on top of
  the shared description: ports, the state object's lifetime, properties
  from the machine file, the time step, and error state.

**Kept friendly to code generation.** The design constraints that follow
from blocks being generated:

- A generated function needs no annotation to become a block. Roles come
  from the signature alone; annotations exist only for documentation,
  units and overrides, and a generator may omit them.
- Nothing has to be written in a second place: no registration call, no
  port list, no name strings.
- The subset of C++ a block may use is small and stated (plain structs,
  arithmetic, fixed-size arrays, `std::expected` for errors), so a
  generator has a simple target and generated blocks are valid port data
  by construction.
- Names are stable and derived: parameter names become port names, the
  function name becomes the block type name. A generator that keeps names
  stable keeps graphs and recordings valid across regenerations.
- Wiring is as easy to generate as blocks. A deployment or subsystem is a
  struct with members and a list of links, with no ordering constraints
  and no imperative set-up code, so it can be emitted directly from a
  graph description. A small tool, `rtt3-gen`, reads a graph in the
  exported graph format (`runtime-layer.md`) and writes the corresponding struct, so a
  controller drawn or assembled in the simulator can be turned into a
  deployment without hand-editing.
- Generated and hand-written blocks are indistinguishable to the
  framework, and can be mixed in one graph.

**Moving a function between the simulator and the real-time system.**

- *Controller into the simulator*: the simulator calls `pid_step`
  directly, with its own state, parameters and time step. Nothing from
  `rtt3` is linked.
- *Simulator model into the real-time system*: a plant or sensor model
  written to the same rules is wrapped with `function_node` and placed in
  a hardware slot as an alternative to the real driver (`deployment.md`). That gives
  an in-process simulation with no network hop, alongside the UDP route.
- *Either direction during development*: because a block is a pure
  function, a recording of its inputs from a real machine (port
  observation, `runtime-layer.md`) can be replayed through it offline and the outputs
  compared, with no framework and no simulator running.

**One block definition for two graph engines.** The simulator is a C++
block graph in the Simulink style with MuJoCo as the physics backend, so
the two systems are structurally alike: both execute a graph of blocks
connected by typed signals. With the block description shared as above,
`blk::describe<^^pid_step>` yields one constant table for a block (inputs,
outputs, state type, parameter type, time step, with names, types, units
and documentation), and both engines build their nodes from it. A block
library has exactly one description of each block, and it is the function
itself.

**Same execution semantics inside a thread.** For a block to behave
identically in both places, the two graphs must agree on when a block sees
its inputs. Inside one `rtt3` thread the rules are those of a synchronous
block diagram, stated here as part of the design, not left as an
implementation detail:

- blocks run once per step in topological order of the links;
- a block sees the outputs its upstream blocks produced in the same step;
- a cycle must contain a link marked as a delay, which delivers the value
  from the previous step and has a declared initial value;
- blocks at a slower harmonic rate run on every n-th step, and links
  between rates are either a hold (slow to fast) or a declared delay
  (fast to slow), never an unspecified sampling.

Links between different threads are different: the reader samples the
latest value at whatever moment it runs, which no fixed-step simulator
reproduces exactly. So a group of blocks that must match the simulator
step for step is kept within one thread, and links that cross threads are
the places where small timing differences between simulation and the real
system are to be expected.

**Subsystems.** A group of blocks with the links between them and a set
of boundary ports is written as a struct, in the same form as a deployment
(`deployment.md`) but using only `blk` types. `rtt3` embeds it as a nested part of a
deployment, flattened into the surrounding thread; the simulator can
instantiate the same struct as a subsystem. Moving a whole controller, or
a whole plant model, between the two is then moving one type.

**MuJoCo.** The physics stays behind a block boundary. Its interface is
arrays inside a large state object, not typed arguments, so it is not
wrapped automatically; one hand-written plant block maps typed inputs to
actuator controls and sensor data to typed outputs. Two ways to use it:

- *In the simulator process*, as today, with the controller reached
  through substituted links over UDP.
- *In the controller process*, for automated tests: the plant block
  occupies the hardware slots, runs in a non-real-time thread, and the
  simulated clock advances both in lock step. That gives a deterministic,
  faster-than-real-time closed-loop test with no network and no second
  process, suitable for CI.

The plant block belongs with the simulator's code, not in `rtt3`; the
plan only makes sure such a block can be placed in a slot. Statements
about MuJoCo's interface here are from general knowledge of the library
and were not checked against its headers.

**Shared types across the UDP link.** The same type definitions are used
on both sides of a substituted link. The wire encoding and schema hash are
generated from the type, and the encoder and decoder are available as a
header-only package the simulator can use without the framework, so the
two ends cannot disagree about a layout without it being detected.

**C functions** are the same mechanism with less information. A C
declaration cannot say whether a pointer is an input, an output or state,
so those are classified where the function is wrapped
(`rtt3::function_node<^^c_step, rtt3::state<"s">, rtt3::out<"y">>`), and
an unclassified pointer is a compile error naming the parameter.
Companion functions map to the lifecycle (`rtt3::init<^^f>`,
`rtt3::cleanup<^^f>`), a pointer and length pair to a bounded array, and
an integer status return to the node's error state. Generated code that
works on global structs is wrapped by naming the globals and is a
singleton.

**When it runs.** By default every activation of its thread, on the
latest input values. `rtt3::when_all_fresh` and `rtt3::when_any_fresh`
restrict it to activations with new input.

## Generated kinematics

Closed-form kinematics generated from a symbolic model (the owner plans
to use SymForce) are an application concern and are described in
`../estimation/kinematics.md`. For the framework the relevant points
are: a generated function is a block like any other; its aligned Eigen
types are kept out of port data by a thin generated shim that takes and
returns boundary types (`numerics.md`); and full and planar variants of
one model can be alternatives in a slot (`deployment.md`).

**Limits.**

- Parameter names come from the declaration; unnamed parameters are
  addressed by position.
- Purity is a convention the framework cannot prove. The test-time
  allocation guard and system call filter (`realtime-internals.md`) catch allocation and I/O;
  hidden static state has to be found by review or by running two
  instances in a test.
- Variadic functions and functions taking callbacks are not wrapped.
- Reflecting free functions and their parameters, including annotations
  on parameters, is on the stage S1 spike list for GCC 16.
