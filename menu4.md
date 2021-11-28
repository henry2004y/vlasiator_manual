@def title = "Internal"
@def hascode = true
@def date = Date(2020, 12, 10)
@def rss = ""

@def tags = ["syntax", "code"]

# Internal Code Structure

\toc

In C++, the included files should be ordered and grouped.
This part can be done with an automatic script.
It should be able to:

* Correct function and file names according to naming standard (`.hpp → .h`, `datareducer → dataReducer`)
* Switch orders of #include if required
* Correct comments indentation and argument indentations.

I need to go through the coding standard.

## Main

`vlasiator.cpp`

```cpp
#include <cstdlib>
#include <iostream>
#include <cmath>
#include <vector>
#include <sstream>
#include <ctime>
#ifdef _OPENMP
   #include <omp.h>
#endif

#include <fsgrid.hpp>

#include "vlasovmover.h"
#include "definitions.h"
#include "mpiconversion.h"
#include "logger.h"
#include "parameters.h"
#include "readparameters.h"
#include "spatial_cell.hpp"
#include "datareduction/datareducer.h"
#include "sysboundary/sysboundary.h"

#include "fieldsolver/fs_common.h"
#include "projects/project.h"
#include "grid.h"
#include "iowrite.h"
#include "ioread.h"

#include "object_wrapper.h"
#include "fieldsolver/gridGlue.hpp"
```

Stages:

1. Initialization
  * Read parameters
  * Open logFile & diagnostic
  * Init project
  * Init fieldsolver grids
  * Init grids
    * `initializeGrids()` creates the `sysBoundaries` object;
    * `isSysBoundaryCondDynamic` determines if we need to update BCs.
  * Init DROs
    * Initialize data reduction operators.
  * getFieldsFromFsGrid
    * Obtain initial B field after calling `propagateFields()` with `dt=0`?
  * compute-dt
  * write-initial-state
  * compute-dt
  * propagate-velocity-space-dt/2
2. Simulation
  * IO
    * Check whether external STOP, KILL or SAVE has been passed
      * `checkExternalCommands()`
    * Profiling and diagnosing
      * phiprof
      * `writeDiagnostic()`
    * Write system
      * not sure if this is the real output?
    * Bailing out from all processes: what does it mean?
      * `MPI-Allreduce()`
    * Write restart if needed
    * Check load balancing
  * Calculate the number of solved local cells
  * Update dt
    * This step is closely related to the leap frog scheme.
  * Propagate for one step
    * Spatial-space translation in Vlasov solver
      * `calculateSpatialTranslation()`: for some reason everything is commented out in this function?
    * Update system boundaries
      * `sysBoundaries.applySysBoundaryVlasovConditions()`
      * This is the first call to the BCs.
    * Compute interpolated moments
      * `calculateInterpolatedVelocityMoments()`
    * Propagate Fields
      * Copy moments onto the fsgrid.
      * `propagateFields()`: calculate field for the next time step.
      * `getFieldsFromFsGrid()`: copy results back from fsgrid.
    * Acceleration term in Vlasov solver
      * `calculateAcceleration()`: accelerate all particle populations to the next time step.
    * Update system boundaries
      * `sysBoundaries.applySysBoundaryVlasovConditions()`
      * This is the second call to the BCs.
    * Compute interpolated velocity moments
      * `calculateInterpolatedVelocityMoments()`
    * Project specified calls
      * `project->hook()`
3. Finalization

Questions:

* `object_wrapper.h`: it defines a struct named ObjectWrapper which contains the AMR criteria, container for user-defined mesh data, parameters for all particle species, projects, and parameters for velocity meshes.
* `fieldsolver/gridGlue.hpp`: names are not following the same standard!
* `phiprof.hpp`
The usage of this profiler is quite similar to Gabor’s library, both of which should be using `mpi_wtime` on the lower level.

During compilation, there is a warning about `narrowing conversion of ...`.
What does it mean and why do we need this?

```cpp
using namespace phiprof
using namespace std
```

Since they are already imported, there is no need to add `std::` in the rest of code.

* `recalculateLocalCellsCache()`: why do we need another local block for the 2 lines?
* What is "ad" in the following comments? "and"?
> needs to be done here already ad the background field will be set right away,
* As one can see, there are many FsGrid objects, but they now all share some common values like dx, dy, dz, etc. Why do we need separate values of those?
* The cubic spatial cell requirement is to restrict. Users have to calculate it themselves and even if the cells are not cubic the code will only give a warning message but still run. Maybe we should just add something to the kernel functions.
* The comments for initializing data reduction operators are confusing.
* The main loop with while: Phiprof diagnostic output is fixed to 10 steps. We should have the flexibility to change that.
* The condition `P::propagateField` comments say that it is only for B field? But I can see all the fields in the argument list of propagteFields?
* `getFieldsFromFsGrid()`: since the field solver is working on a regular Cartesian grid, this is supposed to copy field results back to dccrg grid for vdf?

