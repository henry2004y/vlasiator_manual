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