@def title = "Trouble Shooting"

# Trouble Shooting

\toc

## Memory

Memory is often the bottleneck for Vlasiator. Vlasiator is not constant in memory usage: as the plasma system evolves from initial state, which is mostly described by a Maxwellian distribution, the system shifts away from thermal equilibrium states and the blocks it requires to resolve the nonMaxwellian distribution may drastically increase. The total block counts can easily increase 5x as the system evolves into a quasi-steady state.

All kinds of weird error messages may pop up due to runtime out-of-memory error:

* `Segmentation fault`
* `Out-of-memory`
* `malloc failed`
* `Bus error: nonexistent physical address`

Even if you set the maximum memory limit for bailing-out when compiled with PAPI in configuration files, it may still crash because of memory fragmentations.
Check `memoryallocation.cpp` in the source code if you want to see the details.

One thing to be careful about is that the initial memory usage reported by PAPI significantly underestimates the actual usage, e.g. by 25%. It may actually be better to look at system memory usage for a better estimation.

There are currently two ways to solve the out-of-memory issue:

1. Replace some MPI processes with OpenMP threads.
2. Run with larger memory.

Multithreading runs can save memory mostly because there will be fewer ghost cells required for communication.

## Timestepping

In regions of small densities, the maximum characteristic wave speed may become very large, which in turn limits the discrete time steps constrained by the CFL condition. Surprisingly this has not become a major issue in Vlasiator 5.1, but I did suffer for a long time because of the problematic magnetic field settings near the inner boundary ($\nabla\cdot\mathbf{B}\neq 0$).

Check some ideas in [Low Density Treatment](https://henry2004y.github.io/vlasiator_manual/menu7/#low_density_treatment) if you are interested in implementing a practical solution.

## Staggered Grid

All DCCRG variables are stored at cell centers, while the electric and magnetic field are stored at edge and face centers, respectively. While performing some sensitive analysis such as parallel electric field, mixing `fs_e` with `vg_b_vol` may cause errors up to 1 order of magnitude. The important thing is **not** to process data saved at different spatial locations!

## Velocity Space Resolution

Extra care is needed when resolving the velocity space. Velocity is the 1st moment integral of the distribution function, and temperature determines the width of the distribution: if your velocity is too small compared to the thermal width, you can only resolve part of the phase space; if the velocity space cells are too large, then the integral will become inaccurate.

Even in 1D simulations, Vlasiator treats the other 2 spatial dimensions as 1 cell with periodic boundaries. If the velocity space grid is too small in these two extra dimensions, the sampling scheme used in Vlasiator 5.1 still generate significant discrepancy for the integrals. Under my tests, within 1% error we need 6 blocks (24 cells) in the extra dimensions, and 10 blocks (40 cells) would be better.

## Velocity Space Limit

Currently Vlasiator will generate wrong physics when the velocity space distribution goes beyond the limit. A better question to ask will be: what is a proper boundary condition for the velocity space?

## Ordinary Space Resolution

A person coming from the PIC world may think that it requires resolving the ion inertial length to produce correct ion physics. However, one key claim from the Vlasiator experiments is that this is not necessarily true for a vlasov ion model: _we don't need to fully resolve the ion scales to produce ion physics_. For example, in Maxime's paper about mirror modes and EMIC waves in the meriodional plane 2D3V simulation, he showed that at a resolution of the ion scale 300km given the upstream parameters we could already capture ion-related wave pretty well.

## Error Propagation

I am not a fan of the error propagations used in Vlasiator: for most of the functions I see, they return `false` after something goes wrong, which are unecessary simply because the code will generate wrong results anyway. Error propagation is useful when we want to continue the service even if some parts go down, not in a situation like a physical model where any tiny error will generate completely wrong results.

## AMR Filtering

The Vlasov solver provides moments of the distribution function as input for the field solver, which are spatially filtered to minimize refinement artefacts. In Vlasiator 5.1 it was found to be buggy, but I know little about this currently.

## Units

It seems like SI units are used for both I/O and internal computations. Since electron is treated as massless fluid, no scaling of mass is present.

Double precision is sufficient even if all the physical constants across our known scales are involved. However, if we use single precision floating numbers, we need to be careful about physical constants!

## Magnetic Flux Conservation

When I performed a closed magnetic flux calculation for a 2D meridional run, I found that the total closed dayside magnetic flux is not conserved. Several possibilities:

- The inner boundary, with a fixed line dipole and a perturbed magnetic field shielded by a perfect conductor, acts as a source to maintain the flux inside the closed field region.
- Simply the numerical solver does not consider this at all.
- I made a mistake in extracting the last-closed field line or performing the integral.

## VLSV Writing Bug

The VLSV library has a [file writing bug](https://github.com/fmihpc/vlasiator/issues/698) with OpenMPI4. This happens even for serial runs as long as the code is compiled with MPI. Current workaround it to set

```bash
export OMPI_MCA_io=^ompio
```

## Performance

Vlasiator is distributed with MPI+OpenMP, but the optimal MPI tasks + OpenMP threads combination differs case by case. Hyperthreading may or may not help.

On many clusters with Slurm job scheduler, hyperthreading is on by adding

```shell
#SBATCH --hint=multithread
```

A general MPI + OpenMP with simultaneous multithreading example Slurm script:

```shell
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=large
#SBATCH --time=02:00:00
#SBATCH --nodes=100
#SBATCH --hint=multithread
#SBATCH --ntasks-per-node=16
#SBATCH --cpus-per-task=16

# Note that the ntasks-per-node * cpus-per-task = total core counts with hyperthreading

# Set the number of threads based on --cpus-per-task
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun myprog <options>
```

Slurm now comes with a efficiency report script called `seff`. An example output of `seff $SLURM_JOBID` looks like

```
Job ID: 766237
Cluster: mahti
User/Group: hongyang/hongyang
State: RUNNING
Nodes: 30
Cores per node: 256
CPU Utilized: 680-03:29:31
CPU Efficiency: 14.69% of 4630-20:16:00 core-walltime
Job Wall-clock time: 14:28:17
Memory Utilized: 7.40 TB (estimated maximum)
Memory Efficiency: 107.77% of 6.87 TB (234.38 GB/node)
Job consumed 43414.17 CSC billing units based on following used resources
Billed project: project_2004873
Non-Interactive BUs: 43414.17
```

In general, if the processor usage is far below 100%, the code may not be working efficiently; if the memory usage is far below 100% or above 100%, you may have a problem with the RAM requirements. In this particular case, we have very low CPU efficiency because we were not running with all the cores on a compute node and Vlasiator is a memory-bound program; we have over 100% memory efficiency which indicates that the memory pressure is very high and we may need to consider increase the memory requirement.
