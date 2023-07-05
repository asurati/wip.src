---
title: "Driving the HDMI Display Pipeline"
date: '2021-10-15'
categories:
  - Programming
tags:
  - RPi
  - VC4
  - HDMI
  - Clocks
  - Display
---

The demos have been relying on the firmware to setup the HDMI display pipeline
and to provide them with a frame buffer to draw on.

The program found [here](https://github.com/asurati/x03/blob/main/dev/disp.c)
attempts to take the control of the pipeline from the firmware, and run it
on its own.

Because of the lack of documentation, the Linux `vc4` driver was consulted
to determine the registers to setup and the sequence of steps necessary, to
disable and enable the pipeline.

The program can successfully setup the pipeline to display at resolutions
640x480@60Hz, 800x600@60Hz, and 1920x1080@60Hz.

---

One of the problems faced during this programming was the problem where the
monitor would display a horizontal resolution of +2 pixels. That is, if the
desired resolution were 640x480@60Hz, the monitor would instead run with
642x480@60Hz. The left-most two columns of the pixels were purple/violet in
colour. The first column of the frame buffer was thus at column #2 on the
screen.

A similar problem is described within a bug,
[here](https://bugs.freedesktop.org/show_bug.cgi?id=27452). The bug also
describes the root cause - monitor not being sent the appropriate infoframe
packets.

Following that hint, the program sends a properly configured AVI InfoFrame
corresponding to the resolution being set. It does so by writing into the
appropriate HDMI Packet RAM registers, and asking the HDMI encoder to pull
the infoframe by reading them. With these two steps, the monitor displays the
expected resolutions exactly.

---
