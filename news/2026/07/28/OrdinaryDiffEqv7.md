@def rss_pubdate = Date(2026,7,28)
@def rss = """OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It"""
@def published = " 28 July 2026 "
@def title = "OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It"
@def authors = """<a href="https://github.com/ChrisRackauckas">Chris Rackauckas</a>"""

# OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8: What Breaks, What to Do About It

OrdinaryDiffEq.jl v7 and DifferentialEquations.jl v8 are out, along with SciMLBase v3 and
RecursiveArrayTools v4. This is the first breaking release of the ODE stack in a long while and there is
quite a lot in it. The [full changelog](https://github.com/SciML/OrdinaryDiffEq.jl/blob/master/NEWS.md)
runs to several hundred lines and is organized by subsystem. That is the right way to write a changelog
and the wrong way to plan an afternoon of migration work.

So this post is organized differently: by how likely a change is to bite you without saying anything.
Most of v7's breakage announces itself with an `ArgumentError` that tells you exactly what to write
instead, and those changes are not very interesting: you run your tests, you read the error, you fix it,
you move on. The ones that matter are the handful where your code keeps running and silently
means something else. There are three of those and they are what the first half of this post is about.

Everything below was run against OrdinaryDiffEq v7.1.3, SciMLBase v3.39.1 and RecursiveArrayTools v4.3.4
on Julia 1.11, and re-checked against OrdinaryDiffEq v7.2.0 and SciMLBase v3.40 when those landed. The
outputs are what those versions actually print, not what I remember them printing.

## The silent ones

### `sol.retcode == :Success` is now false

Start here, because this one turns a passing check into a failing one and never says a word about it:

```julia
sol = solve(prob, Tsit5())

sol.retcode == :Success        # false, on a solve that succeeded
successful_retcode(sol)        # true
```

`sol.retcode` has been a `ReturnCode.T` enum rather than a `Symbol` for years now, and every `Symbol`
comparison printed a deprecation warning for the entire v2 series. v3 finally removed the shim. The
trouble is that comparing an enum against a `Symbol` isn't an error in Julia, it's just `false`, so code
shaped like

```julia
if sol.retcode == :Success
    record_result(sol)
else
    @warn "solver failed"
end
```

now takes the failure branch on every single successful solve and nothing anywhere complains about it.
Anyone running a batch job that logs failures is about to log all of them.

The fix is `successful_retcode(sol)`, and I'd suggest using it even after you've dealt with the `Symbol`
problem, in preference to `sol.retcode == ReturnCode.Success`. There are several return codes that mean
success, including `Success`, `StalledSuccess`, `ExactSolutionLeft`, `ExactSolutionRight` and
`FloatingPointLimit`, and `successful_retcode` knows about all of them. A solve that terminated exactly
on a root is a success, and hardcoding the one enum value silently calls it a failure. Grep for
`retcode ==` and `retcode !=` before you upgrade.

### `sol[i]` is no longer the i-th timestep

RecursiveArrayTools v4 makes `AbstractVectorOfArray`, which is the parent type of `ODESolution`, an
honest `AbstractArray`. This is the right call. Every generic `AbstractArray` consumer in the ecosystem
(LinearAlgebra, broadcasting, Zygote adjoints, StructArrays) now works on solutions without SciMLBase
special-casing it, and it let us delete a large pile of hand-written method overrides that existed only
to fake array behavior. But it does change what indexing means.

Here is a 3-variable Lorenz solve with 27 saved timesteps under v7:

```julia
size(sol)         # (3, 27)   it's a matrix now
length(sol)       # 81        total scalar elements
length(sol.u)     # 27        timesteps
sol[1]            # 1.0
sol.u[1]          # [1.0, 0.0, 0.0]
sol[:, 1]         # [1.0, 0.0, 0.0]
first(sol)        # 1.0
first(sol.u)      # [1.0, 0.0, 0.0]
eachindex(sol)    # CartesianIndices, not 1:27
```

`sol[1]` used to hand you a state vector and now hands you a `Float64`. `for u in sol` used to walk
timesteps and now walks scalars in column-major order. `length(sol)` used to count timesteps and now
counts elements. None of this errors; you just get 81 where you expected 27, or a number where you
expected a vector, and the actual failure shows up somewhere far away from the line that caused it.

| Operation | v6 | v7 | Write instead |
|---|---|---|---|
| `sol[i]` | i-th timestep | i-th scalar element | `sol.u[i]` or `sol[:, i]` |
| `length(sol)` | number of timesteps | total scalar count | `length(sol.t)` or `length(sol.u)` |
| `eachindex(sol)` | `1:nsteps` | `CartesianIndices` | `eachindex(sol.u)` |
| `for u in sol` | timesteps | scalars | `for u in sol.u` |
| `first(sol)` / `last(sol)` | first/last timestep | first/last scalar | `first(sol.u)` / `last(sol.u)` |
| `map(f, sol)` | over timesteps | over scalars | `map(f, sol.u)` |
| `maximum(sol)` | over timesteps | over all scalars | `maximum(f, sol.u)` |

The convenient part is that `sol.u[i]` means the same thing on v6 and on v7, so this is a
search-and-replace you can do today, on your current version, and have it be correct before and after the
bump. If you do one thing to prepare for v7, do this one. The same applies to ensembles: `sim[j]` becomes
`sim.u[j]`, `sim[i, j]` becomes `sim.u[j].u[i]`, `length(sim)` becomes `length(sim.u)`.

For a large code base that assumes timestep-first indexing everywhere, there is a way to buy some time.
`RecursiveArrayToolsRaggedArrays.jl` gives you the old semantics back:

```julia
using RecursiveArrayToolsRaggedArrays
sol_old = RaggedVectorOfArray(sol)   # sol_old[i] is the i-th timestep again
```

Treat it as a compatibility layer for unblocking a migration, though, not as something to write new code
against.

### Controller keyword arguments were accepted and then ignored

The adaptive step size controller got refactored from a pile of loose numeric knobs on `solve` into
actual controller objects, so `gamma`, `beta1`, `beta2`, `qmin`, `qmax`, `qsteady_min`, `qsteady_max` and
`qoldinit` now live on `PIController`, `PIDController`, `IController` and `PredictiveController`. This is
a good change, and it's what makes it possible to write your own controller and pass
`controller = MyController(…)` instead of us adding a fourteenth keyword argument to `solve`.

The sharp edge, on SciMLBase v3.39.1 and earlier, is that those old keyword arguments were still on the
accepted list, so passing one didn't error. It just didn't do anything:

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

Identical across a 10x range of `gamma`, where on v6 those four runs would have differed substantially.
`qmin`, `qmax`, `beta1`, `beta2` and `qoldinit` all behaved the same way. A properly unknown keyword like
`totally_bogus_kwarg` errored correctly, so this was specific to the controller names having been left in
the allowlist after the code that consumed them was removed.

SciMLBase v3.40 removes them from the accepted list, so you get a real error instead. On v3.39 and
earlier you don't, so grep for those names in your `solve` calls. Either way the migration is to put
them on the controller object:

```julia
# v6
solve(prob, alg; gamma = 0.9, beta1 = 0.7, beta2 = -0.4)

# v7
using OrdinaryDiffEqCore: PIController
solve(prob, alg; controller = PIController(0.7, -0.4))
```

That `using OrdinaryDiffEqCore` is not a typo, incidentally; see the section on imports below. The whole
thing was tracked as [OrdinaryDiffEq.jl#4027](https://github.com/SciML/OrdinaryDiffEq.jl/issues/4027) and
fixed in [SciMLBase#1471](https://github.com/SciML/SciMLBase.jl/pull/1471).

## The loud ones

Everything from here on errors, so it costs you a test run instead of an afternoon.

The big one is that every `Bool` keyword that used to select between code paths is now a typed object.
`autodiff = true` becomes `autodiff = AutoForwardDiff()`, `verbose = false` becomes
`verbose = DEVerbosity(SciMLLogging.None())`, and `alias = true` becomes
`alias = ODEAliasSpecifier(alias_u0 = true)`. The reason in every case is type stability: a `Bool` picked
between (say) ForwardDiff and FiniteDiff at runtime, which the compiler could not specialize through,
whereas the typed object carries that choice in its type. The error messages are good enough that you
don't really need this post for them. Passing `autodiff = true` on v7 tells you to use an `ADType` from
ADTypes.jl and names two of them.

The `lazy` keyword on `BS5` and `Vern6`–`Vern9` belongs to the same family and the changelog lists it
alongside the others, but be aware that it is not actually enforced yet: `Vern7(lazy = false)` still
constructs and still solves, it just gives you a `Bool` type parameter instead of `Val{false}` and
therefore none of the specialization benefit. Write `lazy = Val{false}()` anyway, since that is where it
is going.

The `autodiff` change is the one with a payoff beyond type stability, though, and it's worth
understanding instead of mechanically fixing. Because the backend is now an `ADTypes` object instead of
a `Bool` plus a scattering of ForwardDiff-specific and FiniteDiff-specific keywords (`chunk_size`,
`diff_type`, `standardtag`), every implicit solver in the library now works with every AD backend for
free. Switching to Enzyme is `autodiff = AutoEnzyme()` and nothing else:

```julia
# v6
Rodas5P(chunk_size = 12, diff_type = Val{:central}, standardtag = true)

# v7
Rodas5P(autodiff = AutoForwardDiff(chunksize = 12))
Rodas5P(autodiff = AutoFiniteDiff(fdtype = Val(:central)))
Rodas5P(autodiff = AutoEnzyme())
```

This is also why the `{CS, AD, FDT, ST, CJ}` type parameters came off of more than a hundred algorithm
structs. Five type parameters collapsed into one field. If you were dispatching on
`SomeAlg{CS, AD, FDT, ST, CJ}` anywhere that will break, and what you almost certainly want now is either
`SomeAlg` with no parameters at all or a dispatch on `alg.autodiff`.

### The umbrella got smaller

`using OrdinaryDiffEq` now loads only the default solver set:

```julia
# v6
using OrdinaryDiffEq
solve(prob, KenCarp4())     # worked

# v7
using OrdinaryDiffEqSDIRK   # required
solve(prob, KenCarp4())
```

You still get `DefaultODEAlgorithm`, `Tsit5`, `AutoTsit5`, `Vern6` through `Vern9`, `AutoVern6` through
`AutoVern9`, `Rosenbrock23`, `Rodas5P` and `FBDF` from the umbrella. Everything else needs its
sublibrary, and the mapping is predictable enough to guess: `KenCarp*` and `TRBDF2` are in
`OrdinaryDiffEqSDIRK`, most of the `Rosenbrock*` and `Rodas*` family is in `OrdinaryDiffEqRosenbrock`,
`RadauIIA*` is in `OrdinaryDiffEqFIRK`, and every family has its own `lib/OrdinaryDiffEq<X>` directory in
the repo if you need to look.

This is the single largest contributor to v7's time-to-first-solve improvement, along with `ODEFunction`
defaulting to `AutoSpecialize` and dropping `Static.jl`, `StaticArrayInterface.jl`, `Polyester.jl` and
`StaticArrays.jl` as direct dependencies. The old umbrella loaded every exponential integrator, every
symplectic method, every stabilized method and every multirate method whether you touched them or not.
To push it further, import a single solver: `using OrdinaryDiffEqTsit5: Tsit5`.

While you're editing those lines anyway: if you're still explicitly asking for `Rodas5`, switch to
`Rodas5P`. Very similar performance profile, considerably more robust on accuracy, and there's not really
a good reason to prefer the older one anymore.

### DAE initialization errors instead of silently fixing

The default is now `CheckInit`, so an inconsistent initial condition gets you

```
DAE initialization failed: your u0 did not satisfy the initialization requirements,
normresid = 0.707… > abstol = 1.0e-6.
```

where v6 would have silently corrected it and carried on. The old behavior sounds friendlier but it
produced wrong answers in the case that actually matters: when your `u0` was correct and a modeling bug
somewhere else made the system look inconsistent, v6 would "fix" the initial condition to match the buggy
model and hand you a plausible-looking trajectory. Erroring surfaces the modeling bug. The old behavior is
still available, you just have to ask for it:

```julia
using DiffEqBase: BrownFullBasicInit
solve(prob, Rodas5P(); initializealg = BrownFullBasicInit())
```

### `VectorContinuousCallback` fires every simultaneous event

This one was a bug fix, not a design change. When several conditions of a single
`VectorContinuousCallback` crossed zero within the same step, only the first crossing's `affect!` ran and
the rest were silently dropped, even though their roots were inside the step and had been located.
Bouncing-ball models, multi-contact mechanics and anything using a callback as a threshold state machine
were all affected.

v7 resolves all of them and dispatches them in one call. That needed a new signature. Instead of being
called once per triggering condition with that condition's index, `affect!` is called once per step with
a `Vector{Int8}` mask over all conditions, where each entry is `0` for "didn't trigger", `+1` for an
upcrossing and `-1` for a downcrossing. Since the sign carries the direction, `affect_neg!` is no longer
used for `VectorContinuousCallback` at all, since one function now handles both. (`ContinuousCallback`
keeps its `affect!` / `affect_neg!` split, so this is not a general change.)

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

The first v7 releases had this sign convention flipped and it was corrected quickly, so make sure you're
on a current v7.1.x before you go debugging against the initial tag.

### Ensembles carry an RNG now

```julia
# v6
prob_func = (prob, i, repeat) -> remake(prob, u0 = rand(length(prob.u0)))

# v7
prob_func = (prob, ctx) -> remake(prob, u0 = rand(ctx.rng, length(prob.u0)))
```

`ctx::EnsembleContext` has `ctx.i` for the trajectory index, `ctx.repeat` for the retry counter and
`ctx.rng`, and `output_func(sol, i)` becomes `output_func(sol, ctx)` in the same way. The old signature
simply had nowhere to thread a reproducible per-trajectory RNG through. Ensemble results therefore
depended on thread count and scheduling, so you could not reproduce your own run on a different machine.
Now `solve(ensemble_prob, alg; seed = 42, trajectories = N)` gives you the same answer regardless of how
many workers happen to pick up the work.

### Renames and removals

These all already exist under their new names on SciMLBase v2 and OrdinaryDiffEq v6 with deprecation
warnings. That matters for the upgrade path below:

| Old | New |
|---|---|
| `sol.destats` | `sol.stats` |
| `has_destats(alg)` | `has_stats(alg)` |
| `integrator.u_modified` | `integrator.derivative_discontinuity` |
| `DEAlgorithm` | `AbstractDEAlgorithm` |
| `DEProblem` | `AbstractSciMLProblem` |
| `DESolution` | `AbstractSciMLSolution` |
| `constructDormandPrince()` | `OrdinaryDiffEqExplicitTableaus.DormandPrince()` |
| `OrdinaryDiffEq.False()` / `True()` | `Serial()` / `Threaded()`, from FastBroadcast |

Around eighty tableau functions dropped their `construct` prefix and are no longer exported, so qualify
them with the sublibrary. DiffEqDevTools v3 went further and removed its own 105 `construct*` re-exports
entirely; the authoritative definitions now live only in `OrdinaryDiffEqExplicitTableaus` and
`OrdinaryDiffEqImplicitTableaus`. `deduce_Butcher_tableau(alg)` is unchanged.

Also gone: `tuples()`, `intervals()`, `QuadratureProblem` (use `IntegralProblem`), `fastpow` (use
`FastPower.fastpower`), `concrete_solve` (use `solve`), the `syms`/`paramsyms`/`indepsym` keyword
arguments, `sol.x` on optimization solutions (use `sol.u`), and the positional
`Alg(stage_limiter!, step_limiter!)` constructors across 99 explicit RK methods. Those want the keyword
form now.

One more default change to flag, because this one is numerical and not just an API rename:
`williamson_condition` now defaults to `false` on all 2N low-storage RK methods. That optimization only
works for mutable `Array`-style state, so having it on by default silently misbehaved for StaticArrays,
GPU arrays and ComponentArrays. Turn it back on when you know your state is a plain `Array`.

## Where things actually live

A wrinkle that will cost you ten minutes if nobody warns you: several of the replacement types are not
reachable from a plain `using OrdinaryDiffEq`. Checked on v7.1.3:

| Symbol | From `using OrdinaryDiffEq`? | Where it lives |
|---|---|---|
| `ODEAliasSpecifier` | yes | the umbrella |
| `successful_retcode`, `ReturnCode` | yes | the umbrella |
| `DEVerbosity` | no | `DiffEqBase`, `OrdinaryDiffEqCore` |
| `PIController`, `PIDController`, `IController` | no | `OrdinaryDiffEqCore` |
| `CheckInit`, `NoInit`, `OverrideInit` | no | `SciMLBase`, `DiffEqBase`, `OrdinaryDiffEqCore` |
| `BrownFullBasicInit` | no | `DiffEqBase`, `OrdinaryDiffEqCore` |
| `Serial`, `Threaded` | no | `DiffEqBase`, originally FastBroadcast |

So the snippet you'd naturally write from reading the changelog,

```julia
using OrdinaryDiffEq
solve(prob, Tsit5(); verbose = DEVerbosity(SciMLLogging.None()))
# ERROR: UndefVarError: `DEVerbosity` not defined in `Main`
```

needs an import:

```julia
using OrdinaryDiffEq, SciMLLogging
using DiffEqBase: DEVerbosity
solve(prob, Tsit5(); verbose = DEVerbosity(SciMLLogging.None()))
```

This is a rough edge left over from the reorganization, not anything intentional, and it's being tidied up.
In the meantime, if a v7 replacement type gives you an `UndefVarError`, try `DiffEqBase` first and
`OrdinaryDiffEqCore` second.

## Do the upgrade in two steps

Don't jump from an old environment straight onto v7. Nearly every rename in the table above already
exists under its new name on SciMLBase v2 and OrdinaryDiffEq v6, with deprecation warnings pointing at
the exact call sites that will break. So stay on v6 first and update to the new names there:
`has_stats`, `sol.stats`, `AbstractDEAlgorithm`, `derivative_discontinuity!`, `ODEAliasSpecifier`,
`DEVerbosity`, `ADTypes`-based `autodiff`, explicit controller objects, the new tableau names. Change
every `sol[i]` to `sol.u[i]` while you're there. Then get your tests passing on v6 with no
deprecation warnings at all, because those warnings are your migration checklist and a clean run means
you've worked through it. Only then bump to v7.

What's left after that is the truly new breakage that no deprecation warning could have told you about:
RAT v4 indexing semantics, the ensemble signature, the struct type parameter removals, the
`VectorContinuousCallback` signature, and the default changes. That's a much shorter list than the one
you started with.

Before you begin, three greps. The deprecation warnings can't help you with the silent changes, so these
cover those:

```
retcode ==
sol[            # also length(sol), eachindex(sol), for … in sol
gamma =         # also qmin, qmax, beta1, beta2, qoldinit in solve calls
```

## DifferentialEquations.jl v8

Separately from all of the above, `using DifferentialEquations` no longer re-exports the whole solver
suite. In v8 it loads `OrdinaryDiffEq` and nothing else, so every other topic is an explicit dependency:
`StochasticDiffEq` for SDEs, `DelayDiffEq` for DDEs, `BoundaryValueDiffEq` for BVPs, `JumpProcesses` for
jumps, `SteadyStateDiffEq` for steady states, `Sundials` for CVODE/IDA/ARKODE, and
`LinearSolve`/`NonlinearSolve`/`Optimization` for the solvers that were never really differential
equations in the first place.

The umbrella made everyone pay the load time for every solver family, and it made it hard to tell which
package a given algorithm came from. We got asked that a lot. Now each topic versions independently and
your `Project.toml` is honest about what your script uses. The DiffEqDocs solver pages
annotate every algorithm with its host package.

v7 and v8 are independent of each other, and for ODE-only work I'd skip the umbrella entirely and depend
on OrdinaryDiffEq directly.

Two smaller things in the same family. `DiffEqBase` is now a sublibrary at `lib/DiffEqBase` inside the
OrdinaryDiffEq monorepo, on the reasoning that it was tightly coupled to OrdinaryDiffEq's internals
anyway and releasing them in lockstep removes an entire class of compatibility bug. And
`StochasticDelayDiffEq.jl` is deprecated. Use `DelayDiffEq.jl` directly, which has handled SDDE problems
for some time now. It will not get a v7-compatible release.

## Was it worth it

Nearly every individual change above is an instance of one of a few things: cutting time to first solve,
making `Bool` switches into types the compiler can specialize on, replacing ForwardDiff-shaped keyword
surfaces with `ADTypes` so every solver generalizes to every backend, turning keyword piles into
extensible objects, and deleting shims that had been warning for several releases.

The migration is real work, and the three silent changes deserve a careful pass instead of a version bump
and a hope. But going through v6's deprecation warnings first makes most of it mechanical, and what you
get out the other side starts faster, specializes properly, and doesn't have ForwardDiff assumptions
baked into a hundred struct signatures.

If you hit something this post doesn't cover, the [full `NEWS.md`](https://github.com/SciML/OrdinaryDiffEq.jl/blob/master/NEWS.md)
is the exhaustive version, and questions are always welcome on
[the SciML Zulip](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged) or as a
[GitHub issue](https://github.com/SciML/OrdinaryDiffEq.jl/issues).
