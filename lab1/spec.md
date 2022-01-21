# EECS 251B Lab 1 - Chipyard: An Agile RISC-V SoC Design Framework

<p align="center">
Profs: Bora Nikolic, Vladimir Stojanovic, Sophia Shao
</p>
<p align="center">
TA: Erik Anderson
</p>
<p align="center">
Department of Electrical Engineering and Computer Science
</p>
<p align="center">
College of Engineering, University of California, Berkeley
</p>

## Overview

<p align="center">
 <img src="figs/chipyard-flow.PNG" alt="train_perf_fig"/>
    <b>
    <em>Fig. 1 - Chipyard Flow</em>
    </b>
</p>

In this lab, we will explore the [Chipyard](https://github.com/ucb-bar/chipyard) framework. 
Chipyard is an integrated design, simulation, and implementation framework for open source hardware development
developed here at UC Berkeley. Chipyard is open-sourced online and is based on the Chisel and FIRRTL hardware 
description libraries, as well as the Rocket Chip SoC generation ecosystem. Chipyard brings together much of
the work on hardware design methodology from Berkeley over the last decade, as well as other useful tools, 
into a single repository that gaurantees version compatibility between the projects it submodules.

A designer can use Chipyard to build, test, and tapeout (manufacture) a RISC-V-based SoC. This includes RTL
development integrated with Rocket Chip, cloud FPGA-accelerated simulation with FireSim, and physical design
with the Hammer framework. Information about Chisel can be found in [https://www.chisel-lang.org/](https://www.chisel-lang.org/).
While you will not be required to write any Chisel code in this lab, basic familiarity with the language will
be helpful in understanding many of the components in the system and how they are put together. An initial introduction
to Chisel can be foud in the Chisel bootcamp: [https://github.com/freechipsproject/chisel-bootcamp](https://github.com/freechipsproject/chisel-bootcamp).
Detailed documentation of Chisel functions can be found in [https://www.chisel-lang.org/api/SNAPSHOT/index.html](https://www.chisel-lang.org/api/SNAPSHOT/index.html).

There is a lot in Chipyard so we will only be able to explore a part of it, but hopefully you will get a brief
sense of its capabilities. Chipyard has great [documentation](https://chipyard.readthedocs.io/en/latest/) and will
be useful to reference throughout the semester. In this lab, you will simulate a Rocket Chip-based design at the 
RTL level, and then elaborate it with SRAMS from the ASAP7 free process design kit (PDK) using part of the 
Hammer back-end flow. Then, you will be challenged to find out how Chipyard is structured and learn how to modify
things in preparation for your own project.

<p align="center">
 <img src="figs/chipyard-components.PNG" alt="train_perf_fig"/>
    <b>
    <em>Fig. 2 - Chipyard Components</em>
    </b>
</p>

## Getting Started
First, we will need to set up our Chipyard workspace. To begin, create an instructional account at 
[https://acropolis.cs.berkeley.edu/~account/webacct/](https://acropolis.cs.berkeley.edu/~account/webacct/). 
After loggin in via "Login using your Berkeley CalNet ID", you should find that you can create an account for
the eecs251b group. All of our work will occur on the instructional servers on the `eda-[1-8].eecs.berkeley.edu` 
machines. You may connect to these machine directly via SSH with X11 Forwarding or via [X2Go](https://wiki.x2go.org/doku.php/download:start).

For this course, it may be wise to work in the `/scratch` directory on the lab machine. Create a folder for yourself
there, and make sure you login to the same server each time. Chipyard may generate too much data for it to fit in
your home directory.

run the commands below in ***Bash terminal only***. This may take about 10 minutes.

```
mkdir /scratch/<your-username>
cd /scratch/<your-username>
git clone /home/ff/eecs251b/sp22-workspace/chipyard
cd chipyard
./scripts/init-submodules-no-riscv-tools.sh
./scripts/init-vlsi.sh asap7
```

You may have noticed while initializing your Chipyard repo that there are many submodules. Chipyard is built
to allow the designer to generate complex configurations from different projects including the in-order Rocket
Chip core, the out-of-order BOOM core, the systolic array Gemmini, and many other components needed to build a chip.
Thankfully, Chipyard has some great documentation, which can be found [here](https://chipyard.readthedocs.io/en/latest/).
You can find most of these in the `chipyard/generators/` directory. All of these modules are built as generators (a
core driving point of using Chisel), which means that each piece is parameterized and can be fit together with some 
of the functionality in Rocket Chip (check out TileLink and Diplomacy references in the Chipyard documentation).
You can find the Chipyard specific code and its configs in `chipyard/generators/chipyard/src/main/scala/config/`.
You can look at examples of how your own Chisel modules or Verilog black-box modules can be integrated into a 
Rocket Chip-based SoC in `chipyard/generators/chipyard/src/main/scala/example/`. Generally, an accelerator block
is connected to the Rocket core with a memory-mapped interface over the system bus. This allows the core to 
configure and read from the block. Again, there is far too much to discuss fully here, but you can really put
together a system very quickly using the infrastructure of Chipyard.


Before continuiing, this environment setup script must be run from the `chipyard` directory every time you use
Chipyard in a new terminal session.

```
source scripts/inst-env.sh
```

## Chipyard Simulation and Design Benchmarking

### RTL Simulation

In general, Chipyard Chisel-based designs look something like a Rocket core connected to some kind of "accelerator"
(e.g. a DSP block like an FFT module). When building something like that, you would typically build your "accelerator"
generator in Chisel, and unit test it using ChiselTesters. You can then write integration tests (e.g. a baremetal C
program) which can then be simulated with your Rocket Chip and "accelerator" block together to test end-to-end system
functionality. Chipyard provides the infrastructure to help you do this for both VCS (Synopsys) and Verilator (open-source).
In this lab, we are just focusing on a Rocket core in isolation, so we will run some assembly tests on a Rocket config.
You can edit the configs being used in `chipyard/variables.mk` or through overriding the make invocation with `CONFIG=YourConfig`.
In this case, we will work with a Rocket config that is a basic Rocket core (includes L1 data and instruction caches) with an
on-chip scratchpad memory. Go to the `chipyard/sims/vcs/` directory and run:

```
cd sims/vcs
make
make BINARY=$RISCV/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple run-binary
```

The first command will elaborate the design and create the Verilog. This is done by converting the Chisel code, embedded
in Scala, into a FIRRTL intermediate representation which is then run through the FIRRTL compiler to generate Verilog.
Next it will run VCS to build a simulator out of the generated Verilog that can run RISC-V binaries. The second command 
will run the test specified by BINARY and output results in the `output/` directory. You can look at the suite of different tests
in the directories specified in the command we ran. The default target uses the scala class RocketConfig to define the parameters 
of the generator. To get a list of all common make variables and targets for simulation run `make help`.

***Q1: In your lab report, include a screenshot of the last 10 lines of the `.out` file generated by the assembly test you ran. 
It should include the *** PASSED *** flag.***

### Firesim

A key component of Chipyard is FireSim, which is an open-source cycle-accurate FPGA-accelerated full-system hardware simulation
platform that runs on cloud FPGAs (Amazon EC2 F1). This allows for simulations of your system at orders-of-magnitude faster speeds
than software simulation. A key point of FireSim is that it is cycle-accurate since it actually models things like DRAM access
latency by transforming parts of your design. Emulating your design normally on an FPGA does not model these system-level aspects 
that your actual chip will run with. Using FireSim is outside the scope of this lab, but it is worth looking in to.

## Design Elaboration and SRAM Mapping

In the previous section, we elaborated the design for just a functional simulation in VCS. To begin the physical design (VLSI)
flow targeting a specific technology, we need to elaborate our design a bit differently. The physical design flow we will use
throughout the semester is integrated into Chipyard in the `vlsi/` folder. After navigating to the base chipyard directory, we can
set up our VLSI back-end by running the following commands:

```
cd vlsi/
make srams INPUT_CONFS=lab1/lab1.yml
```

This will be a multi-step process. First, it will elaborate our Rocket design similarly to the `sims/` directory, but this time,
abstract memory constructs in the source Chisel are translated into hard SRAM macros using the MacroCompiler in the `barstools`
submodule. This MacroCompiler step is required because the memories available in the technology will not exactly match the ones 
required by your design. This step gives you the list of technology-available SRAMS that map to your design and fits the connections 
into the correct parts of your RTL.

Next, the chosen SRAMs from the technology library are inserted into the elaborated Verilog files with the proper connections.
The elaboration results can be found in `vlsi/generated-src/`.

There are many files that are generated in this folder that relate to FIRRTL compilation of the Chisel design and of course the
final Verilog output files. All of the file names are pre-fixed with the name of the config used. For example, in the 
`generated-src/chipyard.TestHarness.RocketConfig` directory, many files are prefixed with `chipyard.TestHarness.Rocketconfig`.

The file `top.mems.conf` describes the parameters of the memories in your design and `top.mems.v` shows the actual Verilog
instantiations of the individual SRAM blocks used (the names of the memory blocks will match the Verilog module names).


***Q2: What is the breakdown of SRAM blocks for each of the memories in the design? This can be found by looking at the files described above.***

***Q3: Now, take a look at the `top.anno.json` file This contains a listing of all the targets (i.e. instances) that need to be
transformed in specific ways by FIRRTL in the Chisel to Verilog elaboration. In your report, show one of the SRAM annotations
and describe the correspondence with what was generated from Q2.***

`generated-src/` also includes your elaborated Verilog in top.v, files related to your system's devoce tree, and even a `graphml`
file that visualizes the diplomacy graph of the different components in your system (can be viewed with an [online viewer](https://www.yworks.com/yed-live/).

Finally, Hammer is executed using the information from the `top.mems.conf` file to gather all of the collateral needed for physical design
with SRAMs. The outputs of Hammer are in the `build/chipyard.TestHarness.RocketConfig-ChipTop` directory. Note that the first time Hammer
invokes the ASAP7 PDK, it extracts the PDK tarball and hacks it into the `tech-asap7-cache/` directory, so it may take a few minutes.

In this build directory, take a look at the `sram-generator-output.json` file. You will find a structure under the key `vlsi.technology.extra_libraries`.
This structure is needed to provide all of the files needed for a physical design flow with IP blocks such as SRAM macros.

***Q4: Choose any library in the list of extra_libraries, and explore the files listed. In your lab report, concisely describe what
each of the files are for and where in a physical design flow they will be used. We will dig deeper into these in an upcoming lab.***

## Exploring the Chipyard Infrastructure

Now, let's delve deeper into how Chipyard is structure by having you go on a scavenger hunt. You may also cross-reference the
[Chipyard documentation](https://chipyard.readthedocs.io/en/latest/) to guide you.

***Q5: In section 3.1, we typed `make` in `sims/vcs` and it elaborated a default Rocket Chip configuration. In your lab report,
copy the section of the file that selects this default subproject and config.***

***Q6: There are many different Rocket-based SoC configs that are shipped with Chipyard. In your lab report, point to the file
that contains all the available Rocket-based configs and list which configs are available. Then, show that `MbusScratchpadRocketConfig`
(this replaces an off-chip memory port with a scratchpad memory) passes the same binary test that we ran in Section 3.1.***

***Q7: By inspecting the available configs, we can try to construct our own custom one. In your lab report, write the config (in the form
 `class <Lab1CustomConfig> extends Config(...)`) that would instantiate the following frankenstein SoC:***

- 2 big Rocket cores and 2 large BOOM cores
- 1 each of Hwacha and Gemmini accelerators
- No L2 cache
- No port for external backing memory

***Q8: It may be useful for your projects to see how Verilog can be integrated into a Chisel system. The GCD MMIO peripheral is a good 
example of this. In your lab report, describe which file allows us to choose whether to use a Verilog or Chisel version of the GCD,
as well as the path to the Verilog file.***

***Q9: To start a new project using Chisel in Chipyard, you will need to do a few things. In your lab report, outline where you will put
your new project code and its directory structure (following convention), and which files you need to change in order to be able to compile 
code from your new project.***

## Conclusion

Chipyard is designed to allow you to rapidly build and integrate your design with general purpose control and compute as well as a whole host of other
generators. You can then take your design, run some RTL simulations, benchmark it with FireSim, and then push it through the VLSI flow with the technology
of your choice using Hammer. The tools integrated with Chipyard, from how you actually build your design (e.g. Chisel and generators), to how you verify
and benchmark its performance, to how you physically implement it, are meant to enable higher design QoR within an agile hardware design process through
increased designer productivity and faster design iteration. We just scratched the surface in this lab, but there are always more interesting features
being integrated into Chipyard. We recommend that you continue to explore what you can build with Chipyard given this introduction!


## Acknowledgements
Thank you to Daniel Grubb and Harrison Liew for authoring this lab, and thank you to Alon Amid and the entire Chipyard dev team for figures and documentation
on Chipyard.
