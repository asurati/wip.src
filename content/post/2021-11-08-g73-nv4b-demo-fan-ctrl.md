---
title: "G73/NV4B demo: Fan Control"
date: '2021-11-08'
categories:
  - Programming
tags:
  - Nvidia
  - G73
  - NV4B
  - GeForce 7600 GT
  - GPU
  - GPIO
---

This is a demonstration of controlling the stock fan. The demo is based on the
DCB details pasted on the
[GPU Passthrough, and EDID access](/wip/post/2021/10/02/qpu-demo-nv-shader)
post.

*Warning: Improperly programming a GPU, or a display, may cause harm to the
devices.*

---
### **The DCB: GPIO Assignment Table**

The offset to the GPIO Assignment table itself lies at offset `0xa`
from the start of the DCB.

The DCB starts at offset `0x8dd6` within my BIOS ROM.

```
$ hexdump -C nv.bios.rom

. . .
00008dd0  2f 21 2c 00 71 00 30 19  0a 08 3f 8e cb bd dc 4e  |/!,.q.0...?....N|
00008de0  78 8e 6c 8e d6 8e e2 8e  f0 8e 05 8f 31 51 59 00  |x.l.........1QY.|
. . .
```

The offset to the GPIO Assignment table is thus `0x8e78`.

```
$ hexdump -C nv.bios.rom

. . .
00008e70  0f 0f 0f 0f 0f 0f 0f 0f  31 06 10 02 9e 8e 00 71  |........1......q|
00008e80  e1 70 e0 07 e0 07 e0 07  85 20 a6 20 e0 07 e0 07  |.p....... . ....|
00008e90  29 89 e0 07 e0 07 e0 07  e0 07 e0 07 e0 07 30 04  |).............0.|
00008ea0  00 02 a6 8e cd 8e 30 07  10 02 00 00 00 00 00 00  |......0.........|
. . .
```

The table begins with a header:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e78`|`0x31`| Version of the GPIO Assignment table.
|`0x8e79`|`0x6`| Size of the Header, in bytes. The table entries immediately follow this header.
|`0x8e7a`|`0x10`| Number of entries inside the table.
|`0x8e7b`|`0x2`| Size of each entry, in bytes.
|`0x8e7c`|`0x8e9e`| Offset to the External GPIO Assignment Master table. It happens to contain no entries. See [here](http://download.nvidia.com/open-gpu-doc/DCB/1/DCB-4.0-Specification.html#_external_gpio_assignment_master_table) for parsing details.

Each entry is of 16 bits in size, with the following layout:

```
[15]:	PWM
[14]:	ON Dir
[13]:	ON Data
[12]:	OFF Dir
[11]:	OFF Data
[10:5]:	Function
[4:0]:	Pin#
```

The description of each of the fields is found
[here](http://download.nvidia.com/open-gpu-doc/DCB/1/DCB-4.0-Specification.html#_gpio_assignment_table). The same page also lists the meanings of the values found
in the `Function` field.

The 16 entries of the GPIO Assignment table are:

|Address|Value|PWM|ON Dir|ON Data|OFF Dir|OFF Data|Function|Pin#|Comment
|---:|---:|---:|---:|---:|---:|---:|---:|---:|-----|
|`0x8e7e`|`0x7100`|`0`|`1`|`1`|`1`|`0`|`8`|`0`|See the table below.
|`0x8e80`|`0x70e1`|`0`|`1`|`1`|`1`|`0`|`7`|`1`|See the table below.
|`0x8e82`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e84`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e86`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e88`|`0x2085`|`0`|`0`|`1`|`0`|`0`|`4`|`5`|See the table below.
|`0x8e8a`|`0x20a6`|`0`|`0`|`1`|`0`|`0`|`5`|`6`|See the table below.
|`0x8e8c`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e8e`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e90`|`0x8929`|`1`|`0`|`0`|`0`|`1`|`9`|`9`|See the table below.
|`0x8e92`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e94`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e96`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e98`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e9a`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.
|`0x8e9c`|`0x7e0`|`0`|`0`|`0`|`0`|`0`|`0x3f`|`0`|Unused.


The entries that are linked to a valid function are:

|Index|Value|PWM|ON Dir|ON Data|OFF Dir|OFF Data|Function|Pin#|Comment
|---:|---:|---:|---:|---:|---:|---:|---:|---:|-----|
|`0`|`0x7100`|`0`|`1`|`1`|`1`|`0`|`8`|`0`|Hotplug B signal.
|`1`|`0x70e1`|`0`|`1`|`1`|`1`|`0`|`7`|`1`|Hotplug A signal.
|`5`|`0x2085`|`0`|`0`|`1`|`0`|`0`|`4`|`5`|Voltage Select (VID) Bit 0.
|`6`|`0x20a6`|`0`|`0`|`1`|`0`|`0`|`5`|`6`|Voltage Select (VID) Bit 1.
|`9`|`0x8929`|`1`|`0`|`0`|`0`|`1`|`9`|`9`|Fan Control.

The speed of the fan is controlled by PWM signals.

The duty cycle of a PWM wave is the fraction of its period during which the
signal is high; the fan is supposed to set its speed according to that
fraction. But notice (ON/OFF Data) that the polarity of the corresponding GPIO
pin is reversed; the fan sets its speed according to the fraction of period
the signal is low. The signal is treated as active low. Thus, to set a duty
cycle of 31%, one must actually program the cycle value as 100 - 31 = 69%.

The values to feed into the controller registers depend on PWM divider value.
It is found by reading the [Performance Table](https://download.nvidia.com/open-gpu-doc/BIOS-Information-Table/1/BIOS-Information-Table.html#BIT_PERF_PTRS_v1) of the
BIOS Information Table Specification. See the function `nvbios_perf_fan_parse`,
[here](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/bios/perf.c). The value on my card happens to be `0x2b4 = 692`.

To set the duty cycle to 31%, the value that needs to be programmed is
`692 - (31 * 692 + 99) / 100 = 477 = 0x1dd`, or `692 - ceil(0.31 * 692) = 477`,
or `floor(0.69 * 692) = 477`.

To establish the duty cycle, set these values (the PWM divider and the duty
cycle), as shown below, and then initiate an Enable control signal to activate
the new duty cycle.
The addresses are relative to the `BAR0` base address, and the assumption is
that the fan control is at GPIO pin #9.

```
// Set the values.
[0x15f8] = PWM Divider = 0x2b4
[0x15f4] = Duty Cycle = 0x1dd

// Enable.
[0x15f4] |= 0x80000000
```

At boot, the PWM Divider register contains `0x2b4`, and the Duty Cycle register
contains `0x1dd`, i.e. 31% duty cycle.

---

The functions `nvkm_therm_fan_set_defaults` and `nvkm_fan_update`, both found
[here](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/therm/fan.c), show how to gradually increase or decrease the fan speed over a 
slow gradient.
