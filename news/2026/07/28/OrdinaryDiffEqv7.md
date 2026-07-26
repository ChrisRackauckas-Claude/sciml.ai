@def rss_pubdate = Date(2026,7,28)
@def rss = """OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It"""
@def published = " 28 July 2026 "
@def title = "OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It"
@def authors = """<a href="https://github.com/ChrisRackauckas">Chris Rackauckas</a>"""

# OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It

OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8 are out. This is the first breaking release of the
ODE stack in a long time, and it comes with a matching bump of **SciMLBase to v3** and
**RecursiveArrayTools to v4**. There is a lot in it, and the [full changelog in
`NEWS.md`](https://github.com/SciML/OrdinaryDiffEq.jl/blob/master/NEWS.md) is long.

This post is not the full changelog. It is the shorter thing you actually want: **the changes most likely
to break your code, ranked by how hard they are to notice.** Everything shown below was run against
OrdinaryDiffEq v7.1.3, SciMLBase v3.39.1, and RecursiveArrayTools v4.3.4 on Julia 1.11, and the outputs
are what those versions actually print.

If you read only one section, read the next one.

## The three changes that break your code silently

Most of v7's breaking changes announce themselves with a loud `ArgumentError` telling you exactly what to
write instead. Those are fine — you run your tests, you get an error, you fix it. The dangerous changes are
the ones where your code keeps running and quietly means something different. There are three of these,
and they are the reason to read a migration guide rather than just bumping and seeing what explodes.

### 1. `sol.retcode == :Success` is now silently `false`

This is the one most likely to bite, because it turns a passing check into a failing one with no error:

```julia
sol = solve(prob, Tsit5())

sol.retcode == :Success        # v7: false — even though the solve succeeded
successful_retcode(sol)        # v7: true  — the correct check
```

`sol.retcode` has been a `ReturnCode.T` enum rather than a `Symbol` for years, with a deprecation warning
on every `Symbol` comparison throughout the v2 series. v3 removed the shim. Comparing an enum to a `Symbol`
is not an error in Julia — it is just `false`. So any code shaped like

```julia
if sol.retcode == :Success
    record_result(sol)
else
    @warn "solver failed"
end
```

now takes the failure branch on every successful solve, and nothing anywhere prints a warning.

**Migration:** use `successful_retcode`, not equality against a specific code.

```julia
successful_retcode(sol)     # ✔ preferred
```

This is better than `sol.retcode == ReturnCode.Success` even after you fix the `Symbol` problem, because
`successful_retcode` accepts every success-ish code — `Success`, `StalledSuccess`, `ExactSolutionLeft`,
`ExactSolutionRight`, `FloatingPointLimit`, and so on. A solve that terminated exactly on a root is a
success, and hardcoding `== ReturnCode.Success` misclassifies it as a failure.

Grep your code base for `retcode ==` and `retcode !=` before you upgrade. It is a two-minute search that
saves a genuinely confusing afternoon.

### 2. `sol[i]` no longer means "the i-th timestep"

RecursiveArrayTools v4 makes `AbstractVectorOfArray` — the parent type of `ODESolution` — an honest
`AbstractArray`. That is the right call: every generic `AbstractArray` consumer (LinearAlgebra,
broadcasting, Zygote adjoints, `StructArrays`) now works on solutions without special-casing, and it
deletes a large pile of manual method overrides in SciMLBase. But it changes what indexing means.

Here is a 3-variable Lorenz solve with 27 saved timesteps, run on v7:

```julia
size(sol)         # (3, 27)   — it's a matrix now
length(sol)       # 81        — total scalar elements (3 × 27)
length(sol.u)     # 27        — number of timesteps
sol[1]            # 1.0       — the first scalar, column-major
sol.u[1]          # [1.0, 0.0, 0.0]  — the first timestep
sol[:, 1]         # [1.0, 0.0, 0.0]  — also the first timestep
first(sol)        # 1.0
first(sol.u)      # [1.0, 0.0, 0.0]
eachindex(sol)    # CartesianIndices — not 1:27
```

So `sol[1]` used to give you a state vector and now gives you a `Float64`. Code that iterates
`for u in sol` used to walk timesteps and now walks scalars. Code that calls `length(sol)` to count
timesteps now gets the element count. None of this errors — you just get 81 where you expected 27, or a
scalar where you expected a vector, and the failure surfaces somewhere far away from the cause.

| Operation | v6 meaning | v7 meaning | Write instead |
|---|---|---|---|
| `sol[i]` | i-th timestep | i-th scalar element | `sol.u[i]` or `sol[:, i]` |
| `length(sol)` | number of timesteps | total scalar count | `length(sol.t)` or `length(sol.u)` |
| `eachindex(sol)` | `1:nsteps` | `CartesianIndices` | `eachindex(sol.u)` |
| `for u in sol` | iterates timesteps | iterates scalars | `for u in sol.u` |
| `first(sol)` / `last(sol)` | first/last timestep | first/last scalar | `first(sol.u)` / `last(sol.u)` |
| `map(f, sol)` | maps over timesteps | maps over scalars | `map(f, sol.u)` |
| `maximum(sol)` | max over timesteps | max over all scalars | `maximum(f, sol.u)` |

**Migration, and the nice part:** `sol.u[i]` means the same thing on v6 *and* v7. So you can do this
search-and-replace today, on your current version, and it is correct before and after the bump. That is
the single highest-value thing you can do to prepare.

The same change applies to `EnsembleSolution`: `sim[j]` → `sim.u[j]`, `sim[i, j]` → `sim.u[j].u[i]`,
`length(sim)` → `length(sim.u)`.

**Escape hatch:** if you have a large code base that assumes timestep-first indexing and you need to buy
time, `RecursiveArrayToolsRaggedArrays.jl` preserves the old semantics:

```julia
using RecursiveArrayToolsRaggedArrays
sol_old = RaggedVectorOfArray(sol)   # sol_old[i] is the i-th timestep, as in v3
```

Treat this as a compatibility layer to unblock a migration, not as the API to write new code against.

### 3. Controller keyword arguments are accepted and then ignored

The adaptive step size controller was refactored from a pile of loose numeric knobs on `solve` into real,
pluggable controller objects. `gamma`, `beta1`, `beta2`, `qmin`, `qmax`, `qsteady_min`, `qsteady_max`, and
`qoldinit` moved onto `PIController` / `PIDController` / `IController` / `PredictiveController`.

This is a good change — you can now write a custom controller and pass `controller = MyController(…)`
instead of the SciML developers adding yet another `solve` kwarg. But there is a sharp edge in the
released version. Those old keyword arguments are still on `solve`'s accepted-keyword list, so passing
them does not error. They simply do nothing:

```julia
for g in (0.1, 0.5, 0.9, 0.99)
    s = solve(prob, Tsit5(); gamma = g, abstol = 1e-8, reltol = 1e-8)
    println("gamma=$g  nsteps=", length(s.t))
end
```

```
gamma=0.1  nsteps=5597
gamma=0.5  nsteps=5597
gamma=0.9  nsteps=5597
gamma=0.99 nsteps=5597
```

Identical step counts across a 10x range of `gamma`. On v6 those four runs would differ substantially.
`qmin`, `qmax`, `beta1`, `beta2`, and `qoldinit` behave the same way — accepted, ignored. (A genuinely
unknown keyword like `totally_bogus_kwarg` *does* error, so this is specific to the controller keywords
still sitting in the allowlist.)

If you tuned your step size controller — and people who tuned it usually had a reason — your solver is
now silently running with default settings. **Grep for `gamma`, `qmin`, `qmax`, `beta1`, `beta2`, and
`qoldinit` in your `solve` calls.**

**Migration:** move them onto a controller object.

```julia
# v6
solve(prob, alg; gamma = 0.9, beta1 = 0.7, beta2 = -0.4)

# v7
using OrdinaryDiffEqCore: PIController
solve(prob, alg; controller = PIController(0.7, -0.4))
```

Note the `using OrdinaryDiffEqCore` — see the import gotcha section below.

## The changes that break loudly

These all raise an error with a helpful message, so they cost you one test run each. Grouped by theme.

### Bool keywords are now typed objects

Every `Bool` solver and `solve` keyword that used to select between code paths is now a typed object.
The reason is type stability: a `Bool` chose between (say) ForwardDiff and FiniteDiff at *runtime*, which
the compiler could not specialize through. A typed object carries the choice in its type.

```julia
# v6
Rosenbrock23(autodiff = true)
solve(prob, alg; verbose = false, alias = true)

# v7
Rosenbrock23(autodiff = AutoForwardDiff())
solve(prob, alg;
    verbose = DEVerbosity(SciMLLogging.None()),
    alias   = ODEAliasSpecifier(alias_u0 = true))
```

The error messages are good. Passing `autodiff = true` on v7 gives you:

```
ArgumentError: Passing a `Bool` for keyword argument `autodiff` is no longer supported.
Use an `ADType` specifier from ADTypes.jl, e.g. `AutoForwardDiff()` or `AutoFiniteDiff()`.
```

The `autodiff` change is the one with a real payoff beyond type stability: because the backend is now an
`ADTypes` object rather than a `Bool` plus a pile of ForwardDiff-specific keywords (`chunk_size`,
`diff_type`, `standardtag`), **every implicit solver now works with every AD backend**. Swapping to Enzyme
is `autodiff = AutoEnzyme()` and nothing else. That also explains why the `{CS, AD, FDT, ST, CJ}` type
parameters were removed from 100+ algorithm structs — five type parameters collapsed into one field.

```julia
# v6
Rodas5P(chunk_size = 12, diff_type = Val{:central}, standardtag = true)

# v7
Rodas5P(autodiff = AutoForwardDiff(chunksize = 12))
Rodas5P(autodiff = AutoFiniteDiff(fdtype = Val(:central)))
Rodas5P(autodiff = AutoEnzyme())     # now just works
```

### `using OrdinaryDiffEq` loads only the default solver set

```julia
# v6
using OrdinaryDiffEq
solve(prob, KenCarp4())     # worked

# v7
using OrdinaryDiffEqSDIRK   # required
solve(prob, KenCarp4())
```

What you still get from the umbrella: `DefaultODEAlgorithm`, `Tsit5`, `AutoTsit5`, `Vern6`–`Vern9`,
`AutoVern6`–`AutoVern9`, `Rosenbrock23`, `Rodas5P`, and `FBDF`. Everything else needs its sublibrary.
The mapping is predictable — `KenCarp*`/`TRBDF2` → `OrdinaryDiffEqSDIRK`, most `Rosenbrock*`/`Rodas*` →
`OrdinaryDiffEqRosenbrock`, `RadauIIA*` → `OrdinaryDiffEqFIRK` — and every family has its own
`lib/OrdinaryDiffEq<X>` directory in the repo.

This is the largest single contributor to v7's time-to-first-solve reduction, together with `ODEFunction`
switching to `AutoSpecialize` by default and the removal of the `Static.jl`, `StaticArrayInterface.jl`,
`Polyester.jl`, and `StaticArrays.jl` dependencies. If you want to go further, import just the one you
need: `using OrdinaryDiffEqTsit5: Tsit5`.

While you are in there: if you are still explicitly choosing `Rodas5`, switch to `Rodas5P`. Very similar
performance profile, considerably more robust accuracy.

### DAE initialization now errors instead of silently fixing

```julia
# v6: inconsistent u0 silently corrected
solve(prob, Rodas5P())

# v7: errors
solve(prob, Rodas5P())
```

On v7 an inconsistent initial condition gives you:

```
DAE initialization failed: your u0 did not satisfy the initialization requirements,
normresid = 0.707… > abstol = 1.0e-6.
```

**Why:** silently fixing an inconsistent initial condition produced wrong answers when the user's `u0` was
actually right and a modeling bug elsewhere made the system look inconsistent. Erroring surfaces the
modeling bug instead of hiding it.

**Migration:** if you want the old behavior, ask for it explicitly.

```julia
using DiffEqBase: BrownFullBasicInit
solve(prob, Rodas5P(); initializealg = BrownFullBasicInit())
```

This one is a loud break rather than a silent one, which is exactly the point of the change.

### `VectorContinuousCallback` fires every simultaneous event

Previously, when several conditions of one `VectorContinuousCallback` crossed zero in the same step, only
the *first* crossing's `affect!` ran. The others were silently dropped even though their roots were inside
the step. Bouncing-ball models, multi-contact mechanics, and threshold state machines were all affected by
this. v7 resolves all simultaneous events and dispatches them in a single `affect!` call — which requires
a new signature:

```julia
# v6: called once per triggering condition, with that condition's index
affect!(integrator, event_index::Int)

# v7: called once per step, with a mask over all conditions
affect!(integrator, simultaneous_events::Vector{Int8})
```

Each entry is `0` (did not trigger), `+1` (upcrossing), or `-1` (downcrossing). Because the sign carries
the crossing direction, `affect_neg!` is no longer used for `VectorContinuousCallback` — one function
handles both directions. (`ContinuousCallback` keeps its `affect!` / `affect_neg!` split.)

```julia
# v6
function affect!(integrator, event_index)
    if event_index == 1
        # ball 1 hit the ground
    elseif event_index == 2
        # ball 2 hit the ground
    end
end
cb = VectorContinuousCallback(condition, affect!, affect_neg!, 2)

# v7
function affect!(integrator, simultaneous_events)
    for i in eachindex(simultaneous_events)
        s = simultaneous_events[i]
        s == 0 && continue
        if i == 1
            # ball 1 hit the ground; s == +1 up, s == -1 down
        elseif i == 2
            # ball 2 hit the ground
        end
    end
end
cb = VectorContinuousCallback(condition, affect!, 2)
```

If you are on an early v7 patch, note that the first releases flipped this sign convention; it was
corrected quickly. Make sure you are on a current v7.1.x.

### Ensemble `prob_func` / `output_func` signatures

```julia
# v6
prob_func = (prob, i, repeat) -> remake(prob, u0 = rand(length(prob.u0)))

# v7
prob_func = (prob, ctx) -> remake(prob, u0 = rand(ctx.rng, length(prob.u0)))
```

`ctx::EnsembleContext` carries `ctx.i` (trajectory index), `ctx.repeat` (retry counter), and `ctx.rng`.
`output_func(sol, i)` becomes `output_func(sol, ctx)` likewise.

**Why this is worth the churn:** the old signature had nowhere to plumb a reproducible per-trajectory RNG,
so ensemble results depended on thread count and scheduling. With `EnsembleContext` you get an RNG derived
from one top-level seed, and `solve(ensemble_prob, alg; seed = 42, trajectories = N)` reproduces regardless
of how many workers run.

### Renames, removals, and tableaus

The renames below all already exist under their new names in SciMLBase v2 / OrdinaryDiffEq v6 with
deprecation warnings, which matters for the upgrade path in the next section:

| Old | New |
|---|---|
| `sol.destats` | `sol.stats` |
| `has_destats(alg)` | `has_stats(alg)` |
| `integrator.u_modified` | `integrator.derivative_discontinuity` |
| `DEAlgorithm` | `AbstractDEAlgorithm` |
| `DEProblem` | `AbstractSciMLProblem` |
| `DESolution` | `AbstractSciMLSolution` |
| `constructDormandPrince()` | `OrdinaryDiffEqExplicitTableaus.DormandPrince()` |
| `OrdinaryDiffEq.False()` / `True()` | `Serial()` / `Threaded()` (from FastBroadcast) |

Tableau functions dropped the `construct` prefix (~80 of them) and are no longer exported, so qualify them
with the sublibrary. DiffEqDevTools v3 removed its own 105 `construct*` re-exports entirely — the
authoritative definitions now live only in `OrdinaryDiffEqExplicitTableaus` /
`OrdinaryDiffEqImplicitTableaus`. `DiffEqDevTools.deduce_Butcher_tableau(alg)` is unchanged.

Also removed: `tuples()`, `intervals()`, `QuadratureProblem` (use `IntegralProblem`), `fastpow` (use
`FastPower.fastpower`), `concrete_solve` (use `solve`), `syms`/`paramsyms`/`indepsym` kwargs, `sol.x` on
optimization solutions (use `sol.u`), and the positional `Alg(stage_limiter!, step_limiter!)` constructors
across 99 explicit RK constructors (use the keyword form).

One more default change worth knowing: **`williamson_condition` now defaults to `false`** on all 2N
low-storage RK methods. That optimization only works for mutable `Array`-style state, so having it on by
default silently misbehaved for `StaticArrays`, GPU arrays, and `ComponentArrays`. Opt back in when you
know your state is a plain `Array`.

## An import gotcha worth knowing before you start

Several of the replacement types are *not* reachable from a plain `using OrdinaryDiffEq`. Checked on
v7.1.3:

| Symbol | Reachable from `using OrdinaryDiffEq`? | Where it lives |
|---|---|---|
| `ODEAliasSpecifier` | yes | — |
| `successful_retcode`, `ReturnCode` | yes | — |
| `DEVerbosity` | **no** | `DiffEqBase`, `OrdinaryDiffEqCore` |
| `PIController`, `PIDController`, `IController` | **no** | `OrdinaryDiffEqCore` |
| `CheckInit`, `NoInit`, `OverrideInit` | **no** | `SciMLBase`, `DiffEqBase`, `OrdinaryDiffEqCore` |
| `BrownFullBasicInit` | **no** | `DiffEqBase`, `OrdinaryDiffEqCore` |
| `Serial`, `Threaded` | **no** | `DiffEqBase` (from `FastBroadcast`) |

So the migration snippet you would naturally write from the changelog:

```julia
using OrdinaryDiffEq
solve(prob, Tsit5(); verbose = DEVerbosity(SciMLLogging.None()))
# ERROR: UndefVarError: `DEVerbosity` not defined in `Main`
```

needs an explicit import:

```julia
using OrdinaryDiffEq, SciMLLogging
using DiffEqBase: DEVerbosity
solve(prob, Tsit5(); verbose = DEVerbosity(SciMLLogging.None()))
```

This is a rough edge in the release rather than an intentional design, and it is being tidied up. Until
then, if a v7 replacement type gives you an `UndefVarError`, the answer is almost always an explicit
import from `DiffEqBase` or `OrdinaryDiffEqCore`.

## The recommended upgrade path: do it in two steps

Do **not** jump from an old environment straight onto v7. Almost every rename above already exists under
its new name in **SciMLBase v2 / OrdinaryDiffEq v6**, with deprecation warnings pointing at the exact call
sites. So:

1. **Stay on v6.** Update to the new names while the deprecation shims still exist: `has_stats`,
   `sol.stats`, `AbstractDEAlgorithm`, `derivative_discontinuity!`, `ODEAliasSpecifier`, `DEVerbosity`,
   `ADTypes`-based `autodiff`, explicit `controller = …` objects, the new tableau names. Change every
   `sol[i]` to `sol.u[i]` — correct on both versions.
2. **Get your tests passing on v6 with zero deprecation warnings.** The warnings are your migration
   checklist.
3. **Then bump to v7.** What is left is the genuinely new breakage: RAT v4 indexing semantics, the ensemble
   `prob_func`/`output_func` signature, struct type parameter removals, the `VectorContinuousCallback`
   signature, and the default changes (`CheckInit`, `williamson_condition`).

Two small steps beat one large one here, mostly because step 2 turns a vague "something changed" into a
concrete list of file-and-line locations.

Before you start, run these three greps. They cover the silent breakages, which the deprecation warnings
cannot help you with:

```
retcode ==
sol[            # and length(sol), eachindex(sol), for … in sol
gamma =         # and qmin, qmax, beta1, beta2, qoldinit in solve calls
```

## DifferentialEquations.jl v8: the umbrella got smaller

Separately from the v7 changes, `using DifferentialEquations` **no longer re-exports the full SciML solver
suite**. In v8 it loads only `OrdinaryDiffEq`. Everything else is now an explicit dependency:

| Topic | v8 |
|---|---|
| ODEs | `using OrdinaryDiffEq` |
| Stochastic ODEs | `using StochasticDiffEq` |
| Delay ODEs | `using DelayDiffEq` |
| Boundary value problems | `using BoundaryValueDiffEq` |
| Jump processes | `using JumpProcesses` |
| Steady state | `using SteadyStateDiffEq` |
| Sundials (CVODE, IDA, ARKODE) | `using Sundials` |
| Linear / nonlinear / optimization | `using LinearSolve` / `NonlinearSolve` / `Optimization` |

**Why:** the old umbrella made every user pay the load time for every solver family, and made it unclear
which package any given algorithm actually came from. Each topic can now version independently, and your
`Project.toml` becomes honest about what your script actually uses. The DiffEqDocs solver pages annotate
every algorithm with its host package.

Note that v7 and v8 are independent — you can use OrdinaryDiffEq v7 directly without the umbrella at all,
which for ODE-only work is what we would suggest.

Two smaller notes in the same family: `DiffEqBase` is now a sublibrary at `lib/DiffEqBase` inside the
OrdinaryDiffEq monorepo, because it was tightly coupled to OrdinaryDiffEq internals and lockstep releases
remove a whole class of compatibility bugs. And `StochasticDelayDiffEq.jl` is deprecated — use
`DelayDiffEq.jl` directly, which has supported SDDE problems for some time. It will not get a
v7-compatible release.

## Was it worth it?

The recurring themes across all of the above are worth stating plainly, because nearly every individual
change is an instance of one of them:

- **Time to first solve.** The scope reduction, `AutoSpecialize` by default, and dropping `Static.jl`,
  `StaticArrayInterface.jl`, `Polyester.jl`, and `StaticArrays.jl` as direct dependencies. Less code
  loaded, more precompilation actually cached.
- **Type stability.** Every `Bool`-as-a-switch became a typed object, so the compiler specializes on the
  choice instead of branching at runtime.
- **Generality beyond ForwardDiff.** `ADTypes` everywhere means every solver gets every AD backend for
  free, instead of a ForwardDiff-shaped keyword surface on every constructor.
- **Objects instead of keyword piles.** Controllers, alias specifiers, and verbosity settings became real
  extensible types rather than growing `solve`'s keyword list forever.
- **Paying off deprecations.** A lot of v7 is simply deleting shims that had been warning for one or more
  minor releases.

The migration is real work, and the three silent breakages in particular deserve a careful pass rather than
a version bump and a hope. But the two-step path through v6's deprecation warnings makes most of it
mechanical, and the result is a stack that starts faster, specializes better, and no longer has
ForwardDiff assumptions baked into a hundred struct signatures.

If you hit something this post does not cover, the [full `NEWS.md`](https://github.com/SciML/OrdinaryDiffEq.jl/blob/master/NEWS.md)
is the exhaustive reference, and questions are welcome on
[the SciML Zulip](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged) or in a
[GitHub issue](https://github.com/SciML/OrdinaryDiffEq.jl/issues).
