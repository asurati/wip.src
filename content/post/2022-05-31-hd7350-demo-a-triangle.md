---
title: "HD 7350 demo: A Triangle"
date: '2022-05-31'
categories:
  - Programming
tags:
  - Radeon
  - AMD
  - HD7350
  - CEDAR
  - Display
  - GPU
---

*Warning: Improperly programming a GPU, or a display, may damage the devices.*

This post demonstrates setting up a HD7350 pipeline to draw a triangle.
The approach adopted here is that of choosing bare-metal programming over
relying on established interfaces like the Linux DRM.
That is not to say that Linux DRM
cannot be used - it is possible to send compute or rendering commands to the
GPU by directly interfacing with the DRM layer on Linux, or by moving up to an
even higher abstraction provided by OpenGL/Vulkan APIs.

The bare-metal program maps the PCIe BARs, POSTs the Video BIOS, configures and
tests the GPU rings (GFX, DMA, INT), parses the EDID blocks of the attached
display, and configures the GPU HW blocks (CRTCs, ENCODERs, TRANSMITTERs)
for running the display at the preferred resolution. The Linux `radeon` driver
and the AMD manuals provide the details on the steps required.

It then prepares a command buffer for a draw call, linking into it
various resources such as the Color Buffer (i.e. the Frame Buffer the GPU is
scanning out), the Vertex Buffer, and the buffers containing the Shader
programs. Finally, it sends the command buffer for execution to the GPU's
Graphics (GFX) ring.

---
### **Setup:**

