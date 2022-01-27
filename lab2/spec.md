# EECS 251B Lab 2 - SHA-3 Accelerator Design in Chisel  

<p align="center">
Profs: Bora Nikolic, Sophia Shao, Vladimir Stojanovic
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

TODO add overview

## Chisel Project Creation

For this project we will be setting up an additional repository besides chipyard. While chipyard
is great, it is not necessarily the easiest environment to begin developing Chisel RTL. 
Thankfully, an official Chisel template repository can be found here: 
[Chisel Project Template](https://github.com/freechipsproject/chisel-template).
This repository makes it incredibly easy to create a new Chisel repo w/o worrying about setting
up all the details of building scala/Chisel code. In your private workflow 
(research, work, hobby), you should follow the instructions given in the 
README of this repository to create your own repo on github. For this lab,
the course staff has done this for you. After logging into the 
instructional machines (`eda-[1-8].eecs.berkeley.edu`) Run the following commands
to clone the Chisel project for lab 2.

```
cd /scratch/<your-username>
git clone /home/ff/eecs251b/sp22-workspace/sp22-chisel-project
```

Lab 1 should have already familiarized you with the sbt directory structure. If you need
refreshing visit this [page](https://www.scala-sbt.org/1.x/docs/Directories.html).
The template repo comes with a couple example Chisel source files 
(`DecoupledGCD.scala` and `GCD.scala`) as well as an example test for the `DecoupledGcd` 
module. To run this test run one of the following commands:

```
sbt test
```

OR

```
sbt 'testOnly gcd.GcdDecoupledTester'
```

The first command will run all tests found in our project while the second command
will run just the `GcdDecoupledTester`.

TODO TODO TODO - Force them to write the main function for simple Chisel elaboration. 
For some reason this is not included???

TODO TODO TODO - Force them to add something to GCD block?? 

## SHA-3 Introduction
Secure hashing algorithms represent a class of hashing functions that provide four
attributes: 
 
1. Ease of hash computation
2. Inability to generate the message from the hash (one-way property)
3. Inability to change the message and not the hash (weakly collision free property) 
4. Inability to find two messages with the same hash (strongly collision free property) 

The National Institute of Standards and Technology (NIST) recently held a competition
for a new algorithm to be added to its set of Secure Hashing Algorithms (SHA). 
In 2012, the winner was determined to be the Keccak hasing function and rough 
specification for SHA-3 was established. The algorithm operates on variable 
length messages with a [sponge function](https://en.wikipedia.org/wiki/Sponge_function),
and thus alternates between absorbing chunks of the message into a set of state bits
and permuting the state. The absorbing is a simple bitwise XOR while the permutation
is amore complex function composed of several operations, &#1D6D8; 

TODODOTOTTOTO add above unicode characters

that all perform various bitwise operations, including rotations, parity calculations,
XORs, etc. The Keccak hashing function is parameterized for different sizes of state
and message chunks, but for this lab we will only support the Keccak-256 variant
with 1600 bits of state and 1088 bit message chunks. In addition, for this lab we
will ignore the variable length portion to avoid one of the most complicated
parts of Keccak: message padding. Our interface, which is discussed further below,
assumes a single chunk of message is ready to be absorbed and hashed. You can
see a block diagram of what your resulting design should look like in Figure 1.

<p align="center">
 <img src="figs/sha3_block_diagram.png" alt="sha3_block_diagram"/>
    <b>
    <em>Fig. 1 - Block Diagram for SHA-3</em>
    </b>
</p>

The SHA-3 standard is documented in FIPS PUB 202. The final approved specification of SHA-3
can be found [here](https://doi.org/10.6028/NIST.FIPS.202). You are encouraged to take
a quick glance at this specification as it is a good example of a standards documents. If you
are new to reading standards documents, you will probably notices that the descriptions
in the document are quite dense and may require some time to understand. This is true
of many standards documents with FIPS PUB 202 being a rather short example 
(the original IEEE 802.11 standard has over 2,700 pages with each addendum such as b, g, 
n, and ac adding more pages). Hardware designers are often responsible for implementing
different standards so learning how to read standards documents is a valuable skill.

Fortunately, you will not need to look at the FIPS PUB 202 too closely for this lab as C
refernce implementation is provided for you. The C implementation constitues the 
software golden reference for the lab. The first step in most projects is to create a
software golden refernce that produces the behavior that is expected from the hardware
design. The results produced by the hardware design are compared against the results 
from the golden reference design to determine if the hardware design is functioning 
as expected. In addition to providing the model by which the hardware design is checked,
the golden reference can also serve as a basis for the initial hardware design.

***Q1: Why might one produce a standard document like FIPS PUB 202 isntead of simply
releasing a C reference? Answer this question at a high level by looking at
the sections of FIPS PUB 202 that detail the Keccak algorithm (sections 3.2-3.4).***

***Q2:What are some benefits of producing a software golden reference before beginning
the hardware design? What are some benefits of this type of verification vs. writing
explicit `poke` and `expect` style tests? What are some dangers of this type of 
verification?***

You will implement the SHA-3 design based on the C reference implementation. You 
are not encouraged to check the consistency of the C implementation with the
standards document as this will take a long time and is not the focus of the lab. 
Your implementation will be compared against the C implementation and not the 
description in FIPS PUB 202. Do not attempt to add any additional features of SHA-3
that are not included in the C implementation.




that 




## SHA-3 Design

## Conclusion 

## Acknowledgements





















## Examples for Erik

***Q9: To start a new project using Chisel in Chipyard, you will need to do a few things. In your lab report, outline where you will put
your new project code and its directory structure (following convention), and which files you need to change in order to be able to compile 
code from your new project.***

<p align="center">
 <img src="figs/chipyard-flow.PNG" alt="train_perf_fig"/>
    <b>
    <em>Fig. 1 - Chipyard Flow</em>
    </b>
</p>

[Chipyard](https://github.com/ucb-bar/chipyard)

