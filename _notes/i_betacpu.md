---
layout: academic
permalink: /notes/betacpu
title: Week 5 Part 1 - Building Beta CPU
description: In-depth study on the Beta CPU's datapath
tags: [instructionset, combinational, sequential, beta, interrupt, exception, datapath]
---

* TOC
{:toc}

**50.002 Computation Structures**
<br>
Information Systems Technology and Design
<br>
Singapore University of Technology and Design


# Building the $$\beta$$ CPU


## Overview

In the previous chapter, we were introduced to the $$\beta$$ ISA, a CPU blueprint that specifies what instructions the CPU can process, how it interacts with the memory unit, the basic CPU components, instruction formats, and many more. 

In this chapter, we will study how each of the 32 $$\beta$$ instructions is supposed to work, and how the $$\beta$$ **CPU** (an implementation of the $$\beta$$ ISA) is able to compute each and every one of them by reprogramming its datapath without physically changing its hardware.  

The key is to have a proper **Control Logic** unit that is able to ***decode***  current instruction's `OPCODE` and give out the correct control signals (PCSEL, RA2SEL, ASEL, etc) to reprogram the datapath. The complete truth table of the control logic unit is as shown below,

<img src="
https://dl.dropboxusercontent.com/s/2txzo6r3aeynguy/CU_2.png?raw=1"  width="80%" height = "80%">

> This unit can be easily implemented using a read only memory. 

We will go through the workings of each instruction and understand how the given $$\beta$$ datapath is able to execute the instruction properly by producing appropriate control signals as shown above. 

  

## Instruction Cycles
### Instruction Fetch

The first thing a CPU must do is to figure out:
* What is the *address* of the instruction to execute next and 
* Fetch (read) them from the Memory Unit  

Instructions are produced by a compiler and are specific to the CPU's ISA. The control unit will know what control signals to produce and which signals need to be *routed* where for each type of instruction.

> For example, when you double-click (run) an executable `.exe` on Windows, the code for that program is moved from Disk into the Memory Unit (RAM), and the CPU is told what address the first instruction of that program starts at. 
> 
> The CPU **always** maintains an internal register called the Program Counter (PC) that holds the memory location of the next instruction to be executed. 
>

Once the CPU knows the address of the first instruction to be executed, it can fetch it from the Memory Unit and execute it. The next steps are easy. 
* The first instruction will then tell the CPU what to do next, where is the second instruction, and so on.  
* The second instruction will also tell the CPU what to do next, where is the third instruction, and so on. 
* This is repeated until the CPU met a `HALT()` instruction.

>As of now, you always assume that the content of the PC register is always initially zero (32-bit of zeroes), and that the first line of your program instruction is always put at memory address zero. 


###  Instruction Decoding

When the CPU has an instruction, it needs to figure out (decode) specifically what type of instruction it is. Each instruction will have a certain set of bits called the `OPCODE` that tells the CPU how to interpret it. In the $$\beta$$ ISA, the `OPCODE` can be found in the 6 most significant bits of the 32-bits instruction. The `OPCODE` is given as an input to the Control Unit, it will compute the appropriate control signals to program the datapath. 

This decoding step depends on how complex the ISA is. An ISA like RISC (e.g: the $$\beta$$ ISA) has a smaller number of instructions (a few dozens) while x86 has thousands. The most common family of instructions are:
* **Memory**: anything regarding loading and storing of data between the REGFILE (CPU internal storage) and the Memory Unit. No other computation is performed.
* **Arithmetic**: anything that requires computation using the ALU, and inputs are taken from the REGFILE.  
* **Branch instructions**: anything pertaining to changing the value of PC Register to load instructions in different Memory Address, (*conditional*) based on a content of a specific register in the REGFILE. 

## Detailed Anatomy of the $$\beta$$ CPU

  
The $$\beta$$ CPU  is comprised of the following standard parts that typically make up a CPU:  PC,  REGFILE,  ALU, and CU.

  

### Program Counter

 
The PC is a 32-bit register (i.e: a set of **32** 1-bit registers). Its job is to store the address of the **current** instruction that is executed. 
>For now, we can safely assume that the initial content of the PC REG is always zero. 

The datapath of the components involving the PC is shown in the figure below:

<img src="https://dl.dropboxusercontent.com/s/mx1cjmc3ugmcdc8/pcreg.png?raw=1"  width="80%" height = "80%">

