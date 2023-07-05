---
title: "QPU Demo: Binary NOT operation"
date: '2021-09-28'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo builds upon
[QPU Demo: DMA Transfers](/wip/post/2021/09/28/qpu-demo-dma-transfers/),
and presents a method of importing four 16-element vectors of integers
from VPM to QPU registers, performing a binary NOT on the elements, and
exporting the results back to the VPM.

---
### **Transfer Methods:**

For QPU to be able to access the vectors and perform any calculations on them,
they must first be brought from the system RAM, where they usually resides,
into the VPM. This was a topic of the second demo.

The transfer of data from the VPM into the QPU registers is VPM-read, or `VPR`.
The reverse transfer, from QPU registers into the VPM is VPM-write, or `VPW`.

The setup register remains the same as in the case of DMA transfers. The
address registers are not involved.

To perform a VPM-read, a set of VPR registers must be configured:

- Initialize the `VPMVCD_RD_SETUP` register with the transfer descriptor.
- Wait for 3 instructions, before reading from the `VPM_READ` register.

A VPM-write can be similarly performed.
- Initialize the `VPMVCD_WR_SETUP` register with the transfer descriptor.
- Write into the `VPM_WRITE` register,

---
### **Transfer Descriptor Formats:**

- For VPM-read, see *Table 33: VPM Generic Block Read Setup Format*.
- For VPM-write, see *Table 32: VPM Generic Block Write Setup Format*.

---

### **VPM-read Transfer Descriptor Format:**

Continuing from the second demo, the vectors are stored vertically within the
last column, starting at the word address `Y=0,X=15`.

Below is the transfer descriptor for reading 4 vectors into QPU registers.

In hexadecimal, the value is `0x41020f`.

```
ID|      -|  NUM|  -| STRIDE| HORIZ| LANED| SIZE|     ADDR|
00| 000000| 0100| 00| 010000|     0|     0|   10| 00001111|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`ID`|`00`| This `VPMVCD_RD_SETUP` configuration is for VPM-to-QPU read, as opposed to RAM-to-VPM read.
|`NUM`|`0100`| Number of vectors to read. Here, four.
|`STRIDE`|`010000`| After each vector is read, the address of the next vector can be had by adding 16 to Y.
|`HORIZ`|`0`| The vectors are stored in a vertical (not horizontal) orientation.
|`LANED`|`0`| Ignored for 32-bit width.
|`SIZE`|`10`| Selects the width of the element, here 32-bit.
|`ADDR`|`00001111`| For the vertical access mode, this 8-bit address is formatted as `ADDR[5:0] = Y[5:4] X[3:0]`. Note that the bits 4 and 5 of the address represent bits 4 and 5 of the Y coordinate, with the bits 0 to 3 of the coordinate assumed to be 0. This forces the coordinate to take on only those values which are multiples of 16.

---
### **VPM-write Transfer Descriptor Format:**

The format is the same as the VPM-read descriptor format, except for the
absence of the `NUM` field.

Below is the transfer descriptor for storing 4 vectors from the QPU registers
into the VPM.
In hexadecimal, the value is `0x1020f`.

```
ID|            -| STRIDE| HORIZ| LANED| SIZE|     ADDR|
00| 000000000000| 010000|     0|     0|   10| 00001111|
```
---
### **QPU Program:**

```
li	vdr_setup, -, 0x8304080f;
ori	vdr_addr, uni_rd, 0;
or	-, vdr_wait, r0;

li	vpr_setup, -, 0x41020f;
;;;	# 3-instruction delay slot before reading.
ori	a0, vpr, 0;
ori	a1, vpr, 0;
ori	a2, vpr, 0;
ori	a3, vpr, 0;

not	a0, a0, a0;
not	a1, a1, a1;
not	a2, a2, a2;
not	a3, a3, a3;

li	vpw_setup, -, 0x1020f;
or	vpw, a0, a0;
or	vpw, a1, a1;
or	vpw, a2, a2;
or	vpw, a3, a3;

# r0 has the setup descriptor.
# r1 has the output address.
# r2 is the counter

li	r0, -, 0x80900078;
ori	r1, uni_rd, 0;
li	r2, -, 0;

loop:
subi	r3, r2, 4	sf;
b.z	done;
;;;
or	vdw_setup, r0, r0;
or	vdw_addr, r1, r1;
or	-, vdw_wait, r0;

