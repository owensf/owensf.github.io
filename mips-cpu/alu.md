---
title:  "The ALU"
parent: "MIPS CPU"
nav_order: 2
---

# The Arithmetic Logic unit

This is it. This is where it things happen. The ALU is the collection of circuitry that actually performs operations on data. The ALU has two input ports for data, one input port to select the desired function, and one output port. There are no flip-flops here, this is purely combination logic (The value of the output is purely a function of the current inputs). Below is the Verilog module definition showing three inputs and one output.


```verilog
module alu (input logic [31:0]  porta, 
            input logic [31:0] 	portb, 
            input logic [3:0] 	aluop, 
            output logic [31:0] outputport);
```

The 4-bit ```aluop``` is an encoding of the function that is desired. Below is a table of operations that this ALU is capable of performing. You'll notice that it closely matches many of the R-type and I-type instructions. The instruction decoding logic is responsible for selecting the correct signals for ports A and B, as well as determining the correct function to perform.

```verilog
  typedef enum logic [3:0] {
    ALU_SLL     = 4'b0000,
    ALU_SRL     = 4'b0001,
    ALU_ADD     = 4'b0010,
    ALU_SUB     = 4'b0011,
    ALU_AND     = 4'b0100,
    ALU_OR      = 4'b0101,
    ALU_XOR     = 4'b0110,
    ALU_NOR     = 4'b0111,
    ALU_SLT     = 4'b1010,
    ALU_SLTU    = 4'b1011
  } aluop_t;
```

The logic of the ALU is contained in one big ```case``` statement. This is very similar to a ```switch``` statement in C. We will "switch" on the value of the ```aluop``` and the value of the output will be determined there.

```verilog
case (aluop)
  ALU_SUB: outputport = porta - portb;
  ALU_ADD: outputport = porta + portb;
  ALU_SLTU: outputport = porta < portb ? 1'b1: 1'b0;
  ALU_AND:  outputport = porta & portb;
  ALU_OR:   outputport = porta | portb;
  ALU_XOR:  outputport = porta ^ portb;
  ALU_NOR:  outputport = ~(porta | portb);
  ALU_SLL:  outputport = portb << porta[4:0];
  ALU_SRL:  outputport = portb >> porta[4:0];
  
  ALU_SLT: begin
    // both positive
    if ((porta[31] == 1'b0) && (portb[31] == 1'b0))
      outputport = porta < portb ? 1'b1: 1'b0;
    
    // a negative, b positive
    if ((porta[31] == 1'b1) && (portb[31] == 1'b0))
      outputport = 1'b1;

    // a positive, b negative
    if ((porta[31] == 1'b0) && (portb[31] == 1'b1))
      outputport = 1'b0;
    
    // both negative
    if ((porta[31] == 1'b1) && (portb[31] == 1'b1))
      // note: when both negative, "larger" number is less negative
      outputport = porta < portb ? 1'b1: 1'b0; 
  end
endcase
```

As you can see, the syntax of Verilog does all the heavy lifting for us. As a hardware engineers using Verilog we are thankfully abstracted away from having to piece together logic gates in order to add numbers. This way the synthesis tool-chains (hardware equivalent of compilers) are responsible for implementing the actual logical functions desired. They can leverage purpose built FPGA blocks, or standard cell macros defined in a fabricator's Process Design Kit (PDK).

You might notice that ```ALU_SLT``` takes many more lines of code than the other operators. This is a signed comparison, using Verilog's unsigned data types. It's not the most elegant way of writing this particular function, but it's fairly straight forward and demonstrates the four possible sign combinations when comparing two numbers. We are using 2's compliment to represent signed numbers, so the most significant bit has a negative value.