Two important things happened **simultaneously** at every clock cycle:
* *As highlighted in red,* the output of the PC is connected to the `IA` port (the input address port) of the Memory Unit (RAM), hence the Memory Unit will produce the content of that address through the `Ins` port. 

* *As highlighted in green*, the output of the PC REG will also be added by 4. 
	* If `PCSEL=0` and `RESET=0`,  this value (old PC + 4) will enter the PC REG in the next clock cycle. This will cause the PC to supply the address of the **subsequent instruction word** in the next clock cycle. 
	*  If `PCSEL!=0` and `RESET=0`, then the value in the PC REG will be equivalent to either of the inputs to the PCSEL mux (depending on what `PCSEL` value is). 

If `RESET=1` then the value of the PC REG in the next cycle will be equivalent to `Reset`. We will learn what `Reset` is in the later weeks -- but in short, `Reset = 0x0000 0000` for $$\beta$$ ISA.  Else if `RESET=0`, the value in PC REG will always be increased by 4 at each clock cycle. 
  

### Register Files

The REGFILE in $$\beta$$ ISA is the CPU's internal storage unit that is comprised of 32 sets of 32-bit registers, denoted as $$R_0, R_1, ...., R_{31}$$. **Each register is addressable in 5 bits**. For example: `00000` is the address of $$R_0$$, `00001` is the address of $$R_1$$, `00010` is the address of $$R_2$$, and so on.

> Remember, a 32-bit register simply means a set of **32** 1-bit registers

The figure below shows the anatomy of $$\beta$$ REGFILE component:

<img src="https://dl.dropboxusercontent.com/s/03v5c3gy4ucoga6/regfiles.png?raw=1"  width="80%" height = "80%">

It has two **combinational** read ports: `RD1` and `RD2`, and one **clocked / sequential** write port: `WD`. 

We can simultaneously, at the same clock cycle, **read** the contents of two selected registers, addressable in 5 bits denoted as `Ra` and `Rb`, :
* The 5-bit address `Ra` is supplied through port `RA1`
* The 5-bit address `Rb` is supplied through `RA2`

We can also **write** data supplied at the `WD` port to any of the registers in the REGFILE:
* In order to write, a valid `1` must be supplied at the `WE` port
* The address of the register to write into is determined by the 5-bit input supplied at the `WA` port. 

#### The Write Enable Signal
Recall that a register / D Flip-Flop "captures" a new input at each CLK rise, and is able to maintain that **stable** value for the period of the CLK. 


However, in practice, we might not want our register to "capture" new input all the time, but only on certain moments. Therefore, there exist a `WE` signal such that:
* When it's value is `1`, the register "captures" and stores the current input.
* Otherwise, the register will ignore the input and will output the last stored value regardless of the CLK edge. 


#### Detailed Anatomy of the REGFILE

To understand how the `WE` signal works more clearly, we need to dive deeper into the inner circuitry of the REGFILE. The figure below shows a more detailed anatomy of the REGFILE unit. 

<div class="orangebox"><code>R31</code>'s <strong>content</strong> is <strong> always </strong> <code>0x00000000</code>, regardless of what values are written to it. Therefore it is not a regular register like the other 30 registers in the REGFILE. It is simply giving out <code>0x00000000</code> as output when <code>RA1</code> or <code>RA2</code> is <code>11111</code>, which is illustrated as the 0 on the rightmost part of each read muxes.</div><br>

<img src="https://dl.dropboxusercontent.com/s/yi9exbpy2vhgz14/regfile_inside.png?raw=1"  width="80%" height = "80%">

The `WE` signal is fed into a 1-to-32 demultiplexer unit. The `WA` signal is the selector of this demux. As a result, only 1 out of the 32 outputs of the demux will follow exactly the value of `WE`. 

The outputs of the demux is used as a selector (`EN` port) to each of the *2-to-1* 32-bit multiplexer connected to each 32-bit register.

> Note: although not drawn (to not clutter the figure further), all the registers are synchronized with the same CLK. 

#### The Static and Dynamic Discipline of the REGFILE

As mentioned above, the REGFILE unit has **2 combinational read ports** that is made up by the two large *32-to-1* 32-bit multiplexers drawn at the bottom of the figure. We can supply two read addresses: `RA1` and `RA2`. They are the selector signals of these two multiplexers. Therefore the time taken to produce valid output (32-bit) data at `RD1` and `RD2` is at least  the $$t_{pd}$$ of the multiplexer and $$t_{pd}$$ of the DFFs depending on when exactly the addresses become valid. 

