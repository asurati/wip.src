---
title: "QPU Demo: Triangle with MSAA 4x"
date: '2021-10-05'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo works with the same triangle as seen in earlier demos; the demo
utilizes the full 3D pipeline, employs Coordinate and Vertex shaders that
are not merely pass-through but that perform the Perspective and ViewPort
transformations (the Model and Camera/View transformations are assumed to have
been performed outside), and enables 4x multi-sampling for a smoother display
of the triangle.

The Coordinate and Vertex shaders perform the Matrix-Vector multiplication, in
order to calculate the Clip Coordinates of each vertex, perform the Perspective
divide to calculate the Normalized Device Coordinates, and perform the
ViewPort Transformation to calculate the Screen Coordinates relative to the
ViewPort Centre. The Vertex Shader additionally passes through the colour
information of each vertex.

To enable MSAA, set the `Multisample Mode (4x)` flag in the
Tile Binning Mode Control Record (0x70), and in the
Tile Rendering Mode Control Record (0x71).
Additionally, set the `Rasteriser Oversample Mode` to `4x` in the
Configuration Bits Control Record (0x60).

A final change to make is to set the tile size to 32x32. The tile size is
reduced to accommodate multiple samples per pixel within the Tile Buffer.

---
### **QPU Program:**

The shaders are found here:

[Coordinate Shader](https://github.com/asurati/x03/blob/main/demo/d52.cs.h)

[Vertex Shader](https://github.com/asurati/x03/blob/main/demo/d52.vs.h)

The Fragment Shader remains the same as before.

---
### **Running the demo:**

The driver program can be found
[here](https://github.com/asurati/x03/blob/main/demo/d53.c).

The Frame Buffer image is found [here](/wip/images/d53.png). Opening it in an
image viewer that can disable its own anti-aliasing support
(for instance, `feh --force-aliasing`) allows one to see the MSAA effect on
individual pixels, especially those at the left and the right edges of the
triangle, clearly.

---
