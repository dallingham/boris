boris
=====

Python script to manage Questa/Modelsim based UVM/OVM simulations

Introduction
------------

``boris`` is a script for managing a Mentor Graphics Questa (Modelsim)
simulation. Compling and running a modern Verilog or SystemVerilog
simulation can be complex. Many RTL files, third party libraries, DPI
object files, and PLI object files are frequently required to run a
single simulation.

Mentor provides two methods of running the simulation. The first is to
use the ``qverilog`` program. qverilog compiles and runs the entire
simulation in a single pass. While this ensures that all changed files
are recompiled, it makes the entire process of running a simulation
very slow. ``qverilog`` provides the ``-incr`` option to attempt to
run an incremental compile. While this makes a bit of an improvement
in performance, the overall performance is still very slow.

The second, and preferred, method to run a simulation is to use the
"three step process". This can be though of as the
compile/optimize/simulation methodology. In this method, each file is
compiled independently using ``vcom``, then all compiled files are
optimized (linked) using ``vopt``, and finally the simulation is run
using ``vsim``. This methodology is much quicker than the single pass,
qverilog approach. However, it makes it difficult to keep track of
which files need to be recompiled when something changes. For example,
if an include file is changed to change the value of a `` `define``,
which of the several hundred files that may comprise a simulation need
to be recompiled. Failing to recompile a file can cause incorrect
simulation results.

``boris`` was written to solve this problem. Its goal is to use the
three step flow and ensure that every file that needs to be recompiled
is recompiled, and no files that do not need to be compiled are
recompiled. It does this by maintaining the state of every file in a
simulation, and determining all other files that may force the file to
be recompiled. If any of the file that a compile depends on changes, a
recompilation is forced. ``boris`` is able to maintain this dynamic
dependency list without any user intervention.

== Configuration files ==

In order to encourage reuse of designs and verification models,
``boris`` uses a set of distributed configuration files. By default,
``boris`` looks in the current working directory for a file called
``build.cfg``. The ``build.cfg`` file is very similar to the
traditional ``.f`` files used in verilog to contain command line
arguments.

A typical ``build.cfg`` file for RTL code would simply list the files to be compiled, along with any command line arguments needed to compile the RTL files. Each RTL file listed is compiled with a separate line.

A typical file would look like:

  +define+RTL
  +incdir+.
  source_1.v
  source_2.v

Boris will execute two ``vcom`` commands, one for each source file,
using the define and include path specified for each compile.

It should be noted that the include path included in any ``+incdir``
option will always be relative to the config file that it is in,
instead of the working directory of the ``boris`` command. This allows
config files to be written in a more reusable manner.

=== Using config files ===

=== Config file commands ===


