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
