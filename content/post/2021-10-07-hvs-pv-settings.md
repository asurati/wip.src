---
title: "HVS and PV Settings"
date: '2021-10-07'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - V3D
---

This post investigates the contents of the registers belonging to the
Hardware Video Scaler (HVS), and to the PixelValve (PV) devices.

When a request is made to the
[MailBox](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface#allocate-buffer)
interface for a Frame Buffer with required properties, the RPi firmware
programs the HVS and the PV in order to drive the display accordingly.

The demos have been requesting a resolution of 640x480, 32bpp, with a default
pixel order of BGRA8888 (0xaarrggbb, or ARGB32).

---
### **HVS Registers:**

HVS has 3 FIFOs/channels which it fills with
scaled/transformed/converted/composited pixels. Each FIFO is connected to an
HVS output, and each such output is connected to a PixelValve.

These mappings are quite static within VideoCoreIV, with one-to-one mapping
between the HVS FIFOs/channels and the HVS outputs, and between the HVS outputs
and the PixelValves.

The mapping between the HVS FIFOs and the PixelValves is:

FIFO0/channel0 with PV0.

FIFO1/channel1 with PV2.

FIFO2/channel2 with PV1.

The HDMI display is driven by PV2. The same PixelValve PV2 can also drive the
SDTV output instead.

The (FIFO2, PV1) pairing can be broken to allow the transposer (TXP) to
redirect the corresponding HVS output into the system RAM.

The list of HVS registers, and their contents.

|HVS Word|Comment|
|:-----|-------|
|`[0x0]=0x9a0c0fff`| `DISPCTRL` register.<br/>`[31]` = `1`. Enable bit.<br/>`[19:18]` = `3`. DSP3 output (connected to PV1) is disabled. This setting breaks the pairing between FIFO2 and PV1.<br/> `[15:0]` = `0xfff`. Enable interrupts corresponding to various events.
|`[0x1]=0`| `DISPSTAT` register.
|`[0x2]=0x64647276`| `ID` register.<br/>The ASCII string is `vrdd`.
|`[0x8]=0x0`| `DISPLIST0` register.<br/>The index into the DisplayList Area (itself at index 0x800 from the start of the HVS register area) where the DisplayList for FIFO0/channel0 resides.
|`[0x9]=0x664`| `DISPLIST1` register.<br/>Same as above, but for FIFO1/channel1.
|`[0xa]=0x0`| `DISPLIST2` register.<br/>Same as above, but for FIFO2/channel2.
|`[0xc]=0x0`| `DISPLACT0` register.<br/>The index into the DisplayList Area where the Active DisplayList for FIFO0/channel0 resides.
|`[0xd]=0x994`| `DISPLACT1` register.<br/>Same as above, but for FIFO1/channel1.
|`[0xe]=0x0`| `DISPLACT2` register.<br/>Same as above, but for FIFO2/channel2.

The DisplayLists at index 0x0:

|HVS Word|Comment|
|:-----|-------|
|`[0x800]=0x80000000`| `CTL0` register.<br/>`[31]` = `1`. Marks the end of the list.

The DisplayLists at index 0x664. This list corresponds to the frame buffer setup by U-Boot.

|HVS Word|Comment|
|:-----|-------|
|`[0xe64]=0x4e007807`| `CTL0` register.<br/>`[30]` = `1`. Valid list.<br/>`[29:24]` = `14`. The number of valid words in this list, excluding the word that marks the end of the list.<br/>`[21:20]` = `0`. Linear Tiling.<br/>`[16]` = `0`. No horizontal flip.<br/>`[15]` = `0`. No vertical flip.<br/>`[14:13]` = `3`. Pixel Order is ABGR32. This value is in conflict with the actual frame buffer order ARGB32. Perhaps the HVS is converting from the input format.<br/>`[12:11]` = `3`. RGBA rounding.<br/>`[10:8]` = `0`. SCL1 (CbCr plane) scaling. Set to the same as SCL0 scaling.<br/>`[7:5]` = `0`. SCL0 scaling (RGB, or Y plane). Horizontal and Vertical scaling is to be performed by a Polyphase Filter (PPF).<br/>`[4]` = `0`. Unity not set.<br/>`[3:0]` = `7`. Pixel Format is RGBA8888. This value is in conflict with the actual frame buffer format BGRA8888.
|`[0xe65]=0xff016000`| `POS0` register.<br/>`[31:24]` = `0xff`. Value of alpha when the alpha is considered fixed.<br/>`[23:12]` = `22`. Pixel position in the Y direction where the first row of pixels starts.<br/>`[11:0]` = `0`. Pixel position in the X direction where the first column of the pixels starts.
|`[0xe66]=0x40b0780`| `POS1` register.<br/>`[27:16]` = `1035`. Destination/Output Height in pixels.<br/>`[11:0]` = `1920`. Destination/Output Width in pixels.
|`[0xe67]=0x43d80720`| `POS2` register.<br/>`[31:30]` = `1`. Source Alpha Mode is Fixed Mode.<br/>[27:16] = `984`. Source/Input Height in pixels.<br/>`[11:0]` = `1824`. Source/Input Width in pixels.
|`[0xe68]=0x3d70000`| `POS3` register.<br/>Context, written by HVS.
|`[0xe69]=0x9e513000`| `POINTER` register.<br/>The bus address of the source/input frame buffer.
|`[0xe6a]=0x9ebe9f80`| `POINTER CONTEXT` register.<br/>Written by HVS.
|`[0xe6b]=0x1c80`| `PITCH0` register.<br/>`1824 * 4` = `7296`.
|`[0xe6c]=0x8300`| `LBM` register.<br/>Line Buffer Memory, utilized as temporary storage by HVS when performing the scaling/conversion.
|`[0xe6d]=0x40f33261`| `CH0-HPPF0` register.<br/>The scaling factor for Horizontal scaling using a PPF, for channel0.<br/>`[24:8]` = `0xf332`. Source Width / Destination Width * 2^16.
|`[0xe6e]=0x40f36260`| `CH0-VPPF0` register.<br/>The scaling factor for Vertical scaling using a PPF, for channel0.<br/>`[24:8]` = `0xf362`. Source Height / Destination Height * 2^16.
|`[0xe6f]=0x17d36`| `CH0-VPPF1` register.<br/>The context register for the PPF scaler.
|`[0xe70]=0xff4`| The index into the DisplayList Area where the PPF kernel parameters for horizontal scaling are stored. For the RGB plane.
|`[0xe71]=0xff4`| The index into the DisplayList Area where the PPF kernel parameters for vertical scaling are stored. For the RGB plane.
|`[0xe72]=0x80000000`| `CTL0` register.<br/>`[31]` = `1`. Marks the end of the list.

The PPF being utilized is the Mitchell-Netravali filter. The parameters are packed as 9-bit signed integers - 3 such integers per word. Each such integer is the value of the function, multiplied by 256, at some x in [-2, 2].

|HVS Word|
|:-----|
|`[0x17f4]=7ebfc00`
|`[0x17f5]=7e3edf8`
|`[0x17f6]=4805fd`
|`[0x17f7]=1dca432`
|`[0x17f8]=355769b`
|`[0x17f9]=1c6e3`
|`[0x17fa]=355769b`
|`[0x17fb]=1dca432`
|`[0x17fc]=4805fd`
|`[0x17fd]=7e3edf8`
|`[0x17fe]=7ebfc00`


The DisplayLists at index 0x994. This list corresponds to the frame buffer setup by the demos.

|HVS Word|Comment|
|:-----|-------|
|`[0x1194]=0x4e007807`| `CTL0` register.
|`[0x1195]=0xff0000f0`| `POS0` register.<br/>`[23:12]` = `0`. Pixel position in the Y direction where the first row of pixels starts.<br/>`[11:0]` = `240`. Pixel position in the X direction where the first column of the pixels starts.
|`[0x1196]=0x43805a0`| `POS1` register.<br/>`[27:16]` = `1080`. Destination/Output Height in pixels.<br/>`[11:0]` = `1440`. Destination/Output Width in pixels.
|`[0x1197]=0x41e00280`| `POS2` register.<br/>[27:16] = `480`. Source/Input Height in pixels.<br/>`[11:0]` = `640`. Source/Input Width in pixels.
|`[0x1198]=0x8d0000`| `POS3` register.<br/>Context, written by HVS.
|`[0x1199]=0x9eac0000`| `POINTER` register.<br/>The bus address of the source/input frame buffer.
|`[0x119a]=0x9eb52e00`| `POINTER CONTEXT` register.<br/>Written by HVS.
|`[0x119b]=0xa00`| `PITCH0` register.<br/>`640 * 4` = `2560`.
|`[0x119c]=0xa800`| `LBM` register.<br/>Line Buffer Memory, utilized as temporary storage by HVS when performing the scaling/conversion.
|`[0x119d]=0x4071c660`| `CH0-HPPF0` register.<br/>The scaling factor for Horizontal scaling using a PPF, for channel0.<br/>`[24:8]` = `0x71c6`. Source Width / Destination Width * 2^16.
|`[0x119e]=0x4071c660`| `CH0-VPPF0` register.<br/>The scaling factor for Vertical scaling using a PPF, for channel0.<br/>`[24:8]` = `0x71c6`. Source Height / Destination Height * 2^16.
|`[0x119f]=0xc000b46a`| `CH0-VPPF1` register.<br/>The context register for the PPF scaler.
|`[0x11a0]=0xff4`| The index into the DisplayList Area where the PPF kernel parameters for horizontal scaling are stored. For the RGB plane.
|`[0x11a1]=0xff4`| The index into the DisplayList Area where the PPF kernel parameters for vertical scaling are stored. For the RGB plane.
|`[0x11a2]=0x80000000`| `CTL0` register.<br/>`[31]` = `1`. Marks the end of the list.


---
### **PixelValve Registers:**

|PV Word|Comment|
|:-----|-------|
|`[0x0]=0x177005`| `CONTROL` register.<br/>`[23:21]` = `0`. Format 24.<br/>`[20:15]` = `46`. FIFO Level.<br/>`[14]` = `1`. Clear at Start.<br/>`[13]` = `1`. Trigger Underflow.<br/>`[12]` = `1`. Wait for HStart.<br/>`[5:4]` = `0`. No Pixel Repetition.<br/>`[3:2]` = `1`. Pixel Clock select. Selects HDMI encoder clock.<br/>`[0]` = `1`. PV Enable.
|`[0x1]=0x3`| `V_CONTROL` register.<br/>`[1]` = `1`. Continuous, as opposed to Interlaced signal.<br/>`[0]` = `1`. Video Enable.
|`[0x3]=0x94002c`| `HORZA` register.<br/>`[31:16]` = `148`. Horizontal Back Porch, in pixels.<br/>`[15:0]` = `44`. Horizontal Sync, in pixels.
|`[0x4]=0x580780`| `HORZB` register.<br/>`[31:16]` = `88`. Horizontal Front Porch, in pixels.<br/>`[15:0]` = `1920`. Horizontal Active, in pixels.
|`[0x5]=0x240005`| `VERTA` register.<br/>`[31:16]` = `36`. Vertical Back Porch, in lines.<br/>`[15:0]` = `5`. Vertical Sync, in lines.
|`[0x6]=0x40438`| `VERTB` register.<br/>`[31:16]` = `4`. Vertical Front Porch, in lines.<br/>`[15:0]` = `1080`. Vertical Active, in lines.

---
