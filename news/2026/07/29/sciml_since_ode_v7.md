@def rss_pubdate = Date(2026,7,29)
@def rss = """What's New in SciML Since OrdinaryDiffEq v7: Continuation Solvers, Multirate Integrators, and a Pure-Julia Sparse LU"""
@def published = " 29 July 2026 "
@def title = "What's New in SciML Since OrdinaryDiffEq v7: Continuation Solvers, Multirate Integrators, and a Pure-Julia Sparse LU"
@def authors = """<a href="https://github.com/ChrisRackauckas">Chris Rackauckas</a>"""

# What's New in SciML Since OrdinaryDiffEq v7

OrdinaryDiffEq v7 and DifferentialEquations v8 shipped at the end of April, and last week we
[wrote up what breaks and how to migrate](https://sciml.ai/news/2026/07/28/OrdinaryDiffEqv7/). That was
a long list of things you have to go fix, which is a fair description of a breaking release but not a
very encouraging one. The reason we put everyone through it was to be able to build things that the old
foundation could not support, so three months on, here is what got built.

Everything below was run locally against the currently registered versions. Where something is sitting on
`master` but not tagged, or where two packages that both sound useful cannot currently be installed
together, I've said so. Those are exactly the details a summary written from commit messages gets wrong,
and you would find them out the hard way about ten minutes after reading this.

## Continuation methods, everywhere

If the last three months had a theme, this is it. Homotopy continuation went from something SciML didn't
really do to a first-class problem type with a solver family behind it, and then started showing up
underneath things you already call.

The foundation is `HomotopyProblem` in SciMLBase, along with `HomotopyNonlinearFunction`. Instead of
asking Newton to jump straight to the answer from whatever guess you had lying around, you describe a
path: a family `H(u, p, λ)` where `λ = 0` is something trivially solvable and `λ = 1` is the problem you
care about, and the solver tracks the solution along it. Newton's method is only locally convergent, and
continuation is the standard answer to that. It's how you get global convergence from a bad guess, and
more importantly it's how you handle a solution branch that folds back on itself, which parameter
marching cannot do no matter how small you make the steps.

The fold case is the one to look at. Take `u³ - 3u = -3 + 6λ` over `λ ∈ [0, 1]`. There are
turning points at `u = ±1`, which is `λ = 1/6` and `λ = 5/6`. Getting from the `λ = 0` root on the lower
sheet to the `λ = 1` root on the upper sheet along the connected branch means `λ` has to climb to 5/6,
reverse all the way back down to 1/6, and only then go up to 1:

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

`ArcLengthContinuation` parameterizes by arclength along the solution curve instead of by `λ`, so
reversing direction isn't a special case that needs handling, it's just continuing to walk forward along
the curve. Instrumenting the residual evaluations confirms it climbs past the first turning point and
then heads back down, exactly as the geometry demands. A natural-parameter method that only ever
increases `λ` walks off the end of the lower sheet at the fold and fails.

Currently released, alongside `ArcLengthContinuation` (which also takes `predictor = :tangent` for a true
tangent predictor): `HomotopyPolyAlgorithm`, a staged polyalgorithm that tries the cheap strategies before
the expensive ones; `SimpleHomotopySweep`, an allocation-free version for StaticArrays and small systems
in hot loops; and `PseudoTransient`, which picked up mass-matrix damping. Two more are on `master` and
worth watching but not yet tagged: `TaylorHomotopyContinuationJL`, which polynomializes non-polynomial
systems so the polynomial homotopy machinery applies to them, and `FastShortcutHomotopyPolyalg`, the
autodiff-aware default.

The part that matters for people who don't want to think about any of this is that continuation is being
wired in as an implementation strategy underneath existing APIs. OrdinaryDiffEq now has
`HomotopyNonlinearSolveAlg`, which solves the implicit stage equations of a stiff method by step-size
homotopy continuation. When a Newton iteration inside an implicit solver fails the traditional response
is to cut `dt` and try again, and this is a more principled version of the same instinct. It ships in
`OrdinaryDiffEqNonlinearSolve` v2.4.0 and not the umbrella, so you want
`using OrdinaryDiffEqNonlinearSolve: HomotopyNonlinearSolveAlg`. Meanwhile ModelingToolkit now routes DAE
and ODE initialization through the continuation solver, which is a natural fit: consistent initialization
of a DAE is exactly the problem of solving a hard nonlinear system from a guess that might be poor, and
initialization failure has been one of the most common ways a large acausal model refuses to run at all.

So if you've ever stared at an "initialization failed" message on a big model, or watched a stiff solve
die on Newton convergence, this is the work aimed squarely at you.

## Multirate integrators

`OrdinaryDiffEqMultirate` is a new sublibrary of multirate infinitesimal methods for split problems
`du/dt = f₁(u,t) + f₂(u,t)` where `f₁` is fast and `f₂` is slow. The slow term is frozen across a macro
step while the fast term gets `m` micro-steps, so the expensive slow right-hand side is evaluated once per
macro step instead of once per micro-step.

What's in v2.6.0: `MRIGARKERK22a` and `MRIGARKERK22b` (order 2), `MRIGARKERK33a` (order 3) and
`MRIGARKERK45a` (order 4) from the Sandu 2019 explicit MRI-GARK family; `MRIGARKIRK21a` and
`MRIGARKESDIRK34a` for the solve-decoupled implicit versions; plus `MRAB` (multirate Adams–Bashforth),
`MREEF` (Richardson extrapolation on an Euler base, adaptive) and `MIS` (Wensch–Knoth–Galant). It's a
standard `SplitODEProblem`, with `m` as a required keyword:

```julia
using OrdinaryDiffEqMultirate, SciMLBase

f_fast!(du, u, p, t) = @. du = -50.0 * u
f_slow!(du, u, p, t) = @. du = -u

prob = SplitODEProblem(f_fast!, f_slow!, [1.0, 2.0, 3.0], (0.0, 1.0))
sol  = solve(prob, MRIGARKERK33a(m = 10); dt = 0.01, adaptive = false)
```

Halving `dt` on that problem confirms the order claim, giving observed orders of 3.31, 3.15 and 3.08 as
the step size comes down, converging on 3 as advertised. Counting right-hand side calls at `dt = 0.01`
with `m = 10` shows the structural property these methods exist for. `MRAB` does 1001 fast evaluations
against 101 slow ones, `MRIGARKERK33a` does 9001 against 301, and `MRIGARKERK45a` does 20001 against 501.
Somewhere between twenty and forty times fewer evaluations of the expensive half.

Now the caveat, because it decides whether any of this helps you. That ratio is a structural property of
the method, not a speedup. It turns into wall-clock time only if your slow term is expensive relative to
the fast one *and* your macro step is limited by the slow dynamics instead of the fast ones.
I tried to build a synthetic benchmark showing a clean win over `Tsit5` on the combined right-hand side
and could not manage it: on a fast oscillation riding a slow decay the macro step stays pinned by the
need to resolve the oscillation, and single-rate `Tsit5` was competitive per unit of accuracy. These
methods are aimed at things like atmospheric dynamics, where the fast acoustic modes are cheap and local
and the slow physics is expensive and global. If your problem doesn't have that shape, benchmark before
switching.

## New solvers

`Rodas3d` in `OrdinaryDiffEqRosenbrock` v2.6.0 is an L-stable, stiffly accurate Rosenbrock method built
for DAEs that are integrating toward a steady state. On the Robertson problem out to `t = 1e5` it agrees
with `Rodas5P` to six significant figures and satisfies the algebraic constraint to `1.1e-16`, taking
1109 steps against Rodas5P's 193, about what its lower order implies. If you integrate DAEs to steady
state and the higher-order Rodas methods have given you trouble there, try it.

Also newly available: `Rodas4PW`, a W-method variant of `Rodas4P`, in the same package. In
`OrdinaryDiffEqSDIRK` v2.8.1 there's a batch of IMEX Runge–Kutta schemes: `ARS222`, `ARS232`, `ARS343`
and `ARS443` from Ascher–Ruuth–Spiteri, plus Boscarino–Russo's `BHR553`. Those are the right shape for
convection–diffusion problems where you want one operator implicit and the other explicit. And
`ESDIRK325L2SA`, the Kennedy–Carpenter 2019 method, joins the existing `ESDIRK436L2SA2`,
`ESDIRK437L2SA`, `ESDIRK547L2SA2`, `ESDIRK54I8L2SA` and `ESDIRK659L2SA` family, which also picked up a
generic stage-predictor menu.

On `master` but not yet tagged: `MSRK10`, Stepanov's order-10 explicit RK, and the Runge–Kutta–Gegenbauer
stabilized methods.

## LinearSolve v5

`SupernodalLUFactorization` is a pure-Julia supernodal LU, a Schenk–Gärtner-style algorithm via
PurePardiso.jl. The interesting thing about it isn't raw speed, it's that there's no SuiteSparse C
dependency anywhere in the path. That matters if you are doing static compilation or trimming, and it
matters if your element type is something the C libraries have never heard of.

On a 2D five-point Laplacian, median of three runs after warmup, it's competitive. At n = 4,900 it takes
0.019s against UMFPACK's 0.174s and KLU's 0.012s, and by n = 10,000 the three have converged to roughly
0.047s, 0.049s and 0.039s respectively, with relative residuals around 1e-13 for all of them. That's why
the structured-sparse default LU now routes to it. On an unstructured matrix with random fill I measured
it slower than UMFPACK (1.12s against 0.52s at n = 4,000), which is roughly what you'd expect from a
supernodal algorithm handed a matrix with no supernodes to find. The default polyalgorithm exists so you
don't have to make this call yourself.

The other addition is a whole new problem type. `EigenvalueProblem`, with `EigenvalueSolution` and
`EigenvalueTarget`, backed by dense, Arpack, ArnoldiMethod, KrylovKit and Jacobi–Davidson solvers, brings
eigenvalue computations into the same swappable-algorithm world as linear and nonlinear solves:

```julia
using LinearSolve, SciMLBase

sol = solve(EigenvalueProblem([2.0 1.0; 1.0 3.0]))
sol.u        # [3.618033988749895, 1.381966011250105]  eigenvalues
sol.vectors  # eigenvectors
sol.alg      # DenseEigen()
sol.retcode  # Success
```

Also in v5: solutions got lighter, since `solve!` no longer populates `LinearSolution.cache` and so no
longer retains a large chunk of memory you probably didn't want; `KrylovJL_GMRES` and `FGMRES` gained
`warm_start`; BLAS LU workspaces are reused across refactorizations; and nonstructural zeros are handled
automatically.

One thing to know before you reach for any of it: LinearSolve v5 cannot currently be installed alongside
OrdinaryDiffEq v7.1.3. The ODE sublibraries `OrdinaryDiffEqDefault`, `OrdinaryDiffEqDifferentiation`,
`OrdinaryDiffEqNonlinearSolve` and `OrdinaryDiffEqRosenbrock` all cap LinearSolve at v4, so asking for
both silently resolves you to v4.3.0 with no `SupernodalLUFactorization` in sight. Use it standalone for now.
The compat bump is in progress, and the OrdinaryDiffEq change that makes Newton–Krylov integrators
default to a Hegedüs warm start is already sitting on `master` waiting on LinearSolve 5.1.

## Adjoints through fully implicit DAEs

SciMLSensitivity gained adjoint sensitivity support for fully implicit `DAEProblem`s, covering index-1
and Hessenberg index-2 systems. This closes a gap that had been open a long time: you could already
differentiate mass-matrix ODEs, but fully implicit `f(du, u, p, t) = 0` formulations weren't covered,
which meant anyone whose model was naturally written that way had to reformulate it before they could
calibrate or optimize anything. Alongside it there's `SundialsAdjoint`, which drives the CVODES C adjoint
interface directly instead of reimplementing it, and an `EnzymeVJP` dispatch for SDE adjoints with
non-diagonal noise.

## Compile time, again

`AutoDePSpecialize` is a new specialization level, landed across SciMLBase, OrdinaryDiffEq and
NonlinearSolve. Where `AutoSpecialize`, the v7 default, cuts recompilation across right-hand side
function types, this one goes after the parameters, de-specializing non-`isbits` `p` behind an
`OpaqueRef` so the solver stops recompiling for every distinct parameter container type. It's aimed at
large struct-of-arrays parameter objects and ModelingToolkit-generated containers, where you were
otherwise paying a fresh compilation for each one. Opt-in.

## The unglamorous half

A good fraction of the merged work in this window adds no API at all, which makes it easy to leave out of
a post like this, and is also where most of your day-to-day solve time comes from.

ROCK2, ROCK4 and RKC now rotate their stage buffers instead of copying them in the in-place loops, and
recompute the spectral radius every 25 steps by default instead of far more often. These are stabilized
explicit methods, so they get used on exactly the large problems where both of those were real costs.
Rosenbrock stage accumulation loops were fused into single-sweep SIMD kernels. The EPIRK, Exp4 and
EXPRB53s3 steppers had their per-step allocations removed and their residual column-slice updates turned
into views. On the implicit side, redundant sparse-Jacobian structure rebuilds are skipped in `calc_J!`,
W gets refactorized on the linear path when `γdt` drifts, and the SDIRK error estimate is smoothed by
reusing the inner W factorization. `NonlinearSolveAlg` lost an inner termination check that could never
fire, started surfacing inner failures instead of swallowing them, and now lets the integrator decide
convergence instead of the inner solver.

Structurally, `GlobalDiffEq.jl` moved into the OrdinaryDiffEq monorepo as `lib/GlobalDiffEq`, following
`DiffEqBase`, for the same reason as before: packages that have to version in lockstep should release in
lockstep.

## What to do with this

If you're already on v7 and hitting DAE initialization failures or Newton convergence failures on stiff
problems, the continuation work is aimed at you and a good deal of it is already wired in underneath. If
you integrate DAEs to steady state, try `Rodas3d`. If you're solving structured sparse systems and would
rather have fewer C dependencies, LinearSolve v5's supernodal LU, with the co-installability caveat
above. If you're differentiating a fully implicit DAE, that works now. And if you have a real fast/slow
split with an expensive slow term, the multirate family is worth benchmarking, with the emphasis on
benchmarking.

One last thing, which is really just the v7 advice continuing to pay off: depend on the specific
sublibrary and not the umbrella. Beyond the load-time argument, `OrdinaryDiffEqRosenbrock` at v2.6.0
has `Rodas3d` while the `OrdinaryDiffEq` umbrella at v7.1.3 still resolves it back to v2.4.2. Naming the
sublibrary you need is how you get new solvers first.

Per-repo release notes remain the exhaustive record, and questions are welcome on
[the SciML Zulip](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged).
