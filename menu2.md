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
Besides, it does not have auto-completion for file names.
Maybe we can set some default locations, like looking for cfg files in the current directory?


The speed difference between `-O0` and `-O3` can be 120/9=13 times! This is quite significant. Thanks compiler!

## Input Configuration File

Apparently there is no manual for the configuration parameters.

* Can I switch the orders of commands?
* Are the commands case-sensitive, e.g. `Outflow`, `outflow`?
* Can change the normalizations?
* Urgent need to describe all the available options for each command.
* I am curious about this `proton_` syntax for many tags. Is it a common pattern for all the available commands?


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
* `dynamic_timestep`: time stepping option?
* `ParticlePopulations`: ion species name (Is there an option list? Or arbitrary name?)

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
* `dt`
* `timestep_max`

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

* `mass_units`: do we have other options?

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

* `ohmHallTerm`: ?
* `minCFL`: ?
* `maxCFL`: ?

### Boundary Conditions

* `precedence`: ?
* `+`,`-`: what is the convention?

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

* `boundary`: type of boundary according to the sequence.
  * `Outflow`: float I guess?
```YAML
[outflow]
precedence = 3
[proton_outflow]
face = x-
face = y-
face = y+
```
  * `Maxwellian`: which value to take? From the input `*.dat` file?
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

Yes I can guess, but shall there be better alternatives?
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

* `emptyBox`: ?
* `densityModel`: ? 

```YAML
[proton_Flowthrough]
T = 100000.0         # initial temperature
rho  = 1000000.0     # initial number density
VX0 = 4e5            # initial velocity in x
VY0 = 4e5            # initial velocity in y
VZ0 = 4e5            # initial velocity in z
nSpaceSamples = 2    # ?
nVelocitySamples = 2 # ?
```

```YAML
[proton_Magnetosphere]
T = 100000.0         # initial temperature?
rho = 1.0e5          # initial number density
VX0 = -5.0e5         # initial velocity in x
VY0 = 0.0            # initial velocity in x
VZ0 = 0.0            # initial velocity in x
nSpaceSamples = 1    # ?
nVelocitySamples = 1 # ?
```

What does this command mean?
Why do we need two identical commands with different names?

Background field
```YAML
[Magnetosphere]
constBgBX = -3.5355339e-9 # ?
constBgBY = 3.5355339e-9  # ?
noDipoleInSW = 1.0        # ? float but not integer?
```

### Parallelization

```YAML
[loadBalance]
rebalanceInterval = 10 # time step interval for load balance?
```

### IO

There is a diagnostic file and a log file.

```YAML
[io]
diagnostic_write_interval = 1 # time step interval for diagnostic output 
write_initial_state = 0       # initial value output on/off

system_write_t_interval = 649.0 # ?
system_write_file_name = bulk
system_write_distribution_stride = 1
system_write_distribution_xline_stride = 0
system_write_distribution_yline_stride = 0
system_write_distribution_zline_stride = 0
```

* `system_write_file_name`: is it arbitrary?
* `system_write_distribution_stride`: does it affect all the values or only f? If it's all the output quantities, why do we include "distribution" in the name?


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
* `diagnostic`: what is the difference compared to `output` other than writing to different files? 

All the output files are in vlsv format, no matter it's the restart file or regular outputs.
The restart file is used for continuing a run, which contains the full distribution, field information and meta data.
Typical output file is called bulk file, which may contain various derived quantities every few cells.
