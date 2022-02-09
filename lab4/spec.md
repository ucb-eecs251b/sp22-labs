# EECS 251B Lab 4 - Introduction to Custom Design Flow  

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

# TODO create an overview that covers both sections of lab (3 and 4)

## Overview
In Lab 3, you used the digital design flow to place-and-route a design using a 
pre-existing library of standard cells based, then pushed it through DRC and 
LVS. In this lab, we will walk you through the basics of custom IC design. 
First, we will place-and-route a 4-16 decoder using the Hammer flow in Chipyard.
We will then extract the design and simulate both the non-extracted and 
extracted netlist. With that, we can compare the timing and energy results 
between pre- and post-extraction.  

In real PDKs, the standard cell library consists of many more cells than a
single flip-flop.  Each of these cells have different functions and therefore
different timing parameters beyond setup time.  Furthermore, it is important to
know how much power each cell consumes, so that the synthesis and P\&R tools can
minimize overall power consumption. This is all compounded by the fact that
libraries must be characterized at multiple operating conditions (process,
voltage, and temperature), which at the very minimum need to support setup and
hold time calculations.  With ever-increasing complexity and end-applications,
successively smaller CMOS nodes have required an characterizing an exponentially
larger set of cells and operating conditions, as shown in Fig.
\ref{fig:characterization}.

In this lab, we will generate Liberty Timing Files (LIBs) and Library Exchange
Format files (LEFs), the two main abstraction collaterals for the VLSI flow.  We
will also generate the remaining pieces of collateral needed for the VLSI flow
and see how this is applicable to not only the standard cells, but also
non-digital blocks (such as analog IP) that need to be integrated into a design.

Then, we will take a closer look at the ASAP7 standard cells we have been using 
throughout the course and start a custom cell design in Cadence Virtuoso. You 
will inspect a simple D flip-flop, run DRC to verify the layout is 
manufacturable, and LVS to verify that your layout matches your schematic. 
Lastly, you will run parasitic extraction on your design and run some basic 
simulations to estimate some of its characteristics.

In Lab 3, you learned how to do extracted simulations on a placed-and-routed
block, then learned how to run a standard cell through the the first part of the
custom design flow.  At the end, you set up extracted simulations of the
flip-flop to calculate its setup time.  The next task is to bridge the gap from
designing the cells to abstracting then in a form that the VLSI flow tools can
efficiently consume and process.  For example, synthesis and P\&R tools must be
able to calculate path delay without running a Spice simulation itself,
necessitating timing models.  Similarly, P\&R only cares about the boundary,
pins, and blockages of cells instead of the entire transistor-level layout,
necessitating layout abstracts. 

## Getting Started: Decoder Extraction

First, pull the latest changes to the lab Chipyard repository:

```
cd chipyard
git pull
```

Notice that there is now a folder called `vlsi/lab4`. Inside, there is a 
`decoder.yml` file, which contains the Hammer inputs for the 4-16 decoder 
we will be working with (it is pretty similar to `gcd.yml` from the last lab, 
minus simulation keys).

We will need to push the decoder through the tools using the Hammer flow.
Refer to the previous lab for help on how to invoke Hammer. Synthesis,
PAR, and LVS all need to be run before you can proceed.

Before continuing, examine the `example-vlsi` file more closely. This file 
contains a HammerDriver that extends the base HammerDriver class. This is how 
the Hammer flow allows for flexibility in the design. If you want to edit the 
default steps that Hammer runs, you can do it like this. You may add Hammer 
[hooks](https://hammer-vlsi.readthedocs.io/en/latest/Hammer-Use/Hooks.html) here
to inject custom TCL that doesn't fit into a Hammer API (any real tapeout will 
certainly have some custom TCL). You may insert hooks before or after any 
default Hammer step (or another hook), you may replace a default Hammer step, or
you may remove a Hammer step. In the `get_extra_par_hooks` method, we have 
instructed Hammer to add a `visualize_floorplan` hook, which generates a 
pictorial floorplan in `par-rundir/all.svg` (this will become more useful with 
more complicated floorplans). We also remove the `place_bumps` and `clock_tree` 
as they are not useful for this lab.

***Q1: Why do we need to remove the `clock_tree` step for this decoder?***

## PEX: Parasitic Extraction

We will be using the Calibre tool PEX to run parasitic extraction. Parasitic 
extraction will produce a netlist with more accurately modeled capacitances and
resistances for the nets in the design. We can then simulate this netlist with 
Spice to get more accurate results than just the original post P&R netlist. 
PEX will first run LVS (again) before exporting the extracted netlist.

We will start by opening up the decoder in CalibreDRV:

```
cd build/custom-decoder/lvs-rundir
./generated-scripts/view_lvs
```

In CalibreDRV, click `Verification` > `Run PEX...` on the toolbar. A dialog will 
open asking for a runset which you can exit out of. Click on `Rules`, then 
navigate to the `vlsi/asap7_pex` folder in your repository and select 
`rcxControl_calibre_asap7.rul`. This is the PEX rule deck for ASAP7. Then go to 
`Inputs`. For the Layout, PEX should extract it from the layout of the decoder 
it already has. For the Netlist, select `decoder.include.sp` in your 
`lvs-rundir` (make sure "Export from schematic viewer" is unselected). Under 
`Outputs`, make sure the Netlist Format is set to `HSPICE`. Finally, click 
`Run PEX`.

***Q2: Submit the first 40 lines of the "decoder.pex.netlist" file generated by 
PEX.***

***Q3: Simulate the decoder and measure the delay from \texttt{A[3]} rising 
(the rest are 0) to \texttt{Z[8]} rising and the average power for both the 
extracted netlist and the original LVS netlist to compare them (some tips 
below). State the output load capacitance you used.***

