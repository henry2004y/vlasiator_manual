@def title = "Postprocessing"
@def hascode = true
@def rss = ""
@def rss_title = "Postprocessing"
@def rss_pubdate = Date(2020, 12, 8)

@def tags = ["syntax", "code", "image"]

# Analyzing Results

\toc

## Tests

Inside `small_test_definitions.sh`, the tests are explicitly listed with orders, which I think is not a good idea if the number of tests goes beyond the roof.

The vlsvdiff tool is also written in C++, which compares each variable and output the "distance" between them.
It is an overkill to compare files using C++, but writing another program for this purpose is just reinventing the wheel.
What’s probably needed to be done here is to properly move these test suites into make.

An ideal way to do tests is to save the SHA256 results as references, and compare any new results against them.
Nice thing about this is that it is a light weight reference to the "correct" result; bad thing is that one cannot tell directly which quantity goes wrong.
But it may not be so bad: when you debug the code, you typically generate an old reference data and compare against it directly.
Eventually there should be some automated checks, bundled in the CI workflow.

The first thing that works:

```shell
../../../vlsvdiff_DP bulk.0000001.vlsv bulk.0000001.vlsv proton/vg_rho 0
```

I made some changes to the run_tests.sh.

When I ran the `run_tests.sh` script with creating the reference solutions, I found that I couldn’t completely kill the process with a single `ctrl-c` command.
Typing it twice might work though.

The standard tests are too long: up to the small magnetosphere test (8th), it already takes 1 hour using one i5 core running at 3.3GHz!

Stupid shell scripts does not allow whitespaces between assignment!

I can now successfully compare results1.


## File Format

VLSV is a self-created XML based file format for Vlasiator, and the first format I know that outputs metadata at the end of file.

Key concepts:

* One vlsv file may contain multiple meshes (fsgrid, dccrg grid, etc.)
* Each mesh may contain multiple zones.
* Each mesh is logically Cartesian.
* Zones are decomposed into irregular domains.

I am confused by the [vlsv description manual](https://github.com/fmihpc/vlsv/blob/master/doc/user%20manual.pdf).

Questions:

1. What is the correct hierarchy order of concepts? Is it `Mesh → zone → domain`, or `Mesh → domain → zone`? According to the first figure in the manual the latter should be the case. However, in the description of Multi-Domain Meshes section, it says
> zones are decomposed into independent domains.
In the caption of the first figure
> Cartesian mesh decomposed into four domains

2. For each domain, zones are separated into locals and ghosts. According to my understanding, mesh is logically Cartesian, but not zones. Then what is the number of zones in x/y/z direction, as shown in the demo code in the Write Mesh Bounding Box section?

3. In the end of Write Bounding Box Node Coordinates section, it is mentioned that 
> Each zone has 8 nodes → cuboid.
I am again puzzled. My current understanding of the dccrg grid is that on a lower level, each block contains $4
\times 4 \times 4$ cubic cells, and the whole simulation domain is organized by blocks.
Load balancing is done on the block level.
I am not sure if this is correct, and how can this map to the "mesh, zone, domain" concept?

My general impression of the vlsv format is that conceptually it is very similar to the rectilinear format in VTK (with the suffix `.vtr`) and it correponding brother `PRectilinerarGrid` (where prefix "P" stands for parallel).
There is already a VisIt plugin for vlsv, but I don't know how hard it is to directly generate `*.vtr` files from Vlasiator.
Another thing to note is that the newer XML-based VTK files can take advantage of the compression algorithms that can shrink the file size significantly depending on the statistics of data, but currently vlsv does not have anything like that.

## Visualization

There are currently three main tools for plotting:

* [Analysator](https://github.com/fmihpc/analysator) in Python;
* VisIt with [VLSV plugin](https://github.com/fmihpc/vlsv/tree/master/visit-plugin);
* [Vlasiator.jl](https://henry2004y.github.io/Vlasiator.jl/dev/).

### Reader

Readers are coded in C++, Python and Julia.
Writer is only available in C++.

Analysator is based on Mayavi (deprecated) and Python packages like Matplotlib and Numpy.
Mayavi requires VTK and PyQt5 as dependencies.
It took me some time to figure out how to install mayavi, VTK and PyQt5.
The easiest way to get VTK is through pip.
Note that as of now, it only supports up to Python3.6.
I tried the latest Python3.9 and got no compatible version error.
If you install it from source, it requires a rather recent cmake version, and there is a lot of issue for installing the latest cmake version on Ubuntu.
PyQt5 cannot be installed from pip, because it does not provide a `setup.py` file which is needed.
However, it can be installed through `apt-get`.

Questions:

1. In the Python version, I notice a `cellids` dictionary which is a lookup table for data ordering. Does this correspond to the concept of blocks as described in vlsv? There is a sorting call somewhere in the plotting function like `plot_colormap`. It may be better to do the sorting while reading the `cellids` and save it as an array.
2. It is hard to tell what can I do if someone randomly throws me a vlsv file. For example, I don't know if it's 1D, 2D, or 3D, and what kind of plots are available.
This may be partly solved with more examples.
3. In Python we may be able to take advantage of the`*args` and `**keyargs` of functions to release the burden of rewriting every existing arguments of the plotting functions in Matplotlib. As some new members like me who has used Matplotlib before but haven't touched the Analysator wrappers so often, it would be very straightforward to return the PyPlot objects from the wrappers, and then I can just rely on the Matplotlib official documentation for the rest.
4. One Python plotting package worth mentioning is yt for astrophysics. I am not familiar with it, but I am impressed by its automatic unit conversion, simple rendering and some other stuff. Maybe worth keeping an eye on that. 

### Analysator

The VDF plotting function has bugs in the normal direction. The selection criteria by default assumes a certain cell width in the normal direction centered at a peak location instead of full projection.

### VisIt

There is a VisIt plugin written in C++ and described in the VLSV manual.

I learned before that VisIt does a better job than ParaView for AMR data when the ghost cell info are neglected in the files which are essentially not needed.
There are bugs in ParaView for, e.g., isosurface contour for parallel VTK format.

### Julia

See more in the Vlasiator.jl [document](https://henry2004y.github.io/Vlasiator.jl/dev/). I recommend Matplotlib for the plotting backend, because as of early 2021, it is still the most complete package for visualization.

### ParaView

ParaView now provides [a way to import VisIt plugins](https://www.paraview.org/Wiki/VisIt_Database_Bridge), but it requires to build Paraview from scratch.

In [Vlasiator.jl](https://henry2004y.github.io/Vlasiator.jl/dev/), there is a method `write_vtk` for converting VLSV format into VTK format. With this we can directly visualize Vlasiator outputs in ParaView except phase space information. However, note that as of ParaView 5.9 there are bugs in supporting the UniformOverlappingAMR class.
