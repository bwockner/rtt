# rtt3: compile-time deployment

Part of the `rtt3` plan; see [README.md](README.md) for the index, the stages and how they depend on each other.

Stages: S8 (deployment struct, fold, graph export) and S9 (slots and
substitutable links).

A deployment is a struct. Its data members are the nodes, annotations give
each node its activity, and a constant member lists the links:

```cpp
struct App {
    [[=rtt3::periodic(1ms), =rtt3::priority(80)]] Sensor     sensor;
    [[=rtt3::periodic(1ms), =rtt3::priority(79)]] Controller ctrl;
    [[=rtt3::triggered_by(&Driver::command_in)]]  Driver     left, right;

    static constexpr auto links = rtt3::links(
        rtt3::link(&App::sensor, &Sensor::state,       &App::ctrl,  &Controller::state),
        rtt3::link(&App::ctrl,   &Controller::command, &App::left,  &Driver::command_in, rtt3::policy::buffer<8>{}),
        rtt3::link(&App::ctrl,   &Controller::command, &App::right, &Driver::command_in, rtt3::policy::buffer<8>{}));
};

int main() { return rtt3::run<App>(); }
```

Instances are identified by their data member, so several nodes of one
component type need no tags. What the compiler does with it:

- **Validation**: value types match, every required input is connected, no
  input has two writers unless its policy allows it, triggering ports
  belong to their node. Errors name the member, for example
  "input `App::right.command_in` has no writer".
- **Storage**: components, channels and buffers have sizes known at
  compile time. Nothing is allocated at start-up and the deployment can sit
  in static storage.
- **Scheduling**: nodes that share a period and priority can be folded into
  one thread with a fixed execution order, computed from the link graph
  (topological order; cycles are errors unless broken with a delayed link).
- **Channel selection**: where the graph proves one writer and one reader,
  a single-producer single-consumer channel is used.
- **Start-up**: the configure, start and stop order is a constant sequence.
  `rtt3::run` returns the first `expected` error, with the node it came from.
- **Dead code**: no factories, name tables or type descriptions are linked
  unless `rtt3-reflect` is used.

## Same-thread links compile to function calls

**Goal.** The framework should add no overhead where none is required.
When two nodes run in the same thread, passing a value from one to the
other should cost what an inlined function call costs: if the compiler
can keep the value in a register it does, nothing is written to a port or
channel object on the way, and there is no copy that a hand-written
`consume(produce())` would not also make.

This is a design goal, not a hard requirement. Where it proves too
difficult for a particular feature or component style, the fallback is
the one-store form described below, and the cost is documented and
measured instead of hidden. The checks at the end of this section report
how close each case gets; they gate regressions, not the first release.

**Why a "plain variable" is not enough.** An earlier part of this plan
said a same-thread link is a plain variable without atomics. That still
forces a store and a load: a port object that is a member of a component
in static storage has an address other code may hold (inspection, other
threads), so the compiler has to assume the store is observable and cannot
remove it. To get register passing, the value must have no addressable
home at all on the normal path.

**How the fold is generated.** For each thread, the deployment generates
one step function in which every same-thread link is a local value:

```cpp
// what the compiler sees for a thread holding sensor -> estimator -> controller -> drive
void step() noexcept {
    const auto raw   = sensor.update();
    const auto state = estimate(est_state, est_params, raw, dt);
    const auto cmd   = pid_step(pid_state, pid_params, target, state, dt);
    drive.apply(cmd);
}
```

- Links are single-assignment locals, in topological order. There are no
  port objects, queues, status words or reference counts for them.
- The framework's own layers between the blocks (node shims, lifecycle,
  argument binding) are forced inline and contain no type erasure, no
  virtual calls, no function pointers and no `std::function`.
- Small values travel in registers. A value too large for registers, such
  as a covariance matrix, is constructed once, in the place its consumer
  reads it from, and is not copied between nodes.
- A value with several readers in the thread is computed once and read by
  each; fan-out costs nothing.

**What a component has to look like to get this.** The value has to flow
through arguments and return values, which is exactly the block form
(`blocks.md`): a free function, or a member function whose object is the node's
state. For these, the fold above is what is generated.

Components written with port members (`state.get()`,
`command.write(x)`) cannot be given the same guarantee, because the port
is an object with an address. For them a same-thread link is one store by
the writer into a slot and direct reads of that slot by the readers: no
copy, no atomic, no queue, but one memory write. The recommendation is
therefore: algorithmic nodes are blocks; port members are for nodes at the
edges that talk to devices or other threads, where a memory write is not
what limits them.

**Things on a link that would force the value into memory, and how each
is kept off the normal path.**

| Feature | Handling |
|---|---|
| Port observation (`runtime-layer.md`) | A read-mostly flag is tested; only when an observer is attached does a cold, out-of-line path copy the value out. With no observer the value never leaves its register |
| `rtt3::finite` check | Operates on the value itself; no storage needed |
| Substitutable link (`deployment.md`) | A branch selects between the computed value and the bridge's buffer. The memory read happens only on the substituted side |
| Slot (`deployment.md`) | Dispatch is a switch over the alternatives, each inlined in its own branch, not a call through a pointer, so arguments and results still pass in registers |
| Lifecycle | One test per thread per step that all its nodes are running selects the straight-line path; if any node is not running, a slower per-node path is taken |
| Freshness and `when_fresh` conditions | Carried as ordinary values alongside the data |
| Per-node counters | Per thread by default; per-node counting is opt-in because it adds a memory increment per node per step |

