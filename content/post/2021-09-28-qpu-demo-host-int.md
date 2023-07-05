---
title: 'QPU Demo: Interrupting the host'
date: '2021-09-28'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This is the first in hopefully a few more articles demonstrating one of the
ways in which the RPi's VideoCoreIV (vc4) GPU can be programmed.

The 3D engine, V3D, is driven by the SIMD16 (physically SIMD4, but multiplexed
over four consecutive clock cycles) processors known as QPUs, Quad Processing Units,
or Quad Processors.

In February 2014, Broadcom
[released](https://www.raspberrypi.org/blog/a-birthday-present-from-broadcom/)
a [specification](https://docs.broadcom.com/doc/12358545)
detailing the vc4 V3D architecture. That document is the primary basis
supporting the work demonstrated here.

Note that vc4 is an older version; the most recent RPi, the RPi4
and its variants, ships with VideoCoreVI (vc6) GPU. One of the differences
between vc4 and vc6 is the inclusion of a MMU within the vc6 GPU.

---
### **Hardware Setup:**

Raspberry Pi 1 Model B, [512MB](https://www.raspberrypi.org/blog/model-b-now-ships-with-512mb-of-ram).
The particular [model](https://www.raspberrypi.org/documentation/computers/raspberry-pi.html#:~:text=000f) I happen to have with me has the (old-style)
revision code `000f`. There are other RPi generations/models also that ship
with vc4.

The RPi PL011 UART serves as the main input and output interface, with the
HDMI output connected to a monitor.
### **Software Setup:**

A bare-metal environment, with U-Boot. The ability of U-Boot, to load files
over `TFTP`, is utilized.

The QPU assembly is written, to be assembled using a custom, but incomplete,
[assembler](https://github.com/asurati/qas).

---

### **Demo Setup:**

Before running a QPU program proper, the 3d engine must be powered up. Else,
a read of its registers will likely return an error code, `0xdeadbeef`, and
a write to them may be ignored.

Request the firmware to power up the [V3D power domain](https://www.kernel.org/doc/Documentation/devicetree/bindings/soc/bcm/raspberrypi%2Cbcm2835-power.txt#:~:text=RPI_POWER_DOMAIN_V3D).

Once it has power, read the engine's `V3D_IDENT0` register to verify
that the read returns a valid value advertising the version of the hardware.

The read returns `0x2443356` on RPi1B. In bytes, it is `0x56 0x33 0x44 0x02`
(note that the 32-bit value is treated as little-endian),
or, `V3D2`, after converting the printable ASCII codes to their respective
characters. This version refers to the `v3d 2.x` hardware.

---

### **QPU Program:**

```
# Write a non-zero value into the HOST_INT register, to interrupt the host
# CPU.
ori	host_int, 1, 1;

# NOPs, where the first one signals the end of the program, and the next two
# are the instructions that can safely fill-in the two mandatory delay slots
# following the instruction that signals the end of the program.
pe;
;
;
```

The binary code that the assembler generates is given below. Each instruction
is 64 bits, or two words, long. The first word is the LSW and the second word
is the MSW of the corresponding instruction.

``` text
0x159c1fc0, 0xd00209a7, // ori	host_int, 1, 1;
0x009e7000, 0x300009e7, // pe;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
```

---

The first instruction, `0xd00209a7159c1fc0`, is decoded as:


``` text
 sig| unpack| pm| pack| cond_add| cond_mul| sf| ws| waddr_add| waddr_mul|
1101|    000|  0| 0000|      001|      000|  0|  0|    100110|    100111|

op_mul| op_add| raddr_a| small_immed| add_0| add_1| mul_0| mul_1|
   000|  10101|  100111|      000001|   111|   111|   000|   000|
```

> Note that the `[add|mul]_[a|b]` fields from the specification are
> replaced here by `[add|mul]_[0|1]` to avoid confusing the `a` and `b`
> suffixes with the register files A and B. Those suffixes here represent,
> respectively, the first and the second inputs to each unit.
>
> Contrast that usage of those suffixes with the usage in `raddr_[a|b]`
> fields, where they do indeed represent the register files A and B,
> respectively.

Decoding in more details:

|Field|Value|Comment|
|-----:|-----:|-------|
|`sig`|`1101`| This is an ALU instruction (as opposed to a load-immediate, or a branch, or a semaphore instruction). The value in the `raddr_b` field represents a small immediate value, instead of a register from the register file B.
|`unpack`|`000`| nop.
|`pm`|`0`|Selects between register file A, or r4/MUL unit for pack/unpack operations. Irrelevant here, as both pack and unpack operations are set to nop.
|`pack`|`0000`|	nop.
|`cond_add`|`001`|Conditional execution switch for the ADD unit. The value `1` says `execute always`.
|`cond_mul`|`000`|Conditional execution switch for the MUL unit. The value `0` says `execute never`.
|`sf`|`0`|The instruction should not set any condition flags.
|`ws`|`0`|The ADD and the MUL units write to their default destination register files, A and B, respectively.
|`waddr_a`|`100110`|Address of the register the ADD unit writes its output to (assuming that the conditional execution flags allow the execution). The address is `0x26(=38)`. Since `ws == 0`, the register belongs to the register file A. Thus, the register the ADD unit writes to is `a38`, or the `HOST_INT` IO register.
|`waddr_b`|`100111`|Address of the register the MUL unit writes to. The address is `0x27=39`. Since `ws == 0`, the register belongs to the register file B. Thus, the register the MUL unit writes to is `b39`, a NOP register which likely results in the output being discarded. Note that the `cond_mul` prevents the execution of the MUL unit altogether.
|`op_mul`|`000`|The operation that the MUL unit should perform. The value `0` represents a nop.
|`op_add`|`10101`|The operation that the ADD unit should perform. The value `0x15(=21)` is the binary `or` operation.
|`raddr_a`|`100111`|In case any of the ALU or the MUL units reads from the register file A for their sources/inputs, this field gives the address of the particular register being read. As can be seen later, no unit reads from the register file A, so this address is irrelevant here. The NOP register `a39` is used as a placeholder for no read.
|`small_immed`|`000001`|The embedded immediate `1` represents the integer `1`.
|`add_0`|`111`|The first source/input to the ADD unit. The value `7` asks the unit to read from the register file B field, `raddr_b`. Since the field contains an integer `1`, that integer becomes the first source/input to the ADD unit.
|`add_1`|`111`|The second source/input to the ADD unit. Same value as `add_0`. Reads the embedded integer, `1`. As `op_add = or`, the ADD unit thus calculates `(1 \| 1) = 1` and writes the result into the `HOST_INT` register.
|`mul_0`|`000`|The first source/input to the MUL unit. Reads the accumulator register `r0`. But, as `cond_mul` is set to `execute never`, no read takes place.
|`mul_1`|`000`|The second source/input to the MUL unit. Similar to `mul_0`.

---

The second instruction, `0x300009e7009e7000`, is decoded as:

``` text
 sig| unpack| pm| pack| cond_add| cond_mul| sf| ws| waddr_add| waddr_mul|
0011|    000|  0| 0000|      000|      000|  0|  0|    100111|    100111|

op_mul| op_add| raddr_a| raddr_b| add_0| add_1| mul_0| mul_1|
   000|  10101|  100111|  100111|   000|   000|   000|   000|
```

Since the `cond_add` and `cond_mul` fields are both set to `execute never`, the
instruction is a nop - both units do not execute. The only useful piece of
information is the signal.

`sig:0011`	The value `3` represents the end-of-the-program signal, which
		is required to be present at the end of every QPU program.

---

The third and the fourth instructions are `0x100009e7009e7000`. They are again
nops, with the no-signal signal bits. These nops are instructions that can
safely fill the two delay slots that must follow the instruction that signals
the end of the program.

---
### **Running the demo:**

The driver program can be found [here](https://github.com/asurati/x03/blob/main/demo/d1.c).

The V3D interrupt handler does receive the interrupt raised by the QPU. The
handler is made to print the contents of two registers, `V3D_INTCTL` and
`V3D_DBQITC`. It prints

```
v3dirqh: intctl 4, dbqitc 800
```
The `V3D_DBQITC` register has a single bit set, bit#11. It corresponds to an
interrupt latched from QPU with ID 11. So, *that* QPU, out of all the available
ones, was scheduled to run the program. The program can also read the `b38`
register, named `QPU_NUMBER`, if it wants to know the ID of the QPU it is
running on.

The `V3D_INTCTL` register has Binner-Out-of-Memory set. Not of any
relevance here, it was raised likely because the Binner has not been
provided with sufficient memory (although no Binning/Rendering work has been
scheduled).

---
