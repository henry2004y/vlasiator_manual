@def title = "Run"
@def hascode = true
@def rss = ""
@def rss_title = "Run"
@def rss_pubdate = Date(2022, 07, 29)

@def tags = ["syntax", "code", "image"]

# Running Vlasiator

\toc

To run the code,

```shell
vlasiator --run_config run.cfg
```

To run from restart,

```shell
vlasiator --run_config run.cfg --restart.filename restart.vlsv
```

To check all the available options,

```shell
vlasiator --help
```

The current executable is designed to be able to read in command line arguments.
However, the list is just too long to check and read.
Also, you cannot use tab to auto-complete input file names!
It is really annoying to type the cfg file name as an argument into the command line everytime when I try to run the code.
Maybe we can set some default locations, like looking for cfg files in the current directory?

The speed difference between `-O0` and `-O3` can be 120/9=13 times! This is quite significant. Thanks compiler!

## Input Configuration File

There is no manual for the configuration parameters currently.
However, there is a script helping to identify potential errors in the configuration file, `tools/check_vlasiator_cfg.sh`.
To run that,

```shell
./check_vlasiator_cfg.sh vlasiator test.cfg
```

* Orders of commands
  * On the command line, all options are either one word (top level) or of the format block.option, they can be in any order. See below for composite option.
  * In a cfg: blocks can be in any order. Top-level options should be at the top, anything after a block title will be counted in that block until the next option. Inside one block, single options can be in any order. See below for composite options.
  * Composed options: There is options that can be repeated any number of times, e.g. `variables.output` to specify more than one output variable. Those can be in any order. However there is a few specific cases where we implemented several composed options to go in groups. In particular, the specification of output file types with the root name of the file type, its frequency, and how many VDFs we want to store. There one has to make sure that the options are in order so that the group of options goes together.
* Commands are case-sensitive, except output and diagnostic variable names.
* Normalization
  * All input units are SI.
  * All plasma parameters are set in the project you choose.
* The help message printed from `vlasiator --help` may be incomplete,outdated, wrong, or missing. When in doubt, check out the source code instead.
* There are these `proton_` prefix syntax for many tags. Vlasiator supports more than one ion population. That population's name (e.g. `proton`) gets prepended to a host of options that are specific to that population, like densities, temperatures, boundary behaviour etc.
* How to restart a run?
  * If the options are set so that restart files come out, you will get files called `restart.INDEX.DATE.vlsv`. INDEX is seven digits in seconds of simulation time, DATE is the date of the file.
  * When you want to restart from a given file, use the option `restart.filename` pointing to that file. In job scripts for multi-stage runs one trick is to use the option on the command line/in the job script as `--restart.filename $( ls -tr | restart.*.vlsv | tail -n 1 )`.

### General

```YAML
project = Flowthrough
propagate_field = 0
propagate_vlasov_acceleration = 0
propagate_vlasov_translation = 1
dynamic_timestep = 1
ParticlePopulations = proton
hallMinimumRho = 1
```

* `project`: type of problems to solve (options?)
  * In the git repository under the projects directory you can find the available projects. Some are old and may be unsupported. Projects may support custom options (e.g defining the dipole field for the Magnetosphere project)
* `propagate_field`: field solver on/off
* `propagate_vlasov_acceleration`: acceleration on/off
* `propagate_vlasov_translation`: translation on/off
* `dynamic_timestep`: 0 or 1 to have static or dynamic dt
* `ParticlePopulations`: ion species name, this is arbitrary and is prepended to all the options relevant to that population. Can be repeated to initialize multiple ion species.
* `hallMinimumRho`: minimum rho value used for the Hall and electron pressure gradient terms in the Lorentz force and in the field solver. This is a common trick in avoiding near-vaccum region in the hybrid simulations where extremely small timesteps can occur. However, this may not be enough by itself.

One thing to notice here is that currently local timestepping is not available. This should be added to speed up the steady state runs.

```YAML
[bailout]
max_memory = 55
min_dt = 1e-3
velocity_space_wall_block_margin = 1 # number of vblocks away from the vextent
```

Stop the run if any of the criteria is violated. The memory limit is only working when PAPI is used!

### Grid

All the values are assumed to be in SI units.

