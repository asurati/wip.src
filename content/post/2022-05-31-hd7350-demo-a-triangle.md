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

This post demonstrates setting up a HD7350 to draw a triangle. The approach
adopted here is that of choosing bare-metal programming over relying on
established interfaces like the Linux DRM. That is not to say that Linux DRM
cannot be used - it is possible to send compute or rendering commands to the
GPU by directly interfacing with the DRM layer on Linux.

The bare-metal program maps the PCIe BARs, POSTs the Video BIOS, configures and
tests the rings (GFX, DMA, INT), parses the EDID blocks of the attached
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

##### **VS[0]:**

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
|`addr`|`0`| The `inst` field shows that this is a `CF_INST_CALL_FS` instruction. Hence, this `addr` field is a PC, relative to the start of the FS, that the GPU should call.
|`count`|`1`| The increment to apply to Call Nesting Counter. Used to avoid reaching a call-nesting of unsupported depths.
|`inst`|`0x13`| The opcode for `CF_INST_CALL_FS`.
|`b`|`1`| Barrier. Wait for other CF instructions to complete before beginning this one.

---

##### **VS[1]:**

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
|`sel_x`|`0`| The X channel (`sel_x`) of the XYZW vector being exported must be sourced from the X channel (= `0`) of the source register.
|`sel_y`|`1`| The Y channel (`sel_y`) of the XYZW vector being exported must be sourced from the Y channel (= `1`) of the source register.
|`sel_z`|`2`| . . .
|`sel_w`|`3`| . . .
|`count`|`0`| The number of POSITIONs exported, minus one.
|`inst`|`0x54`| `CF_INST_EXPORT_DONE`. The `DONE` suffix suggests that this is the last (and, in this case, the only) POSITION being exported.

---

##### **VS[1]:**

Similar to the one directly above, but exports COLOR parameter.

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
|`array_base`|`0`| Parameter Index. Must be 0 for the first (and, in this case, the only) PARAMETER being exported. The strict sequence (0, 1, 2, etc..) seems to be necessary if the PS relies on *Direct Parameter Reads*, which are hardware-supported.
|`type`|`2`| `EXPORT_PARAM`.
|`rw_gpr`|`2`| The register#, `R2`, that contains the COLOR parameter, and acts as the source register for the export. As described later, the semantic table guides the FS into placing the COLOR parameter of a vertex into the `R2` register.
|`count`|`0`| The number of PARAMETERs exported, minus one.
|`inst`|`0x54`| `CF_INST_EXPORT_DONE`. The `DONE` suffix suggests that this is the last (and, in this case, the only) PARAMETER being exported.
