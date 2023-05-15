---
title:  "The Register File"
parent: "MIPS CPU"
nav_order: 3
---

# The Register File
This is another easy hardware module. We want to create a small piece of memory that stores the values of each of our registers. Let's jump right in and look at the module definition.

```verilog
module register_file (input logic CLK, nRST, 
                      input logic WEN,
                      input logic [5:0] wsel, rsel1, rsel2,
                      input logic [31:0] wdat,
                      output logic [31:0] rdat1, rdat2);
```
We must include the global clock and reset signals to this module because it contains flip-flops. Our registers will *always* update their value on the rising edge of the ```CLK``` signal, and will *always* reset to zero on the falling edge of the ```nRST``` signal. This is a constant across our entire CPU and it simplifies the synthesis of our circuitry. On an FPGA the clock signals are carried on special wires, and putting logic gates between the clock source and flip-flops will cause a world of headache.

Let's take a look at the code that describes these flip-flops. We'll use a SystemVerilog construct called ```always_ff``` and specify the conditions when the code block should be evaluated (```posedge CLK``` and ```negedge nRST```). Using ```always_ff``` instead of the more common ```always``` is just a hint to the synthesis tool that we want to create a flip-flop inside this code block. If we mess up and don't describe the functionality of a flip flop, the synthesis tool will complain.

```verilog
logic [31:0][31:0] registers, nxt_registers;
	
always_ff@(posedge CLK, negedge nRST) begin
  if (nRST == 1'b0)
    registers <= '0;
  else
    registers <= nxt_registers;
end
```

We've defined two variables on the first line: ```registers``` (which are the actual flip-flops of the register file) and  ```nxt_registers``` (which is the value the registers should take in the *next* clock cycle). Both are 2d arrays of bits, which is at first a little confusing, but remember we want 32 words each 32 bits in length. In the code I wrote for my college course, I used custom types like ```word_t```, but I'm not using that here for the sake of demonstration.

So we've created the code that updates the registers, but we've never assigned any values to ```nxt_registers```. This is still fairly straight forward. Most of the time the next value for a register is the same as the current value. When our CPU needs to change the value of a register write enable ```WEN``` will go high and we'll use the data provided by ```wdat``` to change the value of the register specified by ```wsel```.

```verilog
always_comb begin
  nxt_registers = registers; // default case
  if (WEN)
    nxt_registers[wsel] = wdat; // A write to the register file
  nxt_registers[0] = '0; // $0 is constant value
end
```
We've used an ```always_comb``` block here to guard against accidentally creating a flip-flop. If we describe logic that is impossible to implement without a memory element, our synthesis tool will complain. The synthesis tools are smart enough to recognize that register zero will always have a value of zero. Even though we've explicitly described a flip-flop for that register, the synthesis tools will optimize it away. This is our first deception of the programmer. The ISA promised 32 registers, but only 31 will exist in hardware.

We're almost done with the register file; all that remains now is to define the outputs. The syntax here is very simple; it looks identical to an array index in C. But remember these are wires, not variables! The circuit that each line describes is a multiplexer (MUX) with 32 inputs, each 32 bits in size. That's 1024 input wires, 5 selection wires, and 32 output wires. All of a sudden one million transistors doesn't seem that big anymore...

```verilog
   assign rdat1 = registers[rsel1];
   assign rdat2 = registers[rsel2]; 
```