This unit also have **1 sequential write port**. The write data is always supplied at `WD`. When the `EN` signal of a target register is a valid `1`, we need to wait until the nearest CLK rise edge in order for `WD` to be reflected at the `Q` port of that register. 

<span style="background-color:yellow; color:black"> In register transfer language, the content of register with address `A` is often denoted as : `Reg[A]` </span>

The timing diagram for read and write is shown below. Please take some time to study them: 
<img src="https://dl.dropboxusercontent.com/s/rvpovodxab54ywl/timing_reg.png?raw=1"  width="80%" height = "80%">


Notice how the new data denoted as `new Reg[A]` supplied at port `WD` (to be written onto `Reg[A]`) must fulfill both $$t_S$$ and $$t_h$$ requirement of the hardware. 

  

### Control Logic Unit

The heart of the control logic unit (CU) is a **combinational** logic device that receives 6-bit `OPCODE` signal, 1-bit `z` signal, 1-bit `RESET` signal, and 1-bit `IRQ` signal as input. We will discuss about `RESET`, `z` and `IRQ` much later on.

At each CLK cycle, the PC will supply a new 32-bit address to the Memory Unit, and in turn, 32-bit instruction data is produced by the Memory Unit. The first 6 bits of the instruction, called the `OPCODE` is supplied as an input to the CU. 

The CU will then decode the input combination consisted of `OPCODE`, `z`, `RESET`, and `IRQ`, and produce various control signals as shown in the figure below. In practice, this unit can be made using a ROM.

<img src="https://dl.dropboxusercontent.com/s/p73ywj1ju4fu3ed/Cu.png?raw=1"  width="50%" height = "50%">

Note that the `ALUFN` is 6 bits long, `PCSEL` is 3 bits long, `WDSEL` is 2 bits long, `RA2SEL`, `BSEL` `ASEL`, `WASEL`, `WR`, and `WERF` (`WE` to REGFILE) are all 1 bit long. The total number of output bits of the CU is therefore *at least* 17 bits long 

> In our Lab however, the output signal of the control unit is 18 bits long. We don't have to memorise these, as long as we get the main idea.


 Notice the presence of the 1-bit register that **samples** the IRQ signal. This is  because. the IRQ signal actually an **asynchronous** interrupt trigger. 

> In the later weeks, we will learn that _asynchronous interrupts_ are generated by **other hardware devices** at *arbitrary* times with respect to the CPU clock signals. Therefore, we need another set of devices to **condition** it such that it doesn't cause unwanted changes to the Control Unit in the middle of execution (in the middle of a clock cycle).  These devices allow the CPU to **sample** the input IRQ signal during the beginning of each instruction _cycle_ (e.g: resulting in sample result `IRQ'` denoted in the figure above), and will respond to the trigger only _if_ the signal `IRQ'` is asserted when sampling _occurs_.

In the Lab however, we also simplify this part. We simply assume that the IRQ signal given by the test file is guaranteed to be stable for the entire CPU clock cycle, and already fulfils the $$t_S$$ and $$t_H$$ requirements of the CPU clock cycle.   

**For simplicity, we no longer display this register unit in the diagrams to explain the datapaths below.** The presence of the `CLK` signal there is written to remind you that the CPU should be able to *sample* the asynchronous `IRQ` signal  for each clock cycle. ==However, the heart of the Control Unit  itself is combinational logic device (e.g: ROM) and not a sequential one==. 

## Beta Datapaths

The $$\beta$$ datapath can be reprogrammed by setting the appropriate control signals depending on the current instruction's `OPCODE`. In general, we can separate the instructions into four categories, and explain the datapath for each:
* The `OP` datapath (Type 1)
* The `OPC` datapath (Type 2)
* Memory access datapath (Type 2)
* Control transfer datapath  (Type 2)

 
## OP datapath

This datapath involves:
* Any logical computations using the ALU, and 
* The inputs to the `A` and `B` port of the ALU is taken from the contents of any two registers `Reg[Ra]` and `Reg[Rb]` from the REGFILE.
* The result is stored as a content of `Reg[Rc]`


The instructions that fall under `OP` category are: `ADD, SUB, MUL, DIV, AND, OR, XOR, CMPEQ, CMPLT, CMPLE, SHL, SHR`, and `SRA`. Its general format is:

