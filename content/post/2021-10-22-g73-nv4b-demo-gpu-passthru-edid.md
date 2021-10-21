---
title: "G73/NV4B demo: GPU Passthrough, and EDID access"
date: '2021-10-22'
categories:
  - Programming
tags:
  - Nvidia
  - G73
  - NV4B
  - GeForce 7600 GT
  - Display
  - GPU
---

This demo utilizes the [VFIO](https://www.kernel.org/doc/Documentation/vfio.txt)
feature of the Linux kernel to pass through a
[GeForce 7600 GT](https://www.techpowerup.com/gpu-specs/geforce-7600-gt.c152)
GPU to a guest VM running on the host. Then, the program running inside the
guest VM configures a DDC (an I<sup>2</sup>C) bus on the GPU to access the
EDID blocks of the attached display.

The details about parsing the Video BIOS, and about driving the DDC bus were
found by reading the
[DCB Specification](http://download.nvidia.com/open-gpu-doc/DCB/1/DCB-4.0-Specification.html)
and, the source code for the
[`nouveau`](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/)
driver.

*Warning: Improperly programming a GPU, or a display, may cause harm to the
devices.*

---
### **Hardware Setup:**

The GPU supports 2 dual-link DVI-I ports, and 1 S-Video/TV-out port. A HDMI
display is connected to one of the DVI-I ports, with a DVI-to-HDMI cable. The
cable also works as a HDMI-to-DVI cable (for instance, when connecting a RPi1B
to a DVI display), thanks to the similarities between the two protocols.

The GPU card is inserted into a PCI Express x16 slot on the host machine. The
host machine has its own display running over an integrated GPU.

The CPU and the motherboard on the host machine must both support CPU
virtualization and IOMMU. The BIOS may have these features disabled - they
must be enabled in the BIOS.

---
### **Host Machine Software Setup:**

There are several articles on the internet, about configuring a Linux host
machine to pass through a spare PCI/PCIe GPU to a guest VM running on that host.

For instance, [this](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
article, although about setting up a *UEFI*-based VM, does present the necessary,
initial steps for configuring the host, regardless of the type (UEFI, or BIOS)
of the guest VM that will ultimately receive the GPU.

The `lspci -nnv` command on my host machine shows the GPU as:

```
# lspci -vnn

01:00.0 VGA compatible controller [0300]: NVIDIA Corporation G73 [GeForce 7600 GT] [10de:0391] (rev a1) (prog-if 00 [VGA controller])
        Flags: fast devsel, IRQ 16, IOMMU group 1
        Memory at f6000000 (32-bit, non-prefetchable) [size=16M]
        Memory at e0000000 (64-bit, prefetchable) [size=256M]
        Memory at f5000000 (64-bit, non-prefetchable) [size=16M]
        I/O ports at e000 [size=128]
        Expansion ROM at f7000000 [disabled] [size=128K]
        Capabilities: [60] Power Management version 2
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [128] Power Budgeting <?>
        Kernel driver in use: vfio-pci
        Kernel modules: nouveau
```

The `qemu` command line, to launch the guest, must be run as `root` in order
to allow the `qemu` process to present the GPU to the guest VM.
There may be ways to run the process as a non-root user, but that may require
additional configuration not performed here.

The command line is:

```
# qemu-system-x86_64							\
-m 1g -cpu host -smp 2,cores=2,threads=1 -enable-kvm -machine q35	\
-device vfio-pci,host=01:00.0,bus=pcie.0,addr=1e.0 -vga std		\
-kernel sys.elf -d mmu,int -serial mon:stdio -nodefaults
```

The `sys.elf` binary is the supervisor-mode program that runs within the
guest VM.

The GPU is presented to the guest VM as a device with address `0x1e` on its
`pcie.0` bus. The output from `info qtree` monitor command.

```
(qemu) info qtree

 dev: q35-pcihost, id ""
    . . .
    bus: pcie.0
      type PCIE
      dev: vfio-pci, id ""
        host = "0000:01:00.0"
        sysfsdev = "/sys/bus/pci/devices/0000:01:00.0"
	. . .
        class VGA controller, addr 00:1e.0, pci id 10de:0391 (sub 0000:0000)
        bar 0: mem at 0xfc000000 [0xfcffffff]
        bar 1: mem at 0xe0000000 [0xefffffff]
        bar 3: mem at 0xfd000000 [0xfdffffff]
        bar 5: i/o at 0xc000 [0xc07f]
        bar 6: mem at 0xffffffffffffffff [0x1fffe]
```

The `qemu` BIOS has already mapped the BARs # 0, 1 and 3 at appropriate
physical locations. The program that runs within the guest VM can thus directly
read/write these locations as needed/appropriate.

---

### **The DCB:**

The Video BIOS contains the
[Device Control Block](http://download.nvidia.com/open-gpu-doc/DCB/1/DCB-4.0-Specification.html),
which helps in finding out, among other pieces, the information that is needed
to gather the EDID block of the connected display.

The DCB contains a Header, and a table of Device Entries. The DCB Header
further points towards a [Communications Control Block (CCB)]() table, and
a [Connector Table](). The details of parsing these tables are shown below:

The DCB table starts at offset `0x36` from the start of the BIOS image. See
the function [dcb_table](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/bios/dcb.c)

```
$ hexdump -C nv.bios.rom

. . .
00000030  01 00 00 00 b0 10 d6 8d  30 39 2f 32 37 2f 30 36  |........09/27/06|
. . .
```

The DCB table is thus at the offset `0x8dd6` from the start of the BIOS image.

```
$ hexdump -C nv.bios.rom

. . .
00008dd0  2f 21 2c 00 71 00 30 19  0a 08 3f 8e cb bd dc 4e  |/!,.q.0...?....N|
00008de0  78 8e 6c 8e d6 8e e2 8e  f0 8e 05 8f 31 51 59 00  |x.l.........1QY.|
00008e20  00 00 00 00 00 00 00 0f  00 00 00 00 00 00 00 0f  |................|
00008e30  00 00 00 00 00 00 00 0f  00 00 00 00 00 00 00 30  |...............0|
. . .
```

The structure of the DCB Header in the BIOS:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8dd6`|`0x30`| Version of the DCB table. This GPU being almost two decades old, does not sport a 4.0 DCB.
|`0x8dd7`|`0x19`| Size of the DCB Header, in bytes. The table of DCB Entries immediately follow the header.
|`0x8dd8`|`0xa`| Number of DCB Device Entries inside the table.
|`0x8dd9`|`0x8`| Size of each DCB Device Entry, in bytes.
|`0x8dda`|`0x8e3f`| Offset to the Communications Control Block (CCB) table., from the start of the BIOS image.
|`0x8ddc`|`0x4edcbdcb`| DCB Signature.
|`0x8de0`|`0x8e78`| Offset to the GPIO Assignment table.
|`0x8de2`|`0x8e6c`| Offset to the Input Devices table.
|`0x8de4`|`0x8ed6`| Offset to the Personal Cinema table.
|`0x8de6`|`0x8ee2`| Offset to the Spread Spectrum table.
|`0x8de8`|`0x8ef0`| Offset to the I<sup>2</sup>C Devices table.
|`0x8dea`|`0x8f05`| Offset to the Connector table.
|`0x8dec`|`0x31`| DCB Flags.
|`0x8ded`|`0x5951`| Offset to the HDTV Translation table.

The DCB Device Entries (they describe the output ports on the card) are show
below. Based on the header, there are 10 entries.
Each entry consumes two rows in the table shown below - the first row describes
the Display Path Information for the entry, while the second describes the
Device Specific Information for the same entry. The entries are indexed from 0,
to the number of entries, minus 1.

DCB Device Entry, Index 0:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8def`|`0x1000300`|`[3:0] = 0`. Display Type is `CRT`, or the common analog VGA display.<br><br>`[7:4] = 0`. The EDID Port to use to query EDID of any display connected to this output port. This value is the index into the CCB table.<br><br>`[11:8] = 3`. Head Bitmask describes the fact that the output port isn't exclusively assigned to one of the two heads. Depending on the programming, any one of the two heads (CRTC controllers) can drive it.<br><br>`[15:12] = 0`. Connector Index; the index, into the Connector table, to the entry which describes the connection to this output port.<br><br>`[19:16] = 0`. Logical Bus Number. Used for mutual exclusion.<br>For example, when a single physical port is capable of driving either a digital or an analog output, it will have two entries in the DCB table, both with the same bus number. This common value helps the driver in avoiding erroneously programming the same physical output port twice - once as digital and at the same time, once as analog.<br><br>`[21:20] = 0`. Location; describes an internal encoder.<br><br>`[22]`,`[23]`. TODO.<br><br>`[27:24] = 1`. TODO. Output Resource description.
|`0x8df3`|`0x28`| `CRT` specific information. Based on the [parse_dcb20_entry](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nouveau_bios.c) function, this field is the maximum frequency of the pixel clock (RAMDAC?) when driving the analog display over this output port. The value represents 400000 KHz (`0x28 * 10000`), or 400MHz.

DCB Device Entry, Index 1:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8df7`|`0x3000302`|`[3:0] = 2`. Display Type is `TMDS`, that is, DVI/HDMI. This is the same output port as above, as the bus number is the same. But this instance is responsible for driving a digital display.
|`0x8dfb`|`0`|`[1:0] = 0`. The EDID source is the DDC.

DCB Device Entry, Index 2:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8dff`|`0x4011310`|`[3:0] = 0`. Display Type is `CRT`.<br><br>`[7:4] = 1`. The EDID Port to use to query EDID of any display connected to this output port. This value is the index into the CCB table.<br><br>`[11:8] = 3`. Head Bitmask.<br><br>`[15:12] = 1`. Connector Index; the index, into the Connector table, to the entry which describes the connection to this output port.<br><br>`[19:16] = 1`. Logical Bus Number.<br><br>`[21:20] = 0`. Location; describes an internal encoder.<br><br>`[22]`,`[23]`. TODO.<br><br>`[27:24] = 4`. TODO. Output Resource description.
|`0x8e03`|`0x28`| `CRT` specific information.

DCB Device Entry, Index 3:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e07`|`0x4011312`| The digital counterpart of the analog Index 2.
|`0x8e0b`|`0`|

DCB Device Entry, Index 4:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e0f`|`0x20223f1`|`[3:0] = 1`. Display Type is `TV`.<br><br>`[7:4] = f`. The value suggests that the EDID information is not available through any entry within the CCB table - other methods (straps, SBIOS) may have to be employed.<br><br>`[11:8] = 3`. Head Bitmask describes the fact that the output port isn't exclusively assigned to one of the two heads. Depending on the programming, any one of the two heads (CRTC controllers) can drive it.<br><br>`[15:12] = 2`. Connector Index; the index, into the Connector table, to the entry which describes the connection to this output port.<br><br>`[19:16] = 2`. Logical Bus Number.<br><br>`[21:20] = 0`. Location; describes an internal encoder.<br><br>`[22]`,`[23]`. TODO.<br><br>`[27:24] = 2`. TODO. Output Resource description.
|`0x8e13`|`0xc0c080`| `TV` specific information.<br>`[2:0] = 0`. SDTV Format. The value represents `NTSC_M(US)`.<br><br>`[19:16][7:4] = 0x08`. TVDAC description. The value represents Standard HDTV DAC.<br><br>`[15:8] = 0xc0`. Encoder Identifier. The value represents NVIDIA Internal Encoder.<br><br>`[20] = 0`. The device doesn't use external communication port.<br><br>`[22:21] = 2`. One output device has 3 connectors.<br><br>`[26:23] = 1`. HDTV Format. The value represents HDTV 480i.


DCB Device Entries, Index 5 to 9:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e17`|`0xf`| `DCB_OUTPUT_UNUSED`.
|`0x8e1b`|`0`|
|`0x8e1f`|`0xf`| `DCB_OUTPUT_UNUSED`
|`0x8e23`|`0`|
|`0x8e27`|`0xf`| `DCB_OUTPUT_UNUSED`
|`0x8e2b`|`0`|
|`0x8e2f`|`0xf`| `DCB_OUTPUT_UNUSED`
|`0x8e33b`|`0`|
|`0x8e37`|`0xf`| `DCB_OUTPUT_UNUSED`
|`0x8e3b`|`0`|

When the Linux OS boots, the `nouveau` driver prints these entries as:

```
[    3.097559] nouveau 0000:01:00.0: DRM: DCB version 3.0
[    3.097561] nouveau 0000:01:00.0: DRM: DCB outp 00: 01000300 00000028
[    3.097562] nouveau 0000:01:00.0: DRM: DCB outp 01: 03000302 00000000
[    3.097563] nouveau 0000:01:00.0: DRM: DCB outp 02: 04011310 00000028
[    3.097564] nouveau 0000:01:00.0: DRM: DCB outp 03: 04011312 00000000
[    3.097565] nouveau 0000:01:00.0: DRM: DCB outp 04: 020223f1 00c0c080
```

---

### **The DCB: Connector Table**

Based on the DCB Header, the Connector table is at offset `0x8f05`.

```
$ hexdump -C nv.bios.rom

. . .
00008f00  00 ff 00 00 00 30 05 0a  02 00 30 10 30 21 10 02  |.....0....0.0!..|
00008f10  11 02 13 02 ff 00 ff 00  ff 00 ff 00 ff 00 66 51  |..............fQ|
. . .
```

The Connector table Header:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8f05`|`0x30`| Version of the Connector table.
|`0x8f06`|`0x5`| Size of the Connector table Header, in bytes.
|`0x8f07`|`0xa`| Number of entries in the table. They immediately follow this header.
|`0x8f08`|`0x2`| Size of each entry in the table, in bytes.
|`0x8f09`|`0x0`| Platform. The value represents a normal add-in card.

Connector table entries:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8f0a`|`0x1030`|`[7:0] = 0x30`. Connector type. The value represents a DVI-I connector.<br><br>`[11:8] = 0`. The physical location of the connector. The value represents the connector on the bracket closet to the PCI slot.<br><br>`[12] = 1`. The connector triggers the Hotplug A interrupt.
|`0x8f0c`|`0x2130`| Similar to above, except this connector is located on the middle bracket, and it generates the Hotplug B interrupt.
|`0x8f0e`|`0x210`|`[7:0] = 0x10`. Connector type is TV-Composite-Out. Its location is on the bracket furthest away from the PCI slot.
|`0x8f10`|`0x211`|`[7:0] = 0x11`. Connector type is TV-S-Video-Out. Same location as the TV-Composite-Out.
|`0x8f12`|`0x213`|`[7:0] = 0x13`. Connector type is TV-HDTV-Component-YPrPb. Same location as the other two TV connectors.
|`0x8f14`|`0xff`| Unused.
|`0x8f16`|`0xff`| Unused.
|`0x8f18`|`0xff`| Unused.
|`0x8f1a`|`0xff`| Unused.
|`0x8f1c`|`0xff`| Unused.


When the Linux OS boots, the `nouveau` driver prints these entries as:

```
[    3.097566] nouveau 0000:01:00.0: DRM: DCB conn 00: 1030
[    3.097567] nouveau 0000:01:00.0: DRM: DCB conn 01: 2130
[    3.097568] nouveau 0000:01:00.0: DRM: DCB conn 02: 0210
[    3.097568] nouveau 0000:01:00.0: DRM: DCB conn 03: 0211
[    3.097569] nouveau 0000:01:00.0: DRM: DCB conn 04: 0213
```

---

### **The DCB: Communications Control Block (CCB) Table**

Based on the DCB Header, the CCB table is at offset `0x8e3f`.

```
$ hexdump -C nv.bios.rom

. . .
00008e30  00 00 00 00 00 00 00 0f  00 00 00 00 00 00 00 30  |...............0|
00008e40  05 03 04 02 37 36 00 00  3f 3e 00 00 51 50 00 00  |....76..?>..QP..|
. . .
```

CCB table Header:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e3f`|`0x30`| Version of the CCB table.
|`0x8e40`|`0x5`| Size of the CCB table Header, in bytes.
|`0x8e41`|`0x3`| Number of entries in the table. They immediately follow this header.
|`0x8e42`|`0x4`| Size of each entry in the table, in bytes.
|`0x8e43`|`0x2`| `[3:0] = 2`: Index of the Primary communications port.<br>`[7:4] = 0`: Index of the secondary communication port.

CCB table entry, Index 0:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e44`|`0x37`| CRTC Index register for driving the I<sup>2</sup> bus.
|`0x8e45`|`0x36`| CRTC Index register for sensing the I<sup>2</sup> bus.
|`0x8e46`|`0`| Reserved/Unknown.
|`0x8e47`|`0`| Type of the CCB Entry format. `0 = DCB_I2C_NV04_BIT`.

CCB table entry, Index 1:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e48`|`0x3f`| CRTC Index register for driving the I<sup>2</sup> bus.
|`0x8e49`|`0x3e`| CRTC Index register for sensing the I<sup>2</sup> bus.
|`0x8e4a`|`0`| Reserved/Unknown.
|`0x8e4b`|`0`| Type of the CCB Entry format. `0 = DCB_I2C_NV04_BIT`.

CCB table entry, Index 2:

|Address|Value|Comment|
|-----:|-----:|-------|
|`0x8e4c`|`0x51`| CRTC Index register for driving the I<sup>2</sup> bus.
|`0x8e4d`|`0x50`| CRTC Index register for sensing the I<sup>2</sup> bus.
|`0x8e4e`|`0`| Reserved/Unknown.
|`0x8e4f`|`0`| Type of the CCB Entry format. `0 = DCB_I2C_NV04_BIT`.

When the Linux OS boots, the `nouveau` driver, with debug traces enabled,
prints these entries as:

```
[    5.072755] nouveau 0000:00:1e.0: i2c: ccb 00: type 00 drive 37 sense 36 share ff auxch ff
[    5.073002] nouveau 0000:00:1e.0: i2c: ccb 01: type 00 drive 3f sense 3e share ff auxch ff
[    5.073122] nouveau 0000:00:1e.0: i2c: ccb 02: type 00 drive 51 sense 50 share ff auxch ff
```

The kernel command-line to enable these traces is: `nouveau.debug=trace`.

---

### **Display Path Summary**

A display is plugged onto the connector found on the middle bracket. The
corresponding Connector entry has the index 1, shown in the driver's output
as `DCB conn 01: 2130`.

The DCB Device Entry with Index 3 is the relevant entry which points to this
Connector entry - that entry is for a digital output port, and it
has its Connector Index set to 1. Furthermore, it has its EDID Port set to
Index 1. The DCB Device Entry is shown in the driver's output as
`DCB outp 03: 04011312 00000000`.

The EDID Port with Index 1 is the DDC/I<sup>2</sup>C bus with the same index, displayed
by the driver as:
`i2c: ccb 01: type 00 drive 3f sense 3e share ff auxch ff`.

Thus, one must configure the CRTC indices `0x3e/0x3f` in order to drive the
I<sup>2</sup>C bus, and extract the EDID information of the connected display.

Note that the CCB entry with Index 2 (CRTC indices `0x50/0x51`) is not referenced by any
output device. The correponding bus might be connected to some other
peripheral.

This complete the trace of the display path.

---

### **Guest Machine Software Setup:**

The supervisor-mode program that runs within the guest VM must drive the
I<sup>2</sup>C bus lines (SDA and SCL) on its own. In contrast, the RPi1B
exposes a higher-level abstraction of a FIFO and some control registers,
which frees up a program from driving the bus on its own.

The relevant source code, along with some timing parameters, can be found
[here](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/i2c/bit.c) and
[here](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/i2c/busnv04.c)

The EDID Ports are extended CRTC registers; upon boot, the GPU keeps these
registers locked, preventing access and modification. The DDC bus cannot be
driven if these registers are locked. The locking/unlocking of these registers
can be done by setting the CRTC index register `0x1f` to appropriate values.
See [nvkm_lockvgac](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/engine/disp/vga.c)

The `nouveau` driver unlocks the registers before it begins with the device initialization;
see [nvkm_devinit_preinit](https://lxr.missinglinkelectronics.com/linux/drivers/gpu/drm/nouveau/nvkm/subdev/devinit/base.c)

---

### **Running the demo:**

The driver program can be found [here]()https://github.com/asurati/x04/blob/main/sys/main.c. The output is:

```
00 ff ff ff ff ff ff 00 10 ac 54 d0 00 00 00 00
23 17 01 03 80 30 1b 78 ea 16 c5 a4 56 50 9f 26
12 50 54 a5 4b 00 71 4f 81 80 d1 c0 01 01 01 01
01 01 01 01 01 01 02 3a 80 18 71 38 2d 40 58 2c
45 00 dc 0b 11 00 00 1e 00 00 00 ff 00 31 4d 56
44 31 33 38 53 30 51 50 54 0a 00 00 00 fc 00 44
45 4c 4c 20 53 32 32 34 30 4c 0a 20 00 00 00 fd
00 38 4c 1e 53 11 00 0a 20 20 20 20 20 20 01 10
02 03 1f f1 4c 90 05 04 03 02 07 16 01 14 1f 12
13 23 09 07 07 83 01 00 00 65 03 0c 00 10 00 02
3a 80 18 71 38 2d 40 58 2c 45 00 dc 0b 11 00 00
1e 01 1d 80 18 71 1c 16 20 58 2c 25 00 dc 0b 11
00 00 9e 01 1d 00 72 51 d0 1e 20 6e 28 55 00 dc
0b 11 00 00 1e 8c 0a d0 8a 20 e0 2d 10 10 3e 96
00 dc 0b 11 00 00 18 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 b7
```
