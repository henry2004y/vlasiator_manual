@def title = "Testing"
@def hascode = true
@def date = Date(2021, 02, 20)
@def rss = ""

@def tags = ["syntax", "code"]

# Testing

\toc

There exists some test funtionalities under the `testpackages` folder, but it is not good enough.
* Currently one can compare the output from one version of the code to another version, but obviously there are many problems with this approach. How can you guarantee the version you select generate the correct "reference" solutions?
* The shell scripts are written for specific machines, but none of them are running on a regular basis.
* One has to do many extra work to create tests on a new machine.
* We cannot pick a specific test to run easily with one command.

To tackle these problems, I am thinking about redesign the whole test suites. The goal is to
* define reference solutions to compare with;
* create a flexible test list with `make`;
* introduce continuous integration (CI) to validate new development and changes

There is a tool written in C++ called `vlsvdiff` that can be used to compare two VLSV files. This can be used to investigate the differences between files. For automated tests, it is enough to know that there are differences. The difficulties for VLSV file format result from the random writing order from MPI processes. If you are running with multiple MPI processes, you may get different SHA values out of the VLSV file even if the data are actually identical.

How to store the reference solutions? Well, they should be saved in a different repository to keep the main source repo clean.

## Comparing Differences

This is a big headache now.
I get different results with different code versions on different platforms, even if there shouldn't be any.

1. There are two GCC optimization flags that will affect the results even on the same machine with slightly differernt code versions: `-ffast-math`, which boost the floating point operation performance by loosening the IEEE standard; `-mavx`, which turns on the avx instructions on the target machine. I can understand the first, but not the second one. Both flags have influence on the speed beyond `-O3`, which are about 10%. 
2. Even if I turn off all the optimization flags, there are still differences running on different machines. I don't have any idea how this can happen.

## Demo

Under the `testpackage` folder:

```shell
make
```
runs all the listed tests in `Makefile`.

```shell
make TESTS=flowthrough
```
runs a specific test.