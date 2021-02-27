@def title = "My Personal Vlasiator Manual"
@def tags = ["syntax", "code"]

# Vlasiator

\tableofcontents <!-- you can use \toc as well -->

Vlasiator is a numerical model for *collisionless* *ion*-kinetic plasma physics.

## Features

Vlasov solvers in higher dimensions are known to be slow. But how slow it is?
3D Earth: 1 week of running  = 6 million cpu hours?

One restart file for 3D Earth now is typically ~2TB.

* semi-Lagrangian Vlasov solver for ions
* massless fluid treatment of electrons
* 3D mesh refinement in Cartesian boxes
* Up to isotropic $\nabla P_e$ term in the Ohm's law when solving for the electric field?

The fundamental description of charged particle motion in an electromagnetic field is given by Vlasov's equation
$$
\frac{\partial f_\alpha}{\partial t} + \mathbf{v}\frac{\partial f_\alpha}{\partial \mathbf{r}} + \mathbf{a}\cdot \frac{\partial f_\alpha}{\partial \mathbf{r}} = 0,
$$
where $\mathbf{r}$ and $\mathbf{v}$ are the spatial and velocity coordinates, $f(\mathbf{r},\mathbf{v},t)$ is
the six-dimensional phase-space density of a particle species with mass m and charge q, and acceleration $\mathbf{a}$ is given by the Lorentz force
$$
\mathbf{a} = \frac{q_\alpha}{m_\alpha}<\mathbf{E}_i+\mathbf{v}\times\mathbf{B}>,
$$
where $\mathbf{E}_i$ is the electric field affecting ions and $\mathbf{B}$ is the magnetic field.

The bulk parameters of the plasma, such as the ion charge density $\rho_q$ and current density $\mathbf{j}_i$, are obtained as velocity moments of the ion velocity distribution function
$$
\rho_q = q \int d^3 v f(\mathbf{r},\mathbf{v},t),
$$
$$
\mathbf{j}_i = q \int d^3 v \mathbf{v}f(\mathbf{r},\mathbf{v},t).
$$
These also give the bulk velocity of ions,
$$
\mathbf{V}_i = mathbf{j}_i / \rho_q.
$$

The magnetic field is updated using Faraday's law:
$$
\nabla \times \mathbf{E}_{\text{Ohm}} = -\frac{\partial \mathbf{B}}{\partial t},
$$
and the system is closed by Ohm's law giving the electric field $\mathbf{E}_{\text{Ohm}}$. Depending on the order of approximation, the simplest one only takes the convection electric field
$$
\mathbf{E}_\text{Ohm} = -\mathbf{V}_i \times\mathbf{B},
$$
where $\mathbf{V}_i$ is the ion bulk velocity given previously.
The electric field exerted on the ion is given by
$$
\mathbf{E}_i = -\mathbf{V}_i \times\mathbf{B} + \frac{1}{\rho_q}\mathbf{j}\times\mathbf{B}.
$$
The second term on the right hand side is the Hall term, which must be included otherwide no bulk force is exerted on the ions.
The total current density $\mathbf{j}$ is obtained from Ampère–Maxwell's law where the displacement current has been neglected:
$$
\nabla\times\mathbf{B} = \mu_0 \mathbf{j}.
$$

## Ion propagation

Using finite volume method, the full 6D spatial and velocity space is discretized into ordinary and velocity cells.
Each spatial cell contains the field variables ($\mathbf{B},\mathbf{E}$), which are stored on a staggered grid with $\mathbf{B}$ on the face centers and $\mathbf{E}$ on the edges. Each spatial cell also contains a 3D Cartesian velocity mesh where each velocity cell contains ths volume average $\tilde{f}$ of the distribution function over the ordinary space volume of the spatial cell, and the velocity space volume of the velocity mesh cell:
$$
\tilde{f} = \frac{1}{\Delta^3 r \Delta^3 v}\int_{cell} d^3r d^3 v f(\mathbf{r},\mathbf{v},t),
$$
where $\Delta^3 r = \Delta x \Delta y \Delta z$ and $\Delta v = \Delta v_x \Delta v_y \Delta v_z$ denote the phase space integration volumes, and $\Delta x$ e.t.c are the sizes of the cell in each dimension.
The volume average $\tilde{f}$ is propagated forward in time by calculating fluxes at every cell face in each of the six dimensions. In the case of Vlasov's equation the spatial ($F_x, F_y, F_z$) and velocity ($F_{vx}, F_{vy}, F_{vz}$) fluxes take on particularly simple forms,
$$
\mathbf{F}_\mathbf{r} = \mathbf{v}f,
$$
$$
\mathbf{F}_v = \frac{q}{m}\Big( \mathbf{v} - \mathbf{V}_i + \frac{1}{\mu_0 \rho_q}\nabla\times\mathbf{B} \Big)\times\mathbf{B}f.
$$
\mathbf{B} is a volume average calculated from face averages using divergence-free reconstruction polynomials given in [Balsara
et al. (2009)](https://doi.org/10.1016/j.jcp.2008.12.003). The propagation of $\tilde{f}$ is given by
$$
\tilde{f}(t+\Delta t) = \tilde{f}(t) - \sum_{i=x,y,z}\frac{\Delta t}{\Delta r_i}\big[ F_i(r_i+\Delta r_i) - F_i(r_i) \big] - \sum_{i=vx,vy,vz}\frac{\Delta t}{\Delta v_i}\big[ F_{vi}(v_i+\Delta v_i) - F_{vi}(v_i) \big].
$$
By construction the FVM scheme here guarantees the conservation of mass except at the boundaries of the simulation domain.
This is further split into the spatial translation $S_T$ and acceleration $S_A$ operators.
Both operators propagate the distribution function with a $2^{nd}$ order accurate method based on solving 1D Riemann problems at the cell faces and applying flux limiters to suppress oscillations. Each cell face can contribute to the flux in up to 18 adjacent cells.

[Strang (1968)](https://doi.org/10.1137/0705041) splitting is used to propagate the distribution in time
$$
\tilde{f}(t+N\Delta t) = \Big[ S_T(0.5 \Delta t) S_A(\Delta t) S_T(0.5\Delta t) \Big]^N \tilde{f}(0).
$$
The leap-frog scheme is used for propagation. The algorithm is stable as long as the timestep fulfills the
Courant–Friedrichs–Lewy (CFL) condition. The Courant number is for each dimension given by
$$
C = \frac{u\Delta t}{\Delta s},
$$
where $u$ corresponds to the velocity: $v_x$, $v_y$ and $v_z$ in ordinary space and $a_x$, $a_y$ and $a_z$ in velocity space. $\Delta s$ corresponds to the size of the simulation cell: Δx, Δy and Δz in ordinary space and $\Delta v_x$, $\Delta v_y$
and $\Delta v_z$ in velocity space.

In simulations with strong local fields leading to high acceleration, e.g., Earth's dipole magnetic field, the timestep is limited
by the acceleration $S_A$ operator. To enable larger timesteps we split the propagation in velocity space into shorter substeps,
where the length of each substep is set according to the CFL condition so that the Courant number of the substep is in the
middle of the allowed range. The magnetic field is constant during the substepping, but the effective electric field does change as we
recompute the bulk velocity $\mathbf{V}_i$ in Eq. (12) after each substep. By substepping the acceleration operator we can keep the global timestep reasonable, which minimizes the total number of timesteps in a simulation.
The length of the substep is computed on a cell-by-cell basis, thus substepping only happens close to Earth in magnetospheric
simulations. When we substep, the propagation of the distribution function is not $2^nd$ order accurate in time.

## Field Propagation



## Sparse Velocity Grid