<img src="https://dl.dropboxusercontent.com/s/sufiy5rhdo5k2j0/op_ins.png?raw=1"  width="60%" height = "60%">


The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`Reg[Rc]` $$\leftarrow$$ `Reg[Ra]` `(OP)`   `Reg[Rb]` 
- The corresponding assembly instruction format runnable in BSIM is `OP(Ra, Rb, Rc)`


> **Important:** Read the $$\beta$$ documentation and fully study the functionalities of each instruction. 

The figure below shows the datapath for all `OP` instructions:

<img src="https://dl.dropboxusercontent.com/s/xidshs27kjyzyc6/op.png?raw=1"  width="80%" height = "80%">

The highlighted lines show how the signals should flow in order for the $$\beta$$ to support `OP` instructions. 

 The control signals therefore must be set to: 

-  `ALUFN = F(OP)` 
	> the `ALUFN` signal for the corresponding operation `OP`, for example, if `OPCODE = SUB` then `ALUFN  = 000001`, and so on.

-  `WERF = 1`

- `BSEL = 0`

-  `WDSEL = 01`

-  `MWR = 0`

-  `RA2SEL = 0`

-  `PCSEL = 000`

-  `ASEL = 0` 

-  `WASEL = 0`

> Take some time to understand why the value of these control signals must be set this way to support the `OP` instructions. 



## OPC datapath

 The `OPC` (Type 2 instruction) datapath is similar to the `OP` datapath, except that input to the `B` port of the ALU must comes from `c = I[16:0]`. 

**There is no `Rb` field in Type 2 instruction**. 

The instructions that fall under `OPC` category are: `ADDC, SUBC, MULC, DIVC, ANDC, ORC, XORC, CMPEQC, CMPLTC, CMPLEC, SHLC, SHRC`, and `SRAC`. It's general format is:

<img src="https://dl.dropboxusercontent.com/s/wcirw4bgwhh2xbg/opc_insdfhi9j45vnuw7n0/opc.png?raw=1"  width="60%" height = "60%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`Reg[Rc]` $$\leftarrow$$ `Reg[Ra]` `(OP)`   `SEXT(C)` 

>Again, don't forget to read $$\beta$$ documentation to understand each functionalities. 
- The corresponding assembly instruction format runnable in BSIM  is `OPC(Ra, c, Rc)`

The figure below shows the datapath for all `OPC` instructions:

<img src="https://dl.dropboxusercontent.com/s/dfhi9j45vnuw7n0/opc.png?raw=1"  width="80%" height = "80%">

The control signals for `OPC` instructions are almost identical to `OP` operations, except that we need to have  `BSEL = 1`. 

### Sample Code
Try it yourself by running this code step by step on BSIM and observe the datapath to familiarize yourself with how OP and OPC datapath works.
* At each timestep, be aware of the value of PC and all Registers. 
* Familiarise yourself with how to translate from the assembly language to the 32-bit machine language

```cpp
.include beta.uasm

ADDC(R31, 5, R0)
SUBC(R31, 3, R1)
MUL(R0, R1, R2)
CMPEQ(R1, R1, R4) 
CMPLT(R0, R1, R4)
SHL(R1, R1, R5)
SRAC(R5, 4, R5)
SHRC(R1, 4, R6)
```

## Memory Access Datapath

There are three instructions that involve access to the Memory Unit: `LD`, `LDR` and `ST`. All of them are Type 2 instructions.


### LD Datapath

The general format of the `LD` instruction is:

<img src="https://dl.dropboxusercontent.com/s/bicusis1a1cx707/ld_ins.png?raw=1"  width="60%" height = "60%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`EA` $$\leftarrow$$ `Reg[Ra] + SEXT(C)`
`Reg[Rc]` $$\leftarrow$$ `Mem[EA]` 

-  The LD instruction allows the CPU to **load** one word (32-bit) of data from the Memory Unit and store it to `Rc`

- The **effective address** (`EA`) of the data is computed using the content of `Ra`  (32-bit) added with `c` (sign extended to be 32-bit).
- The corresponding assembly instruction format runnable in BSIM is `LD(Ra, c, Rc)`

The figure below shows the datapath for  `LD`:

<img src="https://dl.dropboxusercontent.com/s/479io11h24i9yl3/ld.png?raw=1"  width="80%" height = "80%">

The control signals therefore must be set to: 

