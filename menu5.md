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

## Demo

```shell
make test_flowthrough
```
