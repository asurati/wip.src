---
title: "QPU Demo: Triangle with NV Shader"
date: '2021-10-02'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo presents a method of programming the GPU to display a triangle, with
the interpolation of the colours at the pixels covered by it. The graphics
concepts used here borrow heavily from those of OpenGL.

The GPU pipeline supports running in non-vertex-shading mode, or the NV mode.
In this mode, pre-shaded vertices are presented to the pipeline. That is,
the jobs of coordinate and vertex shader are done outside of the pipeline. In
this demo, those jobs/calculations are done by hand, to produce the pre-shaded
vertices in the format expected by the mode (see the section
*Shaded Vertex Format in Memory* in the specification for the layout of the
format).

The calculations are presented as comments within the driver program
[here](https://github.com/asurati/x03/blob/main/demo/d50.c).

The Frame Buffer is setup to be of dimensions 640x480, with Colour/Pixel
order BGRA8888, also known as 0xaarrggbb, or ARGB32.
The [Mailbox Property Interface](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface) provides information on the steps needed to setup a
frame buffer.

The Tile-Binning Control List and the Tile-Rendering Control List must be
prepared, the Binner must be run first and then the Renderer. The Binner is
run by thread 0 and the Renderer is run by thread 1. The Renderer itself
is multi-threaded when it runs the Fragment Shader - it provides support for two
threads which are cooperatively scheduled.

See *Table 38: Control Record IDs and Data Summary* in the spec for details.

---
### **Tile-Binning Control List:**

Raw bytes:
```
0x70,0x00,0x15,0x42,0x40,0x00,0x00,0x01,0x00,0x00,0x05,0x42,0x40,0x0a,0x08,0x04,
0x06,0x07,0x66,0x00,0x00,0x00,0x00,0x80,0x02,0xe0,0x01,0x60,0x05,0x00,0x00,0x67,
0x00,0x14,0x00,0x0f,0x69,0x00,0x00,0xa0,0x45,0x00,0x00,0x70,0x45,0x41,0x00,0x14,
0x42,0x40,0x21,0x04,0x03,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x04
```

|Byte(s)|Value|Comment|
|-----:|-----:|-------|
|`0x70`| `0x70` |A Tile Binning Mode Configuration Record follows.
|`0x00,0x15,0x42,0x40`| `0x40421500` | The bus address of the Tile Allocation Buffer. The Primitive Tile Binner (PTB) fills this buffer, for each tile, with the information about the primitives which impact the tile.
|`0x00,0x00,0x01,0x00`| `0x10000` | The size of the Tile Allocation Buffer. It is kept large enough to avoid having to deal with the Binner Out of Memory interrupt for the modest binning job of this demo.
|`0x00,0x05,0x42,0x40`| `0x40420500` | The bus address of the Tile State Data Array - a buffer which needs 48 bytes per tile.
|`0x0a`| `0xa` | The width of the render, in units of tiles. The Frame Buffer width is 640pixels, and the width of a tile is 64 pixels, and `640/64 = 10 = 0xa`.
|`0x08`| `0x8` | The height of the render, in units of tiles. The Frame Buffer height is 480pixels, and the height of a tile is 64 pixels, and `480/64 = 7.5`. Rounding up to the nearest integer gives `8`.<br/>It may seem as if the GPU may go beyond the Frame Buffer boundaries, but other factors such as the ClipWindow and the ViewPort settings help keep the GPU within the bounds and prevent any accesses to the memory area not belonging to the Frame Buffer.
|`0x04`| `0x4` | Flags.<br/>MultiSample Mode is `0` = `off` (A later demo enables MSAA 4x).<br/>Tile Buffer 64-bit Colour Depth is `0` = `off`.<br/>Auto-initialize Tile State Data Array is `1` = `on`.<br/>Tile Allocation Initial Block Size is `00` = `32`.<br/>Tile Allocation Block Size is `00` = `32`.
|`0x06`| `0x6` | Start Tile Binning Control Record.
|`0x07`| `0x7` | Increment Semaphore Control Record.<br/> It requests the Binner to signal the semaphore, which is a resource shared by the Binner and the Renderer threads, after the Tile Lists have been flushed. This semaphore helps in synchronizing between the Binning and the Rendering passes/phases. The increment signals the Renderer thread (which *waits* on this semaphore) that the Tile Lists are all written down in the Tile Allocation Buffer, and that the Renderer thread can now fetch those lists and begin the render phase.
|`0x66`| `0x66` | Clip Window Control Record follows.
|`0x00,0x00`| `0x0` | Clip Window Left Pixel Coordinate is `0`. Note that, for the GPU, the coordinate system has the origin at the bottom left, while the display circuitry assumes the top-left origin coordinate system. This inconsistency requires adjusting the Y-coordinate when performing the ViewPort Transform in order to flip the image along the X-axis.
|`0x00, 0x00`| `0x0` | Clip Window Bottom Pixel Coordinate is `0`.
|`0x80, 0x02`| `0x280` | Clip Window Width in pixels is `640`.
|`0xe0, 0x01`| `0x1e0` | Clip Window Height in pixels is `480`.
|`0x60`| `0x60` | Configuration Bits Control Record follows.
|`0x05,0x00,0x00`| `0x5` | Flags.<br/>Enable Forward Facing Primitive is `1` = `on`<br/>Enable Reverse Facing Primitive is `0` = `off`.<br/>Clockwise Primitives is `1` = `on`.<br/> Because of the differences in the coordinate systems, as mentioned before, flipping the primitives across X-axis also changes their orientation. The input provides the primitives in counter-clockwise (CCW) order. Because of the flip, their order turns into clockwise (CW). The GPU defaults to CCW-is-Front policy. The flip requires it to adopt CW-is-Front policy. These flags enable it to do just that.
|`0x67`| `0x67` | ViewPort Offset Control Record follows.
|`0x00,0x14`| `0x1400` | ViewPort Centre X Coordinate.<br/> These X, and the Y below, coordinates are in signed 12.4 fixed point format. The `Xs` and `Ys` Screen Coordinates, which are calculated by the Coordinate and Vertex Shaders, or which are provided to the NV Mode, are relative to the ViewPort Centre.<br/>The float value is `320.0`.
|`0x00,0x0f`| `0xf00` | ViewPort Centre Y Coordinate.<br/>The float value is `240.0`.
|`0x69`| `0x69` | Clipper XY Scaling Control Record follows.
|`0x00,0x00,0xa0,0x45`| `5120.0` | ViewPort Half-Width in 1/16th of pixel.<br/> The ViewPort size is the same as the Frame Buffer size. The Width of the Frame Buffer is 640 pixels. The Half-Width in 1/16th of pixel is `(640/2) * 16.0f = 5120.0f`.
|`0x00,0x00,0x70,0x45`| `3840.0` | ViewPort Half-Height in 1/16th of pixel.<br/> The ViewPort size is the same as the Frame Buffer size. The Height of the Frame Buffer is 480 pixels. The Half-Height in 1/16th of pixel is `(480/2) * 16.0f = 3840.0f`.
|`0x41`| `0x41` | NV Shader State Control Record follows.
|`0x00,0x14,0x42,0x40`| `0x40421400` | The bus address of the NV Shader State Record.
|`0x21`| `0x21` | Vertex Array Primitives Control Record follows.
|`0x04`| `0x4` | Primitive Mode is `4` = `Triangles`.
|`0x03,0x00,0x00,0x00`| `0x3` | The number of vertices is `3`.
|`0x00,0x00,0x00,0x00`| `0x0` | The index of the first vertex is `0`.
|`0x04`| `0x4` | Flush Control Record.

---
### **NV Shader State Record:**

Raw bytes:
```
0x01,0x18,0x00,0x03,0xf0,0x04,0x41,0x40,0x00,0x00,0x00,0x00,0x68,0x05,0x41,0x40
```

|Byte(s)|Value|Comment|
|-----:|-----:|-------|
|`0x01`| `0x1` | Flags.<br> Fragment Shader is Single Threaded is `1` = `on`.
|`0x18`| `0x18` | Shaded Vertex Data Stride.
|`0x00`| `0x0` | Number of Uniforms (not used currently).
|`0x03`| `0x3` | Number of Varyings is `3`, since each vertex provides Red, Green and Blue colour components.
|`0xf0,0x04,0x41,0x40`| `0x404104f0` | The bus address of the code for the Fragment Shader.
|`0x00,0x00,0x00,0x00`| `0x0` | The bus address of the Uniforms array. The Fragment Shader of this demo doesn't need to access Uniforms.
|`0x68,0x05,0x41,0x40`| `0x40410568` | The bus address of the Shaded Vertex Array.

---
### **Tile-Rendering Control List:**

Raw bytes:
```
0x72,0x00,0xff,0xff,0xff,0x00,0xff,0xff,0xff,0x00,0x00,0x00,0x00,0x00,0x71,0x00,
0x00,0xac,0x5e,0x80,0x02,0xe0,0x01,0x04,0x00,0x73,0x00,0x00,0x1c,0x00,0x00,0x00,
0x00,0x00,0x00,0x73,0x00,0x00,0x08,0x11,0x00,0x15,0x42,0x40,0x18,0x73,0x02,0x00,
0x11,0x40,0x15,0x42,0x40,0x18,....,0x73,0x09,0x07,0x11,0xe0,0x1e,0x42,0x40,0x19
```

|Byte(s)|Value|Comment|
|-----:|-----:|-------|
|`0x72`| `0x72` | A Clear Colours Control Record follows.
|`0x00,0xff,0xff,0xff`| `0xffffff00` | (Even Column?) Clear Colour value in ARGB32. `0xffffff00` is Yellow.
|`0x00,0xff,0xff,0xff`| `0xffffff00` | (Odd Column?) Clear Colour value in ARGB32. `0xffffff00` is Yellow.
|`0x00,0x00,0x00`| `0x0` | Clear Zs. The value the Depth Buffer is cleared to, initialized with. Irrelevant, as the Depth Buffer isn't employed in this demo.
|`0x00`| `0x0` | Clear VG Mask. Irrelevant as this is NV Mode, and not VG mode.
|`0x00`| `0x0` | Clear Stencil. Irrelevant as Stencil isn't employed in this demo.
|`0x71`| `0x71` | A Tile Rendering Mode Configuration Record follows.
|`0x00,0x00,0xac,0x5e`| `0x5eac0000` | The bus address of the start of the Frame Buffer. The GPU considers the starting scanline of the Frame Buffer as the bottom-most scanline of the render.
|`0x80,0x02`| `0x280` | Width of the render in pixels is `640`.
|`0xe0,0x01`| `0x1e0` | Height of the render in pixels is `480`.
|`0x04,0x00`| `0x4` | Flags:<br/>MultiSample Mode is `0` = `off`.<br/>Non-HDR Frame Buffer Colour Format is `01` = `RGBA8888`. Though the format being set here is RGBA8888, the Frame Buffer actually is configured with BGRA8888. Because both the formats are quite closely related, a slight adjustment to the Fragment Shader when writing the colour works.
|`0x73`| `0x73` | A Tile Coordinates Control Record follows.
|`0x00`| `0x0` | Tile Column Number is `0`.
|`0x00`| `0x0` | Tile Row Number is `0`.
|`0x1c`| `0x0` | A Store Tile Buffer General Control Record follows.
|`0x00,0x00,0x00,`<br/>`0x00,0x00,0x00`| `0x0` | Flags:<br/>Buffer To Store is `000` = `None`. A `None` write is required to Clear the Frame Buffer, Depth Buffer, and Stencil Buffer.
|`0x73`| `0x73` | A Tile Coordinates Control Record follows.
|`0x00`| `0x0` | Tile Column Number is `0`.
|`0x00`| `0x0` | Tile Row Number is `0`.
|`0x08`| `0x8` | A Wait On Semaphore Control Record.<br/> This control causes the Renderer thread to wait for the Binner thread to signal the flushing of the Tile Lists, before trying to process them. The wait is required only once per frame. Other Tile Control Records that follow this one doesn't need the wait.
|`0x11`| `0x11` | A Branch To Sub-List Control Record follows.
|`0x00,0x15,0x42,0x40`| `0x40421500` | The bus address of the Tile Control List prepared by the Binner.
|`0x18`| `0x18` | A Store Multisample Resolved Tile Colour Buffer Control Record.
|`...`| `...` | Many such Tile Coordinates + Branch To Sub-List + Store Multisample Resolved Tile Colour Buffer records follow, one for each tile.
|`0x73`| `0x73` | A Tile Coordinates Control Record follows.
|`0x09`| `0x9` | Tile Column Number is `9`.
|`0x07`| `0x7` | Tile Row Number is `7`.
|`0x11`| `0x11` | A Branch To Sub-List Control Record follows.
|`0xe0,0x1e,0x42,0x40`| `0x40421ee0` | The bus address of the Tile Control List prepared by the Binner.
|`0x19`| `0x19` | A Store Multisample Resolved Tile Colour Buffer and Signal End of Frame Control Record.

---
### **QPU Program:**

The Fragment Shader follows. It performs the final part of the Varyings
Interpolation on the R, G, and B colour components, applies an opaque Alpha
component, and stores the resultant colour while also signaling the GPU to
unlock the Tile/Frame Buffer so that other QPUs, assigned to run the shader,
can access it.

```
fmul	r0, vary_rd, a15;	# a15 has W.
fadd	r0, r0, r5;

fmul	r1, vary_rd, a15;
fadd	r1, r1, r5;

fmul	r2, vary_rd, a15;
fadd	r2, r2, r5;

li	r3, -, 0xff000000;	# alpha (= 8d)

fmuli	r3, r0, 1f	pm8c;
fmuli	r3, r1, 1f	pm8b;
fmuli	r3, r2, 1f	pm8a;

or	tlb_clr_all, r3, r3	usb;

ori	host_int, 1, 1;
pe;;;
```

The binary code:

```
0x203e303e, 0x100049e0, // fmul	r0, vary_rd, a15;
0x019e7140, 0x10020827, // fadd	r0, r0, r5;

0x203e303e, 0x100049e1, // fmul	r1, vary_rd, a15;
0x019e7340, 0x10020867, // fadd	r1, r1, r5;

0x203e303e, 0x100049e2, // fmul	r2, vary_rd, a15;
0x019e7540, 0x100208a7, // fadd	r2, r2, r5;

0xff000000, 0xe00208e7, // li	r3, -, 0xff000000;

0x209e0007, 0xd16049e3, // fmuli	r3, r0, 1f	pm8c;
0x209e000f, 0xd15049e3, // fmuli	r3, r1, 1f	pm8b;
0x209e0017, 0xd14049e3, // fmuli	r3, r2, 1f	pm8a;

0x159e76c0, 0x50020ba7, // or	tlb_clr_all, r3, r3	usb;

0x159c1fc0, 0xd00209a7, // ori	host_int, 1, 1;
0x009e7000, 0x300009e7, // pe;
0x009e7000, 0x100009e7, // ;
0x009e7000, 0x100009e7, // ;
```

---
### **Running the demo:**

The driver program can be found
[here](https://github.com/asurati/x03/blob/main/demo/d50.c).

There are three interrupts raised. The first is raised when the Tile-Binning is
complete, with the Tile Lists flushed into the Tile Allocation Buffer.
The second and third are raised because the Fragment Shader requested raising
a host interrupt at the end of the program. The third is also raised because of
another reason - when the Renderer has written out all the tiles into the
Frame Buffer.

The two interrupts - Binning Complete and Rendering Complete - have been
enabled within the `V3D_INTENA` register.

The `V3D_DBQITC` print shows the QPUs which were tasked with running the
Fragment Shader.

```
v3dirqh: errstat 1000, intctl 6, dbqitc 0
v3dirqh: errstat 1000, intctl 0, dbqitc ffe
v3dirqh: errstat 1000, intctl 1, dbqitc 1
d50: err 0
```

The Frame Buffer image is [here](/wip/images/d50.png). Notice the jagged
appearance of the two sides of the triangle. A
[later demo](/wip/post/2021/10/05/qpu-demo-triangle-with-msaa-4x/) attempts to
reduce these aliasing artifacts by enabling MSAA 4x.

---
