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
    * Spatial-space
      * `calculateSpatialTranslation()`: for some reason everything is commented out in this function?
    * Update system boundaries
      * `sysBoundaries.applySysBoundaryVlasovConditions()`
      * This is the first call to the BCs.
    * Compute interp moments
      * `calculateInterpolatedVelocityMoments()`
    * Propagate Fields
      * Copy moments onto the fsgrid.
      * `propagateFields()`: calculate field for the next time step.
      * `getFieldsFromFsGrid()`: copy results back from fsgrid.
    * Update velocity
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

```
using namespace phiprof
using namespace std
```

Since they are already imported, there is no need to add `std::` in the rest of code.

* `recalculateLocalCellsCache()`: why do we need another local block for the 2 lines?
* What is “ad” in the following comments? "and"?
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

Personally I don't like the name of `SetMaxwellian`. We may change it to `Inflow` and then make `Maxwellian` one option among the available choices. So I envision something like this:
* `DoNotCompute`
* `Ionosphere`
* `Outflow`
* `Inflow`
* `User`

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

* `vlasovBoundaryCopyFromTheClosestNbr()`: the last argument, `calculate_V_moments` should be renames, as it looks too similar to a function!


### SysBoundaryCondition

* `addSysBoundary()`: called from `initSysBoundaries()` in the `sysBoundary` class.

* `getSysBoundary()`: get the pointer to a specific BC.

* `initSysBoundary()`: initialize a specific BC.



### SetByUser

