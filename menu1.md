@def title = "Installation"
@def hascode = true
@def date = Date(2020, 12, 8)
@def rss = ""

@def tags = ["syntax", "code"]

# Installation

\toc

## A Brief History of the Build System

There are several issues with the current file system:
* Source files should not be present on the top level.
* External library linking is a nightmare if you start on a new machine.
* The Makefile lists every single source file with repetitive flags. The flags and preprocessor settings are buried in the long Makefile which makes it hard to modify if necessary.
* Vlasiator depends on many packages. It takes a huge effort to install the code on a new machine, mainly due to the dependency hell.

To handle the fore-mentioned problems, I propose three major changes:
1. Reorganize the file levels.
2. Rewrite the Makefile.
3. Add a new configuration file.

I have come across several round of modification trying to make the new build system simple and useful.

1. For a fresh new machine,
```shell
./configure.py -h
```
shows the available options.
By default options with prefix `-` require no arguments, and options with prefix `--` require one argument.

```shell
./configure.py -install
```
tries to install all the dependencies in the new `lib` folder except Boost.
On Ubuntu it is often installed under the default folder; on supercomputers there is usually a module to load.

2. To access an existing customized Makefile in `MAKE` folder with library paths and compiler flags, e.g.
```shell
./configure.py --machine=yann
```
then it will include the preset library paths in the main Makefile.
By default it links to `MAKE/Makefile.default`.

3. To quickly setup the library paths on a new machine, you create a new makefile `MAKE/Makefile.yours` and set the required links.
Then this can be called with `./configure.py --machine=yours`.

I still keep the options of changing order of solvers, floating point precisions, etc.. Even though they rarely change, it is still changeable, but the default is set to the most common options.

I do not provide support for python2, since the official support has also ended by the end of 2020.
It will warn you about obsolete python version.

The tools should in principle also be able to be compiled, but for now it is not guaranteed to work.

This comes together with a new Makefile borrowed from Athena. I need to think about whether or not we should use a hierarchical Makefile architecture.

## DCCRG

DCCRG is the underlying grid and MPI library for Vlasiator. I guess this provides similar functionality compared to **AMReX** and **BATL**.

## Zoltan

Zoltan is used to do loading balancing, or spatial partitioning of the computation.

It can be downloaded from Sandia's website: [Zoltan](http://cs.sandia.gov/Zoltan/Zoltan_Distributions/zoltan_distrib_v3.83.tar.gz).

## Boost

I don't know what Boost is used for. Maybe argument parsing?

## Eigen

Eigen is probably used for some avx instructions in the acceleration part of the Vlasov solver.
There are two version in-use:
* AGNER, which aims at ultimate avx performance;
* FALLBACK, which is homemade and more robust, but may not provide the best performance.

However, the latest Eigen version has compilation error in assertions:
```shell
3.3.8
/home/hyzhou/include/Eigen/src/Core/products/Parallelizer.h:162:40: error: ‘eigen_assert_exception’ is not a member of ‘Eigen’
  162 |   if (errorCount) EIGEN_THROW_X(Eigen::eigen_assert_exception());
      |                                        ^~~~~~~~~~~~~~~~~~~~~~
/home/hyzhou/include/Eigen/src/Core/util/Macros.h:1017:34: note: in definition of macro ‘EIGEN_THROW_X’
```

## PHIPROF

PHIPROF is the profiler library used, which is very similar to the timer library in SWMF written in Fortran.

Currently I need to load the dynamic library like this:
```
export LD_LIBRARY_PATH=/home/hongyang/Vlasiator/vlasiator/lib/phiprof/lib
```

Alternatively, you can use `rpath` to insert the path into the executable, as has been done on Vorna.

## JEMALLOC

JEMALLOC is a library for improved memory allocation.
Rumour has it that Firefox is also using this library to reduce the memory fragmentation during heap allocations.
The idea behind is, I guess, similar to MPI collective IO in essense.
However, if this is consistently performing better than the standard allocator function `malloc`, I don't know why this is not the standard allocator instead.
So apparently it comes with some other drawbacks.
From [an interesting post on StackOverflow](https://stackoverflow.com/questions/13027475/cpu-and-memory-usage-of-jemalloc-as-compared-to-glibc-malloc):
> If you have reasons to think that your application memory management is not effective because of fragmentation, you have to make tests. It is the only reliable information source for your specific needs. If jemalloc is always better that glibc, glibc will make jemalloc its official allocator. If glibc is always better, jemalloc will stop to exist. When competitors exist long time in parallel, it means that each one has its own usage niche.

One interesting finding is that a MPI communication bug in Vlasiator's DCCRG call will only be triggered by JEMALLOC, at least in some small scale tests.

This is also a dynamic library that needs to be loaded:
```
export LD_LIBRARY_PATH=/home/hongyang/Vlasiator/vlasiator/lib/jemalloc/lib
```
