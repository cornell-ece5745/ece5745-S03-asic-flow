
ECE 5745 Section 3: ASIC Automated Flow
==========================================================================

 - Author: Jack Brzozowski, Christopher Batten
 - Date: February 10, 2022

**Table of Contents**

 - Introduction
 - Test, Simulate, and Translate the Design
 - Generating an ASIC Flow
 - Pushing the Design through the Automated ASIC Flow
 - Evaluating Cycle Time
 - Evaluating Area
 - Evaluating Energy

Introduction
--------------------------------------------------------------------------

In the previous sections, we have learned how to manually run most of the
tools we will be using in the course. These tools are shown below.

![](assets/fig/asic-flow.png)

Obviously, entering commands manually for each tool is very tedious and
error prone. An agile hardware design flow demands automation to simplify
rapidly exploring the area, energy, timing design space of one or more
designs. Luckily, Synopsys and Cadence tools can be easily scripted using
TCL, and even better, the ECE 5745 staff have already created these TCL
scripts along with a set of Makefiles to run the TCL scripts using a
framework called mflowgen. In this section, we will learn how to use this
automated flow to evaluate cycle time, area, and energy of a specific
design.

The first step is to start MobaXterm and then `ssh` into `ecelinux`. Once
you are at the `ecelinux` prompt, source the setup script, clone this
repository from GitHub, and define an environment variable to keep track
of the top directory for the project.

    % source setup-ece5745.sh
    % mkdir -p $HOME/ece5745
    % cd $HOME/ece5745
    % git clone https://github.com/cornell-ece5745/ece5745-S03-asic-flow
    % cd ece5745-S03-asic-flow
    % TOPDIR=$PWD

Test, Evaluate, and Translate the Design
--------------------------------------------------------------------------

As in the previous sections, we will be using the following four-stage
registered incrementer as our example design:

![](assets/fig/regincr-nstage.png)

Before we can use the automated flow, we must make sure our design passes
all of our extensive testing. There is no sense in running the flow if
the design is incorrect!

    % mkdir -p $TOPDIR/sim/build
    % cd $TOPDIR/sim/build
    % pytest ../regincr

Next we should rerun all the tests with the `--test-verilog` and
`--dump-vtb` command line options to ensure that the design also works
after translated into Verilog and that we generate a Verilog test-bench
for gate-level simulation. You should do this step even if you are using
Verilog for your RTL design.

    % cd $TOPDIR/sim/build
    % pytest ../regincr --test-verilog --dump-vtb

The tests are for verification. We probably also want to do some
preliminary design-space exploration of execution time in cycles using an
evaluation simulator. You can run the evaluation simulator for our
four-stage registered incrementer like this:

    % cd $TOPDIR/sim/build
    % ../regincr/regincr-sim 0x10 0x20 0x30 0x40
    % less RegIncr4stageRTL__pickled.v

You should now have the Verilog that we want to push through the ASIC
flow.

Generating an ASIC Flow
--------------------------------------------------------------------------

In agile ASIC design, we usually prefer building _chip generators_
instead of _chip instances_ to enable rapidly exploring a design space of
possibilitise. Similarly, we usually prefer using a _flow generator_
instead of a _flow instance_ so we can rapidly generate many different
flows for differnt designs, parameters, and even ADKs. We will use
the mflowgen framework as our flow generator.

  - <https://mflowgen.readthedocs.io/en/latest>

We use a `flow.py` file to configure the flow. Every design you want to
push through the flow should have its own unique subdirectory in the
`asic` directory with its own `flow.py`. Let's take a look at the
`flow.py` for the four-stage registered incrementer here:

   % cd $TOPDIR/asic
   % less regincr-4stage/flow.py

There is quite a bit of information in the `flow.py`, but the important
configuration information is placed at the top:

    #-----------------------------------------------------------------------
    # Parameters
    #-----------------------------------------------------------------------

    adk_name = 'freepdk-45nm'
    adk_view = 'stdview'

    parameters = {
      'construct_path'  : __file__,
      'sim_path'        : "{}/../../sim".format(this_dir),
      'design_path'     : "{}/../../sim/regincr".format(this_dir),
      'design_name'     : 'RegIncr4stageRTL',
      'clock_period'    : 1.0,
      'clk_port'        : 'clk',
      'reset_port'      : 'reset',
      'adk'             : adk_name,
      'adk_view'        : adk_view,
      'pad_ring'        : False,

      # Gather
      'test_filter'     : 'RegIncrNstageRTL',

      # VCS-sim
      'test_design_name': 'RegIncr4stageRTL',
      'input_delay'     : 0.05,
      'output_delay'    : 0.05,

      # Synthesis
      'gate_clock'      : False,
      'topographical'   : False,

      # Hold Fixing
      'hold_slack'      : 0.070,
      'setup_slack'     : 0.035,

      # PT Power
      'saif_instance'   : 'RegIncr4stage_tb/DUT',
    }

