---
title: "QPU Demo: Triangle with a GL Pass-Through Shader"
date: '2021-10-04'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo results in the same output image as displayed in
[Triangle With NV Shader](/wip/post/2021/10/02/qpu-demo-nv-shader), but this
time the full 3D pipeline is employed, with the exception that the Coordinate
and the Vertex Shaders are mere pass-through shaders.

The Coordinate and the Vertex Shaders are each tasked with reading the input
vertex attributes off the VPM (placed there by the Vertex Cache Manager), and
writing shaded vertices in specific output formats back into the VPM.

The input vertex attributes and the output (shaded) vertices are each stored
within the VPM in separate, pre-determined locations/segments (dependent upon
whether any VPM is reserved for User Programs, and perhaps also on the QPU ID).
The shader program doesn't need to know exactly where its inputs and outputs
are stored - it can safely assume that the inputs and outputs are stored
starting at the first row (Y = 0) - the GPU redirects the reads and writes to
appropriate segments. A consequence of this fact is that these shaders *must*
write their output - if a shader does not write its output, the GPU seems to
pick up whatever values it finds in the designated output segment, and to
assume them as valid output. As a result, the screen shows either no render
(may be just the Clear Colour), or random shapes. Because of this reason,
one cannot usually get away with an empty shader.

---
### **Coordinate and Vertex Shaders:**

Coordinate Shaders are stripped-down Vertex Shaders; they perform vertex
transformations only, whereas a full-fledged Vertex Shader additionally
performs colour calculations, and varying transformations also.

The Coordinate Shader runs during the Binning phase, while the Vertex Shader
(and later, the Fragment Shader) runs during the Rendering phase. The output
from the Coordinate Shader helps the PTB decide, for each tile, the list of
primitives that affect the tile. The output of a Coordinate Shader includes
the Clip-Space coordinates of the vertices - this piece of information helps
the PTB perform clipping.

The Vertex Shader must duplicate some of the work that the Coordinate Shader
did already perform. This duplication of work is limited to the primitives which
actually affect a tile, and is the cost the GPU must pay to avoid maintaining
large amounts of processed data; were they kept in system RAM, they would incur
prohibitively large delays of transferring them through DMA.

---
### **Tile-Binning Control List:**

The Tile-Binning Control List remains mostly same as shown in the previous
demo. Instead of a NV Shader State Control Record (0x41), it must now employ
a GL Shader State Control Record (0x40), which links to a GL Shader State
Record.

As the Coordinate Shader demonstrated here is a simple, pass-through shader,
its input is kept in the same format as its output; it needs to do little more
than a copy-paste of its input into its output.

Raw bytes:
```
...,0x40,0x02,0x15,0x44,0x40,...
```

|Byte(s)|Value|Comment|
|-----:|-----:|-------|
|`0x40`| `0x40` | GL Shader State Control Record follows.
|`0x02,0x15,0x44,0x40`| `0x40441502` | The bus address of the GL Shader State Record, aligned to 16-bytes. The least significant 3 bits (zeroes for a well-formed address) store the number of attribute arrays enabled, out of 8 supported.

---
### **GL Shader State Record:**

The GL Shader State Record is comparatively larger than the NV Shader State
Record, as it needs to record information about the three shaders, and about
the layouts and the locations, within the system RAM and within the VPM, of the
input vertex attributes. As the GPU supports OpenGL ES 1.1/2.0, this facility of
providing the layout and the locations of the input vertex attributes is
flexible enough to fulfill the requirements placed forward by the GL specification.