```YAML
[gridbuilder]
x_length = 20      # number of cells in x
y_length = 20      # number of cells in x
z_length = 1       # number of cells in x
x_min = -1.3e8     # minimum x location, [m]
x_max = 1.3e8      # maximum x location, [m]
y_min = -1.3e8     # minimum y location, [m]
y_max = 1.3e8      # maximum y location, [m]
z_min = -6.5e6     # minimum z location, [m]
z_max = 6.5e6      # maximum z location, [m]
t_max = 650        # maximum physical time allowed, [s]
dt = 2.0           # fixed timestep if static dt is applied, [s]
timestep_max = 100 # maximum number of iterations allowed
```

Boundary cells (typically 1 layer?) are included in `x_length`, etc.

My guess is that if any of the limits is reached, the simulation will terminate.

* `dt` is the single time step length (used if dynamic dt is disabled)
* `timestep_max` is the max number of timesteps taken until the run stops
* `t_max` is the max simulaton time until the run stops

Note that if we are running with a fixed timestep, we need to make sure that it is within the CFL condition.
Based on my tests, the scheme for translation is not TVD when the CFL number goes beyond 0.9.
I need to check the numerical scheme used underneath.

```YAML
[AMR]
max_spatial_level = 2
box_half_width_x = 1  # Half width of the refined box [# of base grid cells]
box_half_width_z = 1
box_half_width_y = 1
box_center_x = 0      # x coord of the center of the refined box
box_center_y = 0      # y coord of the center of the refined box
box_center_z = 0      # z coord of the center of the refined box
filterpasses          # AMR filter passes for each individual refinement level?
```

The AMR settings are basically hard-coded in each project, which explains why this input parameter list is so simple:)

Outer boundary cells must have identical refinement levels as the first layer of physical cells as of Vlasiator 5.1.

I also see a parameter for setting the AMR in velocity space, but it's not been used anywhere probably.

Is it possible to do mesh refinement in 1D or 2D? Yes, but it also depends on what kind of boundary we are using.

### Species & Velocity Space

Normalization, mass and charge

```YAML
[proton_properties]
mass = 1
mass_units = PROTON
charge = 1
```

* `mass_units`: PROTON or ELECTRON (at the latest once the electorn branch is merged)

```YAML
[proton_vspace]
vx_min = -600000.0 # minimum vx value resolved, [m/s]
vx_max = +600000.0 # maximum vx value resolved, [m/s]
vy_min = -600000.0 # minimum vy value resolved, [m/s]
vy_max = +600000.0 # maximum vy value resolved, [m/s]
vz_min = -600000.0 # minimum vz value resolved, [m/s]
vz_max = +600000.0 # maximum vz value resolved, [m/s]
vx_length = 15     # number of blocks in x
vy_length = 15     # number of blocks in y
vz_length = 15     # number of blocks in z
```

Each velocity block consists of $4\times 4\times 4$ cells. These so-called cellblocks are aimed for vectorization. The block sizes are defined as constants.
The minimum and maximum values set the extensions of the velocity space cells.
Note that discrepencies between input and output moment values will occur if the velocity space is poorly sampled.
To determine the proper velocity cell resolution, you need to calculate the thermal velocity $\sqrt(k_B T/m_p)$, which acts as a characteristic width of the Maxwellian distribution, and make sure the velocity cell is small enough to resolve the distribution.
Currently this is still empirical; maybe in the future we can add an assertion check for this criterion.

```YAML
[proton_sparse]
minValue = 1.0e-15 # distribution function threshold
dynamicAlgorithm = 0 
dynamicMinValue1 = 1 # The minimum value for the dynamic minValue
```

This command sets the threshold for resolving a distribution.
For typical magnetospheric plasma parameters, phase-space density peaks out between $1\times10^{-12}$ and $1\times 10^{-9}\ \text{m}^{⁻6}\text{s}^3$.

There are some additional options for setting sparsity:

* `dynamicAlgorithm`: type of algorithm used for calculating the dynamic minValue. `0` is none, `1` is linear algorithm based on rho, `2` is linear algorithm based on blocks. Example linear algorithm: y = kx+b, where dynamicMinValue1=k*dynamicBulkValue1 + b, and dynamicMinValue2 = k*dynamicBulkValue2 + b.

