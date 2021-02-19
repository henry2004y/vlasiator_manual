@def title = "My Personal Vlasiator Manual"
@def tags = ["syntax", "code"]

# Vlasiator

\tableofcontents <!-- you can use \toc as well -->

Vlasiator is a numerical model for collisionless ion-kinetic plasma physics.


## Features

Vlasov solvers in higher dimensions are known to be slow. But how slow it is?
3D Earth: 1 week of running  = 6 million cpu hours?

One restart file for 3D Earth now is typically ~2TB.

* semi-Lagrangian Vlasov solver for ions
* massless fluid treatment of electrons
* 3D mesh refinement in Cartesian boxes
* Up to isotropic $\nabla P_e$ term in the Ohm's law when solving for the electric field?

$$
\frac{\partial f_\alpha}{\partial t} + \mathbf{v}\frac{\partial f_\alpha}{\partial \mathbf{r}} + \frac{q_\alpha}{m_\alpha}<\mathbf{E}+\mathbf{v}\times\mathbf{B}>\cdot \frac{\partial f_\alpha}{\partial \mathbf{r}} = 0
$$