The following should be read along with the layout of the vertex attributes
found in the
[driver program](https://github.com/asurati/x03/blob/main/demo/d51.c).

Raw bytes:
```
0x05,0x00,0x00,0x03,0xb0,0x05,0x41,0x40,0x00,0x00,0x00,0x00,0x00,0x00,0x02,0x18,
0x58,0x07,0x41,0x40,0x00,0x00,0x00,0x00,0x00,0x00,0x01,0x1c,0x28,0x06,0x41,0x40,
0x00,0x00,0x00,0x00,0xe0,0x06,0x41,0x40,0x1b,0x28,0x00,0x00,0xf0,0x06,0x41,0x40,
0x17,0x28,0x00,0x00
```

|Byte(s)|Value|Comment|
|-----:|-----:|-------|
|`0x05,0x00`| `0x5` | Flags.<br> Fragment Shader is Single Threaded is `1` = `on`.<br/>Enable Clipping is `1` = `on`.
|`0x00`| `0x0` | Fragment Shader Number of Uniforms (not used).
|`0x03`| `0x3` | Fragment Shader Number of Varyings is `3`, since each vertex provides Red, Green and Blue colour components.
|`0xb0,0x05,0x41,0x40`| `0x404105b0` | The bus address of the code for the Fragment Shader.
|`0x00,0x00`| `0x0` | Vertex Shader Number of Uniforms (not used).
|`0x02`| `0x2` | Vertex Shader Attribute Array Select Bits. The value `2` selects a single array, with index 1.
|`0x18`| `0x18` | Vertex Shader Total Attribute Size.
|`0x58,0x07,0x41,0x40`| `0x40410758` | The bus address of the code for the Vertex Shader.
|`0x00,0x00,0x00,0x00`| `0x0` | The bus address of the Uniforms array for the Vertex Shader. Not used in this demo.
|`0x00,0x00`| `0x0` | Coordinate Shader Number of Uniforms (not used).
|`0x01`| `0x1` | Coordinate Shader Attribute Array Select Bits. The value `1` selects a single array, with index 0.
|`0x1c`| `0x1c` | Coordinate Shader Total Attribute Size.
|`0x28,0x06,0x41,0x40`| `0x40410628` | The bus address of the code for the Coordinate Shader.
|`0x00,0x00,0x00,0x00`| `0x0` | The bus address of the Uniforms array for the Coordinate Shader. Not used in this demo.
|`0xe0,0x06,0x41,0x40`| `0x404106e0` | Attribute Array[0]: The bus address of the Base Memory Address where this array resides in system RAM.
|`0x1b`| `0x1b` | Attribute Array[0]: The total size of attributes covered in this array, minus 1.
|`0x28`| `0x28` | Attribute Array[0]: The size of the memory stride; the amount of bytes to skip within the array, in order to reach the same attributes for the next vertex.
|`0x0`| `0x0` | Attribute Array[0]: The Vertex Shader VPM Offset, usually in multiples of 4. It gives the number of rows in the VPM to skip before storing this attribute array. For instance, setting it to 8 causes this attribute array to start at `Y=2` when the Vertex Shader is run.
|`0x0`| `0x0` | Attribute Array[0]: The Coordinate Shader VPM Offset. Similar to the Vertex Shader VPM Offset, but applicable only when the Coordinate Shader is being run.
|`0xf0,0x06,0x41,0x40`| `0x404106f0` | Attribute Array[1]: The bus address of the Base Memory Address where this array resides in system RAM.
|`0x17`| `0x17` | Attribute Array[1]: The total size of attributes covered in this array, minus 1.
|`0x28`| `0x28` | Attribute Array[1]: The size of the memory stride; the amount of bytes to skip within the array, in order to reach the same attributes for the next vertex.
|`0x0`| `0x0` | Attribute Array[1]: The Vertex Shader VPM Offset, usually in multiples of 4. It gives the number of rows in the VPM to skip before storing this attribute array. For instance, setting it to 8 causes this attribute array to start at `Y=2` when the Vertex Shader is run.
|`0x0`| `0x0` | Attribute Array[1]: The Coordinate Shader VPM Offset. Similar to the Vertex Shader VPM Offset, but applicable only when the Coordinate Shader is being run.

---
### **Tile-Rendering Control List:**

The Tile-Rendering Control List remains the same as before.


---
### **QPU Program:**

The Coordinate Shader:

```
0x00701a00, 0xe0020c67, // li   vpr_setup, -, 0x701a00;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x15c00dc0, 0xd0020027, // ori  a0, vpr, 0;
0x15c00dc0, 0xd0020067, // ori  a1, vpr, 0;
0x15c00dc0, 0xd00200a7, // ori  a2, vpr, 0;
0x15c00dc0, 0xd00200e7, // ori  a3, vpr, 0;
0x15c00dc0, 0xd0020127, // ori  a4, vpr, 0;
0x15c00dc0, 0xd0020167, // ori  a5, vpr, 0;
0x15c00dc0, 0xd00201a7, // ori  a6, vpr, 0;

0x00001a00, 0xe0021c67, // li   vpw_setup, -, 0x1a00;
0x15027d80, 0x10020c27, // or   vpw, a0, a0;
0x15067d80, 0x10020c27, // or   vpw, a1, a1;
0x150a7d80, 0x10020c27, // or   vpw, a2, a2;
0x150e7d80, 0x10020c27, // or   vpw, a3, a3;
0x15127d80, 0x10020c27, // or   vpw, a4, a4;
0x15167d80, 0x10020c27, // or   vpw, a5, a5;
0x151a7d80, 0x10020c27, // or   vpw, a6, a6;

0x159c1fc0, 0xd00209a7, // ori  host_int, 1, 1;
0x009e7000, 0x300009e7, // pe;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
```

Following is the contents of VPM after the Coordinate Shader pasted out its
input into the output. Notice how the input is stored in 7 rows beginning
at `Y=68,X=4`, and the output is stored at `Y=84,X=4`. And the shader read and
wrote assuming `Y=0` - the redirection occurred in the background transparently.

```
[0x44] 0x5e418502 0xf8862a00 0x22449b0b 0x6032eb2c 0x00000000 0xc0200000 0x40200000
[0x45] 0x64018014 0xa681b0c2 0x15a80204 0x24152880 0x40555555 0xc0555555 0xc0555555
[0x46] 0x04843abb 0x98a0b8d8 0x0887b004 0x17515912 0x40600000 0x40600000 0x40600000
[0x47] 0x424584ab 0x00808415 0xc0443a26 0xe6245246 0x40800000 0x40800000 0x40800000
[0x48] 0xc09c300c 0xf98ba871 0x4aa43191 0x13500130 0xf3870000 0x0c79f385 0x0c790c7b
[0x49] 0x8017b650 0x6f430522 0x465834aa 0x802057c0 0x3f70a3d7 0x3f70a3d7 0x3f70a3d7
[0x4a] 0xb4482590 0x10c41400 0x60401145 0x80123219 0x3e800000 0x3e800000 0x3e800000
. . .
[0x54] 0x5e418502 0xf8862a00 0x22449b0b 0x6032eb2c 0x00000000 0xc0200000 0x40200000
[0x55] 0x64018014 0xa681b0c2 0x15a80204 0x24152880 0x40555555 0xc0555555 0xc0555555
[0x56] 0x04843abb 0x98a0b8d8 0x0887b004 0x17515912 0x40600000 0x40600000 0x40600000
[0x57] 0x424584ab 0x00808415 0xc0443a26 0xe6245246 0x40800000 0x40800000 0x40800000
[0x58] 0xc09c300c 0xf98ba871 0x4aa43191 0x13500130 0xf3870000 0x0c79f385 0x0c790c7b
[0x59] 0x8017b650 0x6f430522 0x465834aa 0x802057c0 0x3f70a3d7 0x3f70a3d7 0x3f70a3d7
[0x5a] 0xb4482590 0x10c41400 0x60401145 0x80123219 0x3e800000 0x3e800000 0x3e800000
```

The Vertex Shader:

```
0x00601a00, 0xe0020c67, // li	vpr_setup, -, 0x601a00;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
0x15c00dc0, 0xd0020027, // ori	a0, vpr, 0;
0x15c00dc0, 0xd0020067, // ori	a1, vpr, 0;
0x15c00dc0, 0xd00200a7, // ori	a2, vpr, 0;
0x15c00dc0, 0xd00200e7, // ori	a3, vpr, 0;
0x15c00dc0, 0xd0020127, // ori	a4, vpr, 0;
0x15c00dc0, 0xd0020167, // ori	a5, vpr, 0;

0x00001a00, 0xe0021c67, // li	vpw_setup, -, 0x1a00;
0x15027d80, 0x10020c27, // or	vpw, a0, a0;
0x15067d80, 0x10020c27, // or	vpw, a1, a1;
0x150a7d80, 0x10020c27, // or	vpw, a2, a2;
0x150e7d80, 0x10020c27, // or	vpw, a3, a3;
0x15127d80, 0x10020c27, // or	vpw, a4, a4;
0x15167d80, 0x10020c27, // or	vpw, a5, a5;

0x159c1fc0, 0xd00209a7, // ori	host_int, 1, 1;
0x009e7000, 0x300009e7, // pe;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
```

The Fragment Shader remains the same as before.

---
### **Running the demo:**

The driver program can be found
[here](https://github.com/asurati/x03/blob/main/demo/d51.c).

```
v3dirqh: errstat 1000, intctl 0, dbqitc 800
v3dirqh: errstat 1000, intctl 2, dbqitc 0
v3dirqh: errstat 1000, intctl 0, dbqitc c00
v3dirqh: errstat 1000, intctl 1, dbqitc 3ff
```

The first interrupt is raised because the Coordinate Shader requested for 
interrupting the host. The second is raised when the Tile-Binning is complete.
The third is raised because the Vertex and Fragment Shaders requested for
interrupting the host. The fourth is raised for an additional reason - the
completion of Tile-Rendering phase.

The Frame Buffer image remains the same as before,
found [here](/wip/images/d50.png).
