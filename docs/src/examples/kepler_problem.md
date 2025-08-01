# The Kepler Problem

The Hamiltonian $\mathcal {H}$ and the angular momentum $L$ for the Kepler problem are

$$\mathcal {H} = \frac{1}{2}(\dot{q}^2_1+\dot{q}^2_2)-\frac{1}{\sqrt{q^2_1+q^2_2}},\quad
L = q_1\dot{q_2} - \dot{q_1}q_2$$

Also, we know that

$${\displaystyle {\frac {\mathrm {d} {\boldsymbol {p}}}{\mathrm {d} t}}=-{\frac {\partial {\mathcal {H}}}{\partial {\boldsymbol {q}}}}\quad ,\quad {\frac {\mathrm {d} {\boldsymbol {q}}}{\mathrm {d} t}}=+{\frac {\partial {\mathcal {H}}}{\partial {\boldsymbol {p}}}}}$$

```@example kepler
import OrdinaryDiffEq as ODE, LinearAlgebra, ForwardDiff, NonlinearSolve as NLS, Plots
H(q, p) = LinearAlgebra.norm(p)^2 / 2 - inv(LinearAlgebra.norm(q))
L(q, p) = q[1] * p[2] - p[1] * q[2]

pdot(dp, p, q, params, t) = ForwardDiff.gradient!(dp, q -> -H(q, p), q)
qdot(dq, p, q, params, t) = ForwardDiff.gradient!(dq, p -> H(q, p), p)

initial_position = [0.4, 0]
initial_velocity = [0.0, 2.0]
initial_cond = (initial_position, initial_velocity)
initial_first_integrals = (H(initial_cond...), L(initial_cond...))
tspan = (0, 20.0)
prob = ODE.DynamicalODEProblem(pdot, qdot, initial_velocity, initial_position, tspan)
sol = ODE.solve(prob, ODE.KahanLi6(), dt = 1 // 10);
```

!!! note
    
    Note that NonlinearSolve.jl is required to be imported for ManifoldProjection

Let's plot the orbit and check the energy and angular momentum variation. We know that energy and angular momentum should be constant, and they are also called first integrals.

```@example kepler
plot_orbit(sol) = Plots.plot(sol, idxs = (3, 4), lab = "Orbit", title = "Kepler Problem Solution")

function plot_first_integrals(sol, H, L)
    Plots.plot(initial_first_integrals[1] .- map(u -> H(u.x[2], u.x[1]), sol.u),
        lab = "Energy variation", title = "First Integrals")
    Plots.plot!(initial_first_integrals[2] .- map(u -> L(u.x[2], u.x[1]), sol.u),
        lab = "Angular momentum variation")
end
analysis_plot(sol, H, L) = Plots.plot(plot_orbit(sol), plot_first_integrals(sol, H, L))
```

```@example kepler
analysis_plot(sol, H, L)
```

Let's try to use a Runge-Kutta-Nyström solver to solve this problem and check the first integrals' variation.

```@example kepler
sol2 = ODE.solve(prob, ODE.DPRKN6())  # dt is not necessary, because unlike symplectic
# integrators DPRKN6 is adaptive
@show sol2.u |> length
analysis_plot(sol2, H, L)
```

Let's then try to solve the same problem by the `ERKN4` solver, which is specialized for sinusoid-like periodic function

```@example kepler
sol3 = ODE.solve(prob, ODE.ERKN4()) # dt is not necessary, because unlike symplectic
# integrators ERKN4 is adaptive
@show sol3.u |> length
analysis_plot(sol3, H, L)
```

We can see that `ERKN4` does a bad job for this problem, because this problem is not sinusoid-like.

One advantage of using `DynamicalODEProblem` is that it can implicitly convert the second order ODE problem to a *normal* system of first order ODEs, which is solvable for other ODE solvers. Let's use the `Tsit5` solver for the next example.

```@example kepler
sol4 = ODE.solve(prob, ODE.Tsit5())
@show sol4.u |> length
analysis_plot(sol4, H, L)
```

#### Note

There is drifting for all the solutions, and high order methods are drifting less because they are more accurate.

