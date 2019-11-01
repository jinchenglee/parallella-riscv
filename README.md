# RISC-V on Parallella & ZedBoard Zynq FPGA Boards

Branched off from original author's github repo. Removed Zedboard contents as I don't have one. 

## Contents of this document

[Using the prebuilt images](#using-the-prebuilt-images)

[Instructions for building everything from scratch](#instructions-for-building-everything-from-scratch)

[Submodules Used](#submodules-used)

[Design](#design)

[Project status at the end of GSoC 2016](#project-status-at-the-end-of-gsoc-2016)

[Links](#links)

[Contributors](#contributors)

## Using the prebuilt images

**Out of date but they still work**

If you don't have a working Vivado installation, or you just want to test everything
quickly without the necessary long build times, you can use the prebuilt images that
have been prepared and hosted on the [Parallella RISC-V Prebuilt images]
(https://github.com/eliaskousk/parallella-riscv-images) repository. It contains images
for both types of Zynq devices that the various Paralllella editions contain, along with
complete instructions on how to use them on your board.

## Instructions for building everything from scratch

**Updated to use a modern RV64 Rocket Chip with Rocket Core Generator of April 2018**

This repo has been updated to use a modern Rocket Core using the infrastructure found
in the original fpga-zynq repository of UCB. The work done there was great (thank you!)
but it is now not maintained and doesn't work with newer Xilinx Vivado tools (i.e 2018.2).

### Setup

In order to build a bitstream and / or the necessary host software to boot the board you must first
edit the `BOARD`, `JOBS`, `VIVADO_PATH` and `VIVADO_VERSION` and `PETALINUX_PATH` variables in
`${TOP}/scripts/settings.sh` if you need to change the target board (to e.g `zedboard` instead of the
default `parallella`) or the number of jobs your machine can simultaneously handle while building
(default is `8`) or the installation path and version of Vivado tools you have installed on your system.
The system can be synthesized and implemented with the latest Xilinx Vivado tools (tested with 2018.2).

Important Note: The build system by default generates a ZedBoard build. If you need to build for the
Parallella you should change the `BOARD` variable above. Moreover, you should make sure your Parallella
version (Desktop / Microserver / Embedded or Kickstarter) is properly set using the `BOARD` variable in the
`${TOP}/parallella/Makefile` (top of the file) as explained in the related comments there.

Warning: As of 2018 the smaller Parallella's with the smaller Zynq device might not be able to fit a modern Rocket Core.

### Build Bitstream and Host (ARM) Software

To build the bitstream and the needed host (ARM) software you must do the following:

* Step 1: Build the FPGA Bitstream

This will build the FPGA bitstream for Parallella or Zedboard:

```bash
./scripts/build.fpga.bitstream.sh
```

After it finishes you can view / edit the design in Vivado by opening the
`${TOP}/${BOARD}/fpga/${BOARD}_riscv/system.xpr** project.

Keep in mind that everytime you run the above build bitstream script the project's folder is
deleted so any modifications you have made there will be lost.

* Step 2: Build the Host Software

**Parallella:**

The host software for the ARM dual-core processor of the Zynq device consists of the Linux kernel,
the devicetree DTB file (Device Tree Blob) and the U-Boot bootloader.

- **The Linux kernel (uImage)** is identical with the one provided by the latest E-SDK and it's a
matter of preference to use the one you build or the fefault from the E-SDK.

- **The device tree DTB file** is needed in order to use Rocket Core's Host I/O interface
and thus the RISC-V RV64 core.

- **U-Boot** is the bootloader which programs the FPGA and boots the Linux kernel. A custom U-Boot build
is not really needed for Parallella since the one contained in its Flash chip is fine and there is always
a risk when performing re-flashes.

**ZedBoard:** (Updated November 2018)

ZedBoard host software is now built with [PetaLinux](https://www.xilinx.com/member/forms/download/xef.html?filename=petalinux-v2018.2-final-installer.run)
so you must first download and install it from Xilinx download site. You need the 6 GB Installer and it only works on Ubuntu 16.04 or RHEL/CentoOS 7.2 - 7.4.

The PetaLinux build will use the ZedBoard BSP for ZedBoard in order to build two following two files:

- **The BOOT image BOOT.bin** which contains the Zynq PS FSB (First Stage BootLoader) & U-Boot & FPGA Bitstream
- **The FIT image image.ub** which contains the Linux Kernel & Linux FileSystem Image & Linux Flatten Device Tree (FDT)

You can build the host software for all boards by running the following:

```bash
./scripts/build.host.software.sh
```

**Important Notes**

Before running the build host software script make sure:

1. You have a generated bitstream (with `./scripts/build.fpga.bitstream.sh` as described above) since the bitstream
is needed to build PetaLinux
2. You have downloaded the [ZedBoard BSP](https://www.xilinx.com/member/forms/download/xef.html?filename=avnet-digilent-zedboard-v2018.2-final.bsp) from the Xilinx site and placed it into your PetaLinux installation directory (i.e `/opt/Xilinx/PetaLinux`)
3. When prompted to configure the rootfs image go into FileSystem Packages -> misc -> gcc runtime and select the `libstdc++`
library package (just the normal package not the -dev one)

All the final output files are placed in the `${TOP}/${BOARD}/output/final/` directory. After generation
you should copy them in your SD card's boot partition. To actually test RISC-V you should also build its
software and copy the `${TOP}/${BOARD}/output/final/riscv` folder to your SD as well (more on this below).

### Obtain an ARM Host Linux Root Image for Parallella

For Parallella the software built above (PetaLinux works only for ZedBoard), does not include a root image
for the Linux host running on the ARM cores of Zynq. If you don't want to bother with building your own you
can use an existing from the following repository:

* [Parallella](https://github.com/parallella/pubuntu/releases)

If you use the `Parallella ESDK` image linked  there you also have the choice of NOT building the ARM Linux
kernel yourself. This image contains both the Linux kernel and the root image to boot Parallella so you
only need to build (see above) and copy on your SD card the device tree DTB file (Device Tree Blob) and
of-course the bitstream.

Note: I used the parabuntu-2019.1-beta1-headless-z7020.img.gz version, which works fine to boot Linux on Zynq ARM and it can run bare-metal RISCV programs generated using below steps. 

### Build the RISC-V Toolchain

In order to compile RISC-V programs of your own or the proxy kernel (bbl), the Berkeley bootloader (bbl)
and the RISC-V Linux you must build the RISC-V toolchain by running the following script:

```bash
./scripts/build.riscv.toolchain.sh
```

Building the RISC-V toolchain might take a lot of time and if you change RISC-V core configurations
(using the **IMA** arch in place of the default **IMAFD**) you need to rebuild it after each change.

The toolchain will be installed in the `${TOP}/ip/toolchain/` directory.

You can use this toolchain to build RISC-V baremetal programs or the RISC-V Linux kernel (see below).

### RISC-V Baremetal

In order to run baremetal programs on the board run the following script:

```bash
./scripts/build.riscv.baremetal.sh
```

All the output files will be placed in the  `${TOP}/${BOARD}/final/output/riscv/` folder.

To copy the built files to an already booted board you can run the following (assuming your Parallella
can be reached inside your network with the `parallella` host. You can replace this with parallella@IP
where IP is the IP address of Parallella in your local network):

```bash
cd parallella/output/final/riscv/
scp fesvr parallella@parallella:~/
scp libfesvr.so parallella@parallella:~/
scp pk parallella@parallella:~/
scp hello parallella@parallella:~/
```

All the above tranfered files will be placed in the home directory of the parallella user but you can
of-course transfer them wherever you wish on your SD card's root or boot partition.

For the frontend server don't forget to copy the shared library `libfesvr.so` to `/usr/local/lib` on your
board and running `sudo ldconfig` to update the library cache.

```bash
parallella@parallella:~/$ sudo cp libfesvr.so /usr/local/lib/
parallella@parallella:~/$ sudo ldconfig
```

Then fesvr and pk can be used to load RISC-V baremetal programs like this:

```bash
parallella@parallella:~/$ sudo ./fesvr pk hello
Hello World!
```

### RISC-V Linux And Root Disk Image 

You can build the RISC-V Linux kernel with the following script:

```bash
./scripts/build.riscv.linux.sh
```

Afterwards you can build the RISC-V Linux root disk image with the following script:

```bash
./scripts/build.riscv.root.sh
```

This uses RISC-V Poky and might take a long time depending on the machine's computational power and network
connectivity so please be patient. We suggest you download the prebuilt image from here (coming soon!).

All the output files will be placed in the  `${TOP}/${BOARD}/final/output/riscv/ folder.

**Update November 2018**

If the above doesn't work or you just don't want to wait for such a build, you can use the following two files
that are inside the home folder of root in this RISC-V root filesystem image of UCB's ZedBoard Image Repo:

- The bbl binary that contains the kernel as a payload
- The root filesystem buildroot.rootfs.ext2

You can extract the files either by mounting the image or by using this tool or by just booting the complete release
from the UCB repo and copying the files in your SD card.

To copy the built or extracted files to an already booted board you can run the following (assuming your Parallella
can be reached inside your network with the `parallella` host. You can replace this with parallella@IP
where IP is the IP address of Parallella in your local network):

```bash
cd parallella/output/final/riscv/
scp bbl parallella@parallella:~/
scp vmlinux parallella@parallella:~/   # Update 2018 - Not needed anymore
scp root.bin parallella@parallella:~/  # Update 2018 - You can use the buildroot.rootfs.ext2 file, see above
```

All the above tranfered files will be placed in the home directory of the parallella user but you can
of-course transfer them wherever you wish on your SD card's root or boot partition.

Then fesvr and bbl can be used to boot RISC-V Linux and perform simple tasks like shown below:

```bash
parallella@parallella:~/$ sudo ./fesvr +blkdev=root.bin bbl vmlinux
              vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
                  vvvvvvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrr       vvvvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrr      vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrrrr    vvvvvvvvvvvvvvvvvvvvvvvv
rrrrrrrrrrrrrrrr      vvvvvvvvvvvvvvvvvvvvvv  
rrrrrrrrrrrrr       vvvvvvvvvvvvvvvvvvvvvv    
rr                vvvvvvvvvvvvvvvvvvvvvv      
rr            vvvvvvvvvvvvvvvvvvvvvvvv      rr
rrrr      vvvvvvvvvvvvvvvvvvvvvvvvvv      rrrr
rrrrrr      vvvvvvvvvvvvvvvvvvvvvv      rrrrrr
rrrrrrrr      vvvvvvvvvvvvvvvvvv      rrrrrrrr
rrrrrrrrrr      vvvvvvvvvvvvvv      rrrrrrrrrr
rrrrrrrrrrrr      vvvvvvvvvv      rrrrrrrrrrrr
rrrrrrrrrrrrrr      vvvvvv      rrrrrrrrrrrrrr
rrrrrrrrrrrrrrrr      vv      rrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrr          rrrrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrrrr      rrrrrrrrrrrrrrrrrrrr
rrrrrrrrrrrrrrrrrrrrrr  rrrrrrrrrrrrrrrrrrrrrr

       INSTRUCTION SETS WANT TO BE FREE
[    0.000000] Linux version 4.1.17-g174f395-dirty (lupo@gene) (gcc version 5.3.0 (GCC) ) #9 Mon Jul 25 18:18:14 EEST 2016
[    0.000000] Available physical memory: 1022MB
[    0.000000] Physical memory usage limited to 384MB
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000000200000-0x00000000181fffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000200000-0x00000000181fffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000200000-0x00000000181fffff]
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 96960
[    0.000000] Kernel command line: root=/dev/htifblk0 mem=384M
[    0.000000] PID hash table entries: 2048 (order: 2, 16384 bytes)
[    0.000000] Dentry cache hash table entries: 65536 (order: 7, 524288 bytes)
[    0.000000] Inode-cache hash table entries: 32768 (order: 6, 262144 bytes)
[    0.000000] Sorting __ex_table...
[    0.000000] Memory: 384352K/393216K available (1810K kernel code, 92K rwdata, 396K rodata, 64K init, 210K bss, 8864K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS:2
[    0.000000] clocksource riscv_clocksource: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 191126044627 ns
[    0.150000] Calibrating delay using timer specific routine.. 20.02 BogoMIPS (lpj=100104)
[    0.150000] pid_max: default: 32768 minimum: 301
[    0.150000] Mount-cache hash table entries: 1024 (order: 1, 8192 bytes)
[    0.150000] Mountpoint-cache hash table entries: 1024 (order: 1, 8192 bytes)
[    0.150000] devtmpfs: initialized
[    0.150000] clocksource jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.150000] NET: Registered protocol family 16
[    0.150000] Switched to clocksource riscv_clocksource
[    0.150000] NET: Registered protocol family 2
[    0.150000] TCP established hash table entries: 4096 (order: 3, 32768 bytes)
[    0.150000] TCP bind hash table entries: 4096 (order: 3, 32768 bytes)
[    0.150000] TCP: Hash tables configured (established 4096 bind 4096)
[    0.150000] UDP hash table entries: 256 (order: 1, 8192 bytes)
[    0.150000] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes)
[    0.150000] NET: Registered protocol family 1
[    0.150000] futex hash table entries: 256 (order: 0, 6144 bytes)
[    0.150000] io scheduler noop registered
[    0.150000] io scheduler cfq registered (default)
[    0.180000] htifcon htif1: detected console
[    0.180000] console [htifcon0] enabled
[    0.180000] htifblk htif2: detected disk
[    0.180000] htifblk htif2: added htifblk0
[    0.180000] VFS: Mounted root (ext2 filesystem) readonly on device 254:0.
[    0.180000] devtmpfs: mounted
[    0.180000] Freeing unused kernel memory: 64K (ffffffff80000000 - ffffffff80010000)
INIT: version 2.88 booting
[    0.430000] random: dd urandom read with 23 bits of entropy available
hwclock: can't open '/dev/misc/rtc': No such file or directory
Sun Jul 17 01:37:12 UTC 2016
hwclock: can't open '/dev/misc/rtc': No such file or directory
INIT: Entering runlevel: 5
Configuring network interfaces... ifconfig: SIOCGIFFLAGS: No such device
hwclock: can't open '/dev/misc/rtc': No such file or directory
Starting syslogd/klogd: done

Poky (Yocto Project Reference Distro) 2.0+snapshot-20160717 riscv64 /dev/ttyHTIF0

riscv64 login: root

root@riscv64:~# ls -al /
drwxr-xr-x   18 root     root          1024 Jul 17 01:37 .
drwxr-xr-x   18 root     root          1024 Jul 17 01:37 ..
drwxr-xr-x    2 root     root          2048 Jul 17 01:37 bin
drwxr-xr-x    2 root     root          1024 Jul 17 01:17 boot
drwxr-xr-x    3 root     root         10720 Jul 17 01:37 dev
drwxr-xr-x   21 root     root          1024 Jan  1  1970 etc
drwxr-xr-x    3 root     root          1024 Jul 17 01:37 home
drwxr-xr-x    2 root     root          2048 Jul 17 01:37 lib
drwx------    2 root     root         12288 Jul 17 01:37 lost+found
drwxr-xr-x    2 root     root          1024 Jul 17 01:17 media
drwxr-xr-x    2 root     root          1024 Jul 17 01:17 mnt
dr-xr-xr-x   29 root     root             0 Jan  1  1970 proc
drwxr-xr-x    3 root     root           160 Jul 17 01:37 run
drwxr-xr-x    2 root     root          1024 Jul 17 01:37 sbin
drwxr-xr-x    2 root     root          1024 Jul 17 01:17 sys
drwxrwxrwt    2 root     root          1024 Jul 17 01:17 tmp
drwxr-xr-x    9 root     root          1024 Jul 17 01:34 usr
drwxr-xr-x    8 root     root          1024 Jul 17 01:34 var

root@riscv64:~# uname -a
Linux riscv64 4.1.17-g174f395-dirty #9 Mon Jul 25 18:18:14 EEST 2016 riscv GNU/Linux

root@riscv64:~# shutdown -h now
INIT: Switching to runlevel: 0
INIT: Sending processes the TERM signal
root@riscv64:~# logout
hwclock: can't open '/dev/misc/rtc': No such file or directory
Stopping syslogd/klogd: stopped syslogd (pid 142)
stopped klogd (pid 141)
done
Deconfiguring network interfaces... ifdown: interface eth0 not configured
done.
Sending all processes the TERM signal...
Unmounting remote filesystems...
Deactivating swap...
Unmounting local filesystems...
[4.730000] reboot: Power down

parallella@parallella:~/$
```

Congratulations, you now have a working RISC-V RV64 core running RISC-V Linux on your Parallella!

### Re-Build the RISC-V RV64 Rocket Core (Optional)

In case you want to (re)build the RISC-V RV64 rocket core IP you can run the following script:

```bash
./scripts/build.riscv.rocketcore.sh
```

Do not attempt to generate a new rocket-core with the latest rocket-chip upstream (i.e by updating the
`ip/rocket-chip` git hash) since it might not work with the `ip/rocket-zynq` customization sources for
the generated rocket-core (`ip/rocket-zynq/src`) or the customized fesvr utility and associated library
(`ip/rocket-zynq/csrc` folder).

This repo currently use a rocket-core generated with an older rocket-chip version from April 2018 that is
stable and in sync with the customization sources in these folders (taken from the [UCB-BAR's original
fpga-zynq repo](https://github.com/ucb-bar/fpga-zynq) which is not maintained any more.

### Build the RISC-V Emulator (Optional)

You can build the RISC-V emulator which simulates the rocket core with Verilator by running the following
script:

```bash
./scripts/build.riscv.emulator.sh
```

Building the RISC-V emulator is NOT necessary to build the FPGA bitstream or the RISC-V Toolchain
(see above).

### Run Make Targets Manually

All the above scripts just run the respective make targets inside `${TOP}/scripts/Makefrag` which is
included in `${TOP}/${BOARD}/Makefile`. You can enter your desired `${TOP}/${BOARD}/` folder and manually
run the targets in a the sequence you desire. Make sure you source the `${TOP}/scripts/set.env.sh` first
though to set the necessary environment variables and your Vivado settings64.sh file if you need to use
Vivado to e.g build a bitstream.

## Submodules Used

This repository uses the following Git submodules. You can initialize each one separately if needed
to clean things up or perform any advanced action but this is normally not needed since the build
system will take care of all the submodule initializations.

* [Parallella Open Hardware](https://github.com/parallella/oh)
  - Mapped to `root_dir/parallella/oh/`
  - Run in root dir: `git submodule update --init -- parallella/oh`
  - Needed to build the bitstream for Parallella or Zedboard (dev)
    inside the `root_dir/$board/fpga/` folder

* [Rocket-Chip Generator](https://github.com/ucb-bar/rocket-chip)
  - Mapped to `root_dir/ip/rocket-chip/`
  - Run in root dir: `git submodule update --init -- ip/rocket-chip`
  - Needed to update the generated RISC-V Rocket Core RV64G IP
    inside the `root_dir/ip/RISCV_Rocket_Core_RV64G_1.0/` folder

* [RISC-V Linux](https://github.com/riscv/riscv-linux)
  - Mapped to `root_dir/boot/riscv-linux/`
  - Run in root dir: `git submodule update --init -- boot/riscv-linux`
  - Needed to build the RISC-V Linux kernel
    inside the `root_dir/$board/output/final/riscv/` folder

* [RISC-V Poky](https://github.com/riscv/riscv-poky) # JC: **This doesn't work anymore**. Should be replaced by https://github.com/riscv/meta-riscv
  - Mapped to `root_dir/boot/riscv-linux/`
  - Run in root dir: `git submodule update --init -- boot/riscv-poky`
  - Needed to build the RISC-V Poky root image (distribution) for RISC-V Linux
    inside the `root_dir/$board/output/final/riscv/` folder

In order to build the boot images (U-Boot, Linux Kernel) for Parallella board target:

* [Parallella U-Boot Bootloader](https://github.com/parallella/parallella-uboot)
  - Mapped to `root_dir/boot/parallella-uboot`
  - Run in root dir: `git submodule update --init -- boot/parallella-uboot`
  - Needed to build the Parallella U-Boot bootloader (u-boot.elf)
    inside the `root_dir/$board/output/boot/` folder
  - Currently deprecated since we prefer not having to replace U-Boot
    or the FSBL by flashing the board (unsafe and needs JTAG or serial cable)

* [Parallella Linux Kernel](https://github.com/parallella/parallella-linux)
  - Mapped to `root_dir/boot/parallella-linux`
  - Run in root dir: `git submodule update --init -- boot/parallella-linux`
  - Needed to build the Zynq ARM Linux kernel of Parallella (uImage)
    inside the `root_dir/$board/output/final/` folder

In order to build the boot images (U-Boot, Linux Kernel) for ZedBoard board target (development only):

* [Xilinx U-Boot Bootloader](https://github.com/Xilinx/u-boot-xlnx)
  - Mapped to `root_dir/boot/u-boot-xlnx`
  - Run in root dir: `git submodule update --init -- boot/u-boot-xlnx`
  - Needed to build the ZedBoard U-Boot bootloader (u-boot.elf)
    inside the `root_dir/$board/output/boot/` folder

* [Xilinx Linux Kernel](https://github.com/Xilinx/linux-xlnx)
  - Mapped to `root_dir/boot/linux-xlnx/`
  - Run in root dir: `git submodule update --init -- boot/linux-xlnx`
  - Needed to build the ZedBoard Linux kernel (uImage)
    inside the `root_dir/$board/output/final/` folder

* [Device Tree Compiler](https://git.kernel.org/cgit/utils/dtc/dtc.git)
  - Mapped to `root_dir/boot/dtc`
  - Run in root dir: `git submodule update --init -- boot/dtc`
  - Needed to build the ZedBoard U-Boot bootloader (u-boot.elf)
    inside the `root_dir/board/output/boot/` folder and possibly
    a new device tree blob (devicetree.dtb) inside the
    `root_dir/$board/output/final/` folder

## Design

![Vivado Block Diagram](doc/images/vivado.parallella.riscv.system.bd.png)

### Parallella Base Component

This design contains the Parallella Base component connected to the ARM cores via AXI4.
The Parallella Base component contains the `E-Link` needed for communication with the `Epiphany` chip
on-board Parallella along with `GPIO` single ended passthrough (PL <-> PS connections) and `I2C` bus
connection to the on-board power regulators that power mangage the Epiphany chip.

All these mean that the bitstreams produced here have identical functionality with those provided by
Parallella plus the RISC-V core (see below).

### RISC-V RV64 Core

The design contains two `RISC-V RV64` cores generated by [Rocket Chip Generator](https://github.com/ucb-bar/rocket-chip).

- The default configuration uses a core generated with support for the **IMA** extensions.
**IMA** means the core contains the **I**nteger, **M**ultiply/Division and **A**tomic extensions.

- You can optionally use a core generated with support for the **IMA** extensions just by setting the
following variables inside the `${BOARD}/Makefile`:

* `RISCV_CORE_ARCH   = IMA` (default is `IMAFD`)
* `RISCV_CORE_CONFIG = DefaultFPGAConfig` (defailt is `DefailtFPGANoFPUConfig`) # JC: **Figure out which default**.

**IMAFD** = **G** means the core contains the  **I**nteger, **M**ultiply/Division, **A**tomic and
**F**loating Point with Single or **D**ouble Precision extensions.

### RISC-V RV64 DRAM Memory

The default allocated DRAM to the RISC-V core is 384 MB, starting from the second half of the 1 GB
DDR3 memory of parallella. You can change this by setting the following makefile variables and then
rebuilding the bitstream, the device tree for the ARM host and the RISC-V Linux kernel:

* `RISCV_DRAM_BASE_RTL   = "3\'d1"`
* `RISCV_DRAM_BITS_RTL   = 29`
* `RISCV_DRAM_BASE_DTS   = 0x20000000`
* `RISCV_DRAM_SIZE_DTS   = 0x18000000`
* `RISCV_DRAM_SIZE_LINUX = 384M`

```bash
./scripts/build.fpga.bitstream.sh
./scripts/build.host.software.sh
./scripts/build.riscv.linux.sh
```

### RISC-V RV64 AXI Buses

The RISC-V RV64 core communicates with the rest of the ARM SoC of the Zynq FPGA device using AXI4
interfaces:

* **AXI Master**: RV64 Core to DDR3 DRAM via ARM  (memory access of the core)

* **AXI Slave**:  ARM to HostIO of RV64 Core (boot / control the core)

### Clocks

The Parallella Base component runs with a 100 MHz clock and the RISC-V RV64 core runs with a 50 MHz clock
when configured with the **IMA** extensions or with a 25 MHz clock when configured with the **IMAFD**
extensions.

# GSoC 2016 Documentation

Go to original autho's a series of [blog posts](http://eliaskousk.teamdac.com) that explain more in depth
some parts of the project.

## Links

[FOSSi](http://www.fossi-foundation.org)

[Parallella](https://www.parallella.org)

[RISC-V](http://riscv.org)

[Google Summer of Code](https://developers.google.com/open-source/gsoc/)

[FOSSi GSoC'16 Projects](https://summerofcode.withgoogle.com/organizations/5516229267685376/#projects)

## Contributors

### Code

- Elias Kouskoumvekakis ([Blog](http://eliaskousk.teamdac.com))

### GSoC Mentors

- Olof    Kindgren ([FOSSi Foundation](http://fossi-foundation.org))
- Andreas Olofsson ([Adapteva](http://www.adapteva.com))
