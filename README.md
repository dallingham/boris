# boris #
 
Python script to manage Questa/Modelsim based UVM/OVM simulations 

## Introduction ##

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
compiled independently using ``vlog``, then all compiled files are
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

If you are running multiple UVM or OVM tests one after another, your
code will not change, and boris will bypass the compile and optimize
steps for each run, and only run the simulation stage.

## Configuration files ##

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
    source1.v  
    source2.v  

Boris will execute two ``vlog`` commands, one for each source file,
using the define and include path specified for each compile.

It should be noted that the include path included in any ``+incdir``
option will always be relative to the config file that it is in,
instead of the working directory of the ``boris`` command. This allows
config files to be written in a more reusable manner.

### Including other config files ###

Since RTL designs tend to not be build in a single directory,
``boris`` allows config files to include other config files. These
config files can be located in other directories. Including another
config file is done using the ``@import <path>`` command.

When a config file imports another, the imported file inherits the
environment of the parent config file. This includes any environment
variables, defines, or command line options.  However, options set in
the child config file typcially do not influence the parent's
environment.

This allows a child config file to add additional defines or command
line options without affecting other config files.

For example, if the parent config file is:

    +define+RTL  
    +incdir+.  
       
    @import "../block2/build.cfg"  
     
    block1.v  

And the child config file is:

    +define+FAST  
    +incdir+.  
    block2.v  

Then two commands will be executed:

    vlog +define+RTL +define+FAST +incdir+. +incdir+../block2 ../block2/block2.v  
    vlog +define+RTL +incdir+. block1.v  

Notice that the path of the ``+incdir`` and the path to the source
file in the child config file was changed during execution from paths
relative to the config file to paths relative to the simulation
directory. This allows blocks to be easily reused, since all paths are
specified relative to the child config's directory, and it becomes
``boris``'s responsibility to map those correctly to the simulation
directory.

Also note that the define and include path in the child config did not
affect the command line of the block compiled from the parent config
file.

If it is important that a command in a child config file be added to
the parent's environment, the command may be exported to the parent
using the ``@export`` option. This may be useful if your config file
repesents a simulation model that requires PLI or DPI code to be
specified.

    @export -pli /opt/tools/mypli.so

This push the PLI command to the parent, so that all simulation
environments that use the model do not have to remember to set the PLI
path.

### Using @define and cases to control config files ###

``boris`` allows you to specify a simulation case on the command
line. This defines a token in the config file that can be used for
testing.

Example:

    boris --case=FPGA test1

This will defined the token ``FPGA`` in the config file. This can be
tested using the ``@if`` structure.

    @if FPGA  
      +define+USE_FPGA  
    @else  
      +define+USE_ASIC  
    @endif  

In this case, if the FPGA case is selected, ``+define+USE_FPGA`` will
be added to the command line, otherwise, ``+define+USE_ASIC`` will be
added to the command line.

Since cases can cause the command lines to differ, and for code to be
compiled in different ways, ``boris`` will use a different work
directory for each case. This reduces compilation time by eliminated
the need to recompile when switching cases.

You may also used defines in the config files. These will define
tokens with a preceding ``@`` symbol. Using the ``--define``
command line argument, you can change the values of the tokens.

Example:

    boris --define=DEBUG test1  

This defines the @DEBUG token, and can be checked in the config file.

    @if @DEBUG  
       +acc  
    @endif  

In this case, if @DEBUG is defined the ``+acc`` option is added. 

The major difference between defines and cases is that defines do not
build in a separate work directory. So if you option affects anything
other than ``vsim``, you will probably reoptimize or recompile all
your code everytime you change the command line argument.

Therefore, defines are typically used for change ``vsim`` simulaton
options.

## Config file options ##

``boris`` attempts to figure out which command line option goes with
with which program of the three stage flow. For example, ``-c``, which
means to not use the graphical interface, will only be sent to
``vsim``. ``+define``, which affects compilation, will only be sent to
``vlog``.

But the entire command line option set for Modelsim is vast, and boris
does not always understand all the options. You can force an option
using build in extensions.

    @vlog +this_is_sent_to_vlog
    @vopt -this_is_sent_to_vopt
    @vsim +this_is_sent_to_vsim

Other enhanced commands include:

* @setenv  
  Sets the environment variable in the simulation environment. This is 
  passed to the simulation environment, and defines a value that can 
  be used within the config files.
* @if  
  Conditional test to determine if code should be run. May not be nested.
* @elif  
  Similar to the @else, but allows a condition to be tested
* @else  
  Default if the values in @if or @elif where not met
* @endif  
  Ends a condition block
* @export
  Exports the command to the global space, affecting all config files. 
  Acts as if the command was in the top level config file