-  `ALUFN = ADD (000000)` 

-  `WERF = 1`

- `BSEL = 1`

-  `WDSEL = 10`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL = 000`

-  `ASEL = 0` 

-  `WASEL = 0`
  




### LDR datapath

The `LDR` instruction is similar to the `LD` instruction, except in the method of computing the `EA` of the data loaded. 

The general format of the `LDR` instruction is:

<img src="https://dl.dropboxusercontent.com/s/5kj00vwcw0ghlfp/ldr_inst.png?raw=1"  width="60%" height = "60%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`EA` $$\leftarrow$$ `PC+4*SEXT(C)`
`Reg[Rc]` $$\leftarrow$$ `Mem[EA]` 

*  The `LDR` instruction computes `EA` **relative** to the current address pointed by `PC`. 
* The corresponding assembly instruction format runnable in BSIM is `LDR(label, Rc)`, where `c` is **auto** computed as `(address_of_label - address_of_current_ins)/4-1`


The figure below shows the datapath for `LDR`:

<img src="https://dl.dropboxusercontent.com/s/cuqicqkpj12a6u5/ldr.png?raw=1"  width="80%" height = "80%">

The control signals therefore must be set to: 

-  `ALUFN = 'A' (011010)` 
	> The ALU is simply required to be *transparent*, i.e: "pass" the input at the `A` port through to its output port. 

-  `WERF = 1`

- `BSEL = --`

-  `WDSEL = 10`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL = 000`

-  `ASEL = 1` 

-  `WASEL = 0`


  
  
### ST datapath

The `ST` instruction does the opposite to what the `LD` instruction does. It allows the CPU to store contents from one of its REGFILE registers to the Memory Unit. 

> Note that the instruction `ST` and `LD`/`LDR` allows the CPU to have access to an expandable memory unit without changing its datapath, although the CPU itself has a limited amount of internal storage in the REGFILE. 

The general format of the `ST` instruction is:

<img src="https://dl.dropboxusercontent.com/s/is3q37kvo167325/st_ins.png?raw=1"  width="60%" height = "60%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`EA` $$\leftarrow$$ `Reg[Ra]+SEXT(c)`
`Mem[EA]`  $$\leftarrow$$   `Reg[Rc]`

-  The ST instruction **stores**  data present in `Rc` to the Memory Unit. 

- Similar to how `EA` is computed for `LD`, the **effective address** (`EA`) of where the data is supposed to be stored is computed using the content of `Ra`  (32-bit) added with `c` (sign extended to be 32-bit).
- The corresponding assembly instruction format runnable in BSIM is `ST(Rc, c, Ra)`, notice the swapped `Rc` and `Ra` position. 


The figure below shows the datapath for  `ST`: 

<img src="https://dl.dropboxusercontent.com/s/bbvysz2jvud00s1/st.png?raw=1"  width="80%" height = "80%">

  
The control signals therefore must be set to: 
-  `ALUFN = 'ADD' (000000)` 

-  `WERF = 0`

- `BSEL = 1`

-  `WDSEL = --`

-  `MWR = 1`

-  `RA2SEL = 1`

-  `PCSEL = 000`

-  `ASEL = 0` 

-  `WASEL = --`

### Sample Code
**Try it yourself** by running this code step by step on BSIM and observe the datapath to familiarize yourself with how OP and OPC datapath works.
* At each timestep, be aware of the value of PC and all Registers. 
* Be aware on the value stored at certain memory locations 
* Familiarise yourself with how to translate from the assembly language to the 32-bit machine language using *labels* and *literals*


```cpp
.include beta.uasm

LD(R31, x, R0)
LD(R31, x + 4, R1)
LD(R31, x + 8, R2)
LD(R31, x + 12, R3)
LDR(x, R4)
LDR(x+8, R5)
MUL(R0, R3, R0)
ADD(R1, R1, R1)
ADDC(R31, 12, R6)
ST(R0, x)
ST(R1, x, R6)

x : LONG(15) | this is an array
	LONG(7)
	LONG(9)
	LONG(-1)
```


## Control Transfer Datapath

There are three instructions that involves **transfer-of-control** (i.e: *branching*, or *jumping*), that is to change the value of `PC` so that we can execute instruction from other `EA` in the Memory Unit instead of going to the next line. These instructions are `BEQ`, `BNE`, and `JMP`. 

We will not use the ALU at all when transferring control. 

