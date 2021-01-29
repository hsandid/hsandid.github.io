---
title: "Adding Custom Instructions to the RISC-V GNU-GCC toolchain"
date: 2021-01-27T18:36:23+02:00
draft: false
menu:
  sidebar:
    name: "Adding Custom Instructions to the RISC-V GNU-GCC toolchain"
    identifier: adding-instructions-riscv-gnu-gcc
    weight: 210
---

- This tutorial is an updated version of Nitish Srivastava's tutorial [Adding custom instruction to RISCV ISA and running it on gem5 and spike](https://nitish2112.github.io/post/adding-instruction-riscv/).
- I've also used some tips Jim Wilson gave on the following [mailing list](https://groups.google.com/a/groups.riscv.org/g/sw-dev/c/sL_OHXYj3LY/m/Gsm6sBc9BQAJ).

### Installing the RISC-V GNU/GCC toolchain

We will be building the RISC-V GNU/GCC RV64GC Newlib Cross-compiler on an Ubuntu machine. If you'd like to use another configuration; you can check the repository instructions [here](https://github.com/riscv/riscv-gnu-toolchain).

- Clone the RISC-V GNU/GCC toolchain, along with its submodules (7GB download) :

```
$ git clone https://github.com/riscv/riscv-gnu-toolchain.git
$ git submodule update --init --recursive
```

- Install the pre-requisites for the RISC-V GNU/GCC toolchain (Ubuntu) :

``` 
$ sudo apt-get install autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
```

- Build the RISC-V GNU/GCC Newlib Cross-compiler :

```
./configure --prefix=<installation_path>
make
```

### Cloning the RISC-V Opcodes Tool

- Clone the RISC-V Opcodes Tool :

```
$ git clone https://github.com/riscv/riscv-opcodes
```

### Defining our Instruction

- We'd like to add a "modulo" instruction to the ISA. The instruction and its semantics are given below :

```
mod r1, r2, r3

Semantics:
R[r1] = R[r2] % R[r3]
```

- In the root folder of the RISC-V Opcodes Tool, you can see different files, each pertaining to a set of RISC-V opcodes. 

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-6.jpeg)

- Add your "modulo" instruction to one of the opcodes-xxx files available in your repository :

```
mod     rd rs1 rs2 31..25=1  14..12=0 6..2=0x1A 1..0=3
```

- Run the following command to generate the MATCH and MASK hexadecimal values for your instruction :

```
cat <opcodes-file-you-modified> | ./parse-opcodes -c > instructionInfo.h
```

- Open the instructionInfo.h file, and you will find the following lines :

```
#define MATCH_MOD 0x200006b                                                    
#define MASK_MOD 0xfe00707f
...
DECLARE_INSN(mod, MATCH_MOD, MASK_MOD)
```

The MASK hexadecimal value is also a 32-bit hex, which puts a 'mask' telling where all fields that are associated with your instruction's type. For example, ```mod``` is an R-type instruction with the following hexadecimal  mask : ```0xfe00707f```

The MATCH hexadecimal value is basically a 32-bit hex where the value of your opcodes/funct3/funct7 are set to define your instruction.  For example; ```mod``` has a hexadecimal match of ```0x200006b```; which means that in binary we have```funct7 = 0b0010000```; ```funct3 =  0b000``` and ```opcode = 0b1101011```.

You can use the RISC-V green card to check the MASK and MATCH fields are correct by determining their hex value by-hand. The RISC-V green card can be found [here](https://inst.eecs.berkeley.edu//~cs61c/fa17/img/riscvcard.pdf).

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-7.jpeg)

### Modifying the RISC-V GNU/GCC Binutils

- Since modifying the GCC compiler itself would prove too complex; we will modify the RISC-V GNU/GCC Binutils to add an additional instruction.
- First, we look into riscv-gnu-toolchain/riscv-binutils-gdb/include/opcode/riscv-opc.h; where we will add the ```#define``` and ```DECLARE_INSN()``` elements we generated for our instruction in the appropriate blocks :

Adding the ```#define``` :

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-1.png)

Adding the ```DECLARE_INSN()``` :

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-2.png)

Make sure to plug your modifications are the right place, or you WILL break the assembler !

- Second, we look into the riscv-gnu-toolchain/riscv-binutils-gdb/opcodes/riscv-opc.c file

We need to declare our instruction using a specific set of parameters :

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-3.png)

*name* : self-explanatory, just give your desired instruction's name

*xlen*: specify the width of an integer register in bits (0,32,64)

*ISA* : declare the type of instruction used. You can find more details on the different instruction directives in the same file if you scroll down

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-4.png)

*operands* : The function handling operands is available in riscv-binutils/gas/config/tc-riscv.c. Check it out to check which characters must be used to represent your operands.

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-8.jpeg)

This is the parsing function riscv_ip()'s main switch statement which handles the different operands. It's located somewhere on Line 2000. Once you get the char values associated with your operands of choice, you plug them in the instruction definition here

*match* :  Use the MATCH value declared before for our custom instruction.

*mask* : Use the MASK value declared before for our custom instruction.

*match_opcode* : Pointer to the function you'd like to use to 'detect' which opcode you are using. Haven't experimented with it yet (*Edit : This definition might be incorrect*)

*pinfo* : Used to specify some special behavior. Mainly used with branch instructions and compressed RISC-V instructions. It's set to 0 if there is no special behavior

### Re-building & Testing our custom instruction

- Rebuild the RISC-V GNU/GCC Compiler Toolchain, as we did in the first steps, and you should now be able to use your custom instructions in your programs. To be able to invoke your custom instruction, you should use inline assembly. 
- Here is an example of C code using inline assembly to invoke a custom instruction:

![Risc-v custom](/images/posts/risc-v-custom/risc-v-custom-5.jpeg)

Finally, RISC-V instructions are assembled into machine code. You can use 'obj-dump' to check the generated RISC-V binaries, and check where your custom instruction is invoked.

- We have not implemented any logic related to the instruction in here. Logic needs to be implemented on the simulator side, meaning that if you'd like your custom instruction to work with SPIKE or gem5; you'd have to modify them and add the logic associated with your custom instruction manually.