@def title = "Testing"
@def hascode = true
@def date = Date(2023, 02, 14)
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
* test one thing at a time;
* introduce continuous integration (CI) to validate new development and changes.

There is a tool written in C++ called `vlsvdiff` that can be used to compare two VLSV files. This can be used to investigate the differences between files. For automated tests, it is enough to know that there are differences. The difficulties for VLSV file format result from the random writing order from MPI processes. If you are running with multiple MPI processes, you may get different SHA values out of the VLSV file even if the data are actually identical.

How to store the reference solutions? Well, they should be saved in a different repository to keep the main source repo clean.
To compare results, more often than not we do not need the raw distribution values f: by not saving it we can save a lot of storage!

## Comparing Differences

This is a big headache now.
I get different results with different code versions on different platforms, even if there shouldn't be any.

1. There are two GCC optimization flags that will affect the results even on the same machine with slightly differernt code versions: `-ffast-math`, which boost the floating point operation performance by loosening the IEEE standard; `-mavx`, which turns on the avx instructions on the target machine. I can understand the first, but not the second one. Both flags have influence on the speed beyond `-O3`, which are about 10%.
2. Even if I turn off all the optimization flags, there are still differences running on different machines for multiple tests, e.g. `acctest_1_maxw_500k_30kms_1deg`. Among all the variables being saved there, the largest difference comes from `populations_vg_rho_loss_adjust`, which records the sparse distribution dropping/adding. I don't have any idea how this can happen: maybe the values are so small, and the round-off errors accumulate with timesteps?

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

It would be nice to save the test log somewhere, but for now just ignore it. Also `vlsvdiff` can substitute julia for comparing differences.

## Tests