> So far, we have only seen `PC` to be advanced by 4:  `PC` $$\leftarrow$$ `PC+4`. With instructions involving transfer-of-control or , we are going to set `PC` a little bit differently. 


### BEQ datapath

This instruction allows the `PC` to *branch* to a particular `EA` if the content of `Ra` is zero.  It is commonly used when checking for condition prior to branching, e.g: `if x==0, else`.
  
The general format of the `BEQ` instruction is:

<img src="https://dl.dropboxusercontent.com/s/hla3dyi15xjxocf/beq_inst.png?raw=1"  width="80%" height = "80%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`Reg[Rc]` $$\leftarrow$$ `PC`
`EA` $$\leftarrow$$ `PC + 4*SEXT(C)`
`if (Reg[Ra] == 0)` then `PC` $$\leftarrow$$ `EA`

* The **address** of the instruction following the `BEQ` instruction is written to `Rc`. 
* If the contents of  `Ra` are zero, the `PC` is loaded with the target address `EA`;
* Otherwise, execution continues with the next sequential instruction.
* <mark> The checking of the content of `Ra` is not done through ALU, but rather through the 32-bit NOR gate that produces `Z` (1-bit) </mark>, The value of `Z` is fed to the CONTROL UNIT to determine whether PCSEL should be `001` or `000` depending on the value of `Z`.
* The corresponding assembly instruction format runnable in BSIM is `BEQ(Ra, label, Rc)`* where `c` is **auto** computed as `(address_of_label - address_of_current_ins)/4-1`



The figure below shows the datapath for the `BEQ`: 
<img src="https://dl.dropboxusercontent.com/s/sp2ee8dny2n2qzs/bne.png?raw=1"  width="80%" height = "80%">

  

  

The control signals therefore must be set to: 
-  `ALUFN = --`
	> We aren't using the ALU at all when transferring control.  

-  `WERF = 1`

- `BSEL = --`

-  `WDSEL = 00`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL = Z ? 001 : 000`

-  `ASEL = --` 

-  `WASEL = 0`

  
  
### BNE datapath

`BNE` is similar to  `BEQ`, but branches `PC` in the opposite way, i.e: when `Ra != 0`. It also utilizes the output `Z`.
  
The general format of the `BNE` instruction is:

<img src="https://dl.dropboxusercontent.com/s/wrqpdsusx3g7lkd/bne_ins.png?raw=1"  width="80%" height = "80%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`Reg[Rc]` $$\leftarrow$$ `PC`
`EA` $$\leftarrow$$ `PC + 4*SEXT(C)`
`if (Reg[Ra] != 0)` then `PC` $$\leftarrow$$ `EA`
* The corresponding assembly instruction format runnable in BSIM is `BNE(Ra, label, Rc)`* where `c` is **auto** computed as `(address_of_label - address_of_current_ins)/4-1`


The figure below shows the datapath for the `BNE`: 
<img src="https://dl.dropboxusercontent.com/s/xgapxsuqjnjl4rv/beq.png?raw=1"  width="80%" height = "80%">


The control signals therefore must be set to: 
-  `ALUFN = --`
	> We aren't using the ALU at all when transferring control.  

-  `WERF = 1`

- `BSEL = --`

-  `WDSEL = 00`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL = Z ? 000 : 001`

-  `ASEL = --` 

-  `WASEL = 0`


  
### JMP Datapath

`JMP` also allows the CPU to change its `PC` value, but without any condition (*jump*). 
  
The general format of the `JMP` instruction is:
<img src="https://dl.dropboxusercontent.com/s/94bul2ifo7a3afj/jmp_inst.png?raw=1"  width="80%" height = "80%">

The register transfer language for this instruction is: 
`PC` $$\leftarrow$$ `PC+4`
`Reg[Rc]` $$\leftarrow$$ `PC`
`EA` $$\leftarrow$$ `Reg[Ra] & 0xFFFFFFFC` (masked)
`PC`$$\leftarrow$$ `EA`

* The **address** of the instruction following the `JMP` instruction is written to `Rc`, then  `PC` is loaded with the contents of  `Ra`.
* The low two bits of `Ra` are **masked** to ensure that the target address is aligned on a 4-byte boundary.
* The corresponding assembly instruction format runnable in BSIM is `JMP(Ra, Rc)`.

