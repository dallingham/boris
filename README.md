boris
=====

Python script to manage Questa/Modelsim based UVM/OVM simulations

Introduction
------------

boris is a script for managing a Mentor Graphics Questa (Modelsim)
simulation. Compling and running a modern Verilog or SystemVerilog
simulation can be complex. Many RTL files, third party libraries, DPI
object files, and PLI object files are frequently required to run a
single simulation.

Mentor provides two methods of running the simulation. The first is to
use the qverilog program. qverilog compiles and runs the entire
simulation in a single pass. While this ensures that all changed files
are recompiled, it makes the entire process of running a simulation
very slow. qverilog provides the -incr option to attempt to run an
incremental compile. While this makes a bit of an improvement in
performance, the overall performance is still very slow.

The second, and preferred, method to run a simulation is to use the
"three step process". This can be though of as the
compile/optimize/simulation methodology. In this method, each file is
compiled independently using vsim, then all compiled files are
optimized (linked) using vopt, and finally the simulation is run using
vsim. This methodology is much quicker than the single pass, qveriog
approach. However, it makes it difficult to keep track of which files
need to be recompiled when something changes. For example, if an
include file is changed to change the value of a `define, which of the
several hundred files that may comprise a simulation need to be
recompiled? Failing to recompile a file can cause incorrect simulation
results.

boris was written to solve this problem. Its goal is to use the three
step flow and ensure that every file that needs to be recompiled is
recompiled, and no files that do not need to be compiled are
recompiled. It does this by maintaining the state of every file in a
simulation, and determining all other files that may force the file to
be recompiled. If any of the file that a compile depends on changes, a
recompilation is forced. boris is able to maintain this dynamic
dependency list without any user intervention.
