@def title = "Postprocessing"
@def hascode = true
@def rss = ""
@def rss_title = "Postprocessing"
@def rss_pubdate = Date(2020, 12, 8)

@def tags = ["syntax", "code", "image"]

# Analyzing Results

\toc

## Tests

Inside `small_test_definitions.sh`, the tests are explicitly listed with orders, which is really not a good idea if the number of tests goes beyond the roof.

The vlsvdiff tool is also written in C++, which compares each variable and output the "distance" between them.
It is completely unnecessary to compare files with C++, but I probably won’t change it, because that’s just reinventing the wheel.
What’s needs to be done here is to properly move these test suites into make.

An ideal way to do tests is to save the SHA256 results as references, and compare any new results against them.
Eventually there should be some automated checks!


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

## Visualization

As for the visualization package, the documents are literally none, except for comments within the code.
I have actualy learned that good codes do not need extra comments, and redundant comments often indicates that the logic can be improved.
Seriously, I believe this, because computers does not read comments.
You can have thousand lines of comments showing people what is done and still make mistakes in the source code.

Without even going to details, I can tell that for example `plot_colormap` is unnecessarily complicated, with over 1200 lines just for plotting a contour with possibly some extra stuffs.
As a comparison, my `contourf` wrapper function in VisAna is 18 lines with comments.

Now I kind of understand why we should use pcolor or pcolormesh but not contourf, since we are working with cell based data! It is much easier to control in some sense.

The default colormap in analysator is called `hot_desaturated`, which is a self-defined one from the Themis IDL toolbox.
I don’t like it because it creates discontinuities that don’t exist.

Let me learn how to write wrappers for Matplotlib again and try to improve on Analysator later.

VLSV is a self-created file format for Vlasiator.

Key concepts:
* One vlsv file may contain multiple meshes.
* Each mesh may contain multiple zones.
* Each mesh is logically Cartesian.
* Zones are decomposed into irregular domains.

Mesh → zone → domain? 
Mesh → domain → zone? According to the figure this should be the case.
The manual is really confusing!

For each domain:
Separate zones into locals and ghosts.

What is the number of zones in x/y/z direction?

“Each zone has 8 nodes” → cuboid!

In principle I can now write a vlsv reader in Julia.
What makes it more interesting is to check if I can convert it into any of the VTK file formats.

vlsv is an XML based format, where the meta data is actually saved as file footer in the end.
This is perhaps the first file format I've seen that does this.
Since there is a Python reader, I think it won't be hard to write a Julia version.
This is also the best way to understand the data structure.
Let's do it!

After two days of work, I now have a working vlsv reader in Julia together with a 2D plotting function.
I read through the vlsv reader in Python and identified several issues that may affect speed:
1. In the current design, every actual reading boils down to a `read` function, which contains a lot of `if` statements, check for cell indexes, and XML properties.
Even though this may seem handy at first, this is not a good code design, as the most generic interface should appear in the top level, not the lower level.
Lower level functions should be short and efficient, while higher level functions can be huge and handle different scenarios.
Without even testing I can tell this is a big performance hit, especially because it's Python.
2. Abuse of dictionaries is a sign of bad code design.
Even dictionary may appear handy initially, it is rarely used in scientific computing largely because better types like arrays can often replace dictionaries especially when handling numerics.
One thing that needs special care is the cell indexes, since vlsv format by design allows random sequence.
In the current Python version, there are complicated logics even with `try ... except` syntax to deal with it, and a dictionary is used to store the mapping.
With better design these can all be avoided.
3. Many functions are called only once.

I know currently Julia cannot beat Python in terms of visualization, but it is still quite surprising to see that my first Julia version is already several times faster.
My philosophy here, which is deeply influenced by a Julia expert in the community, is we should not repeat what Matplotlib offers.
If some feature or functionality is thoroughly documented in Matplotlib already, we should just use it, instead of repetition.
This is by design why those `*arg` and `**arg` exist.
Simpler is better.


Vlasiator has two options for postprocessing:
1. VisIt
2. Analysator

ParaView now provides [a way to import VisIt plugins](https://www.paraview.org/Wiki/VisIt_Database_Bridge), but I haven't tried.
However, it just makes me wonder if it is possible to directly output VTK files from Vlasiator, given that it will be converted to VTK internally somehow.
The closest one I found is probably the parallel rectilinear grid (PRectilinearGrid).
Not for now, but if I am really interested and have free time.

Analysator is based on Mayavi (deprecated) and Python packages like Matplotlib and Numpy.
Mayavi requires VTK and PyQt5 as dependencies.
It took me some time to figure out how to install mayavi, VTK and PyQt5.
The easiest way to get VTK is through pip.
Note that as of now, it only supports up to Python3.6.
I tried the latest Python3.9 and got no compatible version error.
If you install it from source, it requires a rather recent cmake version, and there is a lot of issue for installing the latest cmake version on Ubuntu.
PyQt5 cannot be installed from pip, because it does not provide a `setup.py` file which is needed.
However, it can be installed through `apt-get`.

Now I can read in the vlsv files. But still, I literally have no idea what are saved in the file, or how to check what I can do.

The design of the analysator package is similar to my plotting package VisAna.
What I like about it is fulfills the needs of the 2D plots for most team members; what I don't like about it is the redundancy: the unnecessarily complicated function input arguments indicate that we are inventing the wheels for the existing Matplotlib functions.
My idea for the recent versions of VisAna is to inherit what's available in the dependency package and reduce the burden of the wrapper functions, which obviously contradicts what's happening here for these Python scripts.