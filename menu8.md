@def title = "Trouble Shooting"

# Trouble Shooting

\toc

## Memory

Memory is often the bottleneck for Vlasiator. Vlasiator is not constant in memory usage: as the plasma system evolves from initial state, which is mostly described by a Maxwellian distribution, the system shifts away from thermal equilibrium states and the blocks it requires to resolve the nonMaxwellian distribution may drastically increase. The total block counts can easily increase 5x as the system evolves into a quasi-steady state.

All kinds of weird error messages may pop up due to runtime out-of-memory error:
* `Segmentation fault`
* `Out-of-memory`
* `malloc failed`

Even if you set the maximum memory limit for bailing-out when compiled with PAPI in configuration files, it may still crash because of memory fragmentations.
Check `memoryallocation.cpp` in the source code if you want to see the details.

## Timestepping

In regions of small densities, the maximum characteristic wave speed may become very large, which in turn limits the discrete time steps constrained by the CFL condition. Surprisingly this has not become a major issue in Vlasiator 5.1, but I did suffer for a long time because of the problematic magnetic field settings near the inner boundary ($\nabla\cdot\mathbf{B}\neq 0$). 

Check some ideas in [Low Density Treatment](https://henry2004y.github.io/vlasiator_manual/menu7/#low_density_treatment) if you are interested in implementing a practical solution.

## Velocity Space Resolution

Extra care is needed when resolving the velocity space. Velocity is the 1st moment integral of the distribution function, and temperature determines the width of the distribution: if your velocity is too small compared to the thermal width, you can only resolve part of the phase space; if the velocity space cells are too large, then the integral will become inaccurate.

Even in 1D simulations, Vlasiator treats the other 2 spatial dimensions as 1 cell with periodic boundaries. If the velocity space grid is too small in these two extra dimensions, the sampling scheme used in Vlasiator 5.1 still generate significant discrepancy for the integrals. Under my tests, within 1% error we need 6 blocks (24 cells) in the extra dimensions, and 10 blocks (40 cells) would be better.

## AMR Filtering

The Vlasov solver provides moments of the distribution function as input for the field solver, which are spatially filtered to minimize refinement artefacts. In Vlasiator 5.1 it was found to be buggy, but I know little about this currently.


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