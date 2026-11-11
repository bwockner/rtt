# rtt3: compile-time evaluation

Part of the `rtt3` plan; see [README.md](README.md) for the index, the
stages and how they depend on each other.

Stages: S1 for the techniques, S8 for most of the uses. Applies to every
stage as a design habit.

This file collects what `rtt3` computes or checks during compilation,
what else could be moved there, and where the limits are. Statements
about what GCC 16 accepts in constant expressions are unverified, like
the rest of the GCC 16 feature set, and belong to the stage S1 spike.

## Principle: one implementation, two evaluation times

A deployment has a part that is fixed when it is built (the nodes, the
links, the types) and a part that is only known when the machine file is
read (which slot alternative, property values, activity overrides).

Validation and derivation code is written once, as `constexpr` functions
over the plain tables that reflection produces. The compiler runs it over
the build-time part and turns failures into build errors. The same
functions run once at start-up over the part that depends on the machine
file and turn failures into start-up errors. Nothing is checked twice in
two different implementations, and nothing of this runs after start.

Functions that must never run at run time (hashing a name, building a
description table) are `consteval`, so that a mistake which would defer
them to run time is a compile error instead of a silent cost.

## What the plan already does at compile time

| What | Where described |
|---|---|
| Port, property, operation and node discovery; name tables | `reflection.md`, `components-and-ports.md` |
| Type and link checks, required inputs, single-writer rules | `deployment.md` |
| Execution order, thread fold, harmonic rates, cycle detection | `deployment.md` |
| Channel selection from the link graph | `realtime-internals.md` |
| Storage layout by owning thread; cache-line ownership check | `realtime-internals.md` |
| Checking every slot alternative against its links | `deployment.md` |
| Schema and schema hash of each data type; graph hash | `components-and-ports.md`, `runtime-layer.md` |
| Format-string checking and call-site descriptions for logging | `logging.md` |
| Evaluating a folded step in a `static_assert` | `deployment.md` |
| Rejecting unsuitable port data and log arguments through concepts | `components-and-ports.md`, `logging.md` |

## Further opportunities

Ordered roughly by value for the effort.

### 1. Constant-initialised deployments

If every node is constructible in a constant expression, the whole
deployment object can be declared `constinit`. It then has no
construction code at all: its initial state is an image in the
executable, there is no static-initialisation order to worry about, and
start-up cost before `configure` is zero. The framework's own globals
(registries, log ring directory, level atomics) are `constinit` as a
rule. A node that cannot be constant-initialised, for example one holding
a type from a library that is not `constexpr`, falls back to ordinary
construction for that node only; the build reports which nodes those are.

### 2. Unit checking on links

Ports and block parameters already carry an optional unit annotation for
documentation. Comparing the annotations of the two ends of a link is a
few lines of `consteval` code and turns a radians-to-degrees or
position-to-velocity wiring mistake into a build error. A link adaptor
that converts (`rtt3::convert`) declares its input and output unit. Links
where either end has no unit are not checked, so the feature costs
nothing until it is used.

### 3. Property defaults, ranges and the default machine file

- A default value outside its own declared range is a build error.
- The complete machine-file template (every node, slot, property with
  default, unit, range and documentation) is generated as a constant
  string, so `app --dump-defaults` prints a file that is correct by
  construction and never out of date.
- Lookup of names when the machine file is read uses a perfect hash table
  built at compile time, so configuration does no string comparison
  chains and no allocation.

### 4. Parameters fixed at build time

A property read from the machine file is a run-time value, so nothing
derived from it can be folded. Some parameters never vary between
machines. Marking such a property `rtt3::fixed` makes it a compile-time
constant: the optimiser can fold it through the block, and values derived
from it are computed by the compiler. Typical uses are filter
coefficients derived from a cut-off frequency and the node's period,
lookup tables, and kinematic constants of a machine family. A fixed
property still appears in the graph and the inspection service, read-only.
Whether a parameter is fixed or configurable is one annotation, so it can
be changed without touching the block.

### 5. Timing budgets

A node or block may declare an execution-time budget. From the budgets,
periods and the fold, the compiler can check per thread that the budgets
of the nodes folded into it fit within the period with a stated margin,
and that priorities are consistent with rates. The same budgets are the
thresholds for overrun detection at run time. This checks declared
numbers against each other; it does not measure or prove execution time.

### 6. Memory and capacity budgets

All static sizes are constants: the deployment object, each thread's
storage, the block pools sized from the number of asynchronous callers,
the log rings. A deployment can assert an upper bound on its static size,
and the start-up report prints the same constants. The largest datagram
of every substitutable link is known, so exceeding the UDP limit is
already a build error; the default UDP port mapping is a constant table.

### 7. Features removed, not disabled

Code for observation, link substitution and per-node counters is selected
with `if constexpr` on properties of the deployment type. A deployment
that does not expose itself to the runtime layer does not contain a flag
that is always false; it does not contain the code. This is what makes
"no framework overhead where none is required" a property of the type and
not of a run-time setting.

### 8. Blocks evaluated at compile time

A block that is `constexpr` can be tested in `static_assert`s next to its
definition: fixed inputs, expected outputs, sign conventions, saturation
behaviour, a few steps of a filter. Such tests cannot be skipped and need
no test runner. Folded chains of such blocks can be evaluated the same
way.

### 9. Tables instead of code

- Lifecycle transitions as a constant table, with a compile-time check
  that every state handles every event.
- The log decoder's per-call-site plan (parsed format string, argument
  layout) built at compile time, so the drain thread does not parse
  format strings.
- Compile-time type identifiers from reflection in place of run-time type
  information, for the one run-time type check that remains.