## Definitions

* What should go into `common.h`, `definitions.h`, and `parameters.h` respectively?

`definitions.h`

* `namespace vmesh`: doesn’t it look the same with/without AMR? 
I have no clue why this local block ID has to be defined here, since it probably only lives in a local scope.

## Common

`common.h`

`Namespace sysboundarytype`: is SET_MAXWELLIAN for the fixed inflow?

## Parameters

`parameters.h`

* Why are there a max and a min CFL number?

* What is the difference between `vector<CellID>` and `vector<CellID>&`? Does the second one mean that it is essentially the same vector without copying?

## SysBoundary

The code structure here is slightly more complicated than I thought.
In the main function, `sysBoundary.applySysBoundaryVlasovConditions()` is called for setting the BCs.
`sysBoundary` is a high level wrapper class for initializing, storing, and applying all types of system boundary conditions.
The actual classes for BCs are defined in `sysboundarycondition.cpp`, within the scope of `namespace SBC`.
It contains a base abstract class called `SysBoundaryCondition`, and 5 derived classes:

* `DoNotCompute`
* `Ionosphere`
* `Outflow`
* `SetMaxwellian`
* `SetByUser`

The prefix `sys` is used to distinguish between physical simulation domain boundary and MPI process boundary.
However, in most cases MPI process boundary should be renamed to something else, and the boundary will be interpreted by most people the whole mesh's boundary.

Personally I don't like the name of `SetMaxwellian` for two reasons: firstly it is a verb, secondly it is not descriptive enough. We may change it to `Inflow` and then make `Maxwellian` one option among the available choices. Also currently `SetMaxwellian` is a subclass of `SetByUser`, which I think should not be the case. So I envision the subclasses of `sysBoundaryCondition` like this:

* `Nothing`
* `Ionosphere`
* `Outflow`
* `Inflow`
* `Fixed`
* `User`

There are 3 choices in the `Outflow` type:

1. `None`: do nothing
2. `Copy`: copy the values from the last physcial cell
3. `Limit`: multiply the values from the last physcial cell by a factor between (0,1)?

There are currently bugs related to the `Outflow` BC. 

`Fixed` is equivalent to the previous `SetMaxwellian`, `Inflow` corresponds to the time-varying BCs, and `User` is provided as an interface for extrenal implementations.
To do this, I also need to modify the enumerators defined in `common.h`. In the definition of classes, variables should always come first, followed by methods!

The base class has a private variable `sysBoundaryCondList` to keep track of all the boundary conditions as strings.
Can this possess multiple options for the same BC class?

In `Commons.h`, the general information for the grid is defined:

```cpp
struct technical {
   int sysBoundaryFlag;  /* System boundary flags */
   int sysBoundaryLayer; /* System boundary layer index */
   Real maxFsDt;         /* Maximum timestep allowed in ordinary space by fieldsolver for this cell */
   int fsGridRank;       /* Rank in the fsGrids cartesian coordinator */
   uint SOLVE;           /* Bit mask to determine whether a given cell should solve E or B components */
   int refLevel;         /* AMR Refinement Level */
};
```

!!! note
    `Real` is a self-defined type, which can be `float` or `double` depending on the compilation flag. 

* `getParameters()`: reads from the configuration file and sets periodic BC. It may be better to make periodicity just one type of BCs.

* `initSysBoundaries()`: loops through the list of system boundary conditions listed as to be used in the configuration file/command line arguments. For each of these it adds the corresponding instance and updates the member `isThisDynamic` to determine whether any `SysBoundaryCondition` is dynamic in time. This can be modified to remove the periodicity checking, and the boundary list initialization from `sysBoundaryCondList` is unnecessary. All types of boundary conditions should be prescribed in the class.

`P::xcells_ini`: what does this effect?

* `checkRefinement()`: for AMR usage presumably?

* `belongsToLayer()`: checks ghost cell layers?

* `classifyCells()`: loops through all cells and and assigns the correct `sysBoundaryFlag` depending for each BCs. This is a quite long one.

* `applyInitialState()`: loops through all `SysBoundaryConditions` and calls the corresponding `applyInitialState()` function for all existing particle species.

