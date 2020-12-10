@def title = "Run"
@def hascode = true
@def rss = ""
@def rss_title = "Run"
@def rss_pubdate = Date(2020, 12, 8)

@def tags = ["syntax", "code", "image"]

# Running Vlasiator

\toc


The so-called cellblock, with size `4x4x4`, is aimed for vectorization.
The block sizes are defined as constants.

The current executable is designed to be able to read in command line arguments.
However, the list is just too long to check and read.
Also, you cannot use tab to auto-complete input file names!
It is really annoying to type the cfg file name as an argument into the command line everytime when I try to run the code.
Maybe we can set some default locations, like looking for cfg files in the current directory?


The speed difference between `-O0` and `-O3` can be 120/9=13 times! This is quite significant. Thanks compiler!

## Input Configuration File

Apparently there is no manual for the configuration parameters.

* Can I switch the orders of commands?
  * On the command line, all options are either one word (top level) or of the format block.option, they can be in any order. See below for composite option.
  * In a cfg: blocks can be in any order. Top-level options should be at the top, anything after a block title will be counted in that block until the next option. Inside one block, single options can be in any order. See below for composite options.
  * Composed options: There is options that can be repeated any number of times, e.g. `variables.output` to specify more than one output variable. Those can be in any order. However there is a few specific cases where we implemented several composed options to go in groups. In particular, the specification of output file types with the root name of the file type, its frequency, and how many VDFs we want to store. There one has to make sure that the options are in order so that the group of options goes together.
* Are the commands case-sensitive, e.g. `Outflow`, `outflow`?
  * Yes.
* Can I change the normalizations? For example, I may want 1D cells of unit length 1, and fixed dt and velocity.
  * Units are SI, but you can use the values that you like.
  * You can set the option to not use dynamic dt.
  * All plasma parameters are set in the project you choose.
* Urgent need to describe all the available options for each command.
  * `vlasiator --help`, if the help text is incomplete, outdated, wrong, missing that has to be added in the code.
* I am curious about this `proton_` syntax for many tags. Is it a common pattern for all the available commands?
  * Vlasiator supports more than one ion population. That population's name (e.g. `proton`) gets prepended to a host of options that are specific to that population, like densities, temperatures, boundary behaviour etc.
* How to restart a run?
  * If the options are set so that restart files come out, you will get files called `restart.INDEX.DATE.vlsv`. INDEX is seven digits in seconds of simulation time, DATE is the date of the file, as this helps  in a lot of practical matters.
  * When you want to restart from a given file, use the option `restart.filename` pointing to that file. In job scripts for multi-stage runs one trick is to use the option on the command line/in the job script as `--restart.filename $( ls -tr | restart.*.vlsv | tail -n 1 )`.


### General

```YAML
project = Flowthrough
propagate_field = 0
propagate_vlasov_acceleration = 0
propagate_vlasov_translation = 1
dynamic_timestep = 1
ParticlePopulations = proton
```

* `project`: type of problems to solve (options?)
* `propagate_field`: field solver on/off
* `propagate_vlasov_acceleration`: acceleration on/off
* `propagate_vlasov_translation`: translation on/off
* `dynamic_timestep`: 0 or 1 to have static or dynamic dt
* `ParticlePopulations`: ion species name, this is arbitrary and is prepended to all the options relevant to that population.

### Grid

All the values are assumed to be in SI units.
```YAML
[gridbuilder]
x_length = 20      # number of cells in x
y_length = 20      # number of cells in x
z_length = 1       # number of cells in x
x_min = -1.3e8     # minimum x location
x_max = 1.3e8      # maximum x location
y_min = -1.3e8     # minimum y location
y_max = 1.3e8      # maximum y location
z_min = -6.5e6     # minimum z location
z_max = 6.5e6      # maximum z location
t_max = 650        # maximum physical time allowed
dt = 2.0           # ?
timestep_max = 100 # maximum number of iterations allowed
```

I guess there should an internal logic for which restriction comes first for timestepping?
* `dt` is the single time step length (used if dynamic dt is disabled)
* `timestep_max` is the max number of timesteps taken until the run stops
* `t_max` is the max simulaton time in seconds until the run stops

### Species & Velocity Space

Normalization, mass and charge
```YAML
[proton_properties]
mass = 1
mass_units = PROTON
charge = 1

[proton_vspace]
vx_min = -600000.0 # minimum vx value resolved
vx_max = +600000.0 # maximum vx value resolved
vy_min = -600000.0 # minimum vy value resolved
vy_max = +600000.0 # maximum vy value resolved
vz_min = -600000.0 # minimum vz value resolved
vz_max = +600000.0 # maximum vz value resolved
vx_length = 15     # number of cells in x     
vy_length = 15     # number of cells in y
vz_length = 15     # number of cells in z

[proton_sparse]
minValue = 1.0e-15 # distribution function threshold
```

* `mass_units`: PROTON or ELECTRON (at the latest once the electorn branch is merged)

### Solvers

```YAML
[fieldsolver]
ohmHallTerm = 2
minCFL = 0.4
maxCFL = 0.5

[vlasovsolver]
minCFL = 0.8
maxCFL = 0.99
maxSlAccelerationRotation = 22 # maximum allowed rotation per cycle?
```

* `ohmHallTerm`: 0, 1 or 2, that is the spatial order of accuracy of the Hall term in the field solver (0 turns it off altogether)
* `minCFL` and `maxCFL`: set a bracket for the allowed range of the CFL number to run that solver component at (between 0 and 1, 1 meaning that the fastest signal crosses 1 cell per dt). Note that the field solver is instable between 0.5 and 1, so it should not be run at higher than 0.5.