We have supplied some HSPICE files to start with in `vlsi/asap7_pex` that have 
the testbenches mostly setup for you already. For the pre-extraction
simulation, first run `make hspice-netlist` in the `asap7_pex` directory. This 
will generate an HSPICE compatible netlist from the LVS netlist at 
`decoder.hspice.sp`. For this simulation, make sure that the ports match up 
with the instantiation of the decoder in `decoder.sp` (the port order is 
somewhat random in the PEX netlisting and SPICE is sensitive to port ordering).
Make sure that the ASAP7 CDLs are included in your testbench (there are some 
provided helpers that already have this for you). You may then run the 
simulation in the `asap7_pex` folder if you wish.

## Getting Started: Custom Design

In this portion of the lab, we will be exploring custom cell design. Figure 1 
gives an overview of the steps involved in the design flow for creating a 
custom cell and integrating it into a VLSI flow. We will import an existing 
design of a flip-flop in the ASAP7 PDK and introduce the steps you would take 
while building a custom standard cell. We will examine its schematic, export a 
netlist, run DRC and LVS, and simulate it.

<p align="center">
 <img src="figs/custom-cell-design-flow.png" alt="custom_cell"/>
    <b>
    <em>Fig. 1 - Custom Cell Design Flow</em>
    </b>
</p>

Start by running the following commands to setup and run Cadence Virtuoso 
(recommended to run in the `lab4` directory to minimize file clutter).
You should then see 2 windows, a command intepreter window (CIW) and a Library 
Manger window, as shown in Figure 2.

```
source ~eecs251b/sp22-workspace/asap7_virtuoso/sourceme.sh
tcsh      # start C shell
source ~eecs251b/sp22-workspace/asap7PDK_r1p7/cdslib/setup/setup_asap7.csh
exit      # exit C shell
virtuoso &
```

<p align="center">
 <img src="figs/virtuoso.png" alt="virtuoso"/>
    <b>
    <em>Fig. 2 - Virtuoso</em>
    </b>
</p>

In the Library Manager, you will notice that there are several libraries already
setup (in addition to the default Virtuoso libraries). `asap7_TechLib` is the 
ASAP7 library that contains the different transistor flavors available in ASAP7. 
`asap7ssc7p5t` is another library provided by ASAP7 that contains a few example 
standard cell views, including a flip flop, inverter, and tapcell. If you wanted 
to view the layouts for the other standard cells in ASAP7, you could make a new 
library by going to `File` > `New` > `Library`. Call it `asap7_std_cells`. Then 
select "Attach to an existing technology library" and choose `asap7_TechLib`.  

Then in the CIW, go to `File` > `Import` > `Stream` with Stream File as 
`~eecs251b/sp22-workspace/asap7/asap7sc7p5t_27/GDS/asap7sc7p5t_27_R_201211.gds` 
and select the `asap7_std_cells` library. Attach it to `asap7_TechLib` and click 
Ok. This will stream in the layouts for all the existing standard cells in ASAP7.

### Creating the Custom Library

First, create a new library by going to `File` > `New` > `Library` in the
Library Manager and name it `custom_cell`. Then select "Attach to an existing 
technology library" and choose `asap7_TechLib`. We will be exploring the 
flip-flop view that is already provided in ASAP7. In the library manager, go to
`asap7ssc7p5t` and right-click on `DFFHQNx1_ASAP7_75t_R` and click `Copy`. In 
the new window, change the To library to be `custom_cell` and the Cell to be 
`custom_dff_R`, and click Ok. This will give us a pre-made schematic, symbol, 
and layout for our "custom" cell.

### Schematic Entry

For a truly custom cell, you would create a new schematic view in your new
library and build your design using the components in the ASAP7 PDK. Once you
built it, you can simulate it to verify its operation, including Monte Carlo
simulations to test its operation with process variations.