The `adk_name` specifies the targeted technology node and fabrication
process. The `design_path` points to where all of the source files are
and the `design_name` is the name of the corresponding top-level module.
The `clock_period` is the target clock period we want to use for
synthesis and place-and-route.

To get started create a build directory and run mflowgen. Every push
through the ASIC flow should be in its own unique build directory. You
need to explicitly specify which design you want to push through the flow
when you run mflowgen.

    % mkdir -p $TOPDIR/asic/build-regincr-4stage
    % cd $TOPDIR/asic/build-regincr-4stage
    % cd build-regincr-4stage
    % mflowgen run --design ../build-regincr-4stage
    % make list

The `list` Makefile target will display the various steps in the flow.
The Makefile will take care of running the steps in the right order. You
can use the `graph` Makefile target to generate a figure of the overall
ASIC flow.

    % cd $TOPDIR/asic/build-regincr-4stage
    % make graph

You can open the generated `graph.pdf` file to see the figure which is a
much more detailed version of the high-level flow graph shown above. You
can use the `status` Makefile target to see which steps have been
completed.

Pushing the Design through the Automated ASIC Flow
--------------------------------------------------------------------------

The following commands will basically do what we did in the previous
discussion sections: (1) run all of the tests to generate appropriate
Verilog test harnesses; (2) run all of the tests using 4-stage RTL
simulation; (3) perform synthesis (the front-end of the flow); (4) run
all of the test using fast-functional gate-level simulation; (5) perform
place-and-route (the back-end of the flow); (6) run all of the tests
using back-annotated gate-level simulation.

    % cd $TOPDIR/asic/build-regincr-4stage
    % make ece5745-block-gather
    % make brg-rtl-4-state-vcssim
    % make brg-synopsys-dc-synthesis
    % make post-synth-gate-level-simulation
    % make brg-cadence-innovus-signoff
    % make post-pnr-gate-level-simulation

Instead of typing the complete step name, you can also just use the step
number shown when you use the `list` Makefile target. Go ahead and work
through these steps and monitor the output. You can also use the `status`
and `runtimes` Makefile targets to see the status of each step and how
long each step has taken.

    % cd $TOPDIR/asic/build-regincr-4stage
    % make status
    % make runtimes

Make sure the design passes four-state RTL simulation, fast-functional
gate-level simulation, and back-annotated gate-level simulation! Keep in
mind it can take 5-10 minutes to push simple designs completely through
the flow and up to an hour to push more complicated designs through the
flow. Consider using just the ASIC flow front-end to ensure your design
is synthesizable and to gain some rough early intuition on area and
timing. Then you can iterate quickly and eventually focus on the ASIC
flow back-end.

We can now open up Cadence Innovus to take a look at our final design.

    % cd $TOPDIR/asic/build/10-brg-cadence-innovus-signoff
    % innovus -64 -nolog
    innovus> source checkpoints/design.checkpoint/save.enc

The power ring seems so large because the design is so small. You can use
the design browser to help visualize how modules are mapped across the
chip. Here are the steps:

 - Choose _Windows > Workspaces > Design Browser + Physical_ from the menu
 - Hide all of the metal layers by pressing the number keys
 - Browse the design hierarchy using the panel on the left
 - Right click on a module, click _Highlight_, select a color

You can use the following steps in Cadence Innovus to display where the
critical path is on the actual chip.

 - Choose _Timing > Debug Timing_ from the menu
 - Right click on first path in the _Path List_
 - Choose _Highlight > Only This Path > Color_

You can create a screen capture to create an amoeba plot of your chip
using the _Tools > Screen Capture > Write to GIF File_. We recommend
inverting the colors so your amoeba plot looks better in a report.

Evaluating Cycle Time
--------------------------------------------------------------------------

When first pushing a design through the flow, you should check to see if
your design meets timing after synthesis. If it doesn't meet timing after
synthesis it amostly certainly won't meet timing after place-and-route!
It is always a good idea to start with a very conservative cycle time
constraint and then gradually push the tools harder.

We can see that our initial cycle time of 1.0 is probably too
conservative ... try again using 0.3ns ... do synthesis first, look at
timing, then move on to place-and-route

Evaluating Area
--------------------------------------------------------------------------

use area report from post-pnr

Evaluating Energy
--------------------------------------------------------------------------

use post-synth and post-pnr energy