I need to ask about this.

### Solvers

```YAML
[fieldsolver]
ohmHallTerm = 0 # spatial order of the Hall term in Ohm's law, {0: 'off', 1, 2}
minCFL = 0.4
maxCFL = 0.5
resistivity = 0.0 # Resistivity for the ηJ term in Ohm's law.
diffusiveEterms = true # Enable diffusive terms in the computation of E
ohmGradPeTerm = 0 # spatial order of ∇Pₑ/ne in Ohm's law. 0: off, 1: 1st order.
electronTemperature = 0.0 # Upstream electron temperature to be used for ∇Pₑ/ne, [K]
electronDensity = 0.0 # Upstream electron density to be used for ∇Pₑ/ne, [m^-3]
electronPTindex = 0 # Polytropic index for ∇Pₑ/ne. 0: isobaric, 1: isothermal, 1.667: adiabatic
maxWaveVelocity = Inf # ?, [m/s]

[vlasovsolver]
minCFL = 0.8
maxCFL = 0.99
maxSlAccelerationRotation = 25 # maximum allowed rotation per subcycle
maxSlAccelerationSubcycles = 1
```

The default values are shown above.

* `ohmHallTerm`: 0, 1 or 2, that is the spatial order of accuracy of the Hall term in the field solver (0 turns it off altogether)
* `minCFL` and `maxCFL`: set a bracket for the allowed range of the CFL number between [0,1] to run that solver component at. Note that the field solver is instable between 0.5 and 1, so it should not be run at higher than 0.5. I don't know why there is a lower limit for CFL.
* `maxWaveVelocity`: I suspect this sets the maximum wave speed manually in the solver. I will test on this!

### Boundary Conditions

* `precedence`: Don't touch that or it might well be unstable. It says which boundary condition is chosen (higher precedence) over another at corners where two types meet.
* `+`,`-`: (see --help, + is the positive side/highest coordinate, - is the negative/lower side)

Current boundary settings

```YAML
[boundaries]
periodic_x = no
periodic_y = yes
periodic_z = yes
boundary = Outflow
boundary = Maxwellian
boundary = Ionosphere
```

* `boundary`: type of boundary class that will be initialised and usable in the run. The actual boundary type for each face is explicitly controlled by the tag with the same boundary type below.
  * `Outflow`: so far this is a Neumann condition (copy-condition fo the nearest simulation cell)

```YAML
[outflow]
precedence = 3
[proton_outflow]
face = x-
face = y-
face = y+
```

  * `Maxwellian`: Takes the values from the provided `.dat` file, `time density temperature Vx Vy Vz Bx By Bz`, partial support for time-variation is already in the code. It might be helpful to add a header line to the `.dat` file!

```YAML
[maxwellian]
face = x+
precedence = 4
reapplyUponRestart = 1

[proton_maxwellian]
dynamic = 0
file_x+ = sw1.dat
```

Note: `reapplyUponRestart` by default is 0, which means keeping going with the state existing in the restart file. If set to 1, it calls again `applyInitialState()`. This can be used to change boundary condition behaviour during a run, and currently it must be set to 1 for applying time varying boundary condition!

  * `Ionosphere`: ionospheric inner boundary

```YAML
[ionosphere]
centerX = 0.0   # ?
centerY = 0.0   # ?
centerZ = 0.0   # ?
radius = 38.2e6 # ?
precedence = 2  # ?

[proton_ionosphere]
taperRadius = 100.0e6
rho = 1.0e6
```

I can guess the meanings, but shall there be better alternatives? Let's discuss, but some options are population-specific for example.
Maybe something like

```YAML
[boundary]
boundary_x = Outflow
boundary_y = Maxwellian
boundary_z = Periodic
boundary_inner = Ionosphere
```

Internally we almost don't need to change anything, except a check on the strings.

### Initial conditions

```YAML
[Flowthrough]
emptyBox = 0  # ?
Bx = 1.0e-9   # background B field, [T]
By = 1.0e-9   # background B field, [T]
Bz = 1.0e-9   # background B field, [T]
densityModel = SheetMaxwellian # ?
```

