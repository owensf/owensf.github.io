---
title:  "The Instruction Set"
parent: "MIPS CPU"
nav_order: 1
---

# The Instruction Set
The foundation of any CPU design is the Instruction Set Architecture (ISA). This is the first choice a computer architect must make and will influence everything that follows. The ISA is the set of instructions that the CPU is capable of executing. Programs may be written directly using these instructions (Assembly), or by using a compiler that translates a high level language (like C) into these instructions. I could create my own ISA and build a computer around it, but no compilers would be able to create programs for my CPU to run. Thus, great care must be taken when selecting an ISA. The course staff at Purdue opted for a slightly modified MIPS ISA. MIPS is a Reduced Instruction Set Computer (RISC) architecture that attempts to simplify and reduce the set of instructions that hardware engineers must implement. RISC architectures have some great benefits compared to Complex Instruction Set Computers (CISC). For example MIPS only has four instructions that modify memory (RAM), two types of load, and two types of save. This massively simplifies our work at the expense of increasing the number of instructions needed to accomplish a task. That's an okay trade-off because the compiler will typically do all that heavy lifting for us. The C code will look identical across different ISAs.


# Why not RISC-V?
That's a good question. The course staff at Purdue chose MIPS, so that's what I built. As it turned out, we didn't have a MIPS C compiler, so all of the programs we ran on our CPUs were hand-written in Assembly. If I could give one piece of feedback to the course staff, it would be to update the course and use a simplified RV32I ISA.


# Reserved Registers
First, lets look at the register structure of our ISA. MIPS specifies 32 registers each 32bits in size. The zeroth register is not writable and has a constant value of zero. The other reserved registers are used either by a compiler, or by special instructions. However, they have no special hardware associated with them and can be read or written to just like other registers.

```
$0                   zero
$1                   assembler temporary
$29                  stack pointer
$31                  return address
```


# R-type Instructions
These instructions are the bread and butter of the CPU. They take two registers ```$rs,$rt``` and perform some operation on them, saving the result in the destination register ```$rd```. you might recognize basic logical functions like AND, OR, and  XOR. Also included are basic arithmetic functions like addition and subtraction. Noticeably missing is multiply and divide! This is one of the trade offs the MIPS ISA makes in the name of hardware simplicity. The compiler or programmer must write their own multiplication and division subroutines; an easier job for the hardware engineer means slower performance.

The odd one out here is ```JR```. This is an unconditional jump to the value of the register ```$rs```.

```
ADDU   $rd,$rs,$rt   R[rd] <= R[rs] + R[rt] (unchecked overflow)
ADD    $rd,$rs,$rt   R[rd] <= R[rs] + R[rt]
AND    $rd,$rs,$rt   R[rd] <= R[rs] AND R[rt]
JR     $rs           PC <= R[rs]
NOR    $rd,$rs,$rt   R[rd] <= ~(R[rs] OR R[rt])
OR     $rd,$rs,$rt   R[rd] <= R[rs] OR R[rt]
SLT    $rd,$rs,$rt   R[rd] <= (R[rs] < R[rt]) ? 1 : 0
SLTU   $rd,$rs,$rt   R[rd] <= (R[rs] < R[rt]) ? 1 : 0
SLLV   $rd,$rs,$rt   R[rd] <= R[rt] << [0:4] R[rs]
SRLV   $rd,$rs,$rt   R[rd] <= R[rt] >> [0:4] R[rs]
SUBU   $rd,$rs,$rt   R[rd] <= R[rs] - R[rt] (unchecked overflow)
SUB    $rd,$rs,$rt   R[rd] <= R[rs] - R[rt]
XOR    $rd,$rs,$rt   R[rd] <= R[rs] XOR R[rt]
```


# I-type Instructions
These are immediate instructions. This means that the ```imm``` part of the instruction must be known when the program is written, and the value is embedded in the instruction itself. For instance, if the programmer wanted to set the value of register ```$4``` to 512, they could use ```ADDI $4, $0, 512```. Notice that the source register ```$rs``` must still be specified, but we chose ```$0``` which has a constant value of zero.