In this case, you can open up the existing schematic that we copied over (it
should look like Figure 3. Answer the following questions about the schematic:

***Q4: What is the purpose of the tristate buffer that connects
from the "MS" node to the "MH" node (output of the input latch)?  Does this
affect the setup time? Why do these tristate transistors have fewer fins than
the input transistors?***

Notice the top level pins of the schematic.  These are the same pins that will
be the inputs and outputs of our standard cell layout.

<p align="center">
 <img src="figs/dff-schematic.PNG" alt="sch"/>
    <b>
    <em>Fig. 3 - Schematic of the DFF</em>
    </b>
</p>

If you want to instantiate this cell in a testbench or other schematic, you need
to create a symbol view.  We already have one in this case, but to create one
you can go to `Create` > `Cellview` > `From Cellview`, and click Ok in the
dialog box. You can leave the symbol as is, or use the drawing tools to draw
something custom. When you are done, save and close the window.

### Schematic Netlist Export

We will be running LVS/PEX against this schematic later, so we will need to
export this schematic as a netlist. Most physical verification tools perform
verification on generic filetypes, such as GDS files for layout and SPICE
netlists for schematics. To export the DFF schematic, go to `File` > `Export` >
`CDL` in the CIW, then populate the following fields:

- Library Name: Your new Custom Cell library
- Top Cell Name: `custom_dff_R` (or your top cell name)
- View Name: \texttt{schematic}
- Switch View List: `auCdl schematic`
- Stop View List: `auCdl`
- Output CDL Netlist File: `custom_dff_R.cdl`
- Make sure "Map Bus Name from <> to []" is checked.

Hit Apply and wait for the CDL to be exported, then sanity check it.

### Layout Entry}

As discussed above, we will be using the existing DFF layout, so no
modifications are required for this lab.  The following instructions are a
general tutorial for basic layout editing.

<p align="center">
 <img src="figs/dff-layout.PNG" alt="layout"/>
    <b>
    <em>Fig. 4 - DFF Layout</em>
    </b>
</p>

Figure 4 is a screenshot of what your copied DFF layout should look like.  A
great resource for layout in ASAP7 can be found
[here](http://venividiwiki.ee.virginia.edu/mediawiki/index.php/FinFET_ASAP7_CellLayout}).
This describes most of the important layers and provides a layout tutorial for a
gate in ASAP7.  If you are unfamiliar with transistor level layout, it would be
instructive for you to go through the DFF layout and correlate parts of it to
the schematic.  If you're unsure of where to start, you can use the top level
input pins (M1 pins) and the number of fins for different transistors as
references. For simple things, you only need a few commands to change the
layout:

- By pressing `sim`, 1, 2, 3 etc you can show only certain layers.  Press
  Shift-1 to add M1 to whatever is visible. 
- To move something, hover over the object, then press "m" on the keyboard.
- To make something bigger or smaller, hover over the edge of the object, then
  press "s" on the keyboard, then click again at the final location.
- Press "Esc" if you selected something incorrectly.
- To add new wire, press Ctrl-Shift-W (then F3 will change the options such as
  the width).
- To just add a rectangle, select the desired layer in the layer pallete, and
  press "r".  Then click on the two corners to create. 
- Selecting an object and hitting "q" will bring up property info about it and
  allow you to edit certain characteristics.
- "k" allows you to place rulers to measure distances in the layout.

## Cell Verification

When you are done with your schematic and layout, you can run Calibre DRC and
LVS directly from Virtuoso because there is a Calibre plugin accessible from the
toolbar included in our setup.  This lets us use the same Calibre tools we ran
in the previous lab to verify our custom cell.

### DRC

To run DRC, go to `Calibre` > `Run nmDRC` in the layout editor.  Select the rule
file to be
`~eecs251b/sp22-workspace/asap7PDK_r1p7/calibre/ruledirs/drc/drcRules\_calibre\_asap7.rul`.
Make sure that under Inputs, "Export from layout viewer" is selected and then
Run DRC.

Like in the previous lab, you will see an RVE window with the DRC results. Your
design should be DRC clean except for a LUP error.

***Q5:Read this DRC error and explain what the error is. What is the purpose and
operation of a tapcell?  You can look at an ASAP7 tapcell layout in the the same
`asap7ssc7p5t` library we copied the DFF schmatic and layout from.***

### LVS

You don't have to run LVS for this lab, but if you are building a custom cell
from scratch, you can do it similarly to DRC by going to `Calibre` > `Run
nmLVS`. Change the runset file to
`~eecs251b/sp22-labs/asap7PDK_r1p7/calibre/ruledirs/lvs/lvsRules_calibre_asap7.rul`.
Make sure to select the CDL we exported earlier as the netlist.

If you are running into LVS errors, Calibre will describe the discrepancies and
allow you to highlight the offending nets just like when we ran LVS for a larger
design.

### Extraction

To run parasitic extraction, go to `Calibre` > `Run PEX`.  The settings for this
extraction are essentially the same as earlier in the lab, when you extracted
the decoder. Make sure to select the CDL we exported earlier as the netlist. 

To avoid an LVS error due to substrate pins and power pins being physically
separate at the standard cell level (this is a hint for the tapcell question),
you must select an additional option. In the PEX window, select `Setup`
> `PEX Options`. Click the new `PEX Options` button and go to the
`Connect` tab. Then, select `Connect nets named` and enter
`VDD VSS`.

Most of the above PEX setup steps, minus filling in the exported netlist, are
automated with a runset file (in `asap7_pex/pexRunset_asap7`). Next time, when
you run PEX, you can select this runset file instead of clicking all the options
manually (but it is important to know what is happening). Note that when you
have finished extracting and close the PEX window, \textbf{do not save the
changes to the runset file}, otherwise the next time you run PEX, you'll extract
a previous design instead!

## Extracted Simulation

Finally, we will do a couple more simulations on the exported CDL and the
extracted netlist. You should be able to use a very similar testbench to the one
used earlier for the decoder. We have included a spice testbench in
`~eecs251b/sp22-workspace/asap7_virtuoso` that may be useful to you. Remember
that the output of this flip flop is QN, not Q. Make sure to check the ordering
of the ports of your DFF between the different netlists.


***Q6: Estimate the CLK-Q time of your DFF from for both the exported CDL and
the extracted netlist from PEX. Include a screenshot (or the text) of your
spice/otherwise testbench. State the load capacitance and rise/fall times you
used.***

***Q7: Estimate the setup time of your DFF for both the exported CDL and the
extracted netlist from PEX. State the load capacitance and rise/fall times you
used. Also, include a screenshot of the waveforms of CLK, D, and QN for the
simulation that you determined the setup time from (if you're using HSPICE, you
can use WaveView or wv on the command line to load the `.tr0`
waveforms).***

****Q8: Compare your setup time result to the timing parameters for
`DFFHQNx1_ASAP7_75t_R` in
`~eecs251b/sp22-workspace/asap7/asap7sc7p5t_27/LIB/NLDM/asap7sc7p5t_AO_RVT_TT_nldm_201020.lib.gz`.
Copy the entire section of the lib (starting at "timing () {}") that
corresponds to the setup time and try to guess why it is presented as a look-up
table instead of a single value without consulting a LIB reference. Tip: you
can find the table units and templates at the top of the file, and then find the
timing lookup tables for clk-q, setup, etc. by searching for the
DFFHQNx1_ASAP7_75t_R entry.***
















\title{\vspace{-0.4in}\Large \bf Characterization and Abstraction in the Custom Design Flow \vspace{-0.1in}}

\author{EE241B Lab 4\\ Written by Harrison Liew (2021)\vspace*{-0.6in}}

\date{}
\maketitle

\newcommand{\headertext}{EE241B Lab 4, The Custom Design Flow: Characterization and Abstraction, Spring 2021}
\markboth{\headertext}{\headertext}
\thispagestyle{empty}



\section*{Overview}
In Lab 3, you learned how to do extracted simulations on a placed-and-routed block, then learned how to run a standard cell through the the first part of the custom design flow.  At the end, you set up extracted simulations of the flip-flop to calculate its setup time.  The next task is to bridge the gap from designing the cells to abstracting then in a form that the VLSI flow tools can efficiently consume and process.  For example, synthesis and P\&R tools must be able to calculate path delay without running a Spice simulation itself, necessitating timing models.  Similarly, P\&R only cares about the boundary, pins, and blockages of cells instead of the entire transistor-level layout, necessitating layout abstracts. 

In real PDKs, the standard cell library consists of many more cells than a single flip-flop.  Each of these cells have different functions and therefore different timing parameters beyond setup time.  Furthermore, it is important to know how much power each cell consumes, so that the synthesis and P\&R tools can minimize overall power consumption. This is all compounded by the fact that libraries must be characterized at multiple operating conditions (process, voltage, and temperature), which at the very minimum need to support setup and hold time calculations.  With ever-increasing complexity and end-applications, successively smaller CMOS nodes have required an characterizing an exponentially larger set of cells and operating conditions, as shown in Fig. \ref{fig:characterization}.

In this lab, we will generate Liberty Timing Files (LIBs) and Library Exchange Format files (LEFs), the two main abstraction collaterals for the VLSI flow.  We will also generate the remaining pieces of collateral needed for the VLSI flow and see how this is applicable to not only the standard cells, but also non-digital blocks (such as analog IP) that need to be integrated into a design.

\begin{figure}[h!]
\begin{center}
  \includegraphics[width=3in]{figs/growingLibraries.png}
  \caption{Exponential Growth in Library Characterization}
  \label{fig:characterization}
  \vspace{-16pt}
\end{center}
\end{figure}

\section*{Getting Started}

We will once again start with updating our environment.  Pull the latest changes to the lab Chipyard repository.

Then, start your Virtuoso environment as in Lab 3 and make a note of where your exported netlist and extracted netlists are for the flip-flop that you analyzed.

\section*{Cadence Liberate: Characterization}

Cadence Liberate is a tool that can analyze the function of any circuit and output "electrical views", which contain all the relevant timing relationships between pins, the power consumption information, and more. It accomplishes this by intelligently finding timing arcs, setting up Spice simulations, and measuring parameters like delay and currents automatically, thereby vastly improving the time it takes to prepare a cell for the digital VLSI flow.

A summary of all the things that various Liberate tools can characterize is shown in Fig. \ref{fig:liberate_suite}, which is taken from the reference manual at \texttt{/share/instsww/cadence/LIBERATE/doc/liberate/liberate.pdf}. In this lab, we will only use Liberate Characterization to generate LIBs for our flip-flop.

\begin{figure}[h!]
\begin{center}
  \includegraphics[width=6in]{figs/completeSuite.png}
  \caption{The Complete Cadence Liberate Suite}
  \label{fig:liberate_suite}
\end{center}
\end{figure}

\subsection*{LIB Files}

Liberty Timing Files (LIBs) are an IEEE standard. A very high-level introduction of the key parameters and format of the standard is described \href{http://web.engr.uky.edu/~elias/lectures/LibertyFileIntroduction.pdf}{here}, which corresponds to a basic timing model called the Non-Linear Delay Model (NLDM). It is a simple lookup table-based model which describes delays and rise/fall times for various input waveforms and output loads that results in small LIB files at a moderate accuracy loss compared to a Spice simulation.

With deeply-scaled technologies, however, switching events actually generate significant transient current and voltage spikes, which can cause glitching on critical and neighboring nets. As a result, additional LIB models are needed to accurately model these effects, namely Composite Current Source (CCS) and Effective Current Source Model (ECSM), which non-linearly sample the output current and voltage waveforms, respectively, and are shown in Figs \ref{fig:ccs} and \ref{fig:ecsm}. CCS models are generally used for Synopsys VLSI tools (Design Compiler, IC Compiler), while ECSM models are generally used for the Cadence tools that we have used so far.

Finally, there are additional things to characterize: power, noise immunity, signal integrity, process variation, and electromigration. Liberate will by default analyze power and its importance is self-explanatory, but the latter few are used to ensure product reliability. You will learn about these in lecture but they are outside the scope of this lab. 

\begin{figure}[h!]
\begin{center}
  \includegraphics[width=5in]{figs/ccs.png}
  \caption{Composite Current Source Model}
  \label{fig:ccs}
\end{center}
\end{figure}

\begin{figure}[h!]
\begin{center}
  \includegraphics[width=5in]{figs/ecsm.png}
  \caption{Effective Current Source Model}
  \label{fig:ecsm}
\end{center}
\end{figure}

\subsection*{Generating LIBs}

Notice that there is now a folder called \texttt{vlsi/asap7\_lib}. Inside, there are a few TCL files and a Makefile. Let's go through them one-by-one to see how Liberate works in order to characterize the flip-flop we looked at in Lab 3. We will also then compare it against the LIB that comes from the PDK, which is located at \texttt{\~{}ee241/spring21-labs/asap7libs\_24/lib/asap7sc7p5t\_24\_SEQ\_RVT\_TT.lib}.

The \texttt{Makefile} is very simple. You can see that \texttt{CELL\_TYPE} is set to \texttt{DFF}, which passes \texttt{char\_DFF.tcl} to the \texttt{liberate} command. This TCL file is set up for characterizing D flip-flops.

\texttt{char\_DFF.tcl} is divided into a few short sections:
\begin{enumerate}
  \item It sources a template file \texttt{template\_asap7.tcl} that we will look at next.
  \item You then choose between 3 PVT corners: \texttt{TT\_0p7V\_25C}, \texttt{SS\_0p63V\_100C}, and \texttt{FF\_0p77V\_0C}. These PVT corners correspond to the corners that were characterized by the developers of the ASAP7 PDK. The relevant ASAP7 Spice transistor model file is selected based on the process corner.
  \item You then specify the names of the D flip-flops that you want to characterize for this LIB, read the PEX netlists for those cells, and define I/O and template parameters of this cell family, which denotes that \texttt{CLK} is the clock port, \texttt{D} is an input port, \texttt{QN} is the output port, and some delay/constraint/power templates (see below).
  \item Finally, the cells are characterized at the chosen PVT corner and written to a \texttt{.lib} file.
\end{enumerate}

\texttt{template\_asap7.tcl} sets some base units and a bunch of variables and templates that are actually generated from the source LIB for the flip-flip we are analyzing (\texttt{DFFHQNx1\_ASAP7\_75t\_R}). Here's what the variables mean (summarizing from the reference manual):
\begin{tightlist}
  \item \texttt{(measure\_)slew\_lower/upper\_rise/fall}: Together, these 8 variables pass through to the output LIB and tell Liberate that output transition time values are measured between 10\% to 90\% of the supply voltage.
  \item \texttt{delay\_inp/out\_rise/fall}: Together, these tell Liberate to measure the delay from an input to output for both edges at the 50\% of supply voltage point.
  \item \texttt{def\_arc\_msg\_level} and \texttt{process\_match\_pins\_to\_ports}: These tell Liberate to throw an error if no valid arcs are found and if the pin list is incorrect. This can help detect an mismatched \texttt{define\_cell} command setup relative to the cell being characterized.
  \item \texttt{max/min\_transition}: These get encoded in the LIB as a min/max bound for transition times on any pin. These values are passed onto synthesis and P\&R to prevent them from sizing driving cells and loads too large or small.
  \item \texttt{min\_output\_cap}: This sets the minimum output load during the characterization simulations.
  \item \texttt{define\_template -type delay}: This is a table for delay characterization for various values of input slew (\texttt{index\_1}) and output load (\texttt{index\_2}). It is named \texttt{delay\_template\_7x7} which the delay tables will refer to in the generated LIB.
  \item \texttt{define\_template -type constraint}: This is a table for timing constraint (setup, hold, etc.) characterization. Here, \texttt{index\_1} is a range of input slews on the data and \texttt{index\_2} is a range of input slews of the reference signal (clock, reset, etc.).
  \item \texttt{define\_template -type power}: This is a table for switching and internal power consumption. Here, \texttt{index\_1} is a range of input slews and \texttt{index\_2} is a range of output loads.
\end{tightlist}

\begin{qcnt}
\textbf{What is the benefit of specifying min/max transition times? Hint: think about signal integrity and crowbar currents.}
\end{qcnt}

\begin{qcnt}
\textbf{The template tables are optimized for a \texttt{DFFHQNx1\_ASAP7\_75t\_R} cell, which is the smallest available D flip-flop. For larger D flip-flops (i.e. larger internal transistors and output drive strengths), are the provided template tables suitable? If not, which tables might we want to change and how? To verify your hypothesis, check against the PDK's LIB.}
\end{qcnt}

\vspace{10pt}
Now, let's generate the LIB. Copy or symlink the PEX netlists for your flip-flop from Lab 3 into this \texttt{asap7\_lib} folder. There are 3 files you need: the ones that end in \texttt{.pex.netlist}, \texttt{.pex.netlist.pex}, and \texttt{.pex.netlist.<cellname>.pxi}. Then, run:

\vspace{-10pt}
\begin{minted}{sh}
  make gen-libs
\end{minted}
\vspace{-10pt}

After a couple minutes, it will output a \texttt{DFF\_<PVT corner>.lib} file, as well as a log a file. Open up the generated LIB to compare against the PDK's LIB, then answer these questions:

\begin{qcnt}
\textbf{Glance through the two LIBs and note any high-level differences between the characterization results of your flip-flop and the PDK's \texttt{DFFHQNx1\_ASAP7\_75t\_R}.}
\end{qcnt}

\begin{qcnt}
\textbf{Examine the setup and hold time tables. Given the constraint template, explain why there are both positive and negative values.}
\end{qcnt}

\begin{qcnt}
\textbf{Repeat the characterization for the PVT corners that would be used for setup and hold analysis (which corners are these, respectively?). Show some general comparisons of setup/hold timing and active/leakage power between these LIBs and the typical PVT corner LIB we generated first.}
\end{qcnt}

\subsection*{Extra Credit: Decoder LIB}

Characterize the decoder that we extracted in Lab 3 at the setup PVT corner. You will need to make your own \texttt{char\_decoder.tcl} script. Note that the \texttt{clk} pin in the decoder is not a clock, just a regular input. You can also use bus notation such as \texttt{A[3:0]} for the \texttt{A} and \texttt{Z} pins. The characterization will take a few hours with significant compute utilization, so be mindful of other users. Attach your new characterization script to your lab report, then answer these questions:

\begin{qcnt}
\textbf{Compare the leakage power characterized by Liberate against the post-synthesis \texttt{final\_gates.rpt} -- how well do they match up?}
\end{qcnt}

\begin{qcnt}
\textbf{Compare the LIB's combinational delay table against the post-P\&R timing report. Can you estimate the output load capacitance that is modeled in Innovus for the longest delay path?}
\end{qcnt}

\begin{qcnt}
\textbf{Why are synthesis and P\&R tools are able to time this block much faster than Liberate, beyond the fact that standard cell LIBs are available? Is Liberate the right tool for generating LIBs for all kinds of designs? If not, why?}
\end{qcnt}

\section*{Cadence Abstract: Layout Abstracts}

Cadence Abstract is a tool attached to Virtuoso that generates a layout abstraction for P\&R tools. An abstract is a high-level representation of a layout view, containing information about the type and size of cells, positions of pins, and the size and type of blockages. These abstracts are used in place of full physical layouts so that P\&R tools can have better performance. In the P\&R process, only the bare minimum information is needed for it to know where it is allowed to place cells and how to route to them; extracting this information from the full layouts would consume excess resources. Finally, if you want to integrate some IP into your chip, whether it is SRAM, data converters, etc., abstracts are a way to keep everything inside the IP as a blackbox from the perspective of the physical design flow. 

A summary of the Abstract tool and instructions are found in the documentation at\\ \texttt{/share/instsww/cadence/IC617/doc/abstract/abstract.pdf}. If you are going to generate more complex abstracts for your project, this is very important to reference. In this lab, we will use Abstract to generate LEFs for our flip-flop.

\subsection*{LEF Files}

Library Exchange Format (LEF) is an open industry standard that was developed by a company that has since been acquired by Cadence. It is an ASCII text format and is often used in conjunction with the very similar Design Exchange Format (DEF) files in order to fully describe abstract layouts. An introduction to LEF can be found \href{https://www.csee.umbc.edu/courses/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf}{here} and the full LEF/DEF Language Reference Manual can be found at \texttt{/share/instsww/cadence/IC617/doc/lefdefref/lefdefref.pdf}.

In general, VLSI tools need to take a set of abstracts that in aggregate describe the basic technology rules and information about all macros to be used. In the LEF specification, technology rules are specified in a technology LEF that will contain definitions of layers, manufacturing grids, vias, and sites (i.e. the standard cell unit height/width). Macro information is specified in separate LEFs that contain definitions of class (type of cell), size, symmetry, pins, and obstructions (i.e. areas not available for routing). We will now explore how to generate the latter type of LEF.

\subsection*{Generating LEFs}

In the directory which you have set up your Virtuoso environment from Lab 3, source all the necessary environment scripts again, and then run abstract:

\vspace{-10pt}
\begin{minted}{sh}
  abstract &
\end{minted}
\vspace{-10pt}

Be advised that Abstract is quite an old tool, and may display like this in your X2Go session, where a lot of the menu text is invisible:

\begin{figure}[H]
\begin{center}
  \includegraphics[width=4in]{figs/broken_abstract.png}
  \caption{Incorrectly Displayed Abstract}
  \label{fig:bad_abstract}
\end{center}
\end{figure}
\vspace{-24pt}

There are 2 possible solutions. The first is to shutdown your existing X2Go session and re-install your X2Go with all legacy fonts, as shown here for a Windows installer:

\begin{figure}[H]
\begin{center}
  \includegraphics[width=3in]{figs/x2go_fonts.png}
  \caption{Selecting Legacy Fonts in X2Go Installer}
  \label{fig:x2go_fonts}
\end{center}
\end{figure}
\vspace{-24pt}

The other solution is to rely solely on X11 forwarding over an SSH terminal, which may be slow and cause you to lose your work if the connection drops.

Take a look at the toolbar buttons -- these correspond to the steps needed to generate an abstract. We are going to go through them from left to right:

\begin{enumerate}
  \item Click the Library button (looks like a book). Select the Virtuoso library that contains your custom DFF from Lab 3. You should see that the \texttt{custom\_dff\_R} cell appears on the right pane -- click on it. You should also see 1 cell in the \texttt{Core} "bin" on the left pane. The bins correspond to the types of cells that the LEF specification defines; for a standard cell that we have it should be in the \texttt{Core} bin, but for things like larger IP blocks, you would want them in the \texttt{Block} bin. To do this, you would select the cell, go to \texttt{Cell $>$ Move}, and select the destination bin.
  \item Click the Layout button (looks like 2 wires and 2 vias). This is used to import a layout from a GDS file, such as if somebody else gave you a finished layout in GDS format and not as a Virtuoso library. Since our flip-flop already has a layout view in Virtuoso, we can hit Cancel out of this dialog box.
  \item Click the Logical button (looks like an inverter symbol). This tells the tool what the pins are. Since we just generated a LIB, select LIB as the Filetype and browse to the LIB you created just now. For other cells, we can also import Verilog shell instead, which would just define the module and ports but does not need implementation.
  \item Click on the Pins button (a square with an X inside). This is where we select options for how to extract pin shapes from the pin labels in the layout view and pin directions from the logical view. There are 4 tabs: Map, Text, Boundary, and Blocks.
  \begin{tightlist}
     \item In the Map tab, under the field "Map text labels to pins", type \texttt{(M1 M1) (M2 M2) (M3 M3) (M4 M4) (M5 M5) (M6 M6) (M7 M7) (M8 M8) (M9 M9) (Pad Pad)}. This means that all pin labels on each layer \texttt{M1} are mapped to shapes on the same layer. Leave the other fields as-is.
     \item In the Text tab, do not change anything. However, this can be used if you want to use regex to change the pin names between the layout and abstract view (e.g. bus bit designators).
     \item In the Boundary tab, do not change anything. Here, the listed layers will designate the rectangular P\&R boundary of the cell we want to extract. With the "as needed" option, it will create a boundary only if a \texttt{prBoundary} shape is not defined in the layout view. Other designs may want to restrict the layers used.
     \item In the Blocks tab, do not change anything. This is only used to process pre-routed layouts that are imported from a DEF file.
  \end{tightlist}
  Now, click Run. The abstract log should tell you that 5 pins have been created.
  \item Click on the Extract button (like the Layout button without the wire underneath). This is used to trace the connectivity between shapes and terminals, and also generate antenna data. There are 4 tabs: Signal, Power, Antenna, and General. You do not need to change anything in any tab. In the first 3 tabs, for each layer, there exists a "Geometry Specification" field. For advanced usage, you can generate Boolean functions of multiple layers to define connectivity layers. Here are some other pointers for advanced usage later on:
  \begin{tightlist}
    \item Must connect pins are for situations where the same pin name exists in disjoint locations (i.e. not internally connected), and must be connected together in P\&R, which recognizes the \texttt{MUSTJOIN} specification.
    \item Antenna extraction is necessary for larger blocks with pins that have long wires. This information is used for P\&R to decide whether it needs to do things like switch layers to avoid antenna violations. This is beyond the scope of this lab.
    \item The layer connectivity list in the General tab is very important to get right. By default, this is filled in correctly and lists the via layers that join adjacent metal layers.
  \end{tightlist}
  Now, click Run. The abstract log should tell you that all 5 nets have been extracted.
  \item Click on the Abstract button (square nestled in an L-shape route). This step adjusts pins and creates blockages, overlaps, and grids. There are 7 tabs: Adjust, Blockage, Density, Fracture, Site, Overlap, and Grids.
  \begin{tightlist}
    \item In the Adjust tab, do not change anything. However, for larger designs (such as analog blackboxes), you will often want to select the "Create boundary pins" option to confine the generated pin shape to a square abutting the P\&R boundary rather than over the entire block (prerequisite is that all pins are routed to the edge of the boundary).
    \item In the Blockage tab, we encode how shapes on various layers turn into blockages. For this flip-flop, do not change anything; however, for larger designs (such as analog blackboxes), this is a \emph{very important} tab. There are 3 types of blockages that can be generated for each layer: detailed, shrink, and cover. \textbf{Detail} generates a blockage shape for every piece of geometry on that layer -- this is useful only for standard cells and the topmost layers of large blocks. \textbf{Shrink} fills in the smaller, less useful spaces between shapes, which would allow for over-the-cell routing without modeling each obstruction in detail. \textbf{Cover} generates an obstruction covering the entire block for that layer, which is used for layers which should be off-limits for the P\&R tool. In general, you want to cover as much of your block as necessary, and only expose in shrink or detail the layers that are available for P\&R to do routing on. This both increases the efficiency of P\&R and reduces the chances that routing will cause DRC/LVS issues with the abstracted block. In the extreme case, analog IP with structures like inductors at the top-level need to cover block all layers, exposing only boundary pins. Additional advanced options in this tab will generate cutouts around pins and create routing channels on cover block layers.
    \item In the Density tab, do not change anything. This is used to encode density information, which is particularly helpful to help automatic fill tools achieve required min/max density without needing to calculate it from the full layout of any blackboxes.
    \item In the Fracture tab, do not change anything. This is used to fracture pins and blockages from polygons to rectangles if required for certain P\&R tools.
    \item In the Site tab, for the site name, select \texttt{coreSite}. This is required for cells in the \texttt{CORE} bin like our flip-flop in order to tell the tool that it belongs to the \texttt{coreSite} site, like for all standard cells that need to be placed in ordered rows. For other types of cells, do not do this.
    \item In the Overlap tab, do not change anything. This is particularly useful for larger designs where you want to generate a second, more detailed boundary that is a rectilinear polygon. This overlap layer can be generated from specific layers and will be used to test if cells truly overlap when in P\&R they are placed with overlapping boundaries.
    \item In the Grids tab, do not change anything. This is a tool to help check if all geometry (wires and pins) are properly on the manufacturing grids, which is useful for standard cells but less so for larger designs. Our flip-flop is already fine, so we can skip this tool.
  \end{tightlist}
  Now, click Run. The abstract log should tell you that blockages are created for each layer and do a power rail analysis.
  \item Finally, click the Verify button (it has a check mark). This step helps you check your abstract against rules and various P\&R tools. Cancel out of this step, as we will just move onto generating the LEF.
\end{enumerate}

\begin{qcnt}
\textbf{Why do we generally only want to generate pin shapes on metal layers, and not layers such as n-well and p-substrate?}
\end{qcnt}

\begin{qcnt}
\textbf{Say we want to abstract an SRAM block with signal and power pins on M4 and all internal routing on M4 and below. For which layers should we generate cover, shrink, and detailed blockages? Should we create boundary pins for signal and/or power nets?}
\end{qcnt}

\begin{qcnt}
\textbf{Why are there minimum and maximum density requirements for every layer in a technology stackup?}
\end{qcnt}

\begin{qcnt}
\textbf{Provide a simple drawing of a case where creating an Overlap layer would be beneficial.}
\end{qcnt}

Finally, we are ready to export the LEF. Go to \texttt{File $>$ Export $>$ LEF}. In the window that appears, set the LEF filename to \texttt{custom\_dff\_R.lef} (or something else of your choosing). Select 5.8 as the LEF version, then hit OK.

Now, open the LEF you just created alongside the LEF provided in the PDK at\\ \texttt{\~{}ee241/spring21-labs/asap7libs\_24/lef/scaled/asap7sc7p5t\_24\_R\_4x\_170912.lef}. In the latter file, scroll down to the part that starts with \texttt{MACRO DFFHQNx1\_ASAP7\_75t\_R}. As you compare the two files, note that the LEFs provided in the PDK have been manually scaled up by 4x in order to get around needing extra sub-20nm Innovus licenses.

\begin{qcnt}
\textbf{Minus the scaling factor, would the the abstract of the flip-flop we just generated fit properly in the \texttt{coreSite} site, which is 1.08um tall (4x scaled dimension)? If not, why (hint: look at the layout), and what would we need to change when generating the abstract?}
\end{qcnt}

\begin{qcnt}
\textbf{Notice how for the \texttt{CLK} pin it says \texttt{USE CLOCK} for our LEF while in the PDK's LEF it says \texttt{USE SIGNAL}. How did our abstract run know it's a clock pin, and what would we do differently to make it just a signal?}
\end{qcnt}

\begin{qcnt}
\textbf{The \texttt{OBS} section contains our detailed obstructions. What would it look like if instead we did COVER for layer M2? Why would this pose problems for us in P\&R? What about COVER for layer V1?}
\end{qcnt}

All the the steps we took in the Abstract GUI have actually been recorded for us. In the directory of the library containing the flip-flop, there is a \texttt{.abstract.options} file. This file contains all of the options corresponding to the current state of our Abstract GUI after all of our changes. In the directory in which you launched Abstract, there is a \texttt{abstract.record} file. This file contains the commands corresponding to the steps we took. In the future, you can take and modify these two files in order to run Abstract from the command line (see the section "Log, Replay, and Record File Behavior" in the Abstract user guide).

Finally, in the Virtuoso library, for the \texttt{custom\_dff\_R} cell, there is now an \texttt{abstract} view. We can open this in the Virtuoso layout editor!

\begin{qcnt}
\textbf{Submit a screenshot of the abstract view of the flip-flop, as viewed in Virtuoso.}
\end{qcnt}

\subsection*{Extra Credit: Decoder LEF}

Your challenge is to use both Innovus and Abstract to generate a LEF of the decoder from Lab 3. To generate a LEF from Innovus, open the P\&R'd decoder using the \texttt{open\_chip} script. Then, in the Innovus, use the \texttt{write\_lef\_abstract} command with the following options:\\\texttt{-stripe\_pins -pg\_pin\_layers 9}. To figure out how to use this command, type \texttt{man write\_lef\_abstract} first. Study the generated LEF to understand what it specifies.

Note for Abstract: from our P\&R run, we have already generated a GDS and a Verilog file used for LVS. These two files are sufficient to generate a LEF with Virtuoso and Abstract.

\begin{qcnt}
\textbf{In your lab report, detail the steps/options you set in order to get Abstract to generate a LEF that is \emph{as close as possible} to the LEF generated from Innovus. Summarize any remaining discrepancies between the generated LEFs.}
\end{qcnt}

Note that P\&R flows for complex chips require a hierarchical flow, which would require generating all the views of P\&R'd sub-blocks before integrating in a higher level of P\&R. In this case, Innovus will generate all that for you when the hierarchical flow is setup properly.

\section*{Remaining Views}

So far, we have learned how to generate the Spice netlist (CDL), LIB, and LEF of custom designs. The remaining important views are the Verilog netlist (for synthesis and simulation) and the full layout GDS, for merging into the final layout.

\subsection*{Verilog}

There are two cases where Verilog is used. The first is for synthesis, which are just wrappers and are needed for blackboxes (e.g. SRAMs, analog IP). This can be generated using Liberate with the \texttt{write\_top\_netlist} command (see the manual or type \texttt{help write\_top\_netlist} for details). More often, it is done manually in your source netlist, since only the port list needs to be defined.

The second case is for simulation. This Verilog file would describe the behavior of the design, and is often hand-written along with your design during functional verification.

\subsection*{GDS}

This can be generated from Virtuoso. In the CIW, go to \texttt{File $>$ Export $>$ Stream}. Then, select the \texttt{layout} view of the cell to export, change the Stream File name if desired, and select \texttt{asap7\_TechLib} as the Technology Library. Hit \texttt{Translate} and it will generate a GDS file.

\subsection*{Adding an Extra Library in Hammer}

To ensure that the Hammer flow knows about our new custom cell, we would need to append to the list of \texttt{vlsi.technology.extra\_libraries}. An example is found in the \texttt{vlsi/example-asap7.yml} file in your Chipyard repository, starting on line 85 and reproduced here for a dummy DCO. To round it out for the whole flow, you would also include \texttt{spice netlist} for LVS, \texttt{verilog synth} for synthesis (if not already included in your source Verilog), and \texttt{verilog sim} for simulation.

\begin{minted}{yaml}
vlsi.technology.extra_libraries_meta: ["append", "deepsubst"]
vlsi.technology.extra_libraries:
  - library:
      nldm liberty file_deepsubst_meta: "local"
      nldm liberty file: "extra_libraries/example/ExampleDCO_PVT_0P63V_100C.lib"
      lef file_deepsubst_meta: "local"
      lef file: "extra_libraries/example/ExampleDCO.lef"
      gds file_deepsubst_meta: "local"
      gds file: "extra_libraries/example/ExampleDCO.gds"
      corner:
        nmos: "slow"
        pmos: "slow"
        temperature: "100 C"
      supplies:
        VDD: "0.63 V"
        GND: "0 V"
  - library:
      nldm liberty file_deepsubst_meta: "local"
      nldm liberty file: "extra_libraries/example/ExampleDCO_PVT_0P77V_0C.lib"
      lef file_deepsubst_meta: "local"
      lef file: "extra_libraries/example/ExampleDCO.lef"
      gds file_deepsubst_meta: "local"
      gds file: "extra_libraries/example/ExampleDCO.gds"
      corner:
        nmos: "fast"
        pmos: "fast"
        temperature: "0 C"
      supplies:
        VDD: "0.77 V"
        GND: "0 V"
\end{minted}

\section*{Conclusion}

This lab was meant to round out the custom design flow. You now know how to generate all of the required collateral for the VLSI tools for both a standard cell and a bigger custom cell.  You learned how to use Cadence Abstract Generator (to create the LEF) and Cadence Liberate (to create the LIB) in detail, which may come in handy for your projects if you need to rapidly characterize and abstract custom cells.

As described, there are still multiple characterizations that we have not explored in depth, such as power, leakage, electromigration, antenna, and more. We have also not taken into account process variation in the characterization results. These are left for you to explore for your projects or future research.
