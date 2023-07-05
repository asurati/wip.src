---
title: "vc4: cube demo - 'from scratch'"
date: '2023-07-04'
categories:
  - Programming
tags:
  - RPi
  - vc4
  - Display
  - GPU
---

*Warning: Improperly programming a GPU, or a display, may damage the devices.*

This post demonstrates a partial recreation of the
[`vkcube`](https://github.com/KhronosGroup/Vulkan-Tools/blob/main/cube/cube.c)
application, but
directly on the `vc4` GPU, without involving any of the graphics APIs. It is a
partial recreation, as the only behavior it demonstrates is a
[`LunarG`](https://github.com/KhronosGroup/Vulkan-Tools/blob/main/cube/lunarg.ppm.h)
cube, rotating on its Y-axis.

---

### **Software:**

The demo runs within a bare-bones supervisor-mode framework/kernel,
* that can support memory allocation and mapping of various memory types,
* that has basic drivers for devices such as the interrupt-controller, the
`PixelValve` (`PV`), the `Hardware Video Scaler` (`HVS`), and `V3D`,
* that can bring up other CPUs,
* that can enable and utilize the floating-point unit, and
* that has support for threads.

Such a setup allows, for instance, to run the gpu demo on cpu #3 of the
`RPi3B+`, while the V3D and the PV interrupts occur and are handled on
cpu #0. Such a setup forces upon this tiny system the problem of correctly
dealing with any inter-processor communication required
(for e.g. locks and memory barriers) for correct operation.

It also requires one to deal with caching, both from the CPU side and from the
GPU side. For instance, since the cube rotates `4°` every frame, the `MVP`
matrix that is passed to the shaders as `uniform` also changes every frame.
If the matrix is
stored in an
[inner-shareable
region](https://developer.arm.com/documentation/den0024/a/Memory-Ordering/Memory-attributes/Cacheable-and-shareable-memory-attributes)
(as it is, on this setup), the
corresponding CPU cache-lines must be cleared up to the Point of Coherency upon
change, such that, if the GPU were to view the system RAM region housing the
matrix, it would see the updated data. But there's also a `Uniforms Cache (QUC)`
on the *GPU* side that needs to be invalidated of stale data, if the GPU is to
successfully retrieve the updated matrix from the system RAM.

The driver for the PV device allows one to receive the `VBLANK` interrupt, so
that, in servicing it, one can ask the HVS to flip the display-list for a
tear-free animation.

The driver for the V3D device allows one to manage the Binning Memory Pool -
the device raises an interrupt when it runs out of its working memory during the
binning phase. Servicing that interrupt allows one to provide the device with
the memory it needs piece by piece from a large, pre-allocated pool.

The setup does have drawbacks, as it isn't a complete OS. For instance,
rotating the cube requires calculating trigonometric functions, and the lack of
a maths library in this environment forces the use of a pre-built table of
fixed values (here, `sin`/`cos` for every `4°`, starting at `0°`).

It also relies on the RPi firmware to setup the clocks and the initial
display-list. This setup just creates multiple copies of the initial
display-list, each copy with a different, but same-sized frame-buffer region,
to use them as multiple images into which the GPU can render. Once an image is
rendered into, it is sent to the HVS for presenting, and an image that has
already been presented is pulled from the HVS to begin rendering the next frame
into.

---

### **Vertices, and their Winding Order:**

Each of the 6 faces of the cube is viewed from a position where it faces us,
the coordinates are noted down.

`vkcube` defines a `2x2x2`-sized cube, centered at the origin of the
`object-space`. It lists the vertices in a CCW order. The object-space and
the `world-space` coincide - the model-matrix would have been identity, if not
for the requirement of rotating the cube. The `eye/camera` is at `(0,3,5)` in
the world-space coordinates (assuming the `RHS` coordinate system), looking
right at the origin `(0,0,0)` of the world-space coordinate, without any twists
or turns along its (camera's) Z-axis. As a result, the `+Z` cube face is facing
the camera.

As described in the
[vc4: Winding Order](/wip/post/2023/06/30/vc4-winding-order/) post,
this demo chooses to rely on the the default CCW front-winding and the `Y-flip`
of the clip-space coordinates, to draw the cube. This forces the vertices to be
initially provided in the `CW` winding-order; after the multiplication by the
MVP matrix and a Y-flip in the coordinate and vertex shaders, the primitive
processing stage of the GPU sees the default: CCW as front-winding and CW as
back-winding.

Below is the vertex and texture coordinate information for the face situated
at `-Z` axis, i.e. the `XY` face situated at `Z=-1`.

```
                   ^ +y
         A         |            B
    obj(1,1,-1)    |       obj(-1,1,-1)
    tex(0,1)       |       tex(1,1)
                   |
                   |
    +x <-----------+-------------->
                   |
                   |
    tex(0,0)       |       tex(1,0)
    obj(1,-1,-1)   |       obj(-1,-1,-1)
        D          |          C

```

``` c
    // Vertex Coordinates, in CW order.
    1, 1, -1,		// A
    -1, -1, -1,		// C
    1, -1, -1,		// D

    1, 1, -1,		// A
    -1, 1, -1,		// B
    -1, -1, -1,		// C

    // Texture Coordinates, corresponding to the 6 vertices.
    0, 1,
    1, 0,
    0, 0,
    0, 1,
    1, 1,
    1, 0,
```

This CW winding-order, when looked at from the point-of-view of the camera,
turns into a CCW ordering; that turn occurs after multiplication by the MVP
matrix (the View Matrix, specifically) when running the vertex shader.
The clip-space Y-flip then turns the ordering back into the CW winding-order.
As a result, this `-Z` face is treated as a back-face by the primitive
processing stage and is culled, as it should be; asking the GPU to draw only
this face results in a blank image filled with the clear color. Of course, due
to the rotation, when the -Z face happens to face the camera, it is rendered as
expected.

---

### **Model matrix:**

`vkcube`, by default, rotates CW on its Y-axis, when viewed from top
(i.e. from a point on +Y-axis towards the origin).
Considering the RHS coordinate system, and the Y-flips needed reconcile the
differences between the GPU and the display pipeline, the matrix that is needed
is the one that describes a *CCW* rotation by a positive angle around the
Y-axis when viewed from top.

The rotation matrix (all matrices here are row-major order) is as below, where
the angle is the previous angle + 4°:

```
    +---                        ---+
    | cos(angle)  0  -sin(angle)  0|
    | 0           1  0            0|
    | sin(angle)  0  cos(angle)   0|
    | 0           0  0            1|
    +---                        ---+
```

---

### **View matrix:**

The coordinate transformation, based on the properties defined by `vkcube`,
is as shown below:

```
    eye: (0, 3, 5)
    lookat/center: (0, 0, 0)
    up: (0, 1, 0)

    view.zvec = normalize(eye - lookat)
              = normalize(0, 3, 5)
              = (0, 0.5145, 0.8575)

    view.xvec = normalize(up cross view.zvec)
              = (1, 0, 0)

    view.yvec = view.zvec cross view.xvec
              = (0, 0.8575, -0.5145)

    view matrix = basis-vectors * translation

    +---                ---+   +---      ---+
    | 1  0        0       0|   | 1  0  0   0|
    | 0  0.8575  -0.5145  0| * | 0  0  0  -3| =
    | 0  0.5145   0.8575  0|   | 0  0  0  -5|
    | 0  0        0       1|   | 0  0  0   1|
    +---                ---+   +---      ---+

    +---                     ---+
    | 1  0        0        0    |
    | 0  0.8575  -0.5145   0    |
    | 0  0.5145   0.8575  -5.831|
    | 0  0        0        1    |
    +---                     ---+
```

---

### **Projection matrix:**

`vkcube` describes a perspective projection by setting the Y-FOV to 45°,
the aspect ratio to 1, the near-plane at `Z = -0.1` (in the eye-space) and
the far-plane at `Z = -100` (again, in the eye-space). The aspect ratio is 1
because the initial window that `vkcube` creates is a square window. But this
demo has RPi running at a `800 x 600` resolution. The aspect ratio is adjusted
accordingly.

The calculation of the width and height of the near-plane, based on the Y-FOV
and the aspect-ratio:

```
    The eye-space coordinate system:
                                   ^ +y
                                   |
                                   |
                                **** (0,t,-0.1)
                          ***      |
     (0,0,0)       ****            |
     eye      ***                  |
          **                       |
      E *--------------------------+----------------------> -z
        <----------- 0.1 --------->| (0,0,-0.1)
                                   |
                                   |
                                   |
                                   * (0,-t,-0.1)
                                   |
                                   |
                                   v -y

    The triangle formed by (0,0,0) (0,0,-0.1) and (0,t,-0.1) has 45°/2 = 22.5°
    angle at the point E.

    tan(22.5°) = 0.4142 = t / 0.1.
    Hence, t = 0.04142 units.

    The height of the near-plane is thus 0.04142 * 2 = 0.08284 units.
    The aspect ratio of the viewport, and therefore of the near-plane, is
    800/600. This gives the width of the near-plane as
    800 * 0.08284 / 600 = 0.11045
```

The calculation of a perspective projection matrix is described, in great
detail, [here](http://www.songho.ca/opengl/gl_projectionmatrix.html). Based on
its calculations, and those above, the perspective projection matrix is:

```
    +---                             ---+
    | n/r  0     0             0        |
    | 0    n/t   0             0        |
    | 0    0    -(f+n)/(f-n)  -2fn/(f-n)|
    | 0    0    -1             0        |
    +---                             ---+

    where,
    n = 0.1,
    f = 100,
    r = 0.11045 / 2 = 0.05523,
    t = 0.04142,
    resulting in,

    +---                         ---+
    | 1.8106  0       0       0     |
    | 0       2.4143  0       0     |
    | 0       0      -1.002  -0.2002|
    | 0       0      -1       0     |
    +---                         ---+

```

---

### **MVP matrix:**

Since the rotation matrix changes every frame, the MVP matrix is filled in by
multiplying the VP matrix (projection * view in that order) and the rotation
matrix R, like so: VP * R in that order.

---

### **Control Lists:**

The Binner Control List must be provided with an initial Tile Allocation Memory,
even if it is just one page. If the Tile Allocation Memory Base and Size are
both kept 0, and even if the `OUTOMEM` irq handler hands out pages when needed,
the renderer thread enters an error condition as signaled by the `CT1CS.CTERR`
bit.

There are two attribute arrays, one that stores the vertex coordinate
information, and the other stores the texture coordinate information.
The binner needs access to
only the vertex coordinate information to build the tile-lists. The varyings
(such as the texture coordinates) are needed later by the renderer when the
vertex shader runs. By separating the attributes, one can avoid loading
unnecessary data that pollutes the caches.

---

### **Texture:**

The
[`LunarG`](https://github.com/KhronosGroup/Vulkan-Tools/blob/main/cube/lunarg.ppm.h)
texture is converted to the T-format as described in the
[vc4: T-format textures](/wip/post/2023/06/27/vc4-t-format-textures/) post.

Since the linear format, from which the T-format buffer is created, has the
start of the texture buffer storing the top-row of the image instead of the
bottom row, the texture configuration has the `FLIPY` bit enabled to let the
GPU know that it must compensate for the reversed ordering.

If the T-format buffer were to be created from a linear format that had the
image vertically flipped, then the `FLIPY` bit would not need to be set.

---

### **Coordinate and Vertex shaders:**

The coordinate and vertex shaders share similar code.
The format of the coordinate shader output is the described by the
`Shaded Coordinates Format in VPM for PTB` in the
[V3D Architecture Reference Guide](https://docs.broadcom.com/doc/12358545)

The same guide also describes the format of the vertex shader output,
`Shaded Vertex Format in VPM for PSE`.

The vertex shader outputs 5 varyings: the `xyz` clip-coordinates and the `st`
texture-coordinates for each shaded vertex. These varyings are then
interpolated and provided to the fragment shader.

---

### **Pixel and Element relation:**

GPUs seem to prefer rendering pixels in a group of aligned `2x2` block of
pixels, also called a pixel-quad.

With `vc4` GPU too, each QPU processes a pixel-quad when running fragment
shaders. Not only that, since each QPU is considered to be a 16-way SIMD
processor, it processes aligned blocks of `4x4` pixels, one pixel-quad inside
it at a time.

Within an aligned block of `4x4` pixels, which SIMD-element, out of the 16
SIMD-elements of a QPU, is responsible for which pixel, can be known by running
a series of the following fragment shaders:

``` c
    0x159a7d80, 0x10020827, /* or       r0, element_number, element_number; */
    0xff000000, 0xe0020867, /* li       r1, -, 0xff000000; */
    0x0d9c01c0, 0xd00228a7, /* sub      r2, r0, 0 sf; */
    0xffffffff, 0xe0040867, /* li.zs.never r1, -, 0xffffffff; */
    0x159e7240, 0x10020ba7, /* or       tlb_color_all, r1, r1; */
    0x009e7000, 0x500009e7, /* score_board_unlock; */
    0x009e7000, 0x300009e7, /* program_end; */
    0x009e7000, 0x100009e7, /* ;        */
    0x009e7000, 0x100009e7, /* ;        */
```

The shader outputs the color white if the SIMD-element on which this shader
instance is running is 0. The rest (15 in number) of the shader instances,
running on the same QPU as the instance with SIMD-element 0, color their pixel
black. In the rendered frame-buffer, the lone white pixel in a block of aligned
`4x4` pixels reveals the position, within the `4x4` pixels block, of the
SIMD-element that was responsible for coloring the white pixel.

By running such a series of fragment shaders, one for each SIMD-element, the
following pattern emerges, at least around the origin.

```
    ^ +y
    |
    |.
   7|.
   6|.
   5|o...
   4|e...
   3|o...
   2|e...
   1|ooo...
   0|eee...
    +-------------------------> +x
    frame-buffer
    origin
```

A point marked `e` denotes an aligned block of `4x4` pixels that is at an even
Y-position, while a point marked `o` denotes a similar block that is
at an odd Y-position.

The relation between an even block and a QPU's elements is
shown below. Pixels positions are implicit, while the numbers denote the
particular SIMD-element number that was responsible for coloring the
corresponding pixel white.

```
    // Even block

    6   7   10   11
    4   5   08   09
    2   3   14   15
    0   1   12   13
```

Similarly, for odd block of aligned `4x4` pixels.

```
   // Odd block

   10   11   6   7
   08   09   4   5
   14   15   2   3
   12   13   0   1
```

Within each aligned `4x4` block of pixels assigned to a QPU, the QPU processes
one pixel-quad at a time (since, although a QPU is considered to be a
16-way SIMD processor, physically it is a 4-way SIMD processor multiplexed
4x over 4 clock cycles).

Since `vkcube` relies on `dFdx` and `dFdy` in its fragment shader
to perform lighting calculations, the exact layout of a pixel-quad
is needed. From the above layouts, one can derive a finer relation between a
pixel-quad and the 4 physical QPU SIMD-elements:

```
    ^ +y
    |
    |
    |     **   <-- pixel-quad
    |     **
    |
    |
    |
    |
    +---------------------> +x
    frame-buffer
    origin

    // pixel-quad from above, blown up in size, below:

    *       *
    2       3

    *       *
    0       1
```

The number below each pixel is given by the expression `element_numer & 3`.

The 4 pixel-quads, within each aligned `4x4` block of pixels, are each
processed by [similar SIMD-elements](https://github.com/anholt/mesa/issues/12).

This fact is exploited by `vkcube` fragment shader to calculate the partial
derivatives of the clip-space-position varying, with respect to the X and the Y
directions.

For each (marked by `*`) of the 4 pixels of a pixel-quad, the `dX` and `dY`
vectors of a per-pixel or per-fragment varying function
(such as the clip-space-position in `vkcube`) are:

```
    ^ +y                                 ^ +y
    |                                    |
    |       dX                           |         dX
    |     *------->.                     |     .------->*
    |     ^                              |              ^
    |   dY|                              |              | dY
    |     |                              |              |
    |     .        .                     |     .        .
    |                                    |
    +---------------------> +x           +---------------------> +x
    frame-buffer                         frame-buffer
    origin                               origin



    ^ +y                                 ^ +y
    |                                    |
    |                                    |
    |     .        .                     |     .        .
    |     ^                              |              ^
    |   dY|                              |              | dY
    |     |                              |              |
    |     *------->.                     |     .------->*
    |       dX                           |       dX
    +---------------------> +x           +---------------------> +x
    frame-buffer                         frame-buffer
    origin                               origin
```

In each case, the cross product of `dX` and `dY`, in that order, results in the
surface normal at the given pixel.

---

### **Fragment shader:**

As described [here](https://github.com/anholt/mesa/issues/12), the fragment
shader can rely on the `element_number` of SIMD-element running an instance of
the shader, and on the `mul rotation` feature of the QPU, to calculate the
derivatives.

While testing the fragment shader, if the rotations were calculated as shown
below, the rendered output had black pixels due to negative dot products of the
surface normal and the light vector. The rendering was as if a fine grill of
black pixels were laid on top of the cube.

``` c
    /* func_dfdx: */
    . . .
    . . .
    0x809ff000, 0xd00099c0, /* v8min.zs a0, r0, rot15; */
    0x809ff009, 0xd00099c1, /* v8min.zs a1, r1, rot15; */
    0x809ff012, 0xd00099c2, /* v8min.zs a2, r2, rot15; */
    0x02027c00, 0x10040027, /* fsub.zs  a0, a0, r0; */
    0x02067c40, 0x10040067, /* fsub.zs  a1, a1, r1; */
    0x020a7c80, 0x100400a7, /* fsub.zs  a2, a2, r2; */
    . . .
    . . .
```

After hours of debugging, the pattern shown below, worked. It seems that
back-to-back rotation calculations do not give accurate results. The `nop`
after each rotation instruction is required, since otherwise, the `fsub`
instruction would be reading from a location in A-reg-file that was written to
by the immediately preceding rotation instruction.

``` c
    /* func_dfdx: */
    . . .
    . . .
    0x809ff000, 0xd00099c0, /* v8min.zs a0, r0, rot15; */
    0x009e7000, 0x100009e7, /* ;        */
    0x02027c00, 0x10040027, /* fsub.zs  a0, a0, r0; */

    0x809ff009, 0xd00099c1, /* v8min.zs a1, r1, rot15; */
    0x009e7000, 0x100009e7, /* ;        */
    0x02067c40, 0x10040067, /* fsub.zs  a1, a1, r1; */

    0x809ff012, 0xd00099c2, /* v8min.zs a2, r2, rot15; */
    0x009e7000, 0x100009e7, /* ;        */
    0x020a7c80, 0x100400a7, /* fsub.zs  a2, a2, r2; */
    . . .
    . . .
```

> The `v8min.zs a0, r0, rot15;` instruction is really encoded as
> `v8min.zs a0, r0, r0, rot15;`, but the parser in my assembler isn't complete
> enough to parse the latter expression.


---

### **Result:**
[Here](https://drive.google.com/file/d/1Vik78wRVhf6n42fvoIMDjL02OuKxzR8v/view?usp=sharing)
is a video capture of the `RPi3B+` booting and spinning the cube. The
rendering is a bit darker, for some reason, than that of the `vkcube` running
with `mesa` on an `Intel IvyBridge` machine.

> It seems that Google Drive allows online playback of the video only at 360p,
> even though the video resolution is 800x600. Downloading the file first and
> then playing it in the browser, displays the video at its original
> resolution.

---
