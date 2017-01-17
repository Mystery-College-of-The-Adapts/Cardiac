﻿# Cardiac
C# / .NET implementation of the CARDIAC Cardboard computer

## Overview
This project is a compiler, runtime and debugger for the [CARDIAC - A **Cardboard Illustrative Aid to Computation**](https://en.wikipedia.org/wiki/CARDboard_Illustrative_Aid_to_Computation). 
The goal of the project is to implement a simple, but realistic compiler and interpreter.  The compiler implements a simple language I call CARDIASM which
is a simple assembly language derived from the CARDIAC instruction set.  Sadly the limitations of the CARDIAC hardware platform preclude porting a more common
language like C.  

## Compiling the code
The code is a Visual Studio 201S solution primarily providing three applications


## Applications

### cardiasm.exe
This compiler takes CARDIASM files as input and produces compiled images (".cardimg").  These images can be run via cardiac.exe or
debugged via cardb.exe.  The compiler will optionally produce a simple program database (".cardb") if you pass it a "/debug+" flag on the
command line.  This application uses the ANTLR library to parse cardiasm source files and produces executable images as described 
[here](https://www.cs.drexel.edu/~bls96/museum/cardiac.html).
  
### cardiac.exe  
This is the cardiac interpreter, analogous to "java.exe"  Most of the "real" code is in Cardiac.Core.dll; 
cardiac.exe mostly just accepts command-line parameters and invokes the Interpreter.  The Interpreter 
is available as a general-purpose class and is designed to be easily hosted and "tweaked".
cardiac.exe can optionally emit a profile trace via the "/profile+" switch.
  
### cardbg.exe
This is an interactive debugger. Right now it only supports single-stepping through the code.  If no .cardb file is found it will display
disassembly of the raw CARDIAC opcodes; otherwise it will display source code and an enhanced view of CARDIAC memory during debugging.

## Quick tour

### The CARDIASM language

The CARDIASM language is built around the opcodes defined in the [manual](https://www.cs.drexel.edu/~bls96/museum/CARDIAC_manual.pdf). 
However I did not want developers to worry about memory addresses so the language adds a few familiar
features (defining constants, variables and labels) to make it easier to write code for the platform.

For instance, a very simple cardiac program to add 1+1 and output the result is shown below: This program defines
a and b as 1 then loads a into the Accumulator register and adds b.  The result is stored into "result" and result is then output. 
The final instruction (HRS 0) is a Halt-Reset which just ends the program.

```
DEF a 1  
DEF b a  
DEF result 0  
  
CLA a  
ADD b  
STO result  
OUT result  
HRS 0  
```

#### Definitions
DEF statements can appear anyway.  They can reference each other.  For boolean operations **true** can be 
defined as 1 but I recommend defining **false** as -1 because of the way the TAC instruction works.

```
DEF true 1
DEF false -1
DEF a true
DEF b false
```
CARDIASM does not distinguish between constant values and variables.  However the compiler is 
smart enough to omit un-referenced definitions from the compiled image so you can define variables
liberally.  Note that there is no notion of scope; all definitions are visible everywhere. 

#### Comments
Comments in CARDIASM begin with two slashes.  They can be standalone or at the end of a line. 

#### Labels
* Labels can be used anywhere and are denoted via the familiar C syntax of "<label>: statement". 
The follow program displays the values 1 through 100 and uses labels as targets for the JMP and 
TAC statements. This allows code to be *relocatable*.
```
//This program counts to 100

DEF n    100  
DEF cntr 000 

CLA	00              //Initialize the counter to 1 (address 00 always contains 1)
STO	cntr	

loop:
	CLA	n       //Load n into accumulator
	TAC	exit    //If n < 0, exit
	OUT	cntr    //Output cntr
	
        // cntr++
	CLA	cntr    
	ADD	00      //clever way to implement++ (00 always contains the value 1)
	STO	cntr    
	
        // n--
	CLA	n	    
	SUB	00
	STO	n
	
	JMP	loop

exit:	
	HRS	00
```
  
#### Self-modifying code 
CARDIAC has only one register and no stack so self-modifying code can be useful.  For this reason the language
supports storing values to addresses denoted by labels. The following function from fizzbuzz.cardiasm uses this 
capability in the line STO divides_exit.  This provides a crude way of implementing subroutines by replacing
the "Halt-Reset" at the end of the function with a jump back to the calling location.

```
divides_entry:
	CLA 99
	STO divides_exit //override the "HRS" at divides_exit with a jump to the return address
                         //stored in memory location 99

        ...

divides_exit:
	HRS 0  //This will be a JMP instruction by the time it executes
```

### Compiling programs
If you save the first example above to **add.cardiasm** you can compile it via the command line below:
  
``
cardiasm /debug+ add.cardiasm
``
  
This will produce two files:
* **add.cardimg** - an executable image suitable for executing with cardiac.exe
* **add.cardb**   - a cardiac program database (roughly analogous to a ".pdb") useful for debugging

The actual executable image will be in the form described at 
[the Drexel page](https://www.cs.drexel.edu/~bls96/museum/cardiac.html). **.cardimg** files are 
just text files where each line holds a single value (I chose text over binary because it's 
easier to read and manipulate). 


### Running a program
To run the program you can just pass the name of the compiled image to cardiac.exe:
```
C:\> cardiac add.cardimg
2
```

#### I/O
The CARDIAC is not very well-suited for interactive I/O.  The only I/O instruction is 
a blocking read, and the CARDIAC has no way to deal with text.  **cardiac.exe** supports 
passing arguments on the command-line.  It is up to you to pass the right number.  If
a program tries to read input and none is available cardiac.exe will crash with an
InvalidOperationException and the text "The Program requires input but none is available".
  
The following program expects two arguments and outputs their sum:
```
DEF a 1
DEF b 1
DEF result 0

INP a
INP b

CLA a
ADD b
STO result
OUT result
HRS 0
```

To run this program pass a and b on the command line:
```
C:\> cardiac add.cardimg 1 1 
2
C:\> cardiac add.cardimg 2 3 
5
C:\> cardiac add.cardimg 2 
Unhandled Exception: System.InvalidOperationException: The program requires input but no input available

```
  
### Using the debugger
To debug the program run the debugger as follows:
```
C:\> cardbg add.cardimg
```

You should see a display like the following:
```
 001 002 803 198 297 696 596 900 000 000 000 000 000 000 000 000 000 000 000 000
 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000
 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000
 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000
 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 001 001 803
[ACC:  000] OUT:


   [1] DEF a 1
   [2] DEF b 1
   [3] DEF result 0
==>[4] CLA a
   [5] ADD b
   [6] STO result
   [7] OUT result
   [8] HRS 0
```

The top view displays the memory contents of the CARDIAC (all 100 memory cells) along with the accumulator
and any output.  The bottom display is a source code view.  The debugger is currently very limited; simply
press enter to single-step over the code. The debugger will use color to highlight certain areas, namely:
  
* **Dark Gray** for unused memory  
* **Yellow** for highlighting the current instruction
* **Light Gray** for memory cells containing code
* **Dark Magenta** for memory cells containing constants/variables
* **Light Magenta** to highlight the current Operand address if the current instruction references memory.
For example if the current instruction is "CLA a" and "a" is mapped to address 98 then address 98 will be
highlighted in magenta
* **Red** to highlight the most recently-changed value (so if you step-over you can easily see what changed
(if anything).
  
## Other Resources
The CARDIAC instruction Manual can be found [here](https://www.cs.drexel.edu/~bls96/museum/cardiac.html).
The best page for CARDIAC is probably [this page at cs.drexel.edu](https://www.cs.drexel.edu/~bls96/museum/cardiac.html)
