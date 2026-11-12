# Filter arithmetic: factorisation, structure and precision

Part of the state estimation and calibration plan; see [README.md](README.md) for the index and the work streams.

Streams: A3. Notes on the cost and numerical form of Kalman-style updates. Much of this was written before the estimator became a windowed factor graph over a reduced state at 100 Hz, where computation is no longer a constraint; it is kept as reference for the single-step case and for any filter that runs at a fast rate.

Factorisation helps, but for the state sizes usual in control (a handful
to a few tens of states) it is rarely the largest gain. The options
below are ordered by how much work they typically remove. They are
standard results from the estimation literature, stated from general
knowledge; none has been measured on the code in question.

## 1. Do not run the covariance recursion at all, where that is valid

For a time-invariant model with constant noise covariances and every
measurement present at every step, the covariance and the gain converge
to constants. The steady-state gain can be computed once, during
configuration, by solving the discrete Riccati equation (an iterative
calculation, which is acceptable there). The run-time filter is then the
state prediction and `x + K (z - H x)`: no covariance propagation, no
factorisation, no matrix inverse. Many drive-level observers qualify.
Where the model varies slowly with an operating point, a table of gains
indexed by that point is the same idea.

It does not apply when measurements drop out or arrive at varying rates,
when the model is linearised about the current state, or when the
covariance itself is consumed downstream.

## 2. Exploit the structure of F and H in the propagation

`P = F P Fᵀ + Q` is usually the most expensive line, and F is rarely
dense: integrator chains, block structure between subsystems, identity
blocks. H very often selects or differences a few states, so `H P Hᵀ` is
a sub-block of P, not a product.

- With a symbolic model (`kinematics.md`), the
  generator emits the propagation element by element, using the sparsity
  of F and H, computing only one triangle of P, and sharing common
  subexpressions. For filters of this size this is typically the biggest
  single saving available, and it needs no factorisation.
- Without generation, the same structure can be used by hand with block
  operations, at the cost of the code complexity that generation avoids.
- Subsystems that are not coupled through F, Q or H are separate, smaller
  filters. Cost grows with the cube of the state size, so two filters of
  half the size cost about a quarter.

## 3. Sequential scalar updates in place of a matrix solve

When the measurement noise covariance is diagonal, the measurement
update can process one scalar measurement at a time. Each scalar update
needs one division and work proportional to the square of the state
size; no innovation covariance matrix is formed and nothing is
factorised. Correlated measurement noise can be brought to this form by
whitening with a factor of R that is computed once.

This form suits a real-time system for reasons beyond speed:

- a missing or rejected measurement is simply skipped, so a varying
  number of valid measurements needs no dynamically sized matrices;
- gating of outliers is a per-measurement comparison;
- the operation count per measurement is fixed.

## 4. When a matrix solve is needed, use the right factorisation

- The innovation covariance is symmetric positive definite. Compute the
  gain by a Cholesky factorisation and two triangular solves, never by an
  explicit inverse. It costs roughly half of a general factorisation and
  is better conditioned.
- Plain Cholesky has no pivoting, so its operation count is fixed, which
  matters for timing. A non-positive pivot means the matrix has lost
  positive definiteness; that is reported as an error from the block, not
  ignored.
- Compute and store one triangle of symmetric matrices, and use
  symmetric rank updates for the covariance, so that symmetry holds by
  construction instead of being restored afterwards.
- For very small measurement dimensions (up to three or four) a
  closed-form, unrolled solve beats a general routine; generated code
  produces this automatically.
- Factor constant matrices once. Whitening factors of R and Q, for
  parameters fixed at build time, can be compile-time constants
  (`../rtt3/compile-time.md`); otherwise they are computed during configuration.
- If measurements far outnumber states, the information form inverts a
  matrix of the state size instead of the measurement size.

All of this stays within fixed-size types with in-place factorisation, so
there is no allocation.

## 5. Square-root and UD forms

Propagating a factor of the covariance (a Cholesky factor, or the UD
form of Bierman and Thornton) instead of the covariance itself does not
reduce the operation count much. What it buys:

