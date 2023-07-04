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

### **Preparation: Software**

The demo runs within a bare-bones supervisor-mode framework/kernel,
* that can support memory allocation and mapping of various memory types,
* that has basic drivers for devices such as the interrupt-controller, the
`PixelValve` (`PV`), the `Hardware Video Scaler` (`HVS`), and `V3D`,
* that can bring up other CPUs,
* that can enable and utilize the floating-point unit, and
* that has support for threads.

Such a setup allows, for instance, to run the gpu demo on cpu #3 of the
`RPi3B+`, while the V3D and the PV interrupts occur and are handled on
cpu #0. Such a setup forces upon the system the problem of correctly dealing
with any interprocessor communication required
(for e.g. locks and memory barriers) for correct operation of this tiny system.

It also requires one to deal with caching, both from the CPU side and from the
GPU side. For instance, since the cube rotates 4 degrees every frame, the `MVP`
matrix that is passed to the shaders as `uniform` also changes every frame.
If the matrix is
stored in an inner-cacheable region (as it is, on this setup), the
corresponding CPU cache-lines must be cleared up to the Point of Coherency upon
change, such that, if the GPU were to view the system RAM region housing the
matrix, it sees the updated data. But there's also a `Uniforms Cache (QUC)`
on the *GPU* side that needs to be cleared of stale data, if the GPU is to
successfully retrieve the updated matrix.

The driver for the PV device allows one to receive the `VBLANK` interrupt, so
that, in servicing it, one can ask the HVS to flip the display-list for a
tear-free animation.

The driver for the V3D device allows us to manage the Binning Memory Pool -
the device raises an interrupt when it runs out of its working memory during the
binning phase. Servicing that interrupt allows us to provide the device with
the memory it needs piece by piece from a large, pre-allocated pool.

The setup does have drawbacks, as it isn't a complete OS. For instance,
rotating the cube requires calculation of trigonometric functions, and lack of
a `libc` in this environment forces the use of a pre-built table of fixed values
(here, `sin`/`cos` for every `4°`, starting at `0°`).

It also relies on the RPi firmware to setup the clocks and the initial
display-list. This setup just creates multiple copies of the initial
display-list, each copy with a different, but same-sized frame-buffer region,
to use them as multiple images into which the GPU can render. Once an image is
rendered into, it is sent to the HVS for presenting, and an image that has
already been presented is pulled from HVS to begin rendering the next frame
into.

---

### **Preparation: Vertices, and their Winding Order**

Each of the 6 faces of the cube is viewed from a position where it is facing
the `viewer/eye/camera`, and the coordinates are noted down.

`vkcube` defines a `2x2x2`-sized cube centered at the origin of the
`object-space`. It provides the vertices in a CCW order. The object-space and
the `world-space` coincide - the model-matrix would have been identity, if not
for the requirement of rotating the cube. The camera is at `(0,3,5)` in
the world-space coordinates (assuming the `RHS` coordinate system), looking
right at the origin `(0,0,0)` of the world-space coordinate, without any twists
or turns along its (camera's) Z-axis. As a result, the `+Z` cube face is facing
the camera.

As described in the post
[vc4: Winding Order](https://asurati.github.io/wip/post/2023/06/30/vc4-winding-order/),
this demo chooses to rely on the the default CCW front-winding and the `Y-flip`
of the clip-space coordinates, to draw the cube. This forces the vertices to be
initially provided in the `CW` winding-order; after the multiplication by the
MVP matrix and a Y-flip in the vertex shader, the primitive processing stage
of the GPU sees the default: CCW as front-winding and CW as back-winding.

Below is the vertex and texture coordinate information for the face situated
at `-Z` axis, i.e. the `XY` face situated at `Z=-1`.

```
                   ^+y
         A         |            B
    obj(1,1,-1)    |       obj(-1,1,-1)
    tex(0,1)       |       tex(1,1)
                   |
                   |
     +x<-----------+-------------->
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
this face results in a blank image filled with the clear color.

---

### **Preparation: Model matrix**

`vkcube`, by default, rotates CW on its Y-axis, when viewed from top
(i.e. from a point on +Y-axis towards the origin).
Considering the RHS coordinate system, and the Y-flips needed reconcile the
differences between the GPU and the display pipeline, the matrix that is needed
is the one that describes a *CCW* rotation by a positive angle around the
Y-axis when viewed from top.

The rotation matrix (all matrices here are row-major order) is as below, where
the angle is the previous angle + 4°:

```
    +---                              ---+
    | cos(angle)  0  -sin(angle)  0|
    | 0           1  0            0|
    | sin(angle)  0  cos(angle)   0|
    | 0           0  0            1|
    +---                              ---+
```

---

### **Preparation: View matrix**

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

### **Preparation: Projection matrix**

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

### **Preparation: MVP matrix**

Since the rotation matrix changes every frame, the MVP matrix is filled in by
multiplying the VP matrix (projection * view in that order) and the rotation
matrix R, like so: VP * R in that order.

### **Preparation: Binner Control List**
