---
title: "QPU Demo: DMA Transfers"
date: '2021-09-28'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo presents a method of programming the V3D DMA controller to
transfer four 16-element vectors of integers back and forth between the system
RAM and the VPM.

It is natural for the QPU to work with 16-element vectors. To that end, its
registers can accommodate 16 32-bit values (there are indeed a few exceptions,
however).

In AMD terminology, a QPU can be said to process a wavefront made up of 16
workitems.

---
### **VPM:**

See *Section 7: VPM and VCD* of the specification, for more details on VPM.

The `V3D_IDENT1` register exposes the total size of VPM available on the
system. On the RPi1B, the size is 12KB.

The VPM supports both graphics-related accesses and general-purpose accesses,
even simultaneously provided that it is large enough (the 3D pipeline needs
a minimum of 8KB of VPM).

But, for performing general-purpose accesses specifically, an area of VPM,
beginning at its start, must be reserved. The `V3D_VPMBASE` register allows
one to configure the size of that reserved area. The default size configured is
0, for no reservation at all.

For the demo, the reservation is set to 4KB. Treating that reserved area of
VPM as a 2D array of 32-bit words, the size corresponds exactly to a dimension
of 64 rows, each 16 words wide. The interfaces, through which the VPM is
read from and written to, enforce a 2D structure on it. That access
mechanism is very similar to accessing double-dimensional arrays in languages
like C.

---
### **Transfer Methods:**

The transfer of data from the system RAM into the VPM is known as the
DMA-read of the data - *DMA* because the external (to the GPU) system RAM is
involved, and *read* because the system RAM is being read from.

The reverse transfer is known as the DMA-write of the data -
*DMA* because, again, the external system RAM is involved, and *write*
because the system RAM is being written to. The direction of a DMA transfer
is determined based on the point of view of the system RAM.

DMA-reads can also be referred to as *VDR*, for Vertex DMA Read;
similarly, *VDW*, or Vertex DMA Write, for DMA-writes.

To perform a DMA-read, a set of VDR registers must be configured:

- Initialize the `VPMVCD_RD_SETUP` register with the transfer descriptor.
- Write into the `VPM_LD_ADDR` register, the bus address of the location in
the system RAM to read from.
- Either stall, waiting for the transfer to complete, by reading once from the
`VPM_LD_WAIT` register, or poll the `VPM_LD_BUSY` register for a zero value to
appear.

Similarly, a DMA-write can be performed by setting up a few VDW registers:
- Initialize the `VPMVCD_WR_SETUP` register with the transfer descriptor.
- Write, into the `VPM_ST_ADDR` register, the bus address of the location in
the system RAM to write to.
- Either stall, waiting for the transfer to complete, by reading once from the
`VPM_ST_WAIT` register, or poll the `VPM_ST_BUSY` register for a zero value to
appear.

---
### **Transfer Descriptor Formats:**

- For DMA-read, see *Table 36: VCD DMA Load (VDR) Basic Setup Format*.
- For DMA-write, see *Table 34: VCD DMA Store (VDW) Basic Setup Format*.

---

### **DMA-read Transfer Descriptor Format:**

The vectors are stored vertically within the last column, starting at the word
address `Y=0,X=15`. Each column is 64 words high, so these vectors, which
themselves contain a total of 64 words, completely fill it.

The elements of each vector are stored vertically, and the vectors themselves
are one on top of the other.

Below is the transfer descriptor, to read four 16-element vectors of
32-bit integers from the system RAM, and to store them into VPM in vertical
access mode (See *Figure 9: VPM Vertical Access Mode Examples* for its layout).


In hexadecimal, the value is `0x8304080f`.

