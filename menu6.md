@def title = "Misc"


# Miscellaneous

\toc


## eVlasiator: Dealing with Electrons

The main Vlasiator is solving the Vlasov equations for ions; this eVlasiator is conceptually the same, just replace the ion with electron.
The differences are mainly scales.
The normal Vlasiator treats electrons as massless fluid. Possible options to improve that are:
1. fluid with mass;
2. electron macro paticles;
3. electron distribution function

What is the main difference compared with the test particle approach?
Given that the field resolution is on the order of ion scales, how can you guarantee that electron tracing is accurate?

1. B remains static, but E is updated --- this may mean that it is only electrostatic, but to what extent does it matter?

2. Not particles, but distributions functions (by solving electron Vlasov eq.). With enough number of particles, the final result should be the same.

Ion precipitation: sharp cutoff due to sparse velocity grid, need full VDF information (not saved everywhere in the past). Now implemented as an output option, and use power law extrapolation to get high energy e-, for instance.

* Electron energy range and spectrum
* Time resolution and length
* 2D-3D transformation
* File formats

## UPML Boundary

In EM field solvers, often we need boundless free-space simulation to prohibit reflecting waves.
Back in 1993, a techinque called perfectly matched layer (PML) for the absorption of EM waves was proposed to handle this problem, so that we don't necessarily need to set boundaries sufficiently far enough from the scatterer when solving interaction problems.
With the new medium the theoretical reflection factor of a plane wave striking a vacuum-layer interface is null at any frequency and at any incidence angle, contrary to the previously designed medium with which such a factor is null at normal incidence only. 
So, the layer surrounding the computational domain can theoretically absorb without reflection any kind of wave travelling towards boundaries, and it can be regarded as a perfectly matched layer. The new medium as the PML medium and the new technique of free-space simulation as the PML technique.

The theoretical derivation starts in discussing the transverse electric wave propagation. One key concept that confused me was the magnetic conductivity denoted as $\sigma^\ast$. Maybe this is just a jargon: permeability is a magnetic analogy to conductivity in electric circuits. Reluctance in a magnetic circuit is inversely proportional to permeability just as electric resistance is inversely proportional to conductivity. The relationships between length and cross-sectional area are also the same. Consequently calling permeability "magnetic conductivity" is a fine way to reinforce the analogy and understand magnetic circuits using an electronic analogy.

I guess I need to review my EM courses to fully understand the derivations.


## Resources

CSC, LUMI
Now the fastest machine in the US and Europe are both using AMD GPUS.
This is definitely not a headless choice, but a strong sign that the CUDA-equivalent compiler is strong enough to compete and easier enough to use.
(HIP)

On first sight, they may do a pretty decent job: they have preinstalled Julia 1.3!