- The thread table (count, priorities, affinity masks) as a constant.

### 10. Generated text embedded in the binary

The graph description served by `app --graph` is produced at compile time
and stored as a constant, so exporting it starts nothing and cannot
disagree with the running system. Diagnostics from failed checks are
built as constant strings that name the offending member.

## Compile-time linear algebra

Eigen is the obstacle to items 1, 4 and 8 for estimation and control
blocks. It is possible to cover a useful subset at compile time, and
there is precedent for doing so.

**What exists.**

- *Eigen itself* has some support. Its test suite includes a file of
  compile-time checks (coefficient access in `static_assert`, a global
  `constexpr` fixed-size matrix), the class reference for the 5.0 branch
  lists `constexpr` default constructors and coefficient access, and a
  merge request made initialiser-list construction of fixed-size matrices
  usable in constant expressions. That is enough to declare constant
  matrices; it is not arithmetic, products or decompositions. Which
  release contains which part was not established, so it has to be
  checked against the pinned version.
- *WPILib's `ct_matrix`* (the mathematics library of the FIRST robotics
  software) is the closest existing project: a compile-time wrapper for
  `Eigen::Matrix` with `constexpr` versions of multiplication, addition,
  subtraction, dot and cross product, norm, transpose, determinant and
  identity, an implicit conversion back to an Eigen matrix, and
  `constexpr` geometry helpers built on it. It shows the approach works in
  a robotics code base, and it is a reference for the design. Its licence
  and its suitability as a dependency have not been examined.
- *Other libraries*, from general knowledge and not checked: small
  fixed-size linear-algebra libraries aimed at graphics (GLM and similar)
  are largely usable in constant expressions; Blaze has `constexpr`
  construction and access for its static types; GCE-Math provides
  `constexpr` versions of the elementary mathematical functions, which is
  the other half of the problem.

**Proposed approach.** A small compile-time matrix type in the block
package, not a second linear-algebra library:

- `blk::ct_matrix<R, C>` over a `std::array`, with the same layout as
  the unaligned boundary types (`numerics.md`), so conversion in either
  direction is a copy of the scalars or nothing at all.
- A deliberately short list of operations: construction, element access,
  addition, subtraction, scalar and matrix product, transpose, identity,
  dot, cross, norm, determinant and inverse up to 4 by 4, and a Cholesky
  solve for small symmetric systems.
- Each operation is written as one function that branches on
  `if consteval`: during constant evaluation it runs plain loops over the
  array; at run time it forwards to Eigen and so keeps Eigen's
  vectorisation. A block calls one function and gets the right behaviour
  in both situations, with no duplicated block code.
- Elementary functions (`sqrt`, `sin`, `cos`, `exp`) follow the same
  pattern, using the standard library at run time and, at compile time,
  either what the toolchain provides or small series implementations.

**What it is for.** Deriving constants (filter coefficients, fixed
kinematic transforms, gain matrices computed from fixed parameters),
constant-initialising nodes that hold matrices, and compile-time tests of
blocks built from the listed operations.

**What it is not for.** Iterative decompositions (eigenvalues, singular
values), dynamic sizes, or anything large. A block that needs those stays
a run-time block, and the items above simply do not apply to it.

**Risks.**

- Two code paths for the same operation can disagree. Each operation has
  a test that runs both paths on the same inputs and compares within the
  numerical tolerance; they are never compared for equality.
- Scope creep towards a general library. The operation list is fixed in
  the plan and extended only when a concrete block needs it.
- Compile time for larger matrices. Sizes are limited to what control and
  estimation blocks actually use.

## Limits and costs

- **Compile time and compiler memory.** Constant evaluation is an
  interpreter inside the compiler and has operation and loop limits. The
  heavy evaluation is confined to the translation unit that defines the
  deployment, algorithms work on flat index arrays, and compile time is a
  tracked benchmark from stage S0. A deployment that hits the limits
  raises them deliberately in its own build settings.
- **Not everything is `constexpr`.** Eigen supports little beyond
  declaring constant fixed-size matrices in constant expressions. Blocks
  that stay within the compile-time subset above qualify for items 1, 4
  and 8; the rest do not, and the features degrade per node, not per
  deployment.
- **Mathematical functions.** Coefficient calculations need functions
  such as `sin`, `exp` and `sqrt` in constant expressions. C++26 makes
  more of `<cmath>` `constexpr`, and GCC folds many of these as built-ins,
  but what the pinned toolchain supports is to be confirmed in S1. The
  fallback is small `constexpr` implementations in the block package.
- **Compile-time and run-time floating point can differ in the last
  bits.** The compiler evaluates with its own arithmetic, the target with
  its hardware and library. Under the tolerance policy (`numerics.md`)
  that is acceptable, but a test must not compare a compile-time result
  with a run-time one for equality.
- **Tension with install-time variation.** Anything read from the machine
  file cannot be a compile-time constant. The fixed and configurable
  split in item 4 is the control for that, and the principle at the top
  keeps the checks identical on both sides.
- **Diagnostics.** A failed constant evaluation can produce long compiler
  output. Checks report through one function that formats a short
  message, and the conformance tests include expected-failure cases to
  keep the messages readable.

## Open points

Recorded as M22 in `questions.md`.

- Are there parameters that are the same on every machine and worth
  fixing at build time?
- Is unit checking on links wanted, and are units already written down
  anywhere for existing ports?
- Are execution-time budgets known for existing components, so that the
  budget check would have real numbers to work with?
- Which matrix operations do the estimation and control blocks use
  where their inputs are fixed at build time? That sets the operation
  list of the compile-time matrix type.
- Which Eigen version is pinned? Its own `constexpr` support differs by
  release.
