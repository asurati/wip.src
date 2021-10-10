---
title: "QPU Demo: Export HVS output to the system RAM"
date: '2021-10-08'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This demo presents a way to configure HVS FIFO2/Channel2 and the Transposer
(TXP) block to gather the scaled/converted/transformed scan-lines into a buffer
inside the system RAM.

The HVS FIFO2 has a choice in how its output is consumed - it can be consumed
either by PV1, or by the TXP block. The RPi firmware, by default, seems to
disable the FIFO2 (`DSP3_MUX`), so that it is not connected to PV1. This is
very fortunate, as the firmware has already done a part of the necessary setup.

The demo is based off of QPU Demo:
[QPU Demo: Triangle with MSAA 4x](/wip/post/2021/10/05/qpu-demo-triangle-with-msaa-4x/). That demo presents a rendering of a triangle stored inside a 640x480 frame
buffer. The default RPi setup (RPi firmware + U-Boot) has the HDMI pipeline
running at 1920x1080 resolution. When the triangle is displayed on the screen,
the image is scaled-up by the HVS to 1440x1080, maintaining the aspect ratio.

This demo here attempts to have the display pipeline write out that scaled
image into a separate, appropriately sized frame buffer in the system RAM.

The format of the HVS registers are shown in 
[HVS and PV Settings](/wip/post/2021/10/07/hvs-and-pv-settings/). The
particular values set can be found in the driver program,
[here](https://github.com/asurati/x03/blob/main/demo/d55.c).

---
### **Transposer Registers:**

|TXP Register|Value|Comment|
|:-----|----:|-------|
|`TXP_DST_POINTER`|`0x404c7ea4`| The bus address of the output frame buffer, into which the TXP will write the scaled pixels.
|`TXP_DST_PITCH`|`0x1680`| The pitch of the output frame buffer. `1440 * 4 = 5760`.
|`TXP_DIM`|`0x43805a0`| `[15:0]=1440`. The Width of the output frame buffer.<br/>`[31:16]=1080`. The Height of the output frame buffer.
|`TXP_DST_CTRL`|`0x545f0d01`| The TXP control register.<br/>`[0]=1`. Go.<br/>`[11:8]=13`. Output in ARGB32 Format.<br/>`[19:16]=0xf`. Byte Enable; `0xf` recommended.<br/>`[20]=1`. Enable writing the Alpha component.

---
### **Running the demo:**

The driver program can be found
[here](https://github.com/asurati/x03/blob/main/demo/d55.c).

The scaled-up image is found [here](/wip/images/d55.png). For comparison, the
source image, which is 640x480 in size, and which is rendered by the 3D
pipeline, is found [here](/wip/images/d53.png).