* `emptyBox`: deprecated/undocumented, remnant from a separate project by Arto
* `densityModel`: deprecated/undocumented, remnant from a separate project by Arto

The deprecated command can be safely ignored for now.

```YAML
[proton_Flowthrough]
T = 100000.0         # initial temperature, [K]
rho  = 1000000.0     # initial number density, [/m^3]
VX0 = 4e5            # initial velocity in x, [m/s]
VY0 = 4e5            # initial velocity in y, [m/s]
VZ0 = 4e5            # initial velocity in z, [m/s]
nSpaceSamples = 2    # number of samples per cell per spatial dimension to take to compute the VDF
nVelocitySamples = 2 # number of samples per cell per veloctiy dimension to take to compute the VDF
```

* The idea is to improve the initial sampling when generating the VDF by sampling/averaging over N^3 points per vcell or spatial cell. However due among others to numerical diffusion and so on, it makes little difference whether you sample with 2³ or 5³ points, except that initialization takes exponentially longer. Thus Yann suggest to drop these features.

```YAML
[proton_Magnetosphere]
T = 100000.0         # initial temperature, [K]
rho = 1.0e5          # initial number density, [/m^3]
VX0 = -5.0e5         # initial velocity in x, [m/s] 
VY0 = 0.0            # initial velocity in x, [m/s]
VZ0 = 0.0            # initial velocity in x, [m/s]
nSpaceSamples = 1    # number of samples per cell per spatial dimension to take to compute the VDF
nVelocitySamples = 1 # number of samples per cell per veloctiy dimension to take to compute the VDF
```

These are two different projects, only one can be active in any given run (and usually only one si set in a given cfg).
However, since they appear essentially the same, we may be able to merge them into one tag.

The temperature determines the initial width of the Maxwellian distribution function.
As an estimation, the thermal speed is
\[
v_{th} = \sqrt{\frac{k_B T}{m}}
\]
with a constant factor depending on the dimension and exact definition.
For instance, protons of 1K has a thermal speed of about 90 m/s. 
This will effect the discrete velocity space in Vlasiator due to finite resolution and width.
For example, if the temperature is very low, the distribution is approximately a Dirac distribution.
Therefore, the distribution will only locate within one velocity cell, and thus taking the velocity value as the cell center value instead of the input value listed above.
The output velocity (and other moments) would be way off if the velocity space resolution is low w.r.t. the thermal speed.

The other thing worths noticing is that there will be numerical errors. Even if you set a periodic boundary with a set of constant input parameters, the output velocity will still fluctuates on the order of machine precision.

If for some reason you need to force the number density, you can use the `conserve_mass` flag which re-scales the VDF to match the original value after it's been populated.

Background field

```YAML
[Magnetosphere]
constBgBX = -3.0e-9 # constant background magnetic field x component
constBgBY = 3.0e-9  # y
constBgBZ = 3.0e-9  # z
noDipoleInSW = 1    # if 1, no dipole B in inflow BC cells
dipoleType = 2      # vector dipole
dipoleMirrorLocationX = 3.88e8
```


* `dipoleType`
  * 0: Normal 3D dipole
  * 1: line-dipole for 2D polar simulations
  * 2: line-dipole with mirror
  * 3: 3D dipole with mirror
* `dipoleMirrorLocationX`: location of the mirror dipole, which is used to cancel the components of the dipole field in the solar wind as if it is a perfect conductor. It should be set to `2*(xmax-2*cellsize)` manually if inflow is coming from the +x direction.

### Parallelization

```YAML
[loadBalance]
rebalanceInterval = 10 # time step interval for load balance
```

### IO

There is a diagnostic file and a log file.

```YAML
[proton_thermal]
radius = 0 # radius of the Maxwellian distribution, [m/s]
vx = -5e5  # center of the Maxwellian distribution, [m/s]
vy = 0.0
vz = 0.0
```

Thermal component paramters used for calculating the thermal/suprathermal moments.
A rule of thumb for the choice of `radius` is to be within a few times the thermal speed obtained upstream.
You can also plot some VDFs and look at how the distribution falls off: for non-thermal particle moments, we want values to not be contaminated by the dense solar wind core.

```YAML
[proton_energydensity]
solarwindspeed = 0
solarwindenergy = 0
```