- the covariance is symmetric and positive semi-definite by construction,
  with no symmetrisation or Joseph-form update needed;
- roughly twice the effective numerical precision, since the factor has
  the square root of the condition number.

The second point is where a speed-up can come from: a square-root filter
can often run in single precision where a conventional one needs double.
On the ARM target that doubles the number of values per vector register
and halves memory traffic. This is a design decision per filter and needs
the filter's consistency checked in simulation, not assumed.

The UD measurement update is naturally sequential and combines with
option 3.

## Single precision with a factored covariance

The idea of factorising so that the filter fits 32-bit lanes without
losing precision is sound, with two qualifications: which factorisation,
and which part of the filter the precision argument covers.

**Which factorisation.** For a covariance the right one is the symmetric
form of LU: `P = U D Uᵀ` with unit-triangular U and diagonal D (or
equivalently `L D Lᵀ`), or the Cholesky factor. A general LU
factorisation with pivoting is not suitable for carrying the covariance:
it does not preserve symmetry or positive definiteness, and it gives no
precision benefit. For solving with a sensor's innovation covariance the
same holds; Cholesky or `L D Lᵀ`, not general LU.

**Why it recovers the precision.** A conventional filter squares the
dynamic range of the problem: the condition number of P is the square of
that of its factor. Single precision carries about seven decimal digits,
so a conventional covariance recursion starts to fail when the ratio
between the best and worst known directions approaches what seven digits
can hold. Working on the factor halves the exponent, which is why a
factored filter in single precision behaves, for the covariance and the
gain, about as well as a conventional one in double.

**What the argument does not cover: the state and the residual.** The
factor only protects the covariance arithmetic. The state vector and the
innovation `z - h(x)` are still limited by seven digits if they are held
in single precision. A position of a kilometre is then resolved to about
a tenth of a millimetre, and subtracting two nearly equal large numbers
to form a residual can lose most of the remaining digits. This, not the
covariance, is usually where single precision actually hurts.

**So: mixed precision.** The parts split cleanly by cost.

| Part | Work per step | Precision |
|---|---|---|
| Nominal state, prediction of the state, measurement prediction and residual | Proportional to the state size: a few scalar operations | Double |
| Covariance factor, its propagation, gains | Proportional to the square or cube of the state size: the bulk | Single |

- An error-state formulation makes the split natural: the nominal state
  is held and propagated in double, the filter estimates a small
  deviation, and only the deviation's covariance factor and the gains are
  single precision. The residual is formed in double and converted.
- Scale the error states so that their standard deviations are of
  comparable magnitude. This improves conditioning in any precision and
  costs nothing at run time when the scaling is part of the symbolic
  model.
- Keep coordinates relative to a local origin where large offsets would
  otherwise eat digits.

**Which factored form.**

- *UD (Bierman and Thornton).* No square roots. The measurement update
  is scalar and sequential, taking each measurement's variance directly,
  which matches the per-sensor structure above. The time update is a
  weighted Gram-Schmidt procedure that is inherently sequential, so it
  vectorises less well than a plain product.
- *Cholesky factor with a QR-based time update.* More regular
  arithmetic, which suits vector code better; needs square roots, which
  are a single cheap instruction on both target architectures.
- Either way, the sparsity of the transition Jacobian is still used: the
  product of the Jacobian with the factor is generated from the symbolic
  model, and only the re-triangularisation is generic.

**How much speed to expect.** Single precision doubles the values per
vector register (four instead of two with NEON, eight instead of four
with AVX2) and halves the memory touched. For large dense blocks that
approaches a factor of two. For a filter with few states, where the
generated code is mostly scalar and already sparse, the gain can be much
smaller, and a factored time update costs somewhat more than the
conventional product it replaces. Whether the change pays for itself is
a measurement, not a given; the benchmark compares the conventional
double filter, the factored double filter and the mixed-precision
factored filter on the target.

**Hazards particular to single precision.**

- *Underflow.* Variances are squares of small numbers, and with
  subnormals flushed to zero (`../rtt3/numerics.md`, "Per-thread floating-point state") a
  very small entry of D becomes exactly zero and is later divided by. A
  floor on the diagonal, and the state scaling above, prevent it.
