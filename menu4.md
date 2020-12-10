@def title = "Internal"
@def hascode = true
@def date = Date(2020, 12, 10)
@def rss = ""

@def tags = ["syntax", "code"]

# Internal Code Structure

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

1. `Initialization`
  * `Read parameters`
  * `open logFile & diagnostic`
  * `Init project`
  * `Init fieldsolver grids`
  * `Init grids`
  * `Init DROs`
  * `getFieldsFromFsGrid`
  * `compute-dt`
  * `write-initial-state`
  * `compute-dt`
  * `propagate-velocity-space-dt/2`
2. `Simulation`
  * `IO`
    * `checkExternalCommands`
    * `logfile-io`
    * `diagnostic-io`
    * `write-system`
    * `Bailout-allreduce`
    * `compute-is-restart-written-and-extra-LB`
    * `write-restart`
  * `update-dt`
  * `Propagate`
    * `Spatial-space`: `calculateSpatialTranslation()`
    * `Update system boundaries (Vlasov post-translation)`: `sysBoundaries.applySysBoundaryVlasovConditions()`
    * `Compute interp moments`: `calculateInterpolatedVelocityMoments()`
    * `Propagate Fields`: `propagateFields()`
      * `fsgrid-coupling-in`
      * `getFieldsFromFsGrid`
    * `Velocity-space`: `calculateAcceleration()`
    * `Update system boundaries (Vlasov post-acceleration)`
    * `Compute interp moments`
3. `Finalization`

Questions:
* `object_wrapper.h`: what is this used for?
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
* `getFieldsFromFsGrid`: since the field solver is working on a regular Cartesian grid, this is supposed to copy field results back to dccrg grid for vdf?


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
