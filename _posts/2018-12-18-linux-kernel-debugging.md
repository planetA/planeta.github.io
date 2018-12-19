---
title: Linux kernel debugging
date: 2018-12-19 14:39
categories: Programming
tags: linux kernel
---

Sometimes you think the only way to solve your problem is to debug the
kernel over serial console. At least, this happened to be the case for
me this week. Here, I describe my small adventures, as I found no
article that addressed all the issues I faced.

# Connecting the machines

My setup is following. I have two equivalent machines, both of them
are equipped with a serial port. At some specific point I want to trap
into the kernel and see what happens there. The idea is to connect
over a serial port to a target machine and run gdb on the host
machine. I used three cables to connect the machines: a converter from
10-pin motherboard serial port to DB-9 male port (for example,
[this](http://www.frontx.com/pro/cpx102_2.html)), a female-female
cable (like
[this](https://www.amazon.com/StarTech-com-Straight-Through-Serial-Cable/dp/B00066HP2E)),
and a serial-to-USB cable to be connected to the debugger host (like
[this](http://www.usconverters.com/index.php?main_page=page&id=62&chapter=0)). Theoretically,
of course even one cable ought to be enough, but I did not have the
right one.

## Potential issues

To check if everything is working, you can do the following
test. First, when connecting serial cables to each machine, without
connecting them both together, make a "loopback test". The test runs
as follows: Connect 2nd and 3rd lanes (there are numbers on a DB-9
connector), and try to send something to the port.

Access the serial port in a terminal emulator (minicom or
picocom). Make sure the emulator does not run in echo mode (usually
the default) and type something. If everything works, you will see
what you type on the screen.

The caveat here, is that I first tried to use gender changer instead
of female-to-female cable, that does not swap Rx and Tx lines, as a
result Tx of a host, instead of being connected to Rx of the target,
was connected to Tx of the target. Because of that, when connecting
the two machines, I was not able to communicate over the serial lane.

To check that everything finally works, configure serial port on host
and target:

```bash
stty -F /dev/ttyS0 115200 cs8 -cstopb -parenb -icanon min 1 time 1
stty -F /dev/ttyUSB0 115200 cs8 -cstopb -parenb -icanon min 1 time 1
```

Now, execute `cat /dev/USB0` on the host and `echo sth > /dev/ttyS0`
on the target. If everything works, on the host, you should see
everything you send from the target.

# Compiling the kernel

To have kgdb running, make sure that following flags are set when
compiling the kernel:

```
CONFIG_FRAME_POINTER=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
```

All the options you can set in "Kernel hacking" section of
"menuconfig". To enable frame pointer, select frame pointer unwinder,
instead of ORC unwinder.

The old config usually is available at `/boot/config-$(uname -r)`.

I prefer to build a kernel into a deb package, and then install it
using dpkg. More details one kernel compilation can be found
[here](https://wiki.debian.org/BuildADebianKernelPackage). Following
is summary of the commands:

```bash
sudo apt install build-essential linux-source bc kmod cpio flex cpio libncurses5-dev
cd linux-source*
cp /boot/config-$(uname -r) ./.config
make menuconfig
make -j`nproc` bindeb-pkg
```

Pay attention, after compilation, the whole build directory occupied
21G. Install all the build packages generated in the parent directory.

```bash
sudo dpkg -i ../linux*deb
```

Modify boot parameters in `/boot/grub/grub.cfg` to linux command:

```
linux /boot/vmlinuz-4.18.10 ... kgdboc=ttyS0,115200 sysrq_always_enabled rodata=off nokaslr
```

The parameters do following:

1. `kgdboc` (KGDB-over-Console) sets up serial line and its speed to be used by the remote debugger
2. `sysrq_always_enabled` enable SysRq commands. We will use it to break into the kernel.
3. `rodata` makes read-only data writable
4. `nokaslr` disables kernel-level kernel address-space layout
   randomisation (remember Spectre and Meltdown?), otherwise
   breakpoints do not work.

The full entry looks like this:

```
menuentry 'Debian GNU/Linux (kgdb)' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-64a018ee-9f2c-4917-9711-79c28a190622' {
        load_video
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1  64a018ee-9f2c-4917-9711-79c28a190622
        else
          search --no-floppy --fs-uuid --set=root 64a018ee-9f2c-4917-9711-79c28a190622
        fi
        echo    'Loading Linux 4.18.10 ...'
        linux   /boot/vmlinuz-4.18.10 root=UUID=64a018ee-9f2c-4917-9711-79c28a190622 ro intel_iommu=on kgdboc=ttyS0,115200 sysrq_always_enabled rodata=off nokaslr 
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-4.18.10
}
```

Now, you can reboot. To check the current kernel parameters look at
`/proc/cmdline`.

# Connecting to the debugger

On the host, change to the directory with the compiled kernel and
launch the debugger:

```
sudo gdb ./vmlinux
```

Trap in the guest system. There are
[multiple](https://www.kernel.org/doc/html/v4.18/dev-tools/kgdb.html#quick-start-for-kdb-on-a-serial-port)
ways. Here is one:

```bash
echo g > /proc/sysrq-trigger
```

Now, connect to the remote target over gdb:

```bash
set serial baud 115200
target remote /dev/ttyUSB0
```

Congratulations! Now you can set breakpoints, inspect the kernel, step
into the function, etc. If everything worked properly, you get
readable stack traces with exact source code lines.

## Issues

Most manuals online configure gdb wrongly, as they suggest to set up baud rate as follows:

```
set remotebaud 115200
```

For me, gdb simply complains that it does not know about variable
`remotebaud` in "this context" and connecting to the target does not
work.

You also can enable debug mode to see what gdb sends over serial line:

```
set debug remote 1
```

If at some point you debug a module and symbols are not leaded, use
command `lx-symbols`. For example, I could not navigate file
`nf_conntrack_netfilter.c` until I executed `lx-symbols
./net/netfilter`.
