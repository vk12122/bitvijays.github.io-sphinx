# Kernel Compiling

Kernel can be compiled by either `gcc` or `clang`. We would compile the Kernel using `clang`.

PS: This guide for compiling, configuring kernel for using in QEMU.

## Pre-req Packages

```text
apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

## clang

### Clang - Official

Follow [Getting Started: Building and Running Clang](https://clang.llvm.org/get_started.html) to install the clang.

Make sure that `llvm/build/bin` is added to your path using export.

We can also add that in the `.bash_profile`

```text
export PATH="your-dir:$PATH
```

### Clang - Cheri

Refer [CTSRD-CHERI LLVM Project](https://github.com/CTSRD-CHERI/llvm-project)

Follow the same step as LLVM.

### Confirm LLVM version

Make sure that clang version is correct and coming from what you build.

```text
clang -v
clang version 11.0.0 (https://github.com/CTSRD-CHERI/llvm-project.git b507d88d2aa61cec27adab60324a04b17911f5e4)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /home/bitvijays/Documents/llvm-project/build/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/6
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/6.3.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

## Linux Mainline

Git clone the [Linux Mainline](https://github.com/torvalds/linux)

### Configure Kernel

#### Make Config

We can create a default config using `make defconfig` or `make x86_64_defconfig`

#### Make Other

```text
"make kvmconfig"   Enable additional options for kvm guest kernel support.
"make xenconfig"   Enable additional options for xen dom0 guest kernel support.
"make tinyconfig"  Configure the tiniest possible kernel.
```

Once the kernel is configured

### Compile Kernel

We can compile the kernel using

```text
make
```

We can provide other arguments to make such as

```text
-j num For processors with multiple cores, make all the cores do the work. Add the option -j(<NUMBER_OF_CORES> + 1). For example, a dual core processor contains two logical cores plus one (2 + 1)
ARCH
CC=clang HOSTCC=clang : If we want to compile the kernel using clang
```

#### Kernel Output

A sample output of the Kernel Compilation is mentioned below:

```text
OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
Setup is 16124 bytes (padded to 16384 bytes).
System is 8673 kB
CRC f5ca994b
Kernel: arch/x86/boot/bzImage is ready  (#5)
```

## Running the Kernel in QEMU

### Install the packages

```text
apt install qemu qemu-system
```

### Running Kernel without RootFS

```text
cd linux
qemu-system-x86_64 -no-kvm -kernel arch/x86/boot/bzImage -hda /dev/zero -append "root=/dev/zero console=ttyS0" -serial stdio -display none
```

As above execution has no file system image which could be mounted in qemu virtual environment, we would get the kernel panic message related to mounting file system.

### Running Kernel with RootFS

#### Creating RootFS

**Clone Repo**

We would be using [BuildRoot](https://buildroot.org/) to create the filesystem.

Clone the Buildroot Github

```text
git clone git://git.buildroot.net/buildroot
cd buildroot
```

**Configure BuildRoot**

```text
make menuconfig
```

1. Select “Target Options” to define the Target Architecture.
2. Select `Filesystem images` to define the format of the FileSystem.

**Make BuildRoot**

```text
make -j <Number of Cores>
```

#### Running QEMU with rootFS

```text
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -boot c -m 4096M -hda ../buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" -serial stdio -display none
```

```text
-m is for the RAM memory to the virtual machine.
```

IF you get an error, maybe try running with `-initrd` option

```text
qemu-system-x86_64 -initrd <ramdisk.img> -kernel arch/x86/boot/bzImage -boot c -m 4096M -hda ../buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" -serial stdio -display none
```

### Creating Minimum Kernel

This is a interesting problem!

First, we need to figure out

* What do we mean by minimum kernel?
* Any limitation on size?
* Any limitation on functionalities?

So, let's say we want to create minimum kernel to work on `x86_64` architecture.

#### Create the Minimum Config

```text
cd linux
rm .config
make tinyconfig ARCH=x86_64
make kvmconfig ARCH=x86_64
```

The above would configure the tiniest possible kernel and enable additional options for kvm guest kernel support \(If we are using KVM\).

Make a copy of this config

```text
cp .config .config.tinykvm
```

**Run the kernel with minimum config in QEMU**

```text
qemu-system-x86_64 -initrd <ramdisk.img> -kernel arch/x86/boot/bzImage -boot c -m 4096M -hda ../buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" -serial stdio -display none
```

Probably, the above won't run and won't produce any messages.

#### Create the Default Working Config

```text
make x86_64_defconfig
make kvmconfig ARCH=x86_64
```

**Run the kernel with default config in QEMU**

```text
qemu-system-x86_64 -initrd <ramdisk.img> -kernel arch/x86/boot/bzImage -boot c -m 4096M -hda ../buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" -serial stdio -display none
```

Probably, the above would run and we should be able to login to the rootfs.

#### Compare the Minimum config with Default Config

Here's the tricky part, based on your experience, we might want to copy the few parts of the default config to minimum config and check if that makes it working. It's painful to make changes and compile everytime. However, the below sections might help

1. Firmware Drivers
2. Partition Types
3. Device Drivers
4. Generic Driver Options
5. SCSI Device Support
6. SCSI Support Type
7. Generic Fallback/ Legacy Drivers
8. Input Device Drivers
9. Hardware I/O ports
10. Character devices
11. Serial Drivers
12. PCMCIA character devices
13. ACPI drivers
14. Native drivers
15. ACPI Drivers
16. Console display driver support
17. USB Host Controller Drivers
18. File Systems
19. CD-ROM/ DVD FileSystems
20. DOS/ FAT/ EXFAT/ NT Filesystems
21. Pseudo Filesystem
22. Kernel Hardening
23. Kernel Hacking
24. Printk and Dmesg options

