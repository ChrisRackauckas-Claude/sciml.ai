@def rss_pubdate = Date(2026,7,29)
@def rss = """What's New in SciML Since OrdinaryDiffEq v7: Continuation Solvers, Multirate Integrators, and a Pure-Julia Sparse LU"""
@def published = " 29 July 2026 "
@def title = "What's New in SciML Since OrdinaryDiffEq v7: Continuation Solvers, Multirate Integrators, and a Pure-Julia Sparse LU"
@def authors = """<a href="https://github.com/ChrisRackauckas">Chris Rackauckas</a>"""

# What's New in SciML Since OrdinaryDiffEq v7

OrdinaryDiffEq v7 and DifferentialEquations v8 shipped at the end of April 2026, and we
[wrote up what breaks and how to migrate](https://sciml.ai/news/2026/07/28/OrdinaryDiffEqv7/). That post is
about paying a cost. This one is about what the cost bought.

A breaking release is only worth it if it clears the way for things you could not do before. Three months
on, here is what has actually been built on the new foundation. Everything below was run locally against
the currently registered versions; where a feature is on `master` but not yet released, or not yet
co-installable with something else, this post says so rather than letting you find out the hard way.

## Homotopy continuation is now everywhere in the stack

If there is one theme to the last three months, this is it. Continuation methods went from "not really a
thing in SciML" to a first-class problem type with a solver family, and then propagated outward into the
ODE solvers and into ModelingToolkit's initialization. This is the change most likely to fix a problem you
currently have.

The foundation is a new problem type in SciMLBase, `HomotopyProblem`, plus `HomotopyNonlinearFunction`.
Rather than asking a Newton solver to jump straight to the answer from your initial guess, you describe a
*path*: a family of problems `H(u, p, λ)` parameterized by `λ`, where `λ = 0` is something you can solve
trivially and `λ = 1` is the problem you care about. The solver then tracks the solution along the path.

Why this matters: Newton's method is only locally convergent. Continuation is how you get a *globally*
convergent method when you have a bad initial guess, and it is how you handle problems where the solution
branch folds back on itself — which plain parameter-marching fundamentally cannot do.

Here is the fold case, which is the one worth seeing. Take `u³ - 3u = -3 + 6λ` on `λ ∈ [0, 1]`. It has
turning points at `u = ±1`, i.e. `λ = 1/6` and `λ = 5/6`. To get from the `λ = 0` root on the lower sheet
to the `λ = 1` root on the upper sheet along the *connected* branch, `λ` must rise to 5/6, **reverse**
down to 1/6, and then climb to 1:

```julia
using NonlinearSolve, SciMLBase

H(u, p, λ) = [u[1]^3 - 3u[1] - (-3 + 6λ)]

prob = HomotopyProblem(H, [-2.1038034]; λspan = (0.0, 1.0))
sol  = solve(prob, ArcLengthContinuation())
```

```
retcode         = Success
u at λ=1        = 2.103803
target residual = 1.78e-15
```

`ArcLengthContinuation` implements pseudo-arclength continuation: it parameterizes by arclength along the
solution curve instead of by `λ`, so reversing direction in `λ` is not a special case — it is just
continuing to walk forward along the curve. Instrumenting the residual evaluations confirms the solver
climbs past the first turning point and then reverses, exactly as the geometry requires. A
natural-parameter method that only ever increases `λ` cannot do this; it walks off the end of the lower
sheet at the fold and fails.

The continuation surface available today:

| Solver | What it is for |
|---|---|
| `ArcLengthContinuation` | Pseudo-arclength continuation; handles folds and turning points. `predictor = :tangent` gives a true tangent predictor. |
| `HomotopyPolyAlgorithm` | Staged polyalgorithm for `HomotopyProblem` — tries cheap strategies before expensive ones |
| `SimpleHomotopySweep` | Allocation-free continuation for `StaticArrays`, for small systems in hot loops |
| `PseudoTransient` | Pseudo-transient continuation, now with mass-matrix damping |

Two more sit on `master` and are worth watching but are not in a registered release as of writing:
`TaylorHomotopyContinuationJL`, a polynomialization front-end that lets you apply polynomial homotopy
machinery to *non*-polynomial systems, and `FastShortcutHomotopyPolyalg`, the named autodiff-aware default.

### Where it shows up without you asking

The part that matters for ordinary users is that continuation is being wired in as an *implementation*
strategy underneath things you already call:

- **OrdinaryDiffEq** gained `HomotopyNonlinearSolveAlg`, which solves the implicit stage equations of a
  stiff method by step-size homotopy continuation. When a Newton iteration inside an implicit solver
  fails, the classic remedy is to cut `dt` and retry; continuation is a more principled version of the
  same idea. It ships in `OrdinaryDiffEqNonlinearSolve` (v2.4.0), not the `OrdinaryDiffEq` umbrella, so
  reach it with `using OrdinaryDiffEqNonlinearSolve: HomotopyNonlinearSolveAlg`.
- **ModelingToolkit** now routes DAE and ODE initialization through the homotopy continuation solver.
  Consistent initialization of a DAE is exactly the "solve a hard nonlinear system from a guess that may
  be poor" problem that continuation is built for, and initialization failures have historically been one
  of the most common ways a large acausal model refuses to run.

If you have ever hit "initialization failed" on a big model, or watched a stiff solve die with a Newton
convergence failure, this is the work aimed at you.

## A multirate integrator family

`OrdinaryDiffEqMultirate` is a new sublibrary implementing multirate infinitesimal methods for split
problems `du/dt = f₁(u,t) + f₂(u,t)`, where `f₁` is fast and `f₂` is slow. The slow term is frozen across
a macro step while the fast term is integrated with `m` micro-steps, so you evaluate the expensive slow
right-hand side once per macro step instead of once per micro-step.

Available today in `OrdinaryDiffEqMultirate` v2.6.0:

| Method | Order | Kind |
|---|---|---|
| `MRIGARKERK22a`, `MRIGARKERK22b` | 2 | Explicit MRI-GARK (Sandu 2019) |
| `MRIGARKERK33a` | 3 | Explicit MRI-GARK |
| `MRIGARKERK45a` | 4 | Explicit MRI-GARK |
| `MRIGARKIRK21a` | 2 | Solve-decoupled implicit MRI-GARK |
| `MRIGARKESDIRK34a` | 3 | Solve-decoupled implicit MRI-GARK |
| `MRAB` | — | Multirate Adams–Bashforth (k-step) |
| `MREEF` | adaptive | Multirate Richardson extrapolation, Euler base |
| `MIS` | — | Multirate infinitesimal step (Wensch–Knoth–Galant) |

Usage is the standard `SplitODEProblem`, with `m` (the micro-step count) as a required keyword:

```julia
using OrdinaryDiffEqMultirate, SciMLBase

f_fast!(du, u, p, t) = @. du = -50.0 * u     # fast
f_slow!(du, u, p, t) = @. du = -u            # slow, and expensive in real problems

prob = SplitODEProblem(f_fast!, f_slow!, [1.0, 2.0, 3.0], (0.0, 1.0))
sol  = solve(prob, MRIGARKERK33a(m = 10); dt = 0.01, adaptive = false)
```

Verifying the order claim on that problem by halving `dt`:

```
dt 0.02  -> 0.01     observed order = 3.31
dt 0.01  -> 0.005    observed order = 3.15
dt 0.005 -> 0.0025   observed order = 3.08
```

Converging at order 3, as advertised. And the structural property these methods exist for, measured by
counting right-hand side calls at `dt = 0.01`, `m = 10`:

| Method | fast RHS calls | slow RHS calls |
|---|---|---|
| `MRAB(m=10)` | 1001 | 101 |
| `MRIGARKERK22a(m=10)` | 4001 | 201 |
| `MRIGARKERK33a(m=10)` | 9001 | 301 |
| `MRIGARKERK45a(m=10)` | 20001 | 501 |

Roughly 20-40x fewer slow-term evaluations than fast-term evaluations.

**An honest caveat, because it decides whether this helps you.** That call-count ratio is a structural
property, not a speedup. Whether it becomes wall-clock time depends on two things being true: your slow
term must be genuinely expensive relative to the fast term, *and* your macro step size must be limited by
the slow dynamics rather than the fast ones. I tried to construct a synthetic benchmark showing a clean
win over `Tsit5` on the combined right-hand side and could not — on a fast oscillation riding a slow
decay, the macro step is still constrained by the need to resolve the oscillation, and single-rate `Tsit5`
was competitive per unit of accuracy. These methods are aimed at problems like atmospheric dynamics, where
the fast acoustic modes are cheap and local while the slow physics is expensive and global. If that is not
your problem shape, measure before switching.

## New solvers

Verified present in the current registered sublibraries:

- **`Rodas3d`** (`OrdinaryDiffEqRosenbrock` v2.6.0) — an L-stable, stiffly accurate Rosenbrock method
  aimed specifically at DAE integration that approaches a steady state. On the Robertson DAE to `t = 1e5`
  it lands on the same answer as `Rodas5P` to six significant figures and satisfies the algebraic
  constraint to `1.1e-16`, taking more steps as its lower order implies. If you integrate DAEs to steady
  state and have had trouble with the higher-order Rodas methods there, this is the one to try.
- **`Rodas4PW`** (`OrdinaryDiffEqRosenbrock` v2.6.0) — a W-method variant of `Rodas4P`.
- **`ARS222`, `ARS232`, `ARS343`, `ARS443`, `BHR553`** (`OrdinaryDiffEqSDIRK` v2.8.1) — IMEX Runge–Kutta
  schemes (Ascher–Ruuth–Spiteri, and Boscarino–Russo's BHR(5,5,3)). Useful for convection–diffusion-shaped
  problems where you want to treat one operator implicitly and the other explicitly.
- **`ESDIRK325L2SA`** — the Kennedy–Carpenter (2019) ESDIRK3(2)5L[2]SA method, joining the existing
  `ESDIRK436L2SA2` / `ESDIRK437L2SA` / `ESDIRK547L2SA2` / `ESDIRK54I8L2SA` / `ESDIRK659L2SA` family, plus a
  generic stage-predictor menu for ESDIRK methods.

On `master` but not yet in a registered release: `MSRK10` (Stepanov order-10 explicit RK) and the
Runge–Kutta–Gegenbauer stabilized methods.

## LinearSolve v5: a pure-Julia sparse LU, and eigenvalue problems

`SupernodalLUFactorization` is a pure-Julia supernodal LU (a Schenk–Gärtner-style algorithm via
PurePardiso.jl). The interesting property is not raw speed — it is that it is *Julia*, with no SuiteSparse
C dependency in the path, which matters for static compilation, trimming, and unusual element types.

Measured on a 2D 5-point Laplacian, median of 3 runs after warmup:

| n | `LUFactorization` (UMFPACK) | `SupernodalLUFactorization` | `KLUFactorization` |
|---|---|---|---|
| 4,900 | 0.174 s | 0.019 s | 0.012 s |
| 10,000 | 0.049 s | 0.047 s | 0.039 s |

Relative residuals were ~1e-13 for all three. It is competitive on structured sparse problems — which is
why the structured-sparse default LU now routes to it — though on an unstructured matrix with random
fill I measured it slower than UMFPACK (1.12 s vs 0.52 s at n=4000). As always, the default polyalgorithm
exists so you do not have to make this call yourself.

The other notable addition is a genuinely new problem type: **`EigenvalueProblem`**, with
`EigenvalueSolution` and `EigenvalueTarget`, backed by dense, Arpack, ArnoldiMethod, KrylovKit, and
Jacobi–Davidson solvers. Eigenvalue computations now get the same swappable-algorithm treatment as linear
and nonlinear solves:

```julia
using LinearSolve, SciMLBase

sol = solve(EigenvalueProblem([2.0 1.0; 1.0 3.0]))
sol.u        # [3.618033988749895, 1.381966011250105]  — eigenvalues
sol.vectors  # eigenvectors
sol.alg      # DenseEigen()
sol.retcode  # Success
```

Also in v5: lightweight solutions (`solve!` no longer populates `LinearSolution.cache`, which removes a
large retention footprint), `warm_start` on `KrylovJL_GMRES`/`FGMRES`, BLAS LU workspace reuse across
refactorizations, and automatic handling of nonstructural zeros.

**The caveat you need before you `Pkg.add`:** LinearSolve v5 is **not currently co-installable with
OrdinaryDiffEq v7.1.3**. The ODE sublibraries (`OrdinaryDiffEqDefault`, `OrdinaryDiffEqDifferentiation`,
`OrdinaryDiffEqNonlinearSolve`, `OrdinaryDiffEqRosenbrock`) cap LinearSolve at v4, so adding both gets you
LinearSolve v4.3.0 and no `SupernodalLUFactorization`. If you want LinearSolve v5 features today, use it
standalone; the compat bump is in progress, and the OrdinaryDiffEq change that makes Newton–Krylov
integrators default to a Hegedüs warm start is already on `master` waiting on LinearSolve 5.1.

## Sensitivity analysis: implicit DAE adjoints

SciMLSensitivity gained adjoint sensitivity support for **fully implicit `DAEProblem`s**, covering both
index-1 and Hessenberg index-2 systems. Getting gradients through a fully implicit DAE has been a
long-standing gap — you could differentiate mass-matrix ODEs, but genuinely implicit `f(du, u, p, t) = 0`
formulations were not covered. If you are calibrating or optimizing a model expressed that way, this is
the unlock.

Alongside it: **`SundialsAdjoint`**, which uses the CVODES C adjoint interface directly rather than
reimplementing it, and an `EnzymeVJP` dispatch for non-diagonal-noise SDE adjoints.

## Compile time: `AutoDePSpecialize`

A new specialization level, `AutoDePSpecialize`, landed across SciMLBase, OrdinaryDiffEq, and
NonlinearSolve. Where `AutoSpecialize` (the v7 default) reduces recompilation across right-hand side
function types, `AutoDePSpecialize` goes after the *parameters*: it de-specializes non-`isbits` `p` via an
`OpaqueRef`, so the solver does not recompile for every distinct parameter container type.

This is aimed at the case where you have a large struct-of-arrays parameter object, or a
ModelingToolkit-generated parameter container, and you were paying a fresh compilation for it. It is
opt-in.

## The unglamorous half: allocations and step counts

A large fraction of the merged work in this window is performance grinding that produces no new API. It is
worth listing because it is where day-to-day solve time actually comes from:

- **ROCK2 / ROCK4 / RKC**: stage buffers are now rotated instead of copied in the in-place loops, and the
  spectral radius is recomputed every 25 steps by default rather than far more often. Stabilized explicit
  methods are used precisely on large problems where both of those were real costs.
- **Rosenbrock**: stage accumulation loops fused into single-sweep SIMD kernels.
- **ExponentialRK**: per-step allocations removed from the EPIRK / Exp4 / EXPRB53s3 steppers, and residual
  column-slice updates changed to views.
- **Implicit solvers**: redundant sparse-Jacobian structure rebuilds skipped in `calc_J!`, W refactorized
  on the linear path when `γdt` drifts, and the SDIRK error estimate smoothed by reusing the inner W
  factorization.
- **`NonlinearSolveAlg`**: an inner termination check that could never fire was removed, inner failures are
  now surfaced instead of swallowed, and the integrator decides convergence rather than the inner solver.

Also structural: `GlobalDiffEq.jl` moved into the OrdinaryDiffEq monorepo as `lib/GlobalDiffEq`, following
`DiffEqBase`, on the same reasoning — packages that must version in lockstep should release in lockstep.

## What to actually do with this

If you are on v7 already:

- **Hitting DAE initialization failures or Newton convergence failures on stiff problems?** The
  continuation work is aimed directly at you, and much of it is already wired in underneath.
- **Integrating DAEs to steady state?** Try `Rodas3d`.
- **Solving structured sparse systems and want fewer C dependencies?** LinearSolve v5's
  `SupernodalLUFactorization`, with the co-installability caveat above.
- **Differentiating a fully implicit DAE?** That now works.
- **Have a genuine fast/slow split with an expensive slow term?** The multirate family is worth
  benchmarking — with emphasis on *benchmarking*.

And the v7 advice that keeps paying off: depend on the specific sublibrary rather than the umbrella. Beyond
the load-time win, `OrdinaryDiffEqRosenbrock` at v2.6.0 has `Rodas3d` while the `OrdinaryDiffEq` umbrella
at v7.1.3 still resolves it to v2.4.2. Naming the sublibrary you need is how you get the newest solvers
first.

As always, the per-repo release notes are the exhaustive record, and questions are welcome on
[the SciML Zulip](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged).