* `applySysBoundaryVlasovConditions()`: applys the Vlasov system boundary conditions to all system boundary cells. It loops through all `SysBoundaryConditions` and calls the corresponding `vlasovBoundaryCondition()` function for all moments for all particle species. I think this is the place where the dynamic BCs should be called.

### SysBoundaryCondition

* `determineFace()`: determine on which faces if any cell at the given coordinates is. I did some syntax optimizations.

* `addSysBoundary()`: called from `initSysBoundaries()` in the `sysBoundary` class.

* `getSysBoundary()`: get the pointer to a specific BC.

* `initSysBoundary()`: initialize a specific BC.

* `vlasovBoundaryCopyFromTheClosestNbr()`: copy the distribution and moments from (one of) the closest cell that is not a boundary. Is this used for the `Outflow` type BC? The last argument, `calculate_V_moments`, is renamed to `doCalcMomentsV` to avoid being too similar to a function.

There are two functions for setting the derivatives to zeroes.

### SetByUser

## Vlasov Solver

Reference paper: A three‐dimensional monotone and conservative semi‐Lagrangian scheme (SLICE‐3D) for transport problems.

Two controlling parameters:

* `P::propagateVlasovTranslation`
* `P::propagateVlasovAcceleration`

Two main steps:

* `calculateSpatialTranslation(mpiGrid, dt)`
* `calculateAcceleration(mpiGrid, dt)`

The top level entrance for the solver is `vlasovmover.cpp`.

I found terrifying `goto` statements for calculating moments. It may look easier to implement, but don't do it: there are always modern alternatives. Otherwise we can implement everything with `goto`, just like the "good" old days.

### Translation

A single-dispatch is used for `calculateSpatialTranslation` for each species.
`trans_map_1d()` is repeated for X, Y, and Z directions if the number of cells in that direction is larger than 1.

### Acceleration

Subcycling is applied for the acceleration term. A single-dispatch is used for `calculateAcceleration` for each species. There is a random number being used inside for specifying the order in which velocity mappings are performed

```cpp
signed int rndInt;
random_r(&rngDataBuffer, &rndInt);
uint map_order = rndInt % 3; // Order in which vx,vy,vz mappings are performed.
cpu_accelerate_cell(mpiGrid[cellID], popID, map_order, subcycleDt);
```

but I don't know why.

Now inside `cpu_accelerate_cell`, first we obtain the velocity mesh information through

```cpp
vmesh::VelocityMesh<vmesh::GlobalID,vmesh::LocalID>& vmesh    = spatial_cell->get_velocity_mesh(popID);
vmesh::VelocityBlockContainer<vmesh::LocalID>& blockContainer = spatial_cell->get_velocity_blocks(popID);
```

The Eigen library seems to provide some basic data structures for transformation. There are two types being used: the forward transformation and backward transformation. The intersections are computed in `map_order`, and then 1D mappings are done in consecutive order. The current implementation explicitly list all 3 possible orderings --- this can be refactored by referencing to variables and write the function call only once.

#### Transformation

`cpu_acc_transform.cpp`

#### Intersection

`cpu_acc_intersections.cpp`

* `compute_intersections_1st`

Computes the first intersection data; this is z~ in section 2.4 in Zerroukat et al (2012). We assume all velocity cells have the same dimensions. Intersection z coordinate for (i,j,k) is
> intersection + i * intersection_di + j * intersection_dj + k * intersection_dk 

  * `spatial_cell`: spatial cell that is accelerated
  * `fwd_transform`: transform that describes acceleration forward in time
  * `bwd_transform`: transform that describes acceleration backward in time, used to compute the lagrangian departure gri
  * `dimension`: along which dimension is this intersection/mapping computation done. It is assumed the three mappings are in order 012, 120 or 201 
  * `refLevel`: refinement level at which intersections are computed
  * `intersection`: intersection z coordinate at i,j,k=0
  * `intersection_di`: change in z-coordinate for a change in i index of 1
  * `intersection_dj`: change in z-coordinate for a change in j index of 1
  * `intersection_dk`: change in z-coordinate for a change in k index of 1

* `compute_intersections_2nd`
  * for x

* `compute_intersections_3rd`
  * for y
  * euclidian y goes from vy_min to vy_max, this is mapped to wherever y plane is in lagrangian (???)

#### Mapping

```cpp
map_1d(
  SpatialCell* spatial_cell,
  const uint popID,     
  Realv intersection, Realv intersection_di, Realv intersection_dj,Realv intersection_dk,
  const uint dimension)
```

Map from the current time step grid, to a target grid which is the lagrangian departure grid (so the grid at timestep +dt, tracked backwards by -dt).
