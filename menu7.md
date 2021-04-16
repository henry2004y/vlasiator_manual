@def title = "Hybrid Simulation"
@def hascode = false
@def rss = ""
@def rss_title = "Method"
@def rss_pubdate = Date(2021, 4, 15)

@def tags = ["syntax", "image"]

# Hybrid Simulation

Currently Vlasiator is still a hybrid code. Therefore it shares many similarity with other hybrid models.
Here is a review on hybrid models in general. As this get larger, I turned to LaTeX notes so check further updates there.

\toc

## Standard hybrid model

The basic equations used in the hybrid model are consisting of equation of motion for individual ions and for a fluid electrons

$$
\frac{d\mathbf{x}_j}{dt} = \mathbf{v}_j,
$$

$$
\frac{d\mathbf{v}_j}{dt} = \frac{q_j}{m_j}\big( \mathbf{E} + \mathbf{v}_j \times \mathbf{B} \big),
$$

$$
\frac{d\mathbf{v}_e}{dt} = -\frac{e}{m_e}\big( \mathbf{E} + \mathbf{v}_e \times \mathbf{B} \big) - \frac{1}{n_e m_e}\nabla\cdot\overleftrightarrow{P}_e - \nu m_e(\mathbf{v}_i - \mathbf{v}_e),
$$
where the subscript j and e indicate the indices for individual ions and the electron fluid and other notations are standard.
In equation (3), the last term represents the collision between electrons and ions (is it correct???), which is often neglected in collisionless plasma.

The electromagnetic fields evolve according to the following Maxwell equations in the Darwin approximation (i.e. no displacement current?)
$$
\frac{\partial \mathbf{B}}{\partial t} = -\nabla\times\mathbf{E},
$$

$$
\nabla\times\mathbf{B} = \frac{1}{\mu_0}\mathbf{J},
$$

and the electric charge density ρ and current density $\mathbf{J}$ are defined as
$$
\rho = \sum_s q_s n_s - en_e,
$$

$$
\mathbf{J} = \sum_s q_s n_s \mathbf{v}_s - e n_e \mathbf{v}_e,
$$
where $q_s, n_s, \mathbf{V}_s$ are the charge, number density and bulk velocity of ion species s calculated by taking moments of the distribution function. Notice that there is no equation to determine the time evolution of the electric field.

The crucial assumption in the hybrid model is the quasi-neutrality, that is, the electrons move fast enough to cancel any charge-density fluctuations and $\rho=0$ is always satisfied.
The electron density thus can be written by using ion densities $n_e \approx n_i \equiv \sum_s q_s n_s /e$.
In addition, the electron bulk velocity may also be eliminated using Ampere's law and the relation $\mathbf{V}_e = \mathbf{V}_i - /mathbf{J}/n_e e$.
Finally, since the conventional hybrid model ignores the inertia of electron completely ($m_e \rightarrow 0$), one can use the equation of motion for the electron fluid to determine the electric field from given ion moment quantities and the magnetic field. This gives the generalized Ohm's law of the form
$$
\mathbf{E} = - \mathbf{V}_e \times \mathbf{B} - \frac{1}{n_i e}\nabla\cdot\overleftrightarrow{P}_e = - \mathbf{V}_i \times \mathbf{B} + \frac{1}{n_i e}(\nabla\times\mathbf{B})\times\mathbf{B} - \frac{1}{n_i e}\nabla\cdot\overleftrightarrow{P}_e.
$$
The second term in the right-hand side is the well-known Hall electric field contribution. Determining the electron pressure tensor by using an appropriate equation of state, the evolution of the system can be followed in time.

## Finite electron inertia

It is well known that the Alfvén wave at short wavelength comparable to ion inertia length has dispersion due to the decoupling between ion and electron dynamics. There thus appears the whistler mode whose frequency diverges as . This means that the maximum phase velocity in the system increases rapidly without bound, implying numerical difficulty. This is probably a part of the reasons for the numerical instability in hybrid simulations. It is thus easy to expect that inclusion of finite electron inertia can help stabilizing the simulation because the maximum phase velocity in this case is limited by roughly the electron Alfvén speed.

