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

### Even better approach

CMake-like tools will make your life much easier.

## DCCRG

DCCRG is the underlying grid and MPI library for Vlasiator. It provides similar functionality compared to **AMReX** and **BATL**.

### Pros & Cons

Pros:

* Simplicity. Given the grid size on the base level, each positive integer corresponds to a unique cell index on any refinement level.
* Localization. There is no need for global information to know the location of a given cell or the neighboring cell indexes.

Cons:

* It is purely cell-based, which makes any spatial-related computation slow
* It is purely based on MPI, without considering multithreading or hetergeneous architectures.

## Zoltan

Zoltan is used to do loading balancing, or spatial partitioning of the computation.

It can be downloaded and installed from Sandia's server.

```shell
git clone git@github.com:sandialabs/Zoltan.git
mkdir zoltan-build
cd zoltan-build
../Zoltan/configure --prefix="current_working_directory/zoltan" --enable-mpi --with-mpi-compilers=yes --with-gnumake --with-id-type=ullong CC=mpicc CXX=mpicxx
make -j 4
make install
cd ..
rm -r zoltan-build
```

It works smoothly on my local Ubuntu, but not on a supercomputer in my first attempt.

## Boost

Boost is used in Vlasiator solely for argument parsing. It is typically installed as a HPC module.

## Eigen

Eigen is used for linear algebra, but I don't know where exactly it is used in Vlasiator.

Assuming you are on the top level and want to install Eigen under directory `lib`,

```shell
wget https://gitlab.com/libeigen/eigen/-/archive/3.2.8/eigen-3.2.8.tar.bz2
tar -xf eigen-3.2.8.tar.bz2
cp -r eigen-3.2.8/Eigen lib
```

However, the latest Eigen version 3.3.8 has compilation error in assertions:

```shell
3.3.8
/home/hyzhou/include/Eigen/src/Core/products/Parallelizer.h:162:40: error: ‘eigen_assert_exception’ is not a member of ‘Eigen’
  162 |   if (errorCount) EIGEN_THROW_X(Eigen::eigen_assert_exception());
      |                                        ^~~~~~~~~~~~~~~~~~~~~~
/home/hyzhou/include/Eigen/src/Core/util/Macros.h:1017:34: note: in definition of macro ‘EIGEN_THROW_X’
```

## Vectorclass

Vectorclass used for some avx instructions in the acceleration part of the Vlasov solver.

Assuming you are on the top level and want to install Vectorclass under directory `lib`,

```shell
git clone https://github.com/vectorclass/version1.git
git clone https://github.com/vectorclass/add-on.git
cp add-on/vector3d/vector3d.h version1/
mv version1 lib/vectorclass
```

There are two versions in-use:

* AGNER, which aims at ultimate avx performance;
* FALLBACK, which is homemade and more robust, but does not provide the best performance.


## PHIPROF

_PHIPROF_ is the profiler library used, which is very similar to the timer library in SWMF written in Fortran.

```shell
git clone https://github.com/fmihpc/phiprof
cd phiprof/src
make clean
make -j 4
cd ../..
```

Currently I need to load the dynamic library like this:

```shell
export LD_LIBRARY_PATH=/home/hongyang/Vlasiator/vlasiator/lib/phiprof/lib
```

Alternatively, you can use `rpath` to insert the path into the executable, as has been done on Vorna.

PHIPROF write outputs to `phiprof_x.txt`, and they get overwritten every diagnostic interval.

## JEMALLOC

_JEMALLOC_ is a library for memory allocation, as an alternative to the standard C allocator _glibc_.
Rumour has it that Firefox is also using this library to reduce the memory fragmentation during heap allocations.
However, if this is consistently performing better in all aspects than the standard allocator function `malloc`, then this would become the standard allocator instead. So apparently it comes with some other drawbacks or trade-offs.
From [an interesting post on StackOverflow](https://stackoverflow.com/questions/13027475/cpu-and-memory-usage-of-jemalloc-as-compared-to-glibc-malloc):
> If you have reasons to think that your application memory management is not effective because of fragmentation, you have to make tests. It is the only reliable information source for your specific needs. If jemalloc is always better that glibc, glibc will make jemalloc its official allocator. If glibc is always better, jemalloc will stop to exist. When competitors exist long time in parallel, it means that each one has its own usage niche.

When you search on the web, you will find all kinds of confusing results. The proper way is to test for our case.
One interesting finding is that a MPI communication bug in Vlasiator's DCCRG call will only be triggered by JEMALLOC, at least in some small scale tests.

```shell
git clone https://github.com/jemalloc/jemalloc/releases/download/4.0.4/jemalloc-4.0.4.tar.bz2
tar -xf jemalloc-4.0.4.tar.bz2
cd jemalloc-4.0.4
./configure --prefix="current_working_directory/jemalloc" --with-jemalloc-prefix="je_"
make
make install
```

This is also a dynamic library that needs to be loaded:

```
export LD_LIBRARY_PATH=/home/hongyang/Vlasiator/vlasiator/lib/jemalloc/lib
```

## PAPI

[PAPI](http://icl.cs.utk.edu/papi/) is a memory tracker, activated only when `-DPAPI_MEM` is added to the compiler flags.

```shell
git clone https://bitbucket.org/icl/papi.git
cd papi/src/
./configure --prefix="current_working_directory/papi" CC=mpicc CXX=mpicxx
make -j 4
make install
```

A typical output of PAPI in Vlasiator looks like the following

```
(MEM) Resident per node (avg, min, max): 150.348 144.319 161.92
(MEM) High water mark per node (GiB) avg: 201.933 min: 185.869 max: 217.764 sum (TiB): 5.91601 on 30 nodes
```

The resident is how much the code uses when the memory report is done. High water mark is the highest it has been[^1]. These differ because the memory usage is very dynamic with blocks being added and then removed in acceleration and block adjustment.

[^1]: So this is different from high water mark in Linux kernel.

MPI communication requires memory buffers where to store received data. These are for the neighbor spatial cells which are empty right after the restart. After the first step with communication they have been initialized and contain block data representing the neighboring cells., therefore the numbers will be larger than the first report after restart, e.g. by a factor of ~30%.