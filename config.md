<!--
Add here global page variables to use throughout your
website.
The website_* must be defined for the RSS to work
-->
@def prepath = "vlasiator_manual"
@def website_title = "Personal Vlasiator Manual"
@def website_descr = "How to use Vlasiator in detail"
@def website_url   = "https://henry2004y.github.io/vlasiator_manual/"

@def author = "Hongyang Zhou"

@def mintoclevel = 2

<!--
Add here files or directories that should be ignored by Franklin, otherwise
these files might be copied and, if markdown, processed by Franklin which
you might not want. Indicate directories by ending the name with a `/`.
-->
@def ignore = ["node_modules/", "franklin", "franklin.pub"]

<!--
Add here global latex commands to use throughout your
pages. It can be math commands but does not need to be.
For instance:
* \newcommand{\phrase}{This is a long phrase to copy.}
-->
\newcommand{\R}{\mathbb R}
\newcommand{\scal}[1]{\langle #1 \rangle}