- *Cancellation in the residual*, addressed by forming it in double.
- *Accumulated drift over long runs.* Single precision errors are larger,
  so the comparison with the reference must include long recordings and
  the worst-conditioned phases: start-up with a large initial
  covariance, very accurate sensors, long periods without a measurement.

**Acceptance.** The mixed-precision filter replaces the double one only
if, on recorded data and in simulation, its state and reconstructed
covariance stay within the agreed tolerance of the double reference and
its consistency statistics are unchanged.

## Applied to the case at hand: an EKF with per-sensor varying covariance

The filter in question is an extended Kalman filter in which each
sensor's measurement covariance varies at run time. That narrows the
options above:

- **Option 1 is out.** The model is linearised about the current state
  and the measurement covariance changes, so there is no steady-state
  gain to precompute.
- **Option 2 is the main saving.** The state-transition Jacobian and each
  sensor's measurement Jacobian come from the symbolic model, so the
  propagation and the per-sensor products are generated with their
  sparsity and symmetry used. This is also where the Jacobians stop being
  hand-derived.
- **Option 3 fits the problem's shape, sensor by sensor.** Different
  sensors are independent of each other, so the joint measurement
  covariance is block diagonal with one block per sensor, and the update
  can be done one sensor at a time:
  - a sensor whose own noise is uncorrelated between its components is
    processed as scalar updates, with the current variances passed in as
    ordinary inputs. A varying variance costs nothing extra;
  - a sensor with correlated components is processed as one small block:
    a Cholesky solve of its innovation covariance, of the sensor's own
    dimension (typically up to three or six), unrolled by the generator.
    No factorisation of anything larger is needed anywhere in the filter;
  - sensors arriving at different rates, or missing in a given step, are
    handled by updating with whichever are present.
- **Option 5** remains available if robustness or single precision
  becomes a concern; the UD form takes a per-measurement variance
  directly.

Points that are specific to doing this with an EKF:

- *Linearisation point.* Updating sensor by sensor is equivalent to one
  joint update only if every sensor's Jacobian is evaluated at the same
  prior state. Re-linearising at the partially updated state between
  sensors is also legitimate, and often slightly more accurate, but the
  result then depends on the order of the sensors. Either is acceptable;
  the choice is made once, written down, and made the same way in the
  simulator.
- *Covariance as data.* Each measurement travels with its covariance and
  a validity flag as one port data type, so the estimator block's inputs
  are fixed-size and a sensor that reports nothing in a step is a flag,
  not a missing message. The covariance is carried as a diagonal or as
  one triangle, matching how the sensor's update is done.
- *Guarding the varying covariance.* A covariance supplied at run time
  can be wrong: zero, negative, non-finite, or absurdly small. The
  estimator checks it before use and treats a bad one as an invalid
  measurement. The innovation test for outliers uses the same per-sensor
  innovation covariance that the update already computes.
- *Covariance update form.* With gating and varying noise, the symmetric
  Joseph form or a square-root form is preferred to the short form,
  which can lose positive definiteness when a gain is modified or a
  measurement is strongly weighted.
- *Worst-case time.* The cost of a step depends on how many sensors
  report. The execution-time budget for the block is that of the step in
  which all of them do, and the per-sensor structure makes that bound
  easy to state.

## Suggested order of attack

For the EKF described above:

1. Generate the propagation and the per-sensor Jacobian products from the
   symbolic model, using structure and symmetry.
2. Update sensor by sensor: scalar updates where a sensor's components
   are uncorrelated, a small Cholesky solve where they are not.
3. Use a covariance update form that keeps symmetry and positive
   definiteness.
4. Consider a square-root form if robustness is a problem or single
   precision is wanted.

For a time-invariant filter elsewhere in the system, check first whether
a steady-state or scheduled gain is valid.

## Checking a restructured filter

A restructured filter is compared with the reference implementation on
recorded data, state and covariance both, within tolerance. In
simulation, where the true state is known, the usual consistency
statistics (normalised estimation error and innovation) show whether the
filter is still correctly tuned; a speed-up that makes the filter
overconfident is not a gain.
