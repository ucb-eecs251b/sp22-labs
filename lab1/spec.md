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
the work on hardware design methodology from Berkeley over the last decade as well as useful tools into a single
that gaurentees version compatibility between the projects it submodules.

A designer can use Chipyard to build, test, and tapeout (manufacture) a RISC-V-based SoC. This includes RTL
development integrated with Rocket Chip, cloud FPGA-accelerated simulation with FireSim, and physical design
with the Hammer framework. Information about Chisel can be found in [https://www.chisel-lang.org/](https://www.chisel-lang.org/).
While you will not be required to write any Chisel code in this lab, basic familiarity with the language will
be helpful in understanding many of the components in the system and how they are put together. An initial introduction
to Chisel can be foud in the Chisel bootcamp: [https://github.com/freechipsproject/chisel-bootcamp](https://github.com/freechipsproject/chisel-bootcamp).
Detailed documentation of Chisel functions can be found in [https://www.chisel-lang.org/api/SNAPSHOT/index.html](https://www.chisel-lang.org/api/SNAPSHOT/index.html).

There is a lot in Chipyard so we will only be able to explore a part of it, but hopefully you will get a brief
sense of its capabilities. We will simulate a Rocket Chip-based design at the RTL level, and then synthesize
and place-and-route it in ASAP7 using the Hammer flow.


<p align="center">
 <img src="figs/chipyard-components.PNG" alt="train_perf_fig"/>
    <b>
    <em>Fig. 2 - Chipyard Components</em>
    </b>
</p>

## Getting Started
First, we will need to setup of Chipyard workspace. For this lab, please work on the **eda-[1-8]@eecs.berkeley.edu** 
machines and in the **/scratch/** directory on the machine. This lab will likely generate too much data for it
to fit in your home directory. Run the following commands to set up your workspace:
```
cd /scratch/<my-username>
git clone ~ee251b/spring22-labs/chipyard
cd chipyard
./scripts/init-submodules-no-riscv-tools.sh
cd vlsi
git submodule update --init hammer-cadence-plugins
source sourceme.sh
```

You may have noticed while initializing your Chipyard repo that there are many submodules. Chipyard is built
to allow the designer to generate complex configurations from different projects including the in-order Rocket
Chip core, the out-of-order BOOM core, the systolic array Gemmini, and many other components needed to build a chip.
Thankfully, Chipyard has some great documentation, which can be found [here](https://chipyard.readthedocs.io/en/latest/).
You can find most of these in the `chipyard/generators/` directory. All of these modules are built as generators (a
core driving point of using Chisel), which means that each piece is parameterized and can be fit together with some 
of the functionality in Rocket Chip (check out TileLinnk and Diplomacy references in the Chipyard documentation).
You can find the Chipyard specific code and its configs in `chipyard/generators/chipyard/src/main/scala/config`.
You can look at examples of how your own Chisel modules or verilog black-box modules can be integrated into a 
Rocket Chip-based SoC in `chipyard/generators/chipyard/src/main/scala/example`. Many time, an accelerator block
is connected to the Rocket core with a memory-mapped interface over the system bus. This allows the core to 
configure and read from the block. Again, there is far too much to discuss fully here, but you can really put
together a system very quickly using the infrastructure of Chipyard.

## Chipyard Simulation and Design Benchmarking
### RTL Simulation
### Firesim

## VLSI Flow
### Design Elaboration
### Synthesis
### Floorplanning
### Place and Route (P&R)

## Rest of the VLSI Flow

## Conclusion