Incoming solar wind velocity magnitude in [m/s] and ram energy in [eV]. Used for calculating energy densities.

```YAML
[io]
diagnostic_write_interval = 1 # time step interval for diagnostic output in diagnostic.txt and PAPI memory report
write_initial_state = 0       # initial value output on/off, before any solver (half)steps are taken to get us to t=0.

system_write_t_interval = 649.0 # Interval in simulation seconds to write that type of file
system_write_file_name = bulk
write_system_stripe_factor = 4

system_write_distribution_stride = 1
system_write_distribution_xline_stride = 0
system_write_distribution_yline_stride = 0
system_write_distribution_zline_stride = 0

restart_walltime_interval = -1 # Save in given wall time interval. Negative numbers disables writes
number_of_restarts = 4294967295 # Exit the simulation after a certain number of saving restarts above

write_restart_stripe_factor = 20

write_as_float = 0 # convert to single precision outputs or not
```

* `system_write_file_name`: it is arbitrary, although we have used `bulk` for the frequent simulation output files for almost 10 years now.
* `system_write_distribution_stride`: write out the velocity distribution function every N cells, this is a modulo on the cell's ID essentially.
* `system_write_distribution_xline_stride`: write out the VDF every N cells in x (same for y and z). For example,
```YAML
system_write_distribution_stride = 0
system_write_distribution_xline_stride = 50
system_write_distribution_yline_stride = 50
system_write_distribution_zline_stride = 1
```
means saving VDFs every 50 cells along X and Y axis.
* `write_restart_stripe_factor` and `write_system_stripe_factor`: speed up lustre IO performance by setting the number of Object Storage Targets (OSTs). See [Lustre File Striping](https://docs.nersc.gov/performance/io/lustre/) for more information. If a file is larger than 10 GB, it would be better to use a larger stripe factor than 1.
* `restart_walltime_interval`: by default this is negative. If this is positive by `number_of_restarts` is positive, then the run will immediately stop.

It is indeed very strange to make outputs of raw distribution functions under the IO tag instead of [`variables`].

```YAML
[variables]
output = vg_rhom            # total mass density, [kg/m^3]
output = fg_e               # electric field, [V/m]
output = fg_b               # magnetic field, [T]
output = vg_pressure        # total thermal pressure, [Pa]
output = populations_vg_rho # number density for each species, [m^-3]
output = populations_vg_v   # velocity for each species, [m/s]
output = vg_boundarytype    # type of boundaries
output = vg_rank            # MPI rank for each cell
output = populations_vg_blocks # number of vblocks
diagnostic = populations_vg_blocks # number of vblocks
```

All the quantities can be listed after either `output` or `diagnostic`.

* `output`: writes to the output vlsv files.
* `diagnostic`: writes out the min, max, mean and sum of that variable over the whole grid into the `diagnostic.txt` file. It's a handy way of monitoring some parameters during a run.

* Variable name conventions:
  * `fg`: variable on the fsgrid field solver grid, at highest resolution throughout.
  * `vg`: variable on the DCCRG Vlasov grid, including AMR in 3D potentially.
  * `populations`: used for moments: write out that (group of) moment(s) for all populations.
  * `rho`: number density, `rhom`: mass density, `rhoq`: charge density, `v`: velocity.
  * `_thermal/_nonthermal`: suffix for isolating thermal/nonthermal moments, used with parameters defined in `[populations_thermal]`.

All the output files are in vlsv format, no matter it's the restart file or regular outputs.
The restart file is used for continuing a run, which contains the full distribution, field information and meta data.
Typical output file is called bulk file, which may contain various derived quantities every few cells.

## Runtime Control

Quick cheat list:

* `touch KILL` stops the run without a restart written.
* `touch STOP` stops the run with a restart.
* `touch SAVE` writes an extra restart (not counted in the number given in the cfg).
* `touch DOLB` triggers an additional load balance.

All of these are going through the once-per-timestep check if any of these files are present on disk. They get renamed to `<file>_<date>` to avoid re-triggering and keep a record. However this relies on timesteps proceeding, so if a run is hanging indefinitely there's nothing you can do bar killing it through linux or queuing system commands.

For more information, check `ioread.cpp`.