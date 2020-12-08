@def title = "Installation"
@def hascode = true
@def date = Date(2020, 12, 8)
@def rss = ""

@def tags = ["syntax", "code"]

# Installation

Vlasiator depends on many packages.
These can now be handled with my `configure.py` at ease.
However, the documentation on the dependencies still needs to be improved!

\toc


Now with some modifications, the process looks like the following:

1. For a fresh new machine,
```shell
./configure.py -h
```
shows the available options.

```shell
./configure.py -install
```
tries to install all the dependencies in the new `lib` folder.

2. To access an existing customized Makefile in `MAKE` folder with library paths and compiler flags, e.g.
```shell
./configure.py --machine=yann
```
then it will incorporate those into generating the main Makefile.

3. To save the configured paths into a new customized Makefile (by default Makefile.new)
```shell
./configure.py -install -save
```
to specify a name
```shell
./configure.py -install -save --save_machine=mymachine
```
or something like
```shell
python3.6 configure.py -debug -save -mpi -omp -distfloat \
    --include=src/projects \
    --include=/usr/include/boost \
    --include=lib/zoltan/include \
    --include=lib/vlsv \
    --include=lib \
    --vectorclass_path=lib/vectorclass \
    --fsgrid_path=lib/fsgrid \
    --dccrg_path=lib/dccrg \
    --include=lib/fsgrid \
    --include=lib/jemalloc/include \
    --include=lib/phiprof/include \
    --include=lib/vectorclass \
    --lib=boost_program_options \
    --lib=zoltan \
    --lib=vlsv \
    --lib=jemalloc \
    --lib=phiprof \
    --boost_path=/usr/lib/x86_64-linux-gnu \
    --zoltan_path=lib/zoltan/lib \
    --vlsv_path=lib/vlsv \
    --jemalloc_path=lib/jemalloc/lib \
    --profile_path=lib/phiprof/lib
```

If you really want to set everything manually, just remove the `-save` above and it will generate the main Makefile.

I still keep the options of changing order of solvers, floating point precisions, etc.. Even though they rarely change, it is still changeable, but the default is set to the most common options.

By default options with one `-` require no arguments, and options with two `--` require one argument.

I do not provide support for python2, since the official support has also ended by the end of last year. Therefore is the default python in `/usr/bin/env` is python2, then the configuration file will raise error at one `print` statement with `end` argument.

The tools should in principle also be able to be compiled, but I am having some issues with those.

## DCCRG

DCCRG is the underlying grid and MPI library for Vlasiator. I guess this provides similar functionality compared to **AMReX** and **BATL**.

## Zoltan

Zoltan is used to do loading balancing, or spatial partitioning of the computation.

It can be downloaded from Sandia's website: [Zoltan](http://cs.sandia.gov/Zoltan/Zoltan_Distributions/zoltan_distrib_v3.83.tar.gz).

## Boost

I don't know what Boost is used for. Maybe argument parsing?

## Eigen

No idea what Eigen is used for. However, the latest Eigen version has compilation error in assertions:

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
export LD_LIBRARY_PATH=/home/local/hongyang/Vlasiator/vlasiator/lib/phiprof/lib
```