Our primary memory access instructions are load word ```LW``` and save word ```SW```. The other two memory access instructions ```LL``` and ```SC``` are used to create atomic actions between two cores in a multi-core CPU. These instructions will be required when we want to program in a multi-threaded environment. I'll discuss these two later on.

Immediate instructions also include conditional branches. These will form the basis of loops and conditional statements. These are branch equal ```BEQ``` and branch not equal ```BNE```. By using a comparison like ```SLT $1,$2,$3``` and then ```BNE $0,$1,label``` we branch to ```label``` if ```$2 < $3```.

```
ADDIU  $rt,$rs,imm   R[rt] <= R[rs] + SignExtImm (unchecked overflow)
ADDI   $rt,$rs,imm   R[rt] <= R[rs] + SignExtImm
ANDI   $rt,$rs,imm   R[rt] <= R[rs] & ZeroExtImm
BEQ    $rs,$rt,label PC <= (R[rs] == R[rt]) ? npc+BranchAddr : npc
BNE    $rs,$rt,label PC <= (R[rs] != R[rt]) ? npc+BranchAddr : npc
LUI    $rt,imm       R[rt] <= {imm,16b'0}
LW     $rt,imm($rs)  R[rt] <= M[R[rs] + SignExtImm]
ORI    $rt,$rs,imm   R[rt] <= R[rs] OR ZeroExtImm
SLTI   $rt,$rs,imm   R[rt] <= (R[rs] < SignExtImm) ? 1 : 0
SLTIU  $rt,$rs,imm   R[rt] <= (R[rs] < SignExtImm) ? 1 : 0
SW     $rt,imm($rs)  M[R[rs] + SignExtImm] <= R[rt]
LL     $rt,imm($rs)  R[rt] <= M[R[rs] + SignExtImm]; rmwstate <= addr
SC     $rt,imm($rs)  if (rmw) M[R[rs] + SignExtImm] <= R[rt], R[rt] <= 1 else R[rt] <= 0
XORI   $rt,$rs,imm   R[rt] <= R[rs] XOR ZeroExtImm
```


# J-type Instructions
Jump type instrucions are fairly simple. ```J``` is an unconditional jump to an immedeate value encoded in the instruction. ```JAL``` takes the address of the *next* instruction and places it in ```$31```, then jumps to ```label```. This is the foundation of a function call. ```JAL``` allows the called function to return and continue execution using ```JR $31```.

```
J      label         PC <= JumpAddr
JAL    label         R[31] <= npc; PC <= JumpAddr
```


# Other Instructions
This is a special one. ```HALT``` does not modify the state of any register, and it causes the value of the program counter to stay the same. This means that the CPU will not advance to the next instruction. There is no way to recover from a ```HALT```. Divine intervention is needed (or a loss of power).

```
HALT
```


# Pseudo Instructions
These instructions are translated by the assembler into a sequence of other instructions. ```PUSH``` and ```POP``` allow the programmer an easy mnemonic to manage a stack and ```NOP``` simply doesn't do anything at all (but the program counter is still incremented). In our case ```NOP``` is translated to ```SLL $0,$0,$0``` which has the serendipitous bit encoding of ```0x00000000```.

```
PUSH   $rs           $29 <= $29 - 4; Mem[$29+0] <= R[rs] (sub+sw)
POP    $rt           R[rt] <= Mem[$29+0]; $29 <= $29 + 4 (add+lw)
NOP                  Nop
```


# Assembler Keywords
Assembler keywords create a layer of abstraction (admittedly a thin one) that the programmer can leverage to set the address of code blocks, and to initialize sections of memory. These tools allow us to configure the state of the entire memory before any instructions are executed.

```
org  Addr         Set the base address for the code to follow
chw  #            Assign value to half word memory
cfw  #            Assign value to word of memory
```