### Conclusion

* * *

Symplectic integrator does not conserve the energy completely at all time, but the energy can come back. In order to make sure that the energy fluctuation comes back eventually, symplectic integrator has to have a fixed time step. Despite the energy variation, symplectic integrator conserves the angular momentum perfectly.

Both Runge-Kutta-Nyström and Runge-Kutta integrator do not conserve energy nor the angular momentum, and the first integrals do not tend to come back. An advantage Runge-Kutta-Nyström integrator over symplectic integrator is that RKN integrator can have adaptivity. An advantage Runge-Kutta-Nyström integrator over Runge-Kutta integrator is that RKN integrator has less function evaluation per step. The `ERKN4` solver works best for sinusoid-like solutions.

## Manifold Projection

In this example, we know that energy and angular momentum should be conserved. We can achieve this through manifold projection. As the name implies, it is a procedure to project the ODE solution to a manifold. Let's start with a base case, where manifold projection isn't being used.

```@example kepler
import DiffEqCallbacks as CB

function plot_orbit2(sol)
    Plots.plot(sol, vars = (1, 2), lab = "Orbit", title = "Kepler Problem Solution")
end

function plot_first_integrals2(sol, H, L)
    Plots.plot(initial_first_integrals[1] .- map(u -> H(u[1:2], u[3:4]), sol.u),
        lab = "Energy variation", title = "First Integrals")
    Plots.plot!(initial_first_integrals[2] .- map(u -> L(u[1:2], u[3:4]), sol.u),
        lab = "Angular momentum variation")
end

analysis_plot2(sol, H, L) = Plots.plot(plot_orbit2(sol), plot_first_integrals2(sol, H, L))

function hamiltonian(du, u, params, t)
    q, p = u[1:2], u[3:4]
    qdot(@view(du[1:2]), p, q, params, t)
    pdot(@view(du[3:4]), p, q, params, t)
end

prob2 = ODE.ODEProblem(hamiltonian, [initial_position; initial_velocity], tspan)
sol_ = ODE.solve(prob2, ODE.RK4(), dt = 1 // 5, adaptive = false)
analysis_plot2(sol_, H, L)
```

There is a significant fluctuation in the first integrals, when there is no manifold projection.

```@example kepler
function first_integrals_manifold(residual, u, p, t)
    residual[1:2] .= initial_first_integrals[1] - H(u[1:2], u[3:4])
    residual[3:4] .= initial_first_integrals[2] - L(u[1:2], u[3:4])
end

cb = CB.ManifoldProjection(first_integrals_manifold, autodiff = NLS.AutoForwardDiff())
sol5 = ODE.solve(prob2, ODE.RK4(), dt = 1 // 5, adaptive = false, callback = cb)
analysis_plot2(sol5, H, L)
```

We can see that thanks to the manifold projection, the first integrals' variation is very small, although we are using `RK4` which is not symplectic. But wait, what if we only project to the energy conservation manifold?

```@example kepler
function energy_manifold(residual, u, p, t)
    residual[1:2] .= initial_first_integrals[1] - H(u[1:2], u[3:4])
    residual[3:4] .= 0
end
energy_cb = CB.ManifoldProjection(energy_manifold, autodiff = NLS.AutoForwardDiff())
sol6 = ODE.solve(prob2, ODE.RK4(), dt = 1 // 5, adaptive = false, callback = energy_cb)
analysis_plot2(sol6, H, L)
```

There is almost no energy variation, but angular momentum varies quite a bit. How about only project to the angular momentum conservation manifold?

```@example kepler
function angular_manifold(residual, u, p, t)
    residual[1:2] .= initial_first_integrals[2] - L(u[1:2], u[3:4])
    residual[3:4] .= 0
end
angular_cb = CB.ManifoldProjection(angular_manifold, autodiff = NLS.AutoForwardDiff())
sol7 = ODE.solve(prob2, ODE.RK4(), dt = 1 // 5, adaptive = false, callback = angular_cb)
analysis_plot2(sol7, H, L)
```

Again, we see what we expect.