The [HD7350](https://en.wikipedia.org/wiki/Ati_gpu#Radeon_7000_series) is a
member, named Cedar, of the VLIW-5 TeraScale 2 Evergreen family of the AMD GPUs.
Since its release, the AMD GPU microarchitectures have progressed, to RDNA 3 at
present, through TeraScale 3, GCN 1-5, RDNA and RDNA 2. The card is indeed old.

It has a single [DMS-59](https://en.wikipedia.org/wiki/DMS-59) port. A
DMS-59-to-dual-DVI break-out cable exposes the more prevelant DVI outputs. A
DVI-to-HDMI cable connects one of those DVI outputs to a cheap USB-based HDMI
Video Capture device. The video-capture device appears to the GPU as a display.

The GPU is passed through to a VM running on the host machine. The host machine
runs the [OBS Studio](https://obsproject.com/) software to capture and display
the GPU output.

---

### **MESA and the Manuals:**

The manuals released by AMD contain information on the architecture of the GPU,
the register and buffer formats, and the shader ISA. The Gallium `r600` driver
within MESA provides the preferred/correct sequence of commands.

These are the manuals that were helpful:

* [Evergreen Family Instruction Set Architecture: Instructions and
Microcode](https://developer.amd.com/wordpress/media/2012/10/AMD_Evergreen-Family_Instruction_Set_Architecture.pdf)
* [Radeon Evergreen/Northern Islands Acceleration](http://developer.amd.com/wordpress/media/2013/10/evergreen_cayman_programming_guide.pdf)
* [Radeon Evergreen 3D Register Reference Guide](http://developer.amd.com/wordpress/media/2013/10/evergreen_3D_registers_v2.pdf)
* [Radeon R5xx Acceleration](http://developer.amd.com/wordpress/media/2013/10/R5xx_Acceleration_v1.5.pdf)
* [R600/R700/Evergreen Assembly Language Format](https://developer.amd.com/wordpress/media/2012/10/R600-R700-Evergreen_Assembly_Language_Format.pdf)
* [Radeon R6xx/R7xx Acceleration](http://developer.amd.com/wordpress/media/2013/10/R6xx_R7xx_3D.pdf)
* [Radeon Sea Islands 3D/Compute Register Reference Guide](CIK_3D_registers_v2.pdf)
* [Radeon Southern Islands 3D/Compute Register Reference Guide](SI_3D_registers.pdf)

---

### **Shaders:**

The shaders, required by a demo as simple as this one, are only a few:
* **VS** (Vertex Shader)
* **FS** (Fetch Shader)
* **PS** (Pixel Shader)

---

### **Vertex Shader:**

The vertices, provided as an input to the GPU, are already in the Normalized
Device Coordinate space. As a result, it suffices for the VS to be a
pass-through shader.

It relies on the FS to pull the vertex properties from the Vertex Buffer. Each
vertex has two properties, a POSITION, and a COLOR parameter. It then exports
them to appropriate buffers - POSITION into the Position Buffer, and the COLOR
parameter into the Parameter Cache.

In order to take advantage of the GPU's support for semantic independence (a
facility to reduce dependencies) between the VS and the FS, the FS utilizes a
semantic table to store the properties into registers designated by the table.

With the help of such a facility, the FS does not need to know the exact
registers in which the VS expects to receive the vertex properties, and the VS
can easily change its register allocation patterns without changing the FS
code.

The binary code:

```
00 00 00 00 00 04 c0 84  3c a0 00 00 88 06 00 95
00 40 01 00 88 06 20 95

PC| Offset|     Instruction Words
--+-------+----------------------
 0|   0x00| 0x00000000 0x84c00400
 1|   0x08| 0x0000a03c 0x95000688
 2|   0x10| 0x00014000 0x95200688
```

PC is in the units of 64 bits, relative to the start of the respective
shader. The instructions in the shader ISA are either 64 or 128 bits in length.
Each instruction must fall on a naturally aligned address in the memory. The
starting address of a shader must be aligned on a 256-byte boundary.

All shaders begin their execution with an instruction of type
*Control Flow (CF)*.

---

#### **VS[0]:**

```
PC| Offset|   CF_WORD0|   CF_WORD1|
--+-------+-----------+-----------+
 0|   0x00| 0x00000000| 0x84c00400|

   res| jts|                          addr|
0 0000| 000| 0000 0000 0000 0000 0000 0000|

b| wqm|      inst| eop| vpm| rsvd|   count| cond|  const| pop|
1|   0| 0001 0011|   0|   0| 0000| 00 0001|   00| 0 0000| 000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`addr`|`0`| The `inst` field shows that this is a `CF_INST_CALL_FS` instruction. Hence, this `addr` field is a PC, relative to the start of the FS, that the GPU should call. The GPU pushes the return address (the address of the instruction next to this one) onto a stack; the FS executes `CF_INST_RETURN` when it wants to return.
|`count`|`1`| The increment to apply to Call Nesting Counter. Used to avoid reaching a call-nesting of unsupported depths.
|`inst`|`0x13`| `CF_INST_CALL_FS`.
|`b`|`1`| Barrier. Wait for other CF instructions to complete before beginning this one.

---

#### **VS[1]:**

The `inst` field of this CF instruction is `0x54`, which stands for
`CF_INST_EXPORT_DONE`. This information suggests that the current instruction
is not only a CF instruction like the one above, but it is a sub-type of the
CF instructions, particularly the
*"CF Allocate, Import, or Export of the Swizzle kind"* instructions. The
format should, therefore, be appropriately decoded.

```
PC| Offset| CF_ALLOC_EXPORT_WORD0| CF_ALLOC_EXPORT_WORD1_SWIZ|
--+-------+----------------------+---------------------------+
 1|   0x08|            0x0000a03c|                 0x95000688|

es|   ix_gpr| rw_rel|   rw_gpr| type|       array_base|
00| 000 0000|      0| 000 0001|   01| 0 0000 0011 1100|

b| m|      inst| eop| vpm| count| rsvd| sel_w| sel_z| sel_y| sel_x|
1| 0| 0101 0100|   0|   0|  0000| 0000|   011|   010|   001|   000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`array_base`|`0x3c`| Position Index. Must be `0x3c == 60` when exporting the POSITION property (see `type` below) of a vertex.
|`type`|`1`| `EXPORT_POS`.
|`rw_gpr`|`1`| The register#, `R1`, that contains the POSITION, and acts as the source register for the export. As described later, the semantic table guides the FS into placing the POSITION property of a vertex into the `R1` register.
|`sel_x`|`0`| The X (`sel_x`) channel of the XYZW vector being exported must be sourced from the channel `0` of the source register, i.e. `R1.X`.
|`sel_y`|`1`| The Y (`sel_y`) channel of the XYZW vector being exported must be sourced from the channel `1` of the source register, i.e. `R1.Y`.
|`sel_z`|`2`| . . .
|`sel_w`|`3`| . . .
|`count`|`0`| The number of POSITIONs exported, minus one.
|`inst`|`0x54`| `CF_INST_EXPORT_DONE`. The `DONE` suffix suggests that this is the last (and, in this case, the only one) of all the POSITIONs being exported.

---

#### **VS[2]:**

Similar to the one directly above, but exports one PARAMETER: the COLOR
parameter.

```
PC| Offset| CF_ALLOC_EXPORT_WORD0| CF_ALLOC_EXPORT_WORD1_SWIZ|
--+-------+----------------------+---------------------------+
 1|   0x08|            0x00014000|                 0x95200688|

es|   ix_gpr| rw_rel|   rw_gpr| type|       array_base|
00| 000 0000|      0| 000 0010|   10| 0 0000 0000 0000|

b| m|      inst| eop| vpm| count| rsvd| sel_w| sel_z| sel_y| sel_x|
1| 0| 0101 0100|   1|   0|  0000| 0000|   011|   010|   001|   000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`array_base`|`0`| Parameter Index. Must be 0 for the first (and, in this case, the only) PARAMETER being exported. The strict sequence (0, 1, 2, etc..) seems to be necessary if the PS relies on *Direct Parameter Reads* that support hardware-assisted addressing of the Parameter Cache.
|`type`|`2`| `EXPORT_PARAM`.
|`rw_gpr`|`2`| The register#, `R2`, that contains the COLOR parameter, and acts as the source register for the export. As described later, the semantic table guides the FS into placing the COLOR parameter of a vertex into the `R2` register.
|`count`|`0`| The number of PARAMETERs exported, minus one.
|`eop`|`1`| End Of this Program.
|`inst`|`0x54`| `CF_INST_EXPORT_DONE`. The `DONE` suffix suggests that this is the last (and, in this case, the only one) of all the PARAMETERs being exported.

---

### **Fetch Shader:**

As can be seen below, the FS consists of two types of instructions: the CF
instructions that begin the FS, which are 64-bit instructions, and the VFC
(Vertex Fetch Clause) instructions, which are 128-bit instructions. The PC
still remains displayed in 64-bit units, however. As before, the PC and the
Offset both are relative to the start of the FS.

```
02 00 00 00 00 04 80 80  00 00 00 00 00 00 00 85
01 1f 00 30 90 10 15 4c  00 00 00 00 00 00 00 00
01 1f 00 30 92 10 15 4c  0c 00 00 00 00 00 00 00

PC| Offset|     Instruction Words
--+-------+--------------------------------------------
 0|   0x00| 0x00000002 0x80800400
 1|   0x08| 0x00000000 0x85000000
 2|   0x10| 0x30001f01 0x4c151090 0x00000000 0x00000000
 4|   0x20| 0x30001f01 0x4c151092 0x0000000c 0x00000000
```

---

#### **FS[0]:**

```
PC| Offset|   CF_WORD0|   CF_WORD1|
--+-------+-----------+-----------+
 0|   0x00| 0x00000002| 0x80800400|

   res| jts|                          addr|
0 0000| 000| 0000 0000 0000 0000 0000 0010|

b| wqm|      inst| eop| vpm| rsvd|   count| cond|  const| pop|
1|   0| 0000 0010|   0|   0| 0000| 00 0001|   00| 0 0000| 000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`addr`|`0`| The `inst` field shows that this is a `CF_INST_VC` instruction. Hence, this `addr` field is a PC, relative to the start of this shader, where begins a Vertex Fetch Clause.
|`count`|`1`| The number of instructions in the clause, minus one. There are 2 instructions in the VFC clause that follows. Note that this count hides the fact that the clause instructions are 128-bit each, while the PC, by being even and by being incremented twice when running the clause, does not. This instruction is can be thought of as a stack-less call; the GPU jumps to the clause, executes `count` number of clause instructions, and then returns back to the instruction next to this.
|`inst`|`2`| `CF_INST_VC`.

---

#### **FS[1]:**

```
PC| Offset|   CF_WORD0|   CF_WORD1|
--+-------+-----------+-----------+
 0|   0x00| 0x00000000| 0x85000000|

   res| jts|                          addr|
0 0000| 000| 0000 0000 0000 0000 0000 0000|

b| wqm|      inst| eop| vpm| rsvd|   count| cond|  const| pop|
1|   0| 0001 0100|   0|   0| 0000| 00 0000|   00| 0 0000| 000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`inst`|`0x14`| `CF_INST_RETURN`. Return back to the VS.

---

#### **FS[2], VFC[0]:**

This is a Vertex Fetch Clause instruction, particulary a fetch instruction
depending on the FS-VS semantic table to determine the register that is to
receive the fetched information.

```
PC| Offset|  VTX_WORD0|  VTX_WORD1_SEM|  VTX_WORD2|     ZEROES|
--+-------+-----------+---------------+-----------+-----------+
 2|   0x10| 0x30001f01|     0x4c151090| 0x00000000| 0x00000000|

    mfc| ssx| sr|  src_gpr|    buf_id| fwq| ft|   inst|
00 1100|  00|  0| 000 0000| 0001 1111|   0| 00| 0 0001|

sma| fca| nfa|     fmt| ucf| dsw| dwz| dwy| dwx| rsvd|    sem_id|
  0|   1|  00| 11 0000|   0| 101| 010| 001| 000|    0| 1001 0000|

       rsvd| bim| alt_const| mf| cbns| es|              offset|
0 0000 0000|  00|         0|  0|    0| 00| 0000 0000 0000 0000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`inst`|`1`| `VC_FETCH_SEMANTIC`. Fetch vertex property, relying on the FS-VS semantic table to direct the fetched property into appropriate register.
|`ft`|`0`| `VTX_FETCH_VERTEX_DATA`. Pass the vertexID (and not the instanceID) to identify the entity for which the fetch is being performed.
|`buf_id`|`0x1f`| Index into the MMIO Resource/Constant Descriptor Table. Each entry in the table is 32-bytes, and provides information on the address and size of the buffer from which to fetch.
|`src_gpr`|`0`| The vertexID is found in `R0`. The Shader Pipe Interpolator (SPI) places vertexID into `R0.X` before calling the VS.
|`ssx`|'0'| The vertexID is found in the X channel of the Source GPR.
|`sem_id`|`0x90`| Programmer-defined semanticID that identifies the vertex property being fetched and also the destination GPR that FS should write into. In this demo, it identifies the POSITION property, and the `R1` register. See the contents of the FS-VS semantic table in the command buffer description, later.
|`dsx`|`0`| Destination GPR's X (`dsx`) channel should be filled with the fetched vector's 0 (`X`)
|`dsy`|`1`| Destination GPR's Y (`dsy`) channel should be filled with the fetched vector's 1 (`Y`)
|`dsz`|`2`| Destination GPR's Z (`dsz`) channel should be filled with the fetched vector's 2 (`Z`)
|`dsw`|`5`| Destination GPR's W (`dsw`) channel should be filled with `1.0`. This is because we only define XYZ co-ordinates for each vertex in the Vertex Buffer, so the `W` channel is absent in the buffer. Moreover, since we disable Clipping, the co-ordinates are taken to be already in the Normalized Device Coordinate space. This channel seems to be ignored under the sparse settings established by this demo. Nevertheless, set it as if the Perspective Divide has been already performed.
|`fmt`|`0x30`| `FMT_32_32_32_FLOAT`. The XYZ channels of a vertex position is in 4-byte floats.
|`nfa`|`0`| `NUM_FORMAT_NORM`. A fraction between [0,1] if unsigned, or between [-1,1] if signed. Note that the demo passes NDCs to the GPU.
|`fca`|`1`| `FORMAT_COMP_SIGNED`. Signed numbers. See also `nfa`.
|`offset`|`0`| Byte offset into the Vertex Buffer, from where to begin fetching this vertex property.

---

#### **FS[4], VFC[1]:**

Similar to above, but fetches COLOR parameter.

```
PC| Offset|  VTX_WORD0|  VTX_WORD1_SEM|  VTX_WORD2|     ZEROES|
--+-------+-----------+---------------+-----------+-----------+
 2|   0x10| 0x30001f01|     0x4c151092| 0x0000000c| 0x00000000|

    mfc| ssx| sr|  src_gpr|    buf_id| fwq| ft|   inst|
00 1100|  00|  0| 000 0000| 0001 1111|   0| 00| 0 0001|

sma| fca| nfa|     fmt| ucf| dsw| dwz| dwy| dwx| rsvd|    sem_id|
  0|   1|  00| 11 0000|   0| 101| 010| 001| 000|    0| 1001 0010|

       rsvd| bim| alt_const| mf| cbns| es|              offset|
0 0000 0000|  00|         0|  0|    0| 00| 0000 0000 0000 1100|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`sem_id`|`0x92`| Programmer-defined semanticID that identifies the vertex property being fetched and also the destination GPR that FS should write into. In this demo, it identifies the COLOR parameter, and the `R2` register. See the contents of the FS-VS semantic table in the command buffer description, later.
|`dsw`|`5`| Destination GPR's W (`dsw`) channel should be filled with `1.0`. This is because we only define RGB values for the COLOR parameter of each vertex in the Vertex Buffer, so the `W`, i.e. the Alpha, channel is absent in the buffer. Set it to 1.0 (completely opaque).
|`offset`|`0xc`| Byte offset into the Vertex Buffer, from where to begin fetching this vertex property. The previous property was POSITION, and was of size `4 * 3 = 12` bytes. The COLOR property immediately follows it, for each vertex.
