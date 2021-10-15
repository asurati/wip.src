---
title: "Pixel Clock: PLLH_PIX"
date: '2021-10-14'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - HDMI
  - Clocks
  - Display
---

This post investigates the programming of the HDMI `pixel` clock, performed by
the RPi firmware to push the pixels into the HDMI encoder at that frequency.

Booting with the RPi firmware and the U-Boot, the actual resolution of the
display is 1920x1080@60Hz. The resolution of the frame buffer might be lower -
the HVS takes care of appropriately scaling and positioning the pixels.

Below is the `Modeline` details about the 1920x1080@60Hz resolution. These
details were extracted out of the [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) information gathered from the monitor:

```
Modeline        "Mode 1" 148.500 1920 2008 2052 2200 1080 1084 1089 1125 +hsync +vsync
```

The break-down of the fields:

|Value|Field|Comment|
|-----:|-----:|-------|
|`148.500`|`clk` |The pixel clock frequency/rate, in MHz.
|`1920`|`hdisp` |The Width of the Active Video region of a scanline, in the units of pixels or pixel-clocks.
|`2008`|`hsyncstart` |The point in time, in the units of pixel or pixel-clock, when the HSync Pulse starts.
|`2052`|`hsyncend` |The point in time, in the units of pixel or pixel-clock, when the HSync Pulse ends.
|`2200`|`htotal` |The total time, in the units of pixels or pixel-clocks, of a scanline, including the blanking interval.
||`+hsync` |The HSync pulse is active high.


The sizes (in the units of pixels or pixel-clocks) of the various portions of
a horizontal blanking interval can be calculated as follows:
|Value|Field|Comment|
|-----:|-----:|-------|
|`88`|`hfp` | Horizontal Front Porch = `hsyncstart - hdisp`.
|`44`|`hsw` | Horizontal Sync Width = `hsyncend - hsyncstart`.
|`148`|`hbp` | Horizontal Back Porch = `htotal - hsyncend`.

|Value|Field|Comment|
|-----:|-----:|-------|
|`1080`|`vdisp` |The Height of the Active Video region of a frame, in the units of scanline.
|`1084`|`vsyncstart` |The time, in the units of scanline, when the VSync Pulse starts.
|`1089`|`vsyncend` |The time, in the units of scanline, when the VSync Pulse ends.
|`1125`|`vtotal` |The total time, in the units of scanlines, of a frame, including the blanking intervals.
||`+vsync` |The VSync pulse is active high.

The sizes (in the units of scanlines) of the various portions of a vertical
blanking interval can be calculated as follows:

|Value|Field|Comment|
|-----:|-----:|-------|
|`4`|`vfp` | Vertical Front Porch = `vsyncstart - vdisp`.
|`5`|`vsw` | Vertical Sync Width = `vsyncend - vsyncstart`.
|`36`|`vbp` | Vertical Back Porch = `vtotal - vsyncend`.

The required pixel clock rate is given by `htotal * vtotal * refreshrate`, or
`2200 * 1125 * 60 Hz = 148500000 Hz = 148.5 MHz`.

---


Below is a snippet of [`bcm2708-rpi-b.dtb`](https://github.com/raspberrypi/firmware/blob/master/boot/bcm2708-rpi-b.dtb):

```
hdmi@7e902000 {
	compatible = "brcm,bcm2835-hdmi";
	. . .
	clocks = <0x07 0x10 0x07 0x19>;
	clock-names = "pixel\0hdmi";
	. . .
};

cprman@7e101000 {
	compatible = "brcm,bcm2835-cprman";
	#clock-cells = <0x01>;
	. . .
	phandle = <0x07>;
};
```

The `hdmi` node consumes two clocks, named `pixel` and `hdmi`. The `hdmi` clock
identifies the clock which drives the HDMI State Machine (HSM). The `pixel`
clock is the pixel clock.

The provider of the pixel clock is identified as the clock with ID `16 (0x10)`
under a `phandle` of `7`, the `CPRMAN`, the Clock, Power, and Reset Manager.

Under Linux, the pixel clock is named
[PLLH_PIX](https://lxr.missinglinkelectronics.com/linux/include/dt-bindings/clock/bcm2835.h).
Its input/source is named
[PLLH](https://lxr.missinglinkelectronics.com/linux/include/dt-bindings/clock/bcm2835.h), which is one of the 5 PLLs that themselves source from the main oscillator
running at 19.2 MHz.

The RPi clock tree, and the details about the circuitry that generates the output
for the PLLs, are listed [here](https://elinux.org/The_Undocumented_Pi).

The Linux source code for the function
[`bcm2835_pll_get_rate`](https://lxr.missinglinkelectronics.com/linux/drivers/clk/bcm/clk-bcm2835.c)
shows the register
locations where the clock parameters for a PLL clock are stored; it also shows the
method to calculate the PLL clock's output frequency.

The register `A2W_PLLH_CTRL` has the value `0x2104d` (placed in there by the
RPi firmware). This value implies that
`NDIV` is set to `0x4d`, and `PDIV` is set to 1. The register `A2W_PLLH_FRAC`
has the value `0x58000`. The variable `using_prediv` has the value `0`.

Then, the rate at which the PLLH clock runs is calculated as:

`19.2 MHz * ((0x4d << 20) + 0x58000)) >> 20 = 1485 MHz`.

The PLLH_PIX clock is a divider clock, sourcing its input from the output of
the PLLH clock. The PLLH_PIX clock has a fixed divider of `10`, and another 8-bit
integer divider stored in the register `A2W_PLLH_PIX`.
Together, these dividers divide the input frequency to generate its output
frequency.
With the fixed divider of `10`, and the 8-bit integer divider set to `1`,
the PLLH_PIX clock thus outputs a frequency of
`1485 MHz / (10 * 1) = 148.5 MHz`, which is exactly the desired frequency for
the pixel clock.

The parameters set for the PLLH and the PLLH_PIX clocks, for a few more
standard resolutions are:


|Resolution|Comment|
|-----:|-------|
|`640x480@60Hz`| PLLH: `NDIV=0x1a`, `FDIV=0x40000`.<br/>PLLH_PIX: 8-bit integer divisor set to `2`.
|`800x600@60Hz`| PLLH: `NDIV=0x14`, `FDIV=0xd5556`.<br/>PLLH_PIX: 8-bit integer divisor set to `1`.
