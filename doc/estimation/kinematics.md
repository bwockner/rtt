# Generated kinematics

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A1. Closed-form kinematics and Jacobians generated from one
symbolic model of the mechanism.

The owner plans to use SymForce in the simulator libraries for
closed-form kinematics. Because blocks are plain functions shared with
the simulator, the same generated functions are the kinematics blocks of
the real-time system. What follows about SymForce is from general
knowledge of the project and was not checked against the version to be
used.

**Why this replaces the two-dimensional special case.** Today a speed-up
is obtained by restricting KDL to two dimensions for approximately planar
calculations, at the cost of maintaining a second, structurally different
code path. With a symbolic model the restriction is an assumption applied
to the model (the out-of-plane coordinates and rotations set to zero, or
to constants), and the generator simplifies everything that follows from
it: the zero terms, the trigonometric identities, the shared
subexpressions. The emitted function keeps the full three-dimensional
interface, so callers do not change and there is no hand-written planar
variant to keep in step.

**What this gives beyond speed.**

- *Both variants from one model.* The full and the planar function are
  generated from the same description. They can be alternatives in a
  kinematics slot (`../rtt3/deployment.md`), selected per machine.
- *The approximation becomes measurable.* A test runs both variants over
  recorded or sampled configurations and reports the difference, so
  "approximately planar" has a number attached and a bound that fails the
  build when a machine variant no longer satisfies it.
- *Derivatives come with it.* Jacobians for estimation and control are
  generated from the same model instead of being derived and maintained by
  hand.
- *Fits the real-time rules.* Closed-form generated code is straight-line
  arithmetic: no allocation, no iteration count that depends on the data,
  no traversal of a run-time chain description.

**How generated functions become blocks.**

- A generated function is a block like any other. It needs named
  parameters, plain value types and no hidden state, which is what such
  generators produce.
- Generated signatures use Eigen's default fixed-size types and the
  generator's own geometry types. Those are aligned types, which the
  boundary rule in `../rtt3/numerics.md` keeps out of port data. So each generated
  function is wrapped by a thin, also generated, shim that takes and
  returns boundary types and calls the function. Inside one thread the
  shim disappears through inlining; it matters only where a value crosses
  to another thread, another library or the simulator link.
- The generator's handling of singular configurations (small offsets
  added so that expressions stay finite) changes results slightly near
  those configurations. That is consistent with agreement within
  tolerance, and the offset is a parameter that must be the same in the
  simulator and on the target.
- Generated blocks are not usable in constant expressions; the
  compile-time items in `../rtt3/compile-time.md` do not apply to them.

**Build and reproducibility.**

- Generation is a build-time step with its own toolchain (Python). It is
  not needed on the target and not in the real-time system's run-time
  dependencies.
- Generated sources are either checked in or produced by a pinned
  generator version, with CI regenerating and comparing, so that the code
  running on a machine can be traced to a model and a generator version.
- Options that make the generator emit message types for structured
  outputs through a separate serialisation tool are avoided; outputs are
  matrices, vectors and plain structs, and the wire encoding is the block
  package's.

**Migration from KDL.** The generated functions are validated against the
existing KDL-based results within a tolerance, over recorded
configurations, before KDL is removed from a calculation. KDL can remain
for cases that have no closed form.