The conventional way to include a finite electron inertia correction into the hybrid model is to introduce the following so-called generalized electromagnetic field $\widehat{\mathbf{E}}, \widehat{\mathbf{B}}$, defined as 
$$
\widehat{\mathbf{E}} = \mathbf{E} - \frac{\partial}{\partial t}\big( \frac{c}{\omega_{pe}^2}\nabla\times\mathbf{B} \big),
$$
$$
\widehat{\mathbf{B}} = \mathbf{B} + \nabla\times\big( \frac{c^2}{\omega_{pe}^2}\nabla\times\mathbf{B} \big),
$$
in which the terms proportional to $\nabla\times\mathbf{B}$ represent electron inertia correction.
It is easy to show that they exactly satisfy Faraday's law:
$$
\frac{\partial\mathbf{B}}{\partial t} = -\nabla\times\mathbf{E}.
$$
From the equation of motion for the electron fluid, it may be shown that
$$
\widehat{\mathbf{E}} = - \mathbf{V}_i \times \mathbf{B} + \frac{1}{n_i e}(\nabla\times\mathbf{B})\times\mathbf{B} - \frac{1}{n_i e}\nabla\cdot\overleftrightarrow{P}_e - \frac{m_e}{e}(\mathbf{V}_e\cdot\nabla)\mathbf{V}_e,
$$
which is similar to the generalized Ohm's law (8) but now with the last term which also represents the correction.
Note that this equation is not exact; we have dropped the terms $\partial n_e/\partial t, \partial n_i/\partial t, \partial n_i\mathbf{V}_i/\partial t$, assuming ion moment quantities do not change during the fast electron time scale.

Given the generalized electric field $\widehat{\mathbf{E}}$, one can advance the generalized magnetic field $\widehat{\mathbf{B}}$ by using Eq. (11).
Further simplifications are commonly adopted; for example, the electric field correction term and electron-scale spatial variation of density are often ignored. In this case, the magnetic field may be recovered by solving the implicit equation (???)
$$
\widehat{\mathbf{B}} = \big( 1 - \frac{c^2}{\omega_{pe}^2}\nabla^2 \big)\mathbf{B},
$$
and $\widehat{\mathbf{E}} = \mathbf{E}$ is assumed. The nice feature with this approach is that the correction can be implemented as a post process to the each integration step of a standard procedure.

###

One typical issue in hybrid simulations is that low density regions, once generated, will pose a limit on timesteps
[Amano+, 2014](Amano+ 2014).
This is obviously due to the division-by-density operation needed to calculate the electric field from ion moment quantities, which makes it impossible to handle such (near) vacuum regions.
In practice, numerical difficulty arises even long before the electron Alfvén speed limit is reached because the Alfvén speed increases as the density decreases, imposing a severe restriction on the simulation time step.

Although the above (or similar) set of equations correctly model finite electron inertia effect on transverse modes and have been used for a variety of problems in space physics, a different form concerning the numerical stability may be applied.
Multiplying $n_e e$ to Eq. (12) and eliminating $\widehat{\mathbf{E}}$ using Eq. (9), one obtains
$$
(\omega_{pe}^2 - c^2\nabla^2)\mathbf{E} = \frac{e}{m_e}\big( \mathbf{J}_e \times\mathbf{B} - \nabla\cdot\overleftrightarrow{P}_e \big) + (\mathbf{V}_e\cdot\nabla)\mathbf{J}_e,
$$
where $/mathbf{J}_e \equiv -e n_e \mathbf{V}_e $ is the electron current density. In deriving this equation, $\nabla\cdot\mathbf{E}\sim \mathcal{O}(V_A^2/c^2)$ has been neglected, which is indeed a reasonable assumption. Once the electric field is determined by solving Eq. (14), the magnetic field may be updated using Eq. (4) without invoking the generalized electromagnetic fields.
The present implementation obviously describes the electron scale physics better than the conventional one because it retains the correction term for the electric field as well. Concerning the ions dynamics, however, the effect will be small as it affects only high frequency waves. Nevertheless, the use of Eq. (14) has a remarkable advantage. It is easy to recognize that the terms in the right-hand side of Eq. (14) are proportional to the density. (Or more precisely, they are first and second order moments of the distribution function.) Therefore, in the limit of low density ($n_e \approx n_i \rightarrow 0$), it correctly reduces to the Laplace's equation:
$$
\nabla\cdot\mathbf{E}=0,
$$
implying that there is no essential difficulty with this equation in dealing with low density (or vacuum) regions. This is reflected by the fact that the division-by-density operation is "almost" eliminated in the calculation procedure.

It is clear from Eq. (14) that the vacuum region and the plasma region are naturally connected with an intermediate region in between where the electron inertia effect dominates.
Strictly speaking, however, one must recognize the fact that dealing with such a low density region in the hybrid model certainly violates its assumptions. Namely, the quasi-neutrality assumption $n_i \approx n_e$ is no longer valid in such a tenuous region because time scale associated with the electron plasma oscillation may ultimately become comparable to the simulation time step, and non-negligible charge density fluctuation would appear in reality. It is thus clear that this model does not necessarily give physically correct description of the interface between the plasma and vacuum regions. However, with typical dynamic range of density and grid sizes in hybrid simulations, such a region is not well resolved anyway. It is thus rather important in practice that a code has capability to handle such regions without numerical problems.


[Amano+ 2014]: https://doi.org/10.1016/j.jcp.2014.06.048