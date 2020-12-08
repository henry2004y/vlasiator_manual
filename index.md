@def title = "My Personal Vlasiator Manual"
@def tags = ["syntax", "code"]

# Vlasiator

\tableofcontents <!-- you can use \toc as well -->

Vlasiator, generally speaking, is a numerical model for collisionless ion-kinetic plasma physics.


## Features

Can a global magnetosphere model simulate the precipitation fluxes?
I think the key question is whether a plasma description is applicable also to these energetic species.

I have seen so many frustrations about using multi-fluid MHD.
It makes wonder if PIC or Vlasov solver can do better in terms of numerical stability.

Other than Vlasiator, Umeda from Japan is the other leader in the Vlasov solver.

Vlasov solvers in higher dimensions are known to be slow. But how slow it is?
3D Earth: 1 week of running  = 6 million cpu hours?

One restart file for 3D Earth now is typically ~2TB.

* semi-Lagrangian Vlasov solver
* 3D mesh refinement in Cartesian boxes
* Up to isotropic $\nabla P_e$ term in the Ohm's law when solving for the electric field?

$$
\frac{\partial f_\alpha}{\partial t} + \mathbf{v}\frac{\partial f_\alpha}{\partial \mathbf{r}} + \frac{q_\alpha}{m_\alpha}<\mathbf{E}+\mathbf{v}\times\mathbf{B}>\cdot \frac{\partial f_\alpha}{\partial \mathbf{r}} = 0
$$


## Workflow with Git

I originally made some mistakes in the git workflow.
Vlasiator team chooses to merge into the dev branch instead of the master branch.
So everytime you fork the repository, commit some changes, and then submit a pull request to the main dev branch.
However, what happens if while waiting for the commits to be merged, I want to work on a different part of the code and commit new changes?
If you stay on your own feature branch, the new commits will go into the previous pull request and in the end you will get a huge pull request.
Similar situation is being described [here](https://stackoverflow.com/questions/56875810/new-pull-request-when-one-is-already-opened).

Now I kind of realize where I was doing wrong.
"Working off a branch" usually means you 
1. clone a repository, e.g. `git clone http://repository`
2. check out the branch you're interested in `git checkout awesome-branchname`,
3. and create a new branch based of that `git checkout -b new-even-more-awesome-branch-name`

Instead of step 3, I directly worked from step 2, and now it becomes an issue.
But what can I do now? The best might be: rename my dev branch to something that contains my latest changes, revert the changes in the original dev branch, and then submit another pull request from the renamed branch to the main dev branch. It’s tricky!

Another approach, which is what I am currently doing, is to rename the local branch to something else and start working from there.
In this way I assume that the previous commits in my dev branch will be merged into the main dev branch eventually, and I will not pollute that PR with my new commits.
This seems easier and safer for me.

As with many peer review projects, pull requests take a long time in the queue waiting to be merged.