The figure below shows the datapath for the `JMP`: 
<img src="https://dl.dropboxusercontent.com/s/gsylkuouphx85ea/jmp.png?raw=1"  width="80%" height = "80%">


The control signals therefore must be set to: 
-  `ALUFN = --`
	> We aren't using the ALU at all when transferring control.  

-  `WERF = 1`

- `BSEL = --`

-  `WDSEL = 00`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL = 010`

-  `ASEL = --` 

-  `WASEL = 0`


### Sample Code
**Try it yourself** by running this code step by step on BSIM and observe the datapath to familiarize yourself with how OP and OPC datapath works.
* At each timestep, be aware of the value of PC and all Registers. 
* Know where is the address of each instruction when loaded to memory
* Note how to translate from `label` to `literal`  when crafting the 32-bit machine language for `BEQ/BNE` instructions.
  
```cpp
.include beta.uasm

ADDC(R31, 3, R0)

begin_check: CMPEQ(R31, R0, R1)
BNE(R1, is_zero, R10)
SUBC(R0, 1, R0)
BEQ(R31, begin_check, R10)

is_zero: JMP(R31)
```




## Exception Handling

**Exceptions** as the name suggests, is an event generated by the CPU when an *error* occurs. 

$$\beta$$ exceptions come in three flavors: traps, faults, and interrupts. 

1.  **Traps** (*intentional*) and **faults** (*unintentional*) are both the **direct outcome of an instruction** and are distinguished by the programmer's **intentions**. 
	- These happens for example when we supply an illegal `OPCODE`, i.e: it does not correspond to any of the instructions defined in the ISA.

2.  **Interrupts**  are **asynchronous** with respect to the instruction stream, and are usually caused by **external events**, for example from I/O devices, or network devices. 	This would require us to "*pause*" the execution of the current program and handle the interrupt. 				
	- At the beginning of each cycle, the CPU will always check whether `IRQ == 1`.  
	- If `IRQ != 1`, the CPU will continue with normal execution. 
	- If `IRQ == 1`, the CPU will *pause* the current execution and handle the interrupt request first (and eventually *resume* back the paused execution *after the interrupt handling is done*). 
 
The datapath that handles **trap/fault** (due to Illegal `OPCODE`)  is shown on the left, and the datapath that handles **interrupt** is shown on the right: 

<img src="https://dl.dropboxusercontent.com/s/hfyi0bx54tptiyr/illopirq.png?raw=1"  width="100%" height = "100%">

> There's only one difference between the two, the datapath at the PCSEL mux. 
  
The PCSEL multiplexer's fourth and fifth input are called `ILLOP` and `XAdr`. These refers to the address of the **instruction branching** to the **`interrupt handling`** code, in the events that trap, fault, or interrupt occurs.  In $$\beta$$ ISA,
-  `ILLOP` is set at `0x80000004`
-  `XAdr` is set at `0x80000008`

The control signals in the events of these exceptions therefore must be set to: 
-  `ALUFN = --`
	> We aren't using the ALU at all when transferring control.  

-  `WERF = 1`

- `BSEL = --`

-  `WDSEL = 00`

-  `MWR = 0`

-  `RA2SEL = --`

-  `PCSEL`:
	-  `Illegal_Opcode ? 011 : 000`
	- `IRQ ? 100 : 000`

-  `ASEL = --` 

-  `WASEL = 1`


## CPU Benchmarking

We always want a CPU that has a high performance (most instruction per second) at a low cost. Unfortunately there will always be a tradeoff between the two. We can benchmark the quality of a CPU by computing its $$MIPS$$ (million instruction per second),

$$MIPS = \frac{Clock Rate }{CPI}$$

where $$CPI$$ means "Clocks per Instruction".

Although it is common to judge a CPU's performance from its *clock rate* (cycles per second, typically ranging between 2-4 GHz per core for modern computers), we also need to consider another metric called the $$CPI$$, that is the *average clock cycles* used to execute a single instruction.

In $$\beta$$ ISA, each instruction requires only 1 clock cycle to complete (atomic execution). It is possible for other ISA to take more than 1 clock cycle *on average* to complete an instruction. 

Typically, one will choose a particular program (with fixed number of instructions) for benchmarking purposes, and **the same benchmark program** is run on different CPUs with potentially different Clock Rate and $$CPI$$. 

The higher the $$MIPS$$, the faster it takes to run the benchmark program. Therefore we can say that a CPU with the highest $$MIPS$$ has the best performance.


