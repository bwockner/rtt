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

## Structured matrices: full 4 by 4 form without redundant operations

**Aim.** A matrix with known structure, such as a projection onto a
plane, an affine transform or an axis selector, should be written and
composed as an ordinary 4 by 4 matrix, while the generated code performs
only the arithmetic that the structure actually requires.

**Why the optimiser will not do this by itself.** With constant entries
and full inlining one might expect the compiler to drop multiplications
by zero. Under IEEE rules it may not: `x * 0.0` is NaN when `x` is NaN or
infinite and has a sign that depends on `x`, and `x + 0.0` changes a
negative zero. The compiler removes these only with the finite-math and
signed-zero relaxations, and finite-math is excluded here
(`numerics.md`). Multiplication by one is removed, but zeros are not. So
if the structure is to be exploited, it has to be known to the type
system, not left to the optimiser.

**Two different kinds of structure.** They need different mechanisms.

*Sparsity: entries known to be zero, one or minus one.* An affine
transform has a last row of 0 0 0 1; a projection onto a coordinate plane
has a zero row and column; a scaling is diagonal.

- `blk::smat<R, C, P>` is a matrix whose pattern `P` is a compile-time
  value recording, per entry, whether it is zero, one, minus one or a
  stored value.
- Sums, products, transposes and application to a vector compute the
  pattern of the result with `consteval` functions (zero times anything
  is zero, one times `x` is `x`, a sum of zeros is zero) and emit
  straight-line scalar code for the remaining terms only. The compiler
  then vectorises and removes common subexpressions as usual.
- The product of two patterned matrices is itself patterned, so structure
  survives through a chain of compositions without the author tracking
  it. Affine composed with affine stays affine; the product with a
  general matrix degrades to general.
- Storage is the full R by C array by default, so the object is a 4 by 4
  matrix in memory and converts to a boundary type or an Eigen matrix
  without work. Only the operations are skipped. A compact storage option
  can come later if size matters.

*Low rank and factored form: dense but cheaper than it looks.* A
projection onto an arbitrary plane with unit normal `n` is `I - n nᵀ`. It
has rank three, but no entry of its upper block is zero, so sparsity
gains nothing. The saving is in the factored form: applying it to a
vector is `x - n (n · x)`, a dot product and a scaled subtraction, and
composing it never needs the nine products of a dense multiply.

- A small set of factored types, each logically a 4 by 4 matrix:
  identity plus rank-one update (projections and reflections), rotation
  with translation (rigid transforms), diagonal scaling, and their
  affine combinations.
- Each provides `matrix()` to materialise the full form, application to
  points and vectors using the factors, and products with each other that
  stay factored where the result has a closed form and fall back to a
  patterned or general matrix where it does not.
- Constructors name the intent: `blk::project_onto_plane(n, d)`,
  `blk::reflect(n, d)`, `blk::rigid(R, t)`, `blk::select_axes(...)`.

**Rank deficiency as a type property.** Because the structure is known,
some mistakes can be rejected at build time: asking for the inverse of a
matrix whose pattern has a zero row or column, or of a projector, is a
compile error with a message, where a dense library would return
non-finite values at run time. A pseudo-inverse or a solve restricted to
the range is offered for the factored types where it has a closed form.

**Semantics to be aware of.** Skipping a multiplication by a structural
zero means a NaN or infinity in the other operand does not propagate
through that term, where the dense computation would have produced NaN.
The result is the mathematically intended one, and non-finite inputs are
caught by the finite check at the block boundary, but it is a difference
from the dense path and is documented as such.

**Compile time and run time.** The patterned and factored types use the
same `if consteval` scheme as `blk::ct_matrix`: plain scalar code is
valid in both, so these types work in constant expressions without a
second implementation.

**Existing work this draws on**, from general knowledge and not checked
for this plan:

- Eigen has fixed special cases but not general patterns: its transform
  class has affine, compact-affine, isometry and projective modes that
  skip work for the last row, and there are diagonal, triangular,
  self-adjoint and permutation types.
- Blaze offers zero, identity, diagonal, symmetric and triangular matrix
  adaptors that are resolved at compile time.
- Geometric-algebra libraries (Klein, Versor) encode which components of
  an element can be non-zero in the type, and projections and reflections
  are their native operations.
- Symbolic code generators (SymForce, CasADi) take the other route: the
  expression is simplified symbolically, zeros and common subexpressions
  included, and plain scalar code is emitted.

**Status: generation first, structured types deferred.** The owner plans
to use SymForce in the simulator libraries for closed-form kinematics.
The motivation is a speed-up already observed from restricting KDL to two
dimensions for approximately planar calculations, which came at the cost
of extra code complexity. Symbolic generation addresses the same need at
the source: the planar restriction is stated once in the symbolic model,
and the generator emits simplified scalar code with the full interface.
See "Generated kinematics" in `blocks.md`.

The patterned and factored matrix types above are therefore not part of
the first version. They remain a documented option for hand-written
blocks and for compositions that happen outside generated code, to be
taken up only if profiling shows such a place. One concrete use has since
appeared: a covariance of the form `B P_q Bᵀ` for a mechanism with few
degrees of freedom (`estimation.md`, "Automatic reduction when the true
degrees of freedom are few").

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
- Which structures matter in practice: projections onto coordinate
  planes or arbitrary planes, rigid and affine transforms, homogeneous
  projective matrices, others?
- SymForce is planned for closed-form kinematics (answered). Is there
  hand-written matrix code outside what it will generate that is hot
  enough to justify the structured types?