li	r3, -, 0x800;
add	r0, r0, r3;	# Add 16 to Y coordinate: bits [13:7].
b	loop;
li	r3, -, 0x40;
add	r1, r1, r3;	# Add 64 to the output address.
addi	r2, r2, 1;	# Increment the counter.

done:
ori	host_int, 1, 1;
pe;;;
```

The binary code.
```
0x8304080f, 0xe0020c67, // li	vdr_setup, -, 0x8304080f;
0x15800dc0, 0xd0020ca7, // ori	vdr_addr, uni_rd, 0;
0x15ca7c00, 0x100209e7, // or	-, vdr_wait, r0;

0x0041020f, 0xe0020c67, // li	vpr_setup, -, 0x41020f;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x15c00dc0, 0xd0020027, // ori	a0, vpr, 0;
0x15c00dc0, 0xd0020067, // ori	a1, vpr, 0;
0x15c00dc0, 0xd00200a7, // ori	a2, vpr, 0;
0x15c00dc0, 0xd00200e7, // ori	a3, vpr, 0;

0x17027d80, 0x10020027, // not	a0, a0, a0;
0x17067d80, 0x10020067, // not	a1, a1, a1;
0x170a7d80, 0x100200a7, // not	a2, a2, a2;
0x170e7d80, 0x100200e7, // not	a3, a3, a3;

0x0001020f, 0xe0021c67, // li	vpw_setup, -, 0x1020f;
0x15027d80, 0x10020c27, // or	vpw, a0, a0;
0x15067d80, 0x10020c27, // or	vpw, a1, a1;
0x150a7d80, 0x10020c27, // or	vpw, a2, a2;
0x150e7d80, 0x10020c27, // or	vpw, a3, a3;

0x80900078, 0xe0020827, // li	r0, -, 0x80900078;
0x15800dc0, 0xd0020867, // ori	r1, uni_rd, 0;
0x00000000, 0xe00208a7, // li	r2, -, 0;

0x0d9c45c0, 0xd00228e7, // loop: subi	r3, r2, 4	sf;
0x00000048, 0xf02809e7, // b.z	done;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x159e7000, 0x10021c67, // or	vdw_setup, r0, r0;
0x159e7240, 0x10021ca7, // or	vdw_addr, r1, r1;
0x159f2e00, 0x100209e7, // or	-, vdw_wait, r0;
0x00000800, 0xe00208e7, // li	r3, -, 0x800;
0x0c9e70c0, 0x10020827, // add	r0, r0, r3;
0xffffff90, 0xf0f809e7, // b	loop;
0x00000040, 0xe00208e7, // li	r3, -, 0x40;
0x0c9e72c0, 0x10020867, // add	r1, r1, r3;
0x0c9c15c0, 0xd00208a7, // addi	r2, r2, 1;

0x159c1fc0, 0xd00209a7, // done: ori	host_int, 1, 1;
0x009e7000, 0x300009e7, // pe;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
```

---
### **Running the demo:**

The driver program can be found
[here](https://github.com/asurati/x03/blob/main/demo/d3.c).

A portion of a sample output. Pairs of input and the output elements were
ORed together to ascertain that the result is indeed `0xffffffff`.

```
v3dirqh: intctl 4, dbqitc 800

d3: [0]: dda83415, 2257cbea, ffffffff
d3: [1]: 3dbac32a, c2453cd5, ffffffff
d3: [2]: cc78151b, 3387eae4, ffffffff
. . .
d3: [f]: 5c4bedfc, a3b41203, ffffffff


d3: [10]: 7bbf4c85, 8440b37a, ffffffff
d3: [11]: eda24ada, 125db525, ffffffff
d3: [12]: df847b0b, 207b84f4, ffffffff
. . .
d3: [1f]: bd64ce2c, 429b31d3, ffffffff


d3: [20]: fcc560f5, 33a9f0a, ffffffff
d3: [21]: f6131e8a, 9ece175, ffffffff
d3: [22]: 422e3cfb, bdd1c304, ffffffff
. . .
d3: [2f]: 6c4e1a5c, 93b1e5a3, ffffffff


d3: [30]: a8717165, 578e8e9a, ffffffff
d3: [31]: a1f83e3a, 5e07c1c5, ffffffff
d3: [32]: 74845aeb, 8b7ba514, ffffffff
. . .
d3: [3f]: a3fad28c, 5c052d73, ffffffff
```

---