These tests are borrowed from [Athena](https://www.astro.princeton.edu/~jstone/Athena/tests/linear-waves/linear-waves.html). We choose 3 reference values: $n_0=10^6\,\mathrm{m}^{-3}$, $B_0 = 10^{-9}\,\mathrm{T}$, $m_0=m_i$. All the other conversion factors from dimensionless to SI can be derived:

* lRef: 100 [m]
* tRef: 0.00458462 [s]
* vRef: 21812 [m/s]
* nRef: 1e+06 [#]
* TRef: 28818.8 [K]
* PRef: 3.97887e-13 [Pa]
* BRef: 1e-09 [T]

To make sure the velocity space is refined as much as possible, we check the ratio between thermal speed and the vspace parameters:

* Vth: 21812 [m/s]
* Vth/dvx = 29.0827
* Vth/Vmax = 0.181767

In principle the choice of basic reference values should not effect the numerical results. As a referece, the single precision epsilon is $\epsilon=1.1920929f-7$, and the double precision epsilon is $\epsilon=2.220446049250313e-16$.

### Free stream test

Everything is uniform, which means nothing should change. Because of the existence of magnetic field, the rotation in the velocity space may have a detectable numerical effect.

$\gamma = 5/3$.

| Variable | Background | Background (SI) |
|----------|------------|-----------------|
|$\rho$    | $1.0$      | $1\times 10^6$  |
|$u_x$     | $0.0$      | $0.0$           |
|$u_y$     | $0.0$      | $0.0$           |
|$u_z$     | $0.0$      | $0.0$           |
|$p$       | $1/\gamma=0.6$ | $2.3873241463784306e-13$ |
|$B_x$     | $1.0$      | $1\times 10^{-9}$ |
|$B_y$     | $\sqrt{2}$ | $1.4142135\times 10^{-9}$ |
|$B_z$     | $0.5$      | $0.5\times 10^{-9}$ |

* Single precision VDF:
  * Number density decreases by $2.5\times 10^{-7}$. The absolute error is on the order of single precision $\epsilon$.
  * Velocity changes from $0$ to $\sim\mathcal{O}(1)\,\mathrm{m/s}$. Not sure is this is ok.
  * Magnetic field keeps the initial values, which is good.

* Double precision VDF:
  * Number density error is below single precision $\epsilon$.
  * Velocity changes from $0$ to $\sim\mathcal{O}(10^{-6})\,\mathrm{m/s}$. I would say this is perfectly fine.
  * Magnetic field keeps the initial values, which is good.

### Linear waves test

The waves are propagating in an oblique direction so as to check grid effects.

The results are really sensitive to the perturbation magnitude $A$. If $A=10^{-3}$, single and double precision results are almost identical; if $A=10^{-6}$, the single precision results are completely off while the double precision results at least still maintain the wave-like pattern. I found $A=10^{-4}$ as roughly when the single and double precision results differ.

The moments are very sensitive to the velocity grid resolutions as well as extent. With double precision VDFs, the initial condition follows the analytical solution well even at $A=10^{-6}$; with single precision VDFs, the initial condition has observable errors at $A=10^{-6}$ and even larger errors at larger $A$.

Double precision VDF runs are in general much better, but we can still see both diffusion errors and dispersion errors.

#### Alfven wave

| Variable | Background | Perturbations |
|----------|------------|---------------|
|$\rho$    | $1.0$      | $0.0$         |
|$\rho u_x$| $0.0$      | $0.0$         |
|$\rho u_y$| $0.0$      | $-3.333333333333333e-1$ |
|$\rho u_z$| $0.0$      | $9.428090415820634e-1$ |
|$\mathcal{E}$ | $\frac{1}{\gamma(\gamma-1)}$ | $0.0$ |
|$B_x$     | $1.0$      | $0.0$ |
|$B_y$     | $\sqrt{2}$ | $-3.333333333333333e-1$ |
|$B_z$     | $0.5$      | $9.428090415820634e-1$ |

Depending on the velocity space resolutions, the single precision VDF may not even has a average value of 0 vy/vz. Double precision does not suffer from this issue.

Both diffusion and dispersion errors are observed.

\figenv{Velocity and magnetic field perturbations of Alfven wave after one period.}{/assets/img/alfven_1d_n100_1period.png}{width:100%;border: 1px solid red;}

#### Fast wave

| Variable | Background | Perturbations |
|----------|------------|---------------|
|$\rho$    | $1.0$      | $4.472135954999580e-1$ |
|$\rho u_x$| $0.0$      | $-8.944271909999160e-1$ |
|$\rho u_y$| $0.0$      | $4.216370213557840e-1$ |
|$\rho u_z$| $0.0$      | $1.490711984999860e-1$ |
|$\mathcal{E}$ | $\frac{1}{\gamma(\gamma-1)}$ | $2.012457825664615$ |
|$B_x$     | $1.0$      | $0.0$ |
|$B_y$     | $\sqrt{2}$ | $8.432740427115680e-1$ |
|$B_z$     | $0.5$      | $2.981423969999720e-1$ |

Both diffusion and dispersion errors are observed. It looks slightly better than the Alfven wave test.

\figenv{Density, velocity and magnetic field perturbations of fast wave after one period.}{/assets/img/fast_1d_n100_2period.png}{width:100%;border: 1px solid red;}

#### Slow wave

| Variable | Background | Perturbations |
|----------|------------|---------------|
|$\rho$    | $1.0$      | $8.944271909999159e-1$ |
|$\rho u_x$| $0.0$      | $-4.472135954999579e-1$ |
|$\rho u_y$| $0.0$      | $-8.432740427115680e-1$ |
|$\rho u_z$| $0.0$      | $-2.981423969999720e-1$ |
|$\mathcal{E}$ | $\frac{1}{\gamma(\gamma-1)}$ | $6.708136850795449e-1$ |
|$B_x$     | $1.0$      | $0.0$ |
|$B_y$     | $\sqrt{2}$ | $-4.216370213557841e-1$ |
|$B_z$     | $0.5$      | $-1.490711984999860e-1$ |

The slow wave test is the strangest among all. At first the wave did propagate to the left as expected, but it stopped propagating after a quarter of the wave period and then behaved like a standing wave.

\figenv{Density, velocity and magnetic field perturbations of slow wave after one period.}{/assets/img/slow_1d_n100_1period.png}{width:100%;border: 1px solid red;}

### GEM Challenge
