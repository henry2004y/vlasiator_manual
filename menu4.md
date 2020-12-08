@def title = "Misc"


# Miscellaneous

**Example**:

* page with tag [`syntax`](/tag/syntax/)
* page with tag [`image`](/tag/image/)
* page with tag [`code`](/tag/code/)

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

1. B remains static, but E is updated --- somehow this may mean that it is only electrostatic, but to what extent does it matter?

2. Not particles, but distributions functions (by solving electron Vlasov eq.). With enough number of particles, the final result should be the same.

Ion precipitation: sharp cutoff due to sparse velocity grid, need full VDF information (not saved everywhere in the past). Now implemented as an output option, and use power law extrapolation to get high energy e-, for instance.

* Electron energy range and spectrum
* Time resolution and length
* 2D-3D transformation
* File formats



## Resources

CSC, LUMI
Now the fastest machine in the US and Europe are both using AMD GPUS.
This is definitely not a headless choice, but a strong sign that the CUDA-equivalent compiler is strong enough to compete and easier enough to use.
(HIP)

I have applied for an account at CSC. On first sight, they may do a pretty decent job: they have preinstalled Julia 1.3!