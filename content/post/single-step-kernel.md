+++
title = "Single-stepping through the Kernel"
date = 2019-02-03T18:57:45+05:30
draft = false
tags = ['tech', 'linux', 'kernel']

+++


There may come a time in a system programmer's life when she needs to leave the civilized safety of the userland and confront the unspeakable horrors that dwell in the depths of the Kernel space. While [higher beings might pour scorn](https://lkml.org/lkml/2000/9/6/65) on the very idea of a Kernel debugger, us lesser mortals may have no other recourse but to single-step through Kernel code when the rivers begin to run dry. This guide will help you do just that. We hope you never actually have to.

Ominous sounding intro-bait notwithstanding, setting up a virtual machine for Kernel debugging isn't really that difficult.  It only needs a bit of preparation. If you just want a copypasta, [skip to the end]({{< relref "#copypasta" >}}). If you're interested in the predicaments involved and how to deal with them, read on.


**N.B.:** "But which kernel are you talking about?", some heathens may invariably ask when it is obvious that Kernel with a capital K refers to the [One True Kernel](https://www.kernel.org/). 


### Building the Kernel

Using a minimal Kernel configuration instead of the kitchen-sink one that distributions usually ship will make life a lot easier. You will first need to grab the source code for the Kernel you are interested in. We will use the latest Kernel release tarball from [kernel.org](https://www.kernel.org/), which at the time of writing is [4.20.6](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.6.tar.xz). Inside the extracted source directory, invoke the following:

``` bash
make defconfig
make kvmconfig
make -j4  
```

This will build a minimal Kernel image that can be booted in QEMU like so:

``` bash
qemu-system-x86_64 -kernel linux-4.20.6/arch/x86/boot/bzImage
```

This should bring up an ancient-looking window with a cryptic error message:

![](/images/kernel_panic.png)

You could try pasting the error message into ~~Google~~ a search engine: Except for the fact that you can't select the text in the window. And frankly, the window just looks annoying! So, ignoring the actual error for a moment, let's try to get QEMU to print to the console instead of a spawning a new graphical window:

``` bash
qemu-system-x86_64 -kernel -nographic linux-4.20.6/arch/x86/boot/bzImage
```

QEMU spits out a single line:
``` bash
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
```

[Htop](https://hisham.hm/htop/) tells me QEMU is using 100% of a CPU and my laptop fan agrees. But there is no output whatsoever and `Ctrl-c` doesn't work! What [does work](https://superuser.com/a/1211516), however, is pressing `Ctrl-a` and then hitting `x`:
``` bash
QEMU: Terminated
```

Turns out that by passing `-nographic`, we have plugged out QEMU's *virtual* monitor. Now, to actually see any output, we need to tell the Kernel to write to a [serial port](https://www.kernel.org/doc/html/v4.20/admin-guide/serial-console.html):

``` bash 
qemu-system-x86_64 -nographic -kernel linux-4.20.6/arch/x86/boot/bzImage -append "console=ttyS0"
```

It worked! Now we can read error message in all its glory:
``` bash
[    1.333008] VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
[    1.334024] Please append a correct "root=" boot option; here are the available partitions:
[    1.335152] 0b00         1048575 sr0 
[    1.335153]  driver: sr
[    1.335996] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    1.337104] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.20.6 #1
[    1.337901] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.2-1ubuntu1 04/01/2014
[    1.339091] Call Trace:
[    1.339437]  dump_stack+0x46/0x5b
[    1.339888]  panic+0xf3/0x248
[    1.340295]  mount_block_root+0x184/0x248
[    1.340838]  ? set_debug_rodata+0xc/0xc
[    1.341357]  mount_root+0x121/0x13f
[    1.341837]  prepare_namespace+0x130/0x166
[    1.342378]  kernel_init_freeable+0x1ed/0x1ff
[    1.342965]  ? rest_init+0xb0/0xb0
[    1.343427]  kernel_init+0x5/0x100
[    1.343888]  ret_from_fork+0x35/0x40
[    1.344526] Kernel Offset: 0x1200000 from 0xffffffff81000000 (relocation range: 0xffffffff80000000-0xffffffffbfffffff)
[    1.345956] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

So, the Kernel didn't find a root filesystem to kick off the user mode and panicked. Lets fix that by creating a root filesystem image.

### Creating a Root Filesystem

Start by creating an empty image:
``` bash
qemu-img create rootfs.img 1G
```

And then format it as [`ext4`](https://en.wikipedia.org/wiki/Ext4) and mount it:
``` bash
mkfs.ext4 rootfs.img
mkdir mnt
sudo mount -o loop rootfs.img mnt/
```

Now we can populate it using [`debootstrap`](https://wiki.debian.org/Debootstrap):
``` bash
sudo debootstrap bionic mnt/
```

This will create a root filesystem based on Ubuntu 18.04 Bionic Beaver. Of course, feel free to replace `bionic` with any release that you prefer.

And unmount the filesystem once we're done. **This is important if you want to avoid corrupted images!**
``` bash
sudo umount mnt
```

Now boot the Kernel with our filesystem. We need to tell QEMU to use our image as a virtual hard drive and we also need to tell the Kernel to use the hard drive as the root filesystem:
``` bash
qemu-system-x86_64 -nographic -kernel linux-4.20.6/arch/x86/boot/bzImage -hda rootfs.img -append "root=/dev/sda console=ttyS0" 
```

This time the Kernel shouldn't panic and you should eventually see a login prompt. We could have setup a user while creating the filesystem but it's annoying to have to login each time we boot up the VM. Let's enable auto login as root instead.

Terminate QEMU (`Ctrl-a`, `x`), mount the filesystem image again and then create the configuration folder structure:
``` bash
sudo mount -o loop rootfs.img mnt/
sudo mkdir -p mnt/etc/systemd/system/serial-getty@ttyS0.service.d
```

Add the following lines to `mnt/etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf`:
``` bash
[Service]
ExecStart=
ExecStart=-/sbin/agetty --noissue --autologin root %I $TERM
Type=idle
```

Make sure to unmount the filesystem and then boot the Kernel again. This time you should be automatically logged in.

Gracefully shutdown the VM:
``` bash
halt -p
```


### Attaching a debugger

Let's rebuild the Kernel with debugging symbols enabled:
``` bash
./scripts/config -e CONFIG_DEBUG_INFO
make -j4
```

Now, boot the Kernel again, this time passing the `-s` flag which will make QEMU listen on TCP port 1234:
``` bash
qemu-system-x86_64 -nographic -kernel linux-4.20.6/arch/x86/boot/bzImage -hda rootfs.img -append "root=/dev/sda console=ttyS0" -s
```

Now, in another terminal start gdb and attach to QEMU:
``` bash
gdb ./linux-4.20.6/vmlinux 
...
Reading symbols from ./linux-4.20.6/vmlinux...done.
(gdb) target remote :1234
Remote debugging using :1234
0xffffffff95a2f8f4 in ?? ()
(gdb)
``` 

You can set a breakpoint on Kernel function, for instance `do_sys_open()`:
``` bash
(gdb) b do_sys_open 
Breakpoint 1 at 0xffffffff811b2720: file fs/open.c, line 1049.
(gdb) c
Continuing.
```

Now try opening a file in VM which should result in `do_sys_open()` getting invoked... And nothing happens?! The breakpoint in gdb is not hit. This due to a Kernel security feature called [KASLR](https://lwn.net/Articles/569635/). KASLR can be disabled at boot time by adding `nokaslr` to the Kernel command line arguments. But, let's actually rebuild the Kernel without KASLR. While we are at it, let's also disable loadable module support as well which will save us the trouble of copying the modules to the filesystem.

``` bash
./scripts/config -e CONFIG_DEBUG_INFO -d CONFIG_RANDOMIZE_BASE -d CONFIG_MODULES
make olddefconfig # Resolve dependencies
make -j4
```

Reboot the Kernel again, attach gdb, set a breakpoint on `do_sys_open()` and run `cat /etc/issue` in the guest. This time the breakpoint should be hit. But probably not where you expected:
``` bash
Breakpoint 1, do_sys_open (dfd=-100, filename=0x7f96074ad428 "/etc/ld.so.cache", flags=557056, mode=0) at fs/open.c:1049
1049    {
(gdb) c
Continuing.

Breakpoint 1, do_sys_open (dfd=-100, filename=0x7f96076b5dd0 "/lib/x86_64-linux-gnu/libc.so.6", flags=557056, mode=0) at fs/open.c:1049
1049    {
(gdb) c
Continuing.

Breakpoint 1, do_sys_open (dfd=-100, filename=0x7ffe9e630e8e "/etc/issue", flags=32768, mode=0) at fs/open.c:1049
1049    {
(gdb)
```

Congratulations! From this point, you can single-step away to your heart's content.

By default, the root filesystem is mounted read only. If you want to be able to write to it, add `rw` after `root=/dev/sda` in the Kernel parameters:
``` bash
qemu-system-x86_64 -nographic -kernel linux-4.20.6/arch/x86/boot/bzImage -hda rootfs.img -append "root=/dev/sda rw console=ttyS0" -s
```

### Bonus: Networking

You can create a point to point link between the QEMU VM and the host using a [TAP interface](https://en.wikipedia.org/wiki/TUN/TAP).

First install `tunctl` and create a persistent TAP interface to avoid running QEMU as root:

``` bash
sudo apt install uml-utilities
sudo sudo tunctl -u $(id -u)
Set 'tap0' persistent and owned by uid 1000
sudo ip link set tap0 up
```

Now launch QEMU with a virtual `e1000` interface connected the host's tap0 interface:

``` bash
qemu-system-x86_64 -nographic -device e1000,netdev=net0 -netdev tap,id=net0,ifname=tap0 -kernel linux-4.20.6/arch/x86/boot/bzImage -hda rootfs.img -append "root=/dev/sda rw console=ttyS0" -s
```

Once the guest boots up, bring the network interface up:
``` bash
ip link set enp0s3 up
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::5054:ff:fe12:3456/64 scope link 
       valid_lft forever preferred_lft forever
```

QEMU and the host can now communicate using their IPv6 Link-local addresses. After all, it is 2019.

### Copypasta

``` bash
# Building a minimal debuggable Kernel
make defconfig
make kvmconfig
./scripts/config -e CONFIG_DEBUG_INFO -d CONFIG_RANDOMIZE_BASE -d CONFIG_MODULES
make olddefconfig
make -j4


# Create root filesystem
qemu-img create rootfs.img 1G
mkfs.ext4 rootfs.img
mkdir mnt
sudo mount -o loop rootfs.img mnt/
sudo debootstrap bionic mnt/

# Add following lines to mnt/etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf
# START
[Service]
ExecStart=
ExecStart=-/sbin/agetty --noissue --autologin root %I $TERM
Type=idle
# END

# Unmount the filesystem
sudo umount mnt

# Boot Kernel with root file system in QEMU
qemu-system-x86_64 -nographic -kernel linux-4.20.6/arch/x86/boot/bzImage -hda rootfs.img -append "root=/dev/sda rw console=ttyS0" -s

# Attach gdb
gdb ./linux-4.20.6/vmlinux 
(gdb) target remote :1234
```

 