### Boundary Conditions

* `precedence`: Don't touch that or it might well be instable. But it says which boundary condition is chosen (higher precedence) over another at corners where two types meet.
* `+`,`-`: what is the convention? (see --help, + is the positive side/highest coordinate, - is the negative/lower side)

Current boundary settings
```YAML
[boundaries]
periodic_x = yes
periodic_y = yes
periodic_z = yes
boundary = Outflow
boundary = Maxwellian
boundary = Ionosphere
```

* `boundary`: type of boundary class that will be initialised and usable in the run.
  * `Outflow`: so far this is a Neumann condition (copy-condition fo the nearest simulation cell)
```YAML
[outflow]
precedence = 3
[proton_outflow]
face = x-
face = y-
face = y+
```
  * `Maxwellian`: Takes the values from the provided dat file, `time density temperature Vx Vy Vz Bx By Bz`, partial support for time-variation is already in the code.
```YAML
[maxwellian]
face = x+
precedence = 4
[proton_maxwellian]
dynamic = 0
file_x+ = sw1.dat
```
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

Yes I can guess, but shall there be better alternatives? Let's discuss, but some options are population-specific for example.
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
Bx = 1.0e-9   # background B field
By = 1.0e-9   # background B field
Bz = 1.0e-9   # background B field
densityModel = SheetMaxwellian # ?
```

* `emptyBox`: deprecated/undocumented, remnant from a separate project by Arto
* `densityModel`: deprecated/undocumented, remnant from a separate project by Arto

```YAML
[proton_Flowthrough]
T = 100000.0         # initial temperature
rho  = 1000000.0     # initial number density
VX0 = 4e5            # initial velocity in x
VY0 = 4e5            # initial velocity in y
VZ0 = 4e5            # initial velocity in z
nSpaceSamples = 2    # number of samples per cell per spatial dimension to take to compute the vdf
nVelocitySamples = 2 # number of samples per cell per veloctiy dimension to take to compute the vdf
```

```YAML
[proton_Magnetosphere]
T = 100000.0         # initial temperature?
rho = 1.0e5          # initial number density
VX0 = -5.0e5         # initial velocity in x
VY0 = 0.0            # initial velocity in x
VZ0 = 0.0            # initial velocity in x
nSpaceSamples = 1    # number of samples per cell per spatial dimension to take to compute the vdf
nVelocitySamples = 1 # number of samples per cell per veloctiy dimension to take to compute the vdf
```

What does this command mean?
* These are two different projects, only one can be active in any given run (and usually only one si set in a given cfg).

Why do we need two identical commands with different names?

Background field
```YAML
[Magnetosphere]
constBgBX = -3.5355339e-9 # constant background magnetic field x component
constBgBY = 3.5355339e-9  # same in y
noDipoleInSW = 1.0        # ? float but not integer? It gets parsed anyway, no impact...
```

### Parallelization

```YAML
[loadBalance]
rebalanceInterval = 10 # time step interval for load balance
```

### IO

There is a diagnostic file and a log file.

```YAML
[io]
diagnostic_write_interval = 1 # time step interval for diagnostic output 
write_initial_state = 0       # initial value output on/off, before any solver (half)steps are taken to get us to t=0.

system_write_t_interval = 649.0 # Interval in simulation seconds to write that type of file
system_write_file_name = bulk
system_write_distribution_stride = 1
system_write_distribution_xline_stride = 0
system_write_distribution_yline_stride = 0
system_write_distribution_zline_stride = 0
```

* `system_write_file_name`: it is arbitrary, although we have used `bulk` for the frequent simulation output files for almost 10 years now.
* `system_write_distribution_stride`: Write out the velocity distribution function every N cells, this is a modulo on the cell's ID essentially.
* `system_write_distribution_xline_stride`: Write out the VDf every N cells in x (same for y and z).


```YAML
[variables]
output = vg_rhom
output = fg_e
output = fg_b
output = vg_pressure
output = populations_vg_rho
output = populations_vg_v
output = vg_boundarytype
output = vg_rank
output = populations_vg_blocks
diagnostic = populations_vg_blocks
```

* `output`:
  * `vg_rhom`: ?
  * `fg_e`: electric field
  * `fg_b`: magnetic field
  * `vg_pressure`: total ion pressure?
  * `populations_vg_rho`: ?
  * `populations_vg_v`: ?
  * `vg_boundarytype`: ?
  * `vg_rank`: ?
  * `populations_vg_blocks`: ?
* `diagnostic`: this writes out the min, max, mean and sum of that variable over the whole grid into the `diagnostic.txt` file, it's a handy way of monitoring some parameters during a run.
* `output` means that the full snapshot of that variable is written to the vlsv file(s).
* Variable name conventions (Markus will write more):
  * `fg`: variable on the fsgrid field solver grid, at highest resolution throughout.
  * `vg`: variable on the DCCRG Vlasov grid, including AMR in 3D potentially.
  * `populations`: used for moments: write out that (group of) moment(s) for all populations.
  * `rho`: number density, `rhom`: mass density, `rhoq`: charge density, `v`: velocity.

All the output files are in vlsv format, no matter it's the restart file or regular outputs.
The restart file is used for continuing a run, which contains the full distribution, field information and meta data.
Typical output file is called bulk file, which may contain various derived quantities every few cells.