**Where memory is inherent.** A delayed link holds the previous step's
value, a hold between rates keeps a value across steps, and a block's
state persists; these are stored because they must outlive the step.
Links to other threads go through channels (`realtime-internals.md`). A call into code the
compiler cannot see (a C library built separately) passes arguments by
the platform's calling convention.

**What the compiler needs.**

- It must see the bodies. Blocks are defined inline or as templates in
  headers, or the deployment is built with link-time optimisation, which
  is already the default for static deployments (`os-layer.md`).
- The generated step function is marked for aggressive inlining of
  everything it calls. A thread can opt out for a very large graph, where
  inlining everything would hurt the instruction cache.
- Blocks are `noexcept` and, where possible, `constexpr`.

**This is checked, not assumed.** No library can promise what an
optimiser will do, so the property is tested:

- *Compile-time evaluation.* For blocks that are `constexpr`, a test
  evaluates a whole folded step at compile time with `static_assert`,
  which proves the glue adds nothing that is not a constant expression.
- *Generated code comparison.* A test compiles a folded chain and the
  equivalent hand-written function with the release flags and fails if
  the fold has more instructions or any store that the reference lacks.
  It runs for x86-64 and AArch64.
- *Benchmark.* A chain of trivial blocks is timed against the direct
  call; the two must be indistinguishable.

## Install-time variation: slots

The current system uses runtime components and the deployer to choose,
per installed machine, which sensor and actuator stacks, which kinematic
variant and whether simulated hardware are used. A purely fixed deployment
struct cannot express that, so the deployment gets a variation mechanism
that keeps compile-time checking:

```cpp
struct App {
    [[=rtt3::periodic(1ms), =rtt3::priority(79)]] Controller ctrl;

    [[=rtt3::triggered_by<&rtt3::slot_port::command_in>]]
    rtt3::slot<EthercatDrive, CanDrive> drive;                  // exactly one is active

    rtt3::slot<ScaraKinematics, GantryKinematics> kinematics;

    rtt3::optional_slot<ForceSensorA, ForceSensorB> force;     // may be absent

    static constexpr auto links = rtt3::links(
        rtt3::link(&App::ctrl, &Controller::command, &App::drive, rtt3::port<"command_in">),
        ...);
};

int main(int argc, char** argv) { return rtt3::run<App>(rtt3::machine_file(argv[1])); }
```

- A slot lists the alternatives that may occupy one position in the
  system. Which one is active is read from the machine configuration file
  at start-up and does not change afterwards.
- The compiler checks every alternative against every link that touches
  the slot, so each selectable combination is validated at build time. A
  machine file can select a combination but cannot create an invalid one.
  Links to a slot name the port by identifier; every alternative must have
  a port of that name and value type, which reflection verifies.
- Storage is sized for the largest alternative, so there is still no
  allocation. The cost is one indirect call per slot per cycle, instead of
  an inlined call; links between fixed nodes are unaffected.
- Hardware simulation is selected by the same file, at the level of
  connections, as the deployer extension does today: the real source or
  sink of a port is disconnected and a UDP-backed port is connected in its
  place. In `rtt3` this is *link substitution*:
  - a link, or all links of a node, is declared substitutable in the
    deployment (`rtt3::substitutable`), so the set of places where
    simulation may cut in is fixed and visible at compile time;
  - the machine file may then replace either end of such a link with a UDP
    endpoint (`runtime-layer.md`). The component on the kept side is unchanged and
    unaware of it;
  - a node whose links have all been substituted can be marked disabled in
    the machine file, so a hardware driver is neither configured nor
    started when its hardware is absent;
  - substitution is resolved once at start-up. A substitutable link costs
    one indirection per port access; links not declared substitutable keep
    the fully static path and cannot be redirected by any file or tool.

  No mock components are needed, generated or hand-written.
- Kinematic variation that is only a difference in parameters is a
  property set in the machine file, not a slot; a slot is for variants
  with different code.
- The machine file also carries property values and, per slot, activity
  overrides (period, priority, CPU). `app --check machine.toml` validates
  a file against the binary and prints the resolved deployment without
  touching hardware, so an installation can be verified before first start.

The approach assumes the set of stacks is known when the software is
built, even though the choice among them is made at install time. Where
that does not hold, a slot can include `rtt3::loaded<Interface>`, an
alternative supplied by a component library loaded at start-up (`runtime-layer.md` and
`conventions.md`) and checked against the slot's port interface at load time instead of
compile time.

Optional `rtt3::autowire<App>` connects outputs to inputs whose name and
value type both match, with ambiguity a compile error. It is off by
default; explicit links are the norm.

`rtt3::reflect::expose(app)` makes a static deployment visible to runtime
tooling and lets dynamically loaded components attach to it.
