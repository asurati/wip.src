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

*[Update #1](#update1) on: 2nd June, 2022.*

*[Update #2](#update2) on: 5nd June, 2022.*

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

It has a single [DMS-59](https://en.wikipedia.org/wiki/DMS-59) port. A
DMS-59-to-dual-DVI break-out cable exposes the more prevalent DVI outputs. A
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
* [Radeon R6xx/R7xx 3D Register Reference Guide](https://developer.amd.com/wordpress/media/2013/10/R6xx_3D_Registers.pdf)
* [Radeon Sea Islands 3D/Compute Register Reference Guide](http://developer.amd.com/wordpress/media/2013/10/CIK_3D_registers_v2.pdf)
* [Radeon Southern Islands 3D/Compute Register Reference Guide](https://developer.amd.com/wordpress/media/2013/10/SI_3D_registers.pdf)

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
 2|   0x10|            0x00014000|                 0x95200688|

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
 1|   0x08| 0x00000000| 0x85000000|

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

This is a Vertex Fetch Clause instruction, particularly a fetch instruction
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

Similar to above, but fetches the COLOR parameter.

```
PC| Offset|  VTX_WORD0|  VTX_WORD1_SEM|  VTX_WORD2|     ZEROES|
--+-------+-----------+---------------+-----------+-----------+
 4|   0x20| 0x30001f01|     0x4c151092| 0x0000000c| 0x00000000|

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

---

### **Pixel Shader:**

Just as there is a support for semantic independence between the FS and the VS,
there is a similar support provided between the VS and the FS. But, the
PARAMETER exports from the VS do not go into GPRs - they go to the Parameter
Cache. The layout of the exported parameters are available in the documents.

It seems that the Parameter Cache fills itself starting at its base; so any
semantic mapping which does not start parameter allocations at the base of the
cache, but which nudges the automatic hardware addressing facility to try and
attempt 'random accesses' into the cache, fail. As a result, this demo keeps
the semantic mapping simple: VS export of the COLOR parameter is associated
with the 0th entry in the VS-PS semantic table. If there were another parameter
being exported by the VS, it would have been associated with the 1st entry, and
so on. The driver in MESA too follows the same consecutive assignment.

As with FS-VS semantic table, the command buffer is the place where the VS-PS
semantic table is setup.

```
02 00 00 00 00 00 14 a0  00 80 00 00 88 0a 20 95
00 04 38 00 90 6c 34 40  00 00 38 80 80 6c 34 60
00 04 38 00 10 6b 34 00  00 00 38 00 10 6b 34 20
00 04 38 00 00 6b 34 40  00 00 38 80 00 6b 34 60

PC| Offset|     Instruction Words
--+-------+----------------------
 0|   0x00| 0x00000002 0xa0140000
 1|   0x08| 0x00008000 0x95200a88
 2|   0x10| 0x00380400 0x40346c90
 3|   0x18| 0x80380000 0x60346c80
 4|   0x20| 0x00380400 0x00346b10
 5|   0x28| 0x00380000 0x20346b10
 6|   0x30| 0x00380400 0x40346b00
 7|   0x38| 0x80380000 0x60346b00
```

---

#### **PS[0]:**

The `inst` field of this CF instruction is `0x81`. This isn't the usual CF
instructions, or the Alloc, Export, Import CF instruction, (check if there are
other types to remove ambiguity), as this value isn't defined.

The instruction belongs to the group of CF instructions that begin an ALU
clause. The `inst` field is shorter, and its value here turns out to be `0x8`,
which represents the `CF_INST_ALU` instruction.

Note also that *CF ALU* instructions are *Control Flow* instructions used to
initiate ALU clauses, while the *ALU* instructions belong only to the ALU
clauses.

```
PC| Offset|   CF_ALU_WORD0|   CF_ALU_WORD1|
--+-------+---------------+---------------+
 0|   0x00|     0x00000002|     0xa0140000|

km0|  kb1|  kb0|                        addr|
 00| 0000| 0000| 00 0000 0000 0000 0000 0010|

b| wqm| inst| alt_const|    count|       ka1|      ka0| km1|
1|   0| 1000|         0| 000 0101| 0000 0000|0000 0000|  00|
```

The KCACHE fields are not relevant here; they are used to access Constant
Buffers (Uniforms), for which this demo has no need. Also note that ALU clause
instructions are 64-bits, while the Vertex Fetch Clause instructions are
128-bits.

|Field|Value|Comment|
|-----:|-----:|-------|
|`addr`|`2`| The PC of the clause to execute, relative to the start of this shader.
|`count`|`5`| The number of instructions to execute in the clause, minus one.
|`inst`|`8`| `CF_INST_ALU`.

As with execution of other types of clauses, the GPU, before moving on to the
next instruction in the program order, jumps to the clause found at `addr` PC,
and executes `count + 1` number of instructions.

---

#### **PS[1]:**

The `inst` field of this CF instruction is `0x54`, or `CF_INST_EXPORT_DONE` -
one of the CF instruction types, seen above in the VS.

```
PC| Offset| CF_ALLOC_EXPORT_WORD0| CF_ALLOC_EXPORT_WORD1_SWIZ|
--+-------+----------------------+---------------------------+
 1|   0x08|            0x00008000|                 0x95200a88|

es|   ix_gpr| rw_rel|   rw_gpr| type|       array_base|
00| 000 0000|      0| 000 0001|   00| 0 0000 0000 0000|

b| m|      inst| eop| vpm| count| rsvd| sel_w| sel_z| sel_y| sel_x|
1| 0| 0101 0100|   1|   0|  0000| 0000|   101|   010|   001|   000|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`array_base`|`0`| The index of the Color Buffer to export into. Color Buffer #0 is the one that the GPU is setup to display.
|`type`|`0`| `EXPORT_PIXEL`.
|`rw_gpr`|`1`| The register#, `R1`, that contains the COLOR property, and acts as the source register for the export. The ALU clause that this instruction initiates interpolates the COLOR property for the current fragment, and stores the resultant color in `R1`.
|`sel_w`|`5`| The W (`sel_w`) (i.e. the Alpha) channel of the XYZW vector being exported must be set to `1.0`. Note that when VS exported the COLOR property for each vertex, it set Alpha to `1.0`, but that was not the property of a *fragment*. Here in the PS, each fragment is shaded; not setting Alpha to `1.0` here causes the GPU to pick-up random/left-over values for the Alpha channel. That causes unwanted transparency effects. Hence, set the Alpha channel to 1.0 for each fragment.
|`count`|`0`| The number of Color Buffers to which this pixel is being exported, minus one. (Verify this)
|`eop`|`1`| End Of this Program.
|`inst`|`0x54`| `CF_INST_EXPORT_DONE`. The `DONE` suffix suggests that this is the last (and, in this case, the only one) of all the Color Buffers to which this pixel is being exported.

---

#### **PS[2], ALU[0]:**

Note that an `ALU_WORD` isn't the same as a `CF_ALU_WORD`.

The ALU instructions interpolate the COLOR parameter for this fragment. To
support the interpolation, the each interpolation instructions receives two of
V0, V1-V0 (also written as V10), and V2-V0 (also written as V20) as PARAMETERS.
It also receives the BaryCentric
co-ordinates I/J (or u/v in other literature) inside `R0.X` and `R0.Y`
respectively. See the section "5.2 Evergreen/Cayman Starting Condition" in the
[Radeon Evergreen/Northern Islands Acceleration](http://developer.amd.com/wordpress/media/2013/10/evergreen_cayman_programming_guide.pdf)
manual.

The Evergreen ISA manual also describes the process of interpolation, and helps
in understand the reason the instructions are laid out the way they are.

In short, interpolating a single channel needs two instructions. The second
instruction carries our the MAD operation: `tmp = V0 + I * V10`. The result of this
operation is sent to the first instruction, which then performs the DOT
operation: `res = tmp + J * V20`. The res is then written out to the
destination channel.

Each instruction in the instruction group can only write to its corresponding
channel. That is, the ALU instruction in the X slot can write to no other
channel except X, etc. (This limitation doesn't apply to the T unit).

The two `INTERP_Z` instructions interpolate the Z channel; These two
instructions must be allotted the ZW slots in the instruction group.

The four `INTERP_XY` instructions, which form another instruction group,
interpolate both X and Y channels. The X channel is interpolated by the
instructions in the XY slots, while the Y channel is interpolated by those in
the ZW slots. In the case of the Y-interpolation, instructions which calculates
the MAD and DOT operations are in the ZW slot, but the Z-slot passes the result
to the Y-slot; hence, the write to the Y channel must be in the Y slot, and not
Z slot (such a write is not even allowed). Details in the Evergreen ISA.

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 2|   0x10| 0x00380400|    0x40346c90|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
0| 00| 000|   0|  00|   0| 1 1100 0000|   0|  01|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 10|  0| 000 0001| 101| 000 1101 1001|   00|  1| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`1`| Select the Y channel of `src0`. `R0.Y` contains the J co-ordinate.
|`s1s`|`0x1c0`| `0x1c0` represents the location the parameter in the Parameter Cache. Since VS exports a single parameter, it is available at index 0 in the Parameter Cache, or, in absolute terms, at location `0x1c0 + 0 = 0x1c0`.
|`l`|`0`| This isn't the last instruction in the current instruction group. This being a VLIW-5 CPU, the ALU instructions are grouped into one each for each of the XYZWT ALU units. In addition, the group can have 2 64-bit constants embedded in the instruction stream. These 7 slots are the maximum supported; a group can have less. The `l` bit marks the end of the group.
|`wm`|`1`| `WRITE_MASK`. It would have been better for it to be named `WRITE_ENABLE`, since it doesn't prevent the write into the destination GPR; on the contrary, it enables the write. Note that this is the `DOT` half of the interpolation.
|`inst`|`0xd9`| `OP2_INST_INTERP_Z`. This (and the next instruction) interpolate the Z channel.
|`bs`|`5`| `ALU_VEC_210` bank swizzle. This instruction works with two source values. The order in which they must be loaded is fixed for this instruction. The `210` says that `src0` (the J coordinate) must be loaded in clock cycle 2, and `src1` (one of V10, V20, but not the one sent to the MAD half) must be loaded in clock cycle 1.
|`dst_gpr`|`1`| Write the result into `R1`.
|`dc`|`2`| Specifically, write the result into `R1.Z`.

The MAD half of this 2-instruction group (only ZW) follows.

---

#### **PS[3], ALU[1]:**

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 3|   0x18| 0x80380000|    0x60346c80|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
1| 00| 000|   0|  00|   0| 1 1100 0000|   0|  00|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 11|  0| 000 0001| 101| 000 1101 1001|   00|  0| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`0`| Select the X channel of `src0`. `R0.X` contains the I co-ordinate.
|`l`|`1`| This is the last instruction in the current instruction group.
|`wm`|`0`| Do not write the destination register. However, the result of this instruction is forwarded to the DOT half (the instruction above).
|`dc`|`3`| Select the `W` channel of the destination register. Although the destination register is not written by this instruction, proper setting of `dst_gpr` and `dc` is required for correct forwarding of the result. Since this instruction is the MAD half of the `INTERP_Z` operation, it must set the destination channel to `W`.

---

#### **PS[4], ALU[2]:**

This instruction begins the XYZW group, which interpolates both X and Y channels.
The first two of the group interpolate the X channel; the last two, the Y
channel.

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 4|   0x20| 0x00380400|    0x00346b10|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
0| 00| 000|   0|  00|   0| 1 1100 0000|   0|  01|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 00|  0| 000 0001| 101| 000 1101 0110|   00|  1| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`1`| Select the Y channel of `src0`. `R0.Y` contains the J co-ordinate.
|`inst`|`0xd6`| `OP2_INST_INTERP_XY`. This (and the next three instructions) interpolate the XY channels. This instruction is the DOT half of the DOT-MAD interpolation instructions.
|`wm`|`1`| `WRITE_ENABLE`. Note that this is the `DOT` half of the X-interpolation. It must write to the X channel.
|`dst_gpr`|`1`| `R1` is the destination register which is, in the end, going to contain the interpolated XYZW values of the PARAMETER.
|`dc`|`0`| Select the `X` channel of the destination register.

---

#### **PS[5], ALU[3]:**

This instruction begins the XYZW group, which interpolates both X and Y channels.

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 5|   0x28| 0x00380000|    0x20346b10|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
0| 00| 000|   0|  00|   0| 1 1100 0000|   0|  00|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 01|  0| 000 0001| 101| 000 1101 0110|   00|  1| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`0`| Select the X channel of `src0`. `R0.X` contains the I co-ordinate.
|`wm`|`1`| `WRITE_ENABLE`. Note that this instruct slot performs double-duty. It not only calculates forms the MAD half of the XY interpolation, it also receives the interpolated Y value from the ZW instruction pairs. As a result, we must select dst.`Y` as writable.
|`dst_gpr`|`1`| `R1` is the destination register which is, in the end, going to contain the interpolated XYZW values of the PARAMETER.
|`dc`|`1`| Select the `Y` channel of the destination register.

---

#### **PS[6], ALU[4]:**

This instruction begins the XYZW group, which interpolates both X and Y channels.
The first two of the group interpolate the X channel; the last two, the Y
channel.

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 6|   0x30| 0x00380400|    0x40346b00|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
0| 00| 000|   0|  00|   0| 1 1100 0000|   0|  01|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 10|  0| 000 0001| 101| 000 1101 0110|   00|  0| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`1`| Select the Y channel of `src0`. `R0.Y` contains the J co-ordinate.
|`wm`|`0`| Do not write the destination register. Note that this is the `DOT` half of the Y-interpolation; it's output is the final result of the Y-interpolation. But since this instruction itself lies in the Z-slot of the group, it cannot select dst.`Y` to write; instead the GPU forwards the output of this instruction to the instruction in the Y-slot, which then writes into the Y channel of the destination register. Nevertheless, this instruction must appropriately select the destination and the channel.
|`dst_gpr`|`1`| `R1` is the destination register which is, in the end, going to contain the interpolated XYZW values of the PARAMETER.
|`dc`|`2`| Select the `Z` channel of the destination register.

---

#### **PS[7], ALU[5]:**

This instruction begins the XYZW group, which interpolates both X and Y channels.

```
PC| Offset|  ALU_WORD0| ALU_WORD1_OP2|
--+-------+-----------+--------------+
 7|   0x38| 0x80380000|    0x60346b00|

l| ps|  im| s1n| s1c| s1r|         s1s| s0n| s0c| s0r|         s0s|
1| 00| 000|   0|  00|   0| 1 1100 0000|   0|  00|   0| 0 0000 0000|

c| dc| dr|  dst_gpr|  bs|          inst| omod| wm|up|uem|s1a|s0a|
0| 11|  0| 000 0001| 101| 000 1101 0110|   00|  0| 0|  0|  0|  0|
```

|Field|Value|Comment|
|-----:|-----:|-------|
|`s0s`|`0`| The first source, `src0` is the `R0` register. It contains the I/J co-ordinates.
|`s0c`|`0`| Select the X channel of `src0`. `R0.X` contains the I co-ordinate.
|`wm`|`0`| Do not write to the destination register. Since this is the MAD half of the Y-interpolation, it forwards its result to the DOT half, i.e. the instruction in the Z-slot. For proper forwarding, this instruction must still select the correct destination register and channel.
|`dst_gpr`|`1`| `R1` is the destination register.
|`dc`|`3`| Select the `W` channel of the destination register.

The entire interpolation of the XYZ coordinates of the COLOR parameter can be
understood as shown below, with `R1.XYZ` containing the resultant interpolated
COLOR channels:

```
interp_z	r1.z, r0.y, param0.x	(r1.z = t0 + v20.z * j)

interp_z	__.w, r0.x, param0.x	(t0 = v0.z + v10.z * i)



interp_xy	r1.x, r0.y, param0.x	(r1.x = t1 + v20.x * j)

interp_xy	r1.y, r0.x, param0.x	(t1 = v0.x + v10.x * i)
					(r1.y = t2)

interp_xy	__.z, r0.y, param0.x	(t2 = t3 + v20.y * j)

interp_xy	__.w, r0.x, param0.x	(t3 = v0.y + v10.y * i)
```
---

### **Command Buffer:**

The Command Buffer consists of some number of words. The commands are
encapsulated within GFX ring packets of type 3. The manuals have the format and
the meaning of various fields.

The command buffer was extracted out of MESA when rendering the same triangle,
using an OpenGL app. Then, the buffer was modified to suit the modest needs of
this demo, and to fix the pointers to various resources.

The buffer is then inserted into the GFX ring for execution as an
Indirect Buffer.

It is available, as a binary file, [here](/wip/data/eg.cmds.0.bin).
Only some of the most obvious commands are shown below.

```
. . .

c0016900 SET_CONTEX_REG
0000023c SQ_VTX_SEMANTIC_CLEAR
ffffffff -> [SQ_VTX_SEMANTIC_CLEAR]
	 Clears the FS-VS semantic table entries.

. . .

c0026900 SET_CONTEXT_REG
00000090 PA_SC_GENERIC_SCISSOR_TL
00000000 -> [PA_SC_GENERIC_SCISSOR_TL]
40004000 -> [PA_SC_GENERIC_SCISSOR_BR]
	 Defines the size of the Generic Scissor, by setting its top-left and
	 bottom-right corners.

. . .

c0026900 SET_CONTEXT_REG
0000000c PA_SC_SCREEN_SCISSOR_TL
00000000 -> [PA_SC_SCREEN_SCISSOR_TL]
40004000 -> [PA_SC_SCREEN_SCISSOR_BR]
	 As above, but for the Screen Scissor.

. . .

c00d6900 SET_CONTEXT_REG
00000318 CB_COLOR0_BASE
00008000 -> [CB_COLOR0_BASE]. The Color Buffers's gpu_addr >> 8.
0000009f -> [CB_COLOR0_PITCH].
	 Width, in units of 8-pixels, minus one. Here, 720/8 - 1.
0000383f -> [CB_COLOR0_SLICE].
         The slice here is made up of the entire buffer. The dimensions of the
	 slice, in the units of 8x8, minus one. Here, 1280 * 720 / 64 - 1.
00000000
0108e168 -> [CB_COLOR0_INFO]
         .format	= COLOR_8_8_8_8;
	 .array_mode	= ARRAY_LINEAR_ALIGNED;
	 .number_type	= NUMBER_SRGB;
	 .comp_swap	= SWAP_ALT;
			  The frame buffer is in B8R8G8A8, or BGRA8888, or
			  ARBG32 format. But when pixel color is exported, it
			  is in the format XYZW=RGBA. Hence, we need to swap
			  the channels such that XYZW=BGRA.
	 .src_format	= EXPORT_4C_16BPC; Only one that is suitable.
	 .blend_clamp	= 1; Must be set for NUMBER_SRGB.
00000010 -> [CB_COLOR0_ATTRIB]
00000000 .non_disp_tiling_order = 1;
	 Must be set since the Color Buffer is in a non-tiled format.
00000000
00000000
00000000
00000000
00000000
00000000

. . .

c0026900 SET_CONTEXT_REG
00000081 PA_SC_WINDOW_SCISSOR_TL
00000000 -> [PA_SC_WINDOW_SCISSOR_TL] (0, 0)
02d00500 -> [PA_SC_WINDOW_SCISSOR_BR] (1280, 720)

. . .

c0086d00 SET_RESOURCE
00001ff8 Buffer_ID 0x1f, from the FS Resources Base (992).
	 ((992 + 31) * 32) / 8.
00004000 LSW(Buffer_Addr)
00000047 sizeof(Buffer) - 1
00001800 stride = 6 * 4, LSB(MSW(Buffer_Addr))
00005440 Channel Selection. XYZ1
00000000
00000000
00000000
c0000000 Valid Buffer.

. . .

c0026900 SET_CONTEXT_REG
000000e0 SQ_VTX_SEMANTIC_0
00000090 -> [SQ_VTX_SEMANTIC_0]. Semantic ID for POSITION.
00000092 -> [SQ_VTX_SEMANTIC_1]. Semantic ID for COLOR.

. . .

c0016900 SET_CONTEXT_REG
0000030f PA_SC_AA_MASK
01010101 -> [PA_SC_AA_MASK].
	 Even with AA disabled, since the pixels are processed in 2x2 quad, we
	 must enable Sample0 (the only sample when AA is disabled) for each of
	 the 4 pixels.

. . .

c0016900 SET_CONTEXT_REG
00000204 PA_CL_CLIP_CNTL
00010000 .clip_disable = 1;

. . .

c0066900 SET_CONTEXT_REG
0000010f PA_CL_VPORT_XCALE_0
44200000 -> [PA_CL_VPORT_XSCALE_0] = 1280.0 / 2, in float.
44200000 -> [PA_CL_VPORT_XOFFSET_0]
c3b40000 -> [PA_CL_VPORT_YSCALE_0] = -720 / 2 (Note the Y-axis being flipped)
43b40000 -> [PA_CL_VPORT_YOFFSET_0] = 720 / 2
3f000000 -> [PA_CL_VPORT_ZSCALE_0] = 0.5
3f000000 -> [PA_CL_VPORT_ZOFFSET_0]
	 These are utilized when performing the ViewPort Transform.

. . .

c0016900 SET_CONTEXT_REG
00000229 SQ_PGM_START_FS
00000054 -> [SQ_PGM_START_FS] = fs_start_gpu_addr >> 8

c0016900 SET_CONTEXT_REG
00000191 SPI_PS_INPUT_CNTL_0
00000092 -> [SPI_PS_INPUT_CNTL_0] = Semantic ID for the COLOR parameter.

c0026900 SET_CONTEXT_REG
000001b3 SPI_PS_IN_CONTROL_0
10000001 -> [SPI_PS_IN_CONTROL_0]
	 .num_interp = 1;
	 .persp_gradient_ena = 1;

. . .

c0016900 SET_CONTEXT_REG
00000213 SQ_PGM_EXPORTS_PS
00000002 -> [SQ_PGM_EXPORTS_PS]. PS exports one Color (Buffer?).

c0026900 SET_CONTEXT_REG
00000210 SQ_PGM_START_PS
00000058 -> [SQ_PGM_START_PS] = ps_start_gpu_addr >> 8
00a00002 -> [SQ_PGM_RESOURCE_PS]
	 .num_gprs = 2; The PS needs 2 GPRs to run.

c00a6900 SET_CONTEXT_REG
00000187 SPI_VS_OUT_ID_0
00000092 -> [SPI_VS_OUT_ID_0].
	 .semantic_0 = Semantic ID for the COLOR export.

. . .

c0016900 SET_CONTEXT_REG
00000218 SQ_PGM_RESOURCES_VS
00200103 -> [SQ_PGM_RESOURCES_VS]
	 .num_gprs = 3; The VS needs 3 GPRs to run.
	 .stack_size = 1; The VS needs 1 stack entry, for calling the FS.

c0016900 SET_CONTEXT_REG
00000206 PA_CL_VTE_CNTL
0000003f -> [PA_CL_VTE_CNTL]. Enable ViewPort Transformation

c0016900 SET_CONTEXT_REG
00000217 SQ_PGM_START_VS
00000050 -> [SQ_PGM_START_VS] = vs_start_gpu_addr >> 8

. . .

c0016800 SET_CONFIG_REG
00000256 VGT_PRIMITIVE_TYPE
00000004 -> [VGT_PRIMITIVE_TYPE] = DI_PT_TRILIST

c0002f00 NUM_INSTANCE
00000001

c0012d00 DRAW_INDEX_AUTO
00000003 # of vertices
00000002 VGT should generate indices (as no Index Buffer has been setup)

c0016800 SET_CONFIG_REG
00000010 WAIT_UNTIL
00008000 -> [WAIT_UNTIL]. Wait for the 3D engine to become idle.

c0034300 SURFACE_SYNC
02000040 CB0_DEST_BASE_ENA, CB_ACTION_ENA
00003840 Size of the Color Buffer in units of 256 bytes.
00008000 cb_gpu_addr >> 8
0000000a
```

---

### **Output:**

A screen capture from OBS Studio may result in an image file that has been
processed by the OBS Studio, and so may not represent the actual frame-buffer
contents. Instead, QEMU's `pmemsave` command is used to dump the frame-buffer.
These raw pixels are then converted to a PNG image with the help of
ImageMagick: `convert -size 1280x720 -depth 8 BGRA:raw.bin eg.0.png`.

The image is available [here](/wip/images/eg.0.png). The vertex buffer that was
provided to the GPU was:

```
static const float verts[] = {
	/* positions */		/* colors */
	0.9, -0.9, 0,		1, 0, 0,	/* BR */
	-0.9, -0.9, 0,		0, 0, 1,	/* BL */
	0.0,  0.9, 0,		0, 1, 0		/* T */
};
```

---

### **Update #1:** <a name="update1"></a>

The vertices given above do not form an equilateral triangle. Moreover, because
the co-ordinates are in the NDC space, and because the aspect ratio of the
ViewPort is not square (1280x720 is 16:9), the ViewPort Transformation causes
the triangle to stretch out in the horizontal dimension with respect to the
vertical.

We still want to keep the vertices in the NDC space, but also want to render a
nice, equilateral triangle. We want to keep the height of the triangle the same,
i.e. 1.8 units in the NDC space. An equilateral triangle with a height of 1.8
units has each of its sides measuring `2 * 1.8 / tan(pi/3) = 2.07846096908`
units.

The transformation stretches the horizontal units by a factor of 16/9, with
respect to the vertical units. Hence, the base of the triangle must be
*compressed* by that same factor before performing the transformation.

That compression gives a base-length of 
`2.07846096908 * 9 / 16 = 1.16913429511` units.
Therefore, the X co-ordinates of the bottom-left and bottom-right vertices
should be changed to `-1.16913429511/2 = -0.58456714755` and
`1.16913429511/2 = 0.58456714755` units, respectively.

```
static const float verts[] = {
	/* positions */		/* colors */
	0.5846, -0.9, 0,	1, 0, 0,	/* BR */
	-0.5846, -0.9, 0,	0, 0, 1,	/* BL */
	0.0,  0.9, 0,		0, 1, 0		/* T */
};
```

The resultant image is available [here](/wip/images/eg.1.png).

---

### **Update #2:** <a name="update2"></a>

Instead of hand-coding the instructions, an assembler was written to help.
The FS was made part of the VS, i.e.`SQ_PGM_FS_START` is made the
same as `SQ_PGM_VS_START`, and any FS-specific offsets were made relative to
the start of the VS.

The instructions for the VS and the FS:
```
vs_start:
c.fs		fs_start	b;
c.xd.pos(1) 	60.xyzw, r1	b;
c.xd.prm(1)	0.xyzw, r2	b, eop;

fs_start:
c.vc(2)	cc.a	vfc_start	b;
c.ret				b;
c.nop;	# To align vfc_start at even PC

vfc_start:
v.s		0x90.xyz1, float3, sn, fs[0x1f][0], r0;
v.s		0x92.xyz1, float3, sn, fs[0x1f][0xc], r0;
```

Code generated by the assembler for the VS and the FS:

```
/*vs_start:*/
0x00000003, 0x84c00000, /*0: c.fs cc.a fs_start b;*/
0x0000a03c, 0x95000688, /*1: c.xd.pos(1) 60.xyzw, r1 b;*/
0x00014000, 0x95200688, /*2: c.xd.prm(1) 0.xyzw, r2 b,eop;*/

/*fs_start:*/
0x00000006, 0x80800400, /*3: c.vc(2) cc.a vfc_start b;*/
0x00000000, 0x85000000, /*4: c.ret b;*/
0x00000000, 0x00000000, /*5: c.nop;*/

/*vfc_start:*/
0x30001f01, 0x4c151090, 0x00000000, 0x00000000,
/*6: v.s 0x90.xyz1, float3, sn, fs[0x1f][0], r0;*/
0x30001f01, 0x4c151092, 0x0000000c, 0x00000000,
/*8: v.s 0x92.xyz1, float3, sn, fs[0x1f][0xc], r0;*/
```

The instructions for the PS:

```
ps_start:
c.alu(6)	alu_start	b;
c.xd.pix(1)	0.xyz1, r1	b, eop;

alu_start:
a.iz		r1.z,	0.y, 0x1c0,v210;
a.iz		$.w,	0.x, 0x1c0,v210	last;

a.ixy		r1.x,	0.y, 0x1c0,v210;
a.ixy		r1.y,	0.x, 0x1c0,v210;
a.ixy		$.z,	0.y, 0x1c0,v210;
a.ixy		$.w,	0.x, 0x1c0,v210	last;
```

Code generated by the assembler for the PS:

```
/*ps_start:*/
0x00000002, 0xa0140000, /*0: c.alu(6) alu_start b;*/
0x00008000, 0x95200a88, /*1: c.xd.pix(1) 0.xyz1, r1 b,eop;*/

/*alu_start:*/
0x00380400, 0x40346c90, /*2: a.iz r1.z, 0.y, 0x1c0,v210;*/
0x80380000, 0x60146c80, /*3: a.iz $.w, 0.x, 0x1c0,v210 last;*/
0x00380400, 0x00346b10, /*4: a.ixy r1.x, 0.y, 0x1c0,v210;*/
0x00380000, 0x20346b10, /*5: a.ixy r1.y, 0.x, 0x1c0,v210;*/
0x00380400, 0x40146b00, /*6: a.ixy $.z, 0.y, 0x1c0,v210;*/
0x80380000, 0x60146b00, /*7: a.ixy $.w, 0.x, 0x1c0,v210 last;*/
```