```
ID| MODEW| MPITCH| ROWLEN| NROWS| VPITCH| VERT|      ADDRXY|
 1|   000|   0011|   0000|  0100|   0000|    1| 00000001111|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`ID`|`1`| The `VPMVCD_RD_SETUP` can also help read data from VPM into a QPU register. This bit distinguishes between the setup of a RAM-to-VPM read, and that of a VPM-to-QPU read.
|`MODEW`|`000`| Selects the element width as 32 bits.
|`MPITCH`|`0011`| Each vector is present in the system RAM side-by-side, consecutively. If each vector is considered as a row, the row-to-row pitch/stride is 64 bytes. The corresponding value to fill here is `3`.
|`ROWLEN`|`0000`| This field provides the size (in units of the width selected within `MODEW`) of the row to access. The row-length can be smaller than the stride/`MPITCH` to accommodate reading only the first few needed words of the row, and skipping the rest, before moving on to the next row. The value `0` represents a length of `16` words, each of a size as programmed in `MODEW`, 32-bit here.
|`NROWS`|`0100`| The number of rows of data in system RAM. Four in this case.
|`VPITCH`|`0000`| Similar to `MPITCH`, this field helps the DMA controller move to the next row inside the VPM to begin storing into it the next row. This value is added to the Y coordinate of the address. Since this demo accesses the VPM in a vertical mode, the appropriate increment is `16`. Correspondingly, the value programmed is `0`.
|`VERT`|`1`|The access mode. One can visualize the layout of the storage as <br>`vpm[0][15] = vec[0][0];` <br>`vpm[1][15] = vec[0][1];` <br>`vpm[2][15] = vec[0][2];` <br>... <br>`vpm[63][15] = vec[3][15];` <br>Each VPM location here is accessed as `vpm[Y=#][X=#]`.
|`ADDRXY`|`00000001111`| The start address, to being writing into the VPM at, is `Y=0,X=15`. The *Figure 9* in the specification shows that this address corresponds to the rightmost column.

---

### **DMA-write Transfer Descriptor Format:**

Although it is possible to DMA-read multiple 16-element vectors and store them
vertically with a *single* configuration of `VPMVCD_RD_SETUP`, it is not
possible to DMA-write multiple, 16-element, vertically-arranged vectors with
a *single* configuration of `VPMVCD_WR_SETUP`.

If such a DMA-write is attempted, say for exporting four vertically-stored
vectors at `Y=0,X=15`, `Y=16,X=15`, `Y=32,X=15`, and `Y=48,X=15`, the DMA
engine writes vectors residing at `Y=0,X=15`, `Y=16,X=0`, `Y=16,X=1`, and
`Y=16,X=2` instead. That is, the DMA engine moves horizontally to the right,
vector after vector, and down-to-up if the X coordinate overflows.

Because of this problem, the `VPMVCD_WR_SETUP` must be separately configured
for transferring each individual vector. The only changes that must be made in
each successive transfer descriptor are the increment of the Y coordinate of
the address to read the vector from, and the increment of the output address.

Below is the transfer descriptor for transferring the first vector.
In hexadecimal, the value is `0x80900078`.

```
|ID |UNITS   |DEPTH   |LANED |HORIZ |VPMBASE     |MODEW
|10 |0000001 |0010000 |0     |0     |00000001111 |000
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`ID`|`10`| Similar to `VPMVCD_RD_SETUP`. The bit distinguishes between the setup of a VPM-to-RAM write, and that of a QPU-to-VPM write.
|`UNITS`|`0000001`| Similar to `NROWS` from `VPMVCD_RD_SETUP`.
|`DEPTH`|`0010000`| Similar to `ROWLEN` from `VPMVCD_RD_SETUP`. The value is in the units of the width selected in the `MODEW` field below.
|`LANED`|`0`| Must be 0.
|`HORIZ`|`0`| Since each vector lies vertically, the DMA controller must be told about the vertical orientation.
|`VPMBASE`|`00000001111`| Similar to `ADDRXY` from `VPMVCD_RD_SETUP`. The start address inside the VPM where the vectors reside; `Y=0,X=15`.
|`MODEW`|`000`| Selects the element width as 32-bit.


After each vector is transferred, the `Y` coordinate must be incremented by
`16`. This amounts to adding `0x800` after every transfer. The vectors are to be
stored into the output buffer in consecutive addresses; that requires
incrementing the output address by `16 * 4 = 64`.

---
### **Uniforms:**

The addresses of the input and output buffers are passed to the QPU program
through a facility known as the
[uniforms](https://www.khronos.org/opengl/wiki/Uniform_(GLSL)). It is usually
utilized to pass such global pieces of information to the QPU; it is a
read-only facility from the perspective of the QPU.

The two buffer addresses are stored inside an array, and the address and the
size of the array are written into `V3D_SRQUA` and `V3D_SRQUL` registers,
respectively.

The QPU program reads the `a32/b32` register, also named `UNIFORM_READ`,to gain
access to the buffer addresses.

---
### **QPU Program:**

```
li	vdr_setup, -, 0x8304080f;
ori	vdr_addr, uni_rd, 0;
or	-, vdr_wait, r0;

# r0 has the setup descriptor.
# r1 has the output address.
# r2 is the counter

li	r0, -, 0x80900078;
ori	r1, uni_rd, 0;
li	r2, -, 0;

loop:
subi	r3, r2, 4	sf;
b.z	done;	# There's 3-instruction delay slot for a branch.
;;;		# 3 NOPs to fill the slot.
or	vdw_setup, r0, r0;
or	vdw_addr, r1, r1;
or	-, vdw_wait, r0;

li	r3, -, 0x800;
add	r0, r0, r3;	# Add 16 to Y coordinate: bits [13:7].
b	loop;	# The following 3 instructions are executed as part of the branch
		# delay slot.
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
[here](https://github.com/asurati/x03/blob/main/demo/d2.c).

A portion of a sample output:

```
v3dirqh: intctl 4, dbqitc 800

d2: [0]: e64fec73, e64fec73
d2: [1]: 7e04e730, 7e04e730
d2: [2]: 25a73fa9, 25a73fa9
. . .
d2: [f]: 8636e692, 8636e692


d2: [10]: 468cd863, 468cd863
d2: [11]: 73787c60, 73787c60
d2: [12]: 72726519, 72726519
. . .
d2: [1f]: 2421842, 2421842


d2: [20]: 9e94a053, 9e94a053
d2: [21]: e153bd90, e153bd90
d2: [22]: a1c9c689, a1c9c689
. . .
d2: [2f]: 41515f2, 41515f2


d2: [30]: 94d64443, 94d64443
d2: [31]: 76d9aac0, 76d9aac0
d2: [32]: bd3463f9, bd3463f9
. . .
d2: [3f]: e33adfa2, e33adfa2
```
