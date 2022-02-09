---
title: Qemu下模拟ARM64搭建GDB Server调试环境
date: 2019-06-22 16:42:43
categories:
- tech
tags:
- linux
- qemu
- arm
- kernel
---

OS： ubuntu 18.04 LTS x64

### Qemu

#### Install

需要模拟arm64平台，所以要安装aarch64版本
`sudo apt-get install qemu-system-aarch64`

### Cross-compile

安装交叉编译工具链，需要把一些依赖的其他库安装好

`sudo apt-get install flex bison build-essential pkg-config libglib2.0-dev libpixman-1-dev libssl-dev`

这里不使用系统仓库自带的`gcc-aarch64-linux-gnu`，仓库里面的gcc版本不好调整为自己需要的，所以直接下载[Linaro网站](http://releases.linaro.org/components/toolchain/binaries/7.4-2019.02/)的.

Linaro网站提供了多个平台的交叉编译工具，也一直有更新，ubuntu 64位的系统选择`x86_64_aarch64-linux-gnu`版本，我用的是
`gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu`

下载到开发目录arm下后，解压

```bash
$ cd arm
$ tar -xvf gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz
```


### Busy Box

下载busybox代码也到arm目录下，解压

```bash
$ cd arm
$ tar -xvf busybox-1.23.1.tar.gz
```
进入busybox根目录，先配置当前的环境变量为arm64

```bash
$ export ARCH=arm64
$ export CROSS_COMPILE=/home/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```
执行`make menuconfig`打开编译配置菜单，其中做以下配置
* 勾选静态编译 `Settings->Build static binary (no shared lib)`
* 指定交叉编译器为：`/home/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-`
* General Configuration --> Dont use /usr
* Busybox Libary Tuning--> 勾选：[\*]Username completion、[\*]Fancy shell prompts 、[\*]Query  cursor  position  from  terminal 

保存配置后，会更新`.config`编译配置文件，可以打开确认编译信息的正确性

开始编译`make -j4`

最后执行`make install`在busybox根目录生成`_install`目录

### Linux kernel

#### Linux Kernel下载

[Kernel官网](https://www.kernel.org/ )下载4.9.11版本的内核，不能下载太旧的版本，例如3.19和最新的gcc7.4不兼容，编译总是失败，提示COMPILE版本的错误信息。最好选择长期支持的版本，这样功能更稳定一些。

解压内核后配置环境变量后，可以对内核进行配置

在执行`make menuconfig`时会遇到
> In file included from scripts/kconfig/mconf.c:23:0:
scripts/kconfig/lxdialog/dialog.h:38:20: fatal error: curses.h: No such file or directory
 include CURSES_LOC                   
compilation terminated.
make[1]: *** [scripts/kconfig/mconf.o] Error 1
make: *** [menuconfig] Error 2

此时需要安装**ncurses devel** `sudo apt-get install libncurses5-dev`

``` bash
tar -xvf linux-4.19.11.tar
cd linux-4.19.11
# 配置环境变量为arm64
export ARCH=arm64
# 配置交叉工具链
export CROSS_COMPILE=/home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
# 根据当前的环境变量的arch类型，到内核的arch目录中把arch/arm64/configs/中的配置作为模板
make defconfig
# 打开配置菜单界面，此时配置菜单中可以看到当前的目标类型和工具链类型
make menuconfig
```

#### 配置Kernel

根据需要把支持的设备勾选，不想支持的就不要勾选，否则编译时间太长.可以第一次多裁减一些，如果需要，后面在配置增加功能，把每一次修改的`.config`文件版本管理起来

Platform Selection只选择`ARMv8 based Freescale Layerscape SoC family`和`ARMv8 software model (Versatile Express)`

Device Driver中普通程序不要支持的也可删除

因为要通过内存镜像启动内核，还需要配置使用内存文件系统

`General setup->Initial RAM filesystem and RAM disk (initramfs/initrd) support`

`Device Drivers->Block devices-><*> RAM block device support`，其中配置1个block`(1)     Default number of RAM disks `内存大小为128M`(131072) Default RAM disk size (kbytes) `

如果需要调试内核，需要打开调试信息
```
kernel hacking-->
    [*]compile the kernel with debug info
```

配置完成后，执行`make -j12` 开始编译内核，时间需要1个多小时

### Run kernel

#### 创建根文件系统

在编译内核的过程中，可以准备内核启动的根文件系统，这里参考了[摩斯电码](https://www.cnblogs.com/pengdonglin137/p/6431234.html)的脚本文件，做了简单修改

```sh
#!/bin/bash

sudo rm -rf rootfs
sudo rm -rf tmpfs
sudo rm -rf ramdisk*
# 创建根文件系统目录
sudo mkdir rootfs
# 把busybox拷贝到这里 _install 里面就2个目录和1个文件`bin\  linuxrc  sbin\`
sudo cp ../busybox-1.23.1/_install/*  rootfs/ -raf
# 初始化根目录结构
sudo mkdir -p rootfs/proc/
sudo mkdir -p rootfs/sys/
sudo mkdir -p rootfs/tmp/
sudo mkdir -p rootfs/root/
sudo mkdir -p rootfs/var/
sudo mkdir -p rootfs/mnt/
# 系统配置目录
sudo cp etc rootfs/ -arf
# 公共库目录
sudo mkdir -p rootfs/lib
# 后续编译程序也要依赖同样的库文件
sudo cp -arf ../gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/lib/* rootfs/lib/
# 删除静态库，文件太大
sudo rm rootfs/lib/*.a
# strip减小so体积
sudo ../gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-strip rootfs/lib/*
# 初始化的设备
sudo mkdir -p rootfs/dev/
sudo mknod rootfs/dev/tty1 c 4 1
sudo mknod rootfs/dev/tty2 c 4 2
sudo mknod rootfs/dev/tty3 c 4 3
sudo mknod rootfs/dev/tty4 c 4 4
sudo mknod rootfs/dev/console c 5 1
sudo mknod rootfs/dev/null c 1 3
# dd Copy a file, converting and formatting according to the operands.
# if 输入文件 /dev/zero 表示一个尽量满足需要的无限大的文件，且文件内容都初始化为0
# of 输出文件 bs : block size  count : num of blocks
# 这里的块数量需要根据rootfs目录文件大小调整，目前我的是57M
sudo dd if=/dev/zero of=ramdisk bs=1M count=64
# mkfs.ext4 will create a file system for use with ext4
sudo mkfs.ext4 -F ramdisk

sudo mkdir -p tmpfs
# -t : fs type -o : option loop : loop device
# 把文件系统镜像文件挂载到一个loop device上，从而可以把roofs的文件拷贝进去
sudo mount -t ext4 ramdisk ./tmpfs/ -o loop

sudo cp -raf rootfs/*  tmpfs/
sudo umount tmpfs

sudo gzip --best -c ramdisk > ramdisk.gz
# 创建镜像文件
sudo mkimage -n "ramdisk" -A arm64 -O linux -T ramdisk -C gzip -d ramdisk.gz ramdisk.img
```

The **loop  device** is a block device that maps its data blocks not to a
physical device such as a hard disk or optical disk drive, but  to  the
blocks  of  a  regular file in a filesystem or to another block device. This can be useful for example to provide a block device for a filesystem image stored in a file, so that it can be mounted with the mount(8)
command


其中etc目录结构如下
```sh
etc
├── init.d   #初始脚本目录
|   └── rcS  #启动时默认执行脚本
├── sysconfig  
|   └── HOSTNAME  #登陆后的主机名保存在这里
├── fstab      # fs mount
├── inittab    # init
└── profile    # shell环境变量
```

* /etc/init.d/rcS
```sh
#!/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
runlevel=S
prevlevel=N
umask 022
export PATH runlevel prevlevel

mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
#mount -n -t usbfs none /proc/bus/usb
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
mkdir -p /var/lock

ifconfig lo 127.0.0.1
ifconfig eth0 192.168.43.202 netmask 255.255.255.0 broadcast 192.168.43.255

/bin/hostname -F /etc/sysconfig/HOSTNAME
```

* /etc/sysconfig/HOSTNAME
```
aarch64
```

* /etc/fstab
```sh
#device		mount-point	type	options		dump	fsck order
proc		/proc		proc	defaults		0	0
tmpfs		/tmp		tmpfs	defaults		0	0
sysfs		/sys		sysfs	defaults		0	0
tmpfs		/dev		tmpfs	defaults		0	0
var		    /dev		tmpfs	defaults		0	0
ramfs		/dev		ramfs	defaults		0	0
debugfs		/sys/kernel/debug	debugfs		defaults	0	0
```

* /etc/inittab
```sh
# /etc/inittab
::sysinit:/etc/init.d/rcS
console::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::restart:/sbin/init
```

* /etc/profile
```sh
USER="root"
LOGNAME=$USER
export HOSTNAME=`/bin/hostname`
export USER=root
export HOME=/root
export PS1="[$USER@$HOSTNAME \W]\# "
PATH=/bin:/sbin:/usr/bin:/usr/sbin
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export PATH LD_LIBRARY_PATH
```

对于生成的image文件可以通过`mkimage -l ramdisk.img`查看文件信息
```
Image Name:   ramdisk
Created:      Sun Jun 23 21:18:57 2019
Image Type:   AArch64 Linux RAMDisk Image (gzip compressed)
Data Size:    15885428 Bytes = 15513.11 kB = 15.15 MB
Load Address: 00000000
Entry Point:  00000000
```

#### 使用Qemu运行

* run.sh
```sh
qemu-system-aarch64 \
    -M  virt \
    -cpu cortex-a53 \
    -smp 2 \
    -m 1024M \
    -kernel ./linux-4.19.11/arch/arm64/boot/Image \
    -nographic \
    -append "root=/dev/ram0 rw rootfstype=ext4 console=ttyAMA0 init=/linuxrc ignore_loglevel" \
    -initrd ./rootfs/ramdisk.img \
    -netdev tap,helper=/usr/lib/qemu/qemu-bridge-helper,id=hn0 -device virtio-net-pci,netdev=hn0,id=nic1 \
    -fsdev local,security_model=passthrough,id=fsdev0,path=/home/edison/develop/arm/nfsroot \
    -device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare
```



### 共享目录

使用9p共享目录，内核在编译时默认是支持的
新建目录
`mkdir nfsroot`

启动时这两个选项

```
-fsdev local,security_model=passthrough,id=fsdev0,path=/home/edison/arm/nfsroot \
-device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare
```

指明了共享目录的位置

在内核启动起来之后，把共享目录挂载上来，就可以看到文件了
也可以把这个mount添加到内核启动程序中，不用每次都执行一遍
```
[root@aarch64 ]# mount -t 9p -o trans=virtio,version=9p2000.L hostshare /mnt
[root@aarch64 ]# ls /mnt/
code   
```

### Network with Qemu

使用网桥方式，可以让qemu和host主机之间直接进行网络通信

1. 安装网桥工具 
`sudo apt install bridge-utils` 和 `sudo apt install uml-utilities`
2. 新建一个网桥 `sudo brctl addbr br0` 网桥会在重启后消失
3. 启用此网桥 `sudo ip link set br0 up` 
4. 确认`/etc/qemu/bridge.conf`中`allow br0`
5. 给帮助程序权限`sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper`
6. qemu 启动时增加`-netdev tap,helper=/usr/lib/qemu/qemu-bridge-helper,id=hn0 -device virtio-net-pci,netdev=hn0,id=nic1`
7. qemu 启动后会自动在host主机上新建一个tap0的网卡
8. 使用`brctl show`查看br0和tap0已经关联上了
9. 把host主机的一个网卡也和br0关联起来，主机wifi的网卡由于是dhcp获取的ip，无法与br0绑定，需要使用有线网卡绑定`sudo brctl addif br0 enp5s0`

```
bridge name	bridge id		STP enabled	interfaces
br0		8000.3860773ac46e	no		enp5s0
							tap0
```

10. host设置各个网卡和网桥的ip，**此处需要注意先设置br0的ip和tap0的ip，再设置host网卡的ip，否则guest里面无法ping外部主机的ip，最终使br0的mac和tap0的mac地址相同**，具体原因还没来及查
`sudo ifconfig br0 192.168.43.210 netmask 255.255.255.0`
`sudo ifconfig tap0 192.168.43.51 netmask 255.255.255.0`
`sudo ifconfig enp5s0 192.168.43.50 netmask 255.255.255.0`

```
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.43.210  netmask 255.255.255.0  broadcast 192.168.43.255
        inet6 fe80::1429:b3ff:fe07:5f92  prefixlen 64  scopeid 0x20<link>
        ether fe:16:30:37:22:4f  txqueuelen 1000  (Ethernet)

tap0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.43.51  netmask 255.255.255.0  broadcast 192.168.43.255
        inet6 fe80::fc16:30ff:fe37:224f  prefixlen 64  scopeid 0x20<link>
        ether fe:16:30:37:22:4f  txqueuelen 1000  (Ethernet)

enp5s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.43.50  netmask 255.255.255.0  broadcast 192.168.43.255
        ether 38:xx:xx:xx:xx:xx  txqueuelen 1000  (Ethernet)
```

11. guest设置eth0的ip 与br0的ip在一个网段内 例如 192.168.43.202

`qemu-bridge-helper`使用`/etc/qemu-ifup`和`/etc/qemu-ifdown`来控制虚拟虚拟机网卡tap0启动

* 如果想使用其他定义的网桥, `/etc/qemu/bridge.conf`中添加`allow qemubr0`
```
qemu linux.img 
-netdev tap,helper="/usr/local/libexec/qemu-bridge-helper --br=qemubr0",id=hn0 -device virtio-net-pci,netdev=hn0,id=nic1
```

### Gdbserver

到GDB网站下载gdb的源码，其中gdbserver在里面的子目录gdbserver中，进入gdbserver的源码目录

```bash
$ cd ~/develop/arm/gdb-8.3/gdb/gdbserver
$ export CC=/home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc
$ export CXX=/home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-g++

$ ./configure --target=aarch64-linux-gnu --host=aarch64-linux-gnu
```

把编译出来的gdbserver放到共享目录

qemu 作为客户机执行

`#./gdbserver 192.168.43.202:10000 all` 

192.168.43.202 is guest ip address
output:
```
Process /mnt/code/all created; pid = 1066
Listening on port 10000
Remote debugging from host 192.168.43.210, port 51730
```

主机host run:

`/home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gdb all`

in gdb console, connect to the guest gdbserver:
```sh
(gdb) target remote 192.168.43.202:10000
Reading /lib/ld-linux-aarch64.so.1 from remote target...
warning: File transfers from remote targets can be slow. Use "set sysroot" to access files locally instead.
Reading /lib/ld-linux-aarch64.so.1 from remote target...
Reading symbols from target:/lib/ld-linux-aarch64.so.1...(no debugging symbols found)...done.
0x0000ffffbf6d3d00 in ?? () from target:/lib/ld-linux-aarch64.so.1
# 设置一个目录，否则看不到库函数
(gdb) set sysroot /home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/
warning: .dynamic section for "/home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/lib/ld-linux-aarch64.so.1" is not at the expected address (wrong library or version mismatch?)
Reading symbols from /home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/lib/ld-linux-aarch64.so.1...done.
Reading symbols from /home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/lib/ld-linux-aarch64.so.1...done.
(gdb) b main
Breakpoint 1 at 0x4005f4: file main.cpp, line 7.
(gdb) b func(int) 
Breakpoint 2 at 0x400630: file main.cpp, line 16.
(gdb) r
The "remote" target does not support "run".  Try "help target" or "continue".
(gdb) c
Continuing.

Breakpoint 1, main () at main.cpp:7
7	  int i = 25;
(gdb) list
2	 
3	int func(int i);
4	 
5	int main(void)
6	{
7	  int i = 25;
8	  int v = func(i);
9	  printf("value is %d\n", v);
10	  getchar();
11	  return 0;
(gdb) c
Continuing.

Breakpoint 2, func (i=25) at main.cpp:16
16	  int a = 2;
(gdb) c
Continuing.
[Inferior 1 (process 1066) exited normally]

```

#### 测试程序

```c++
#include <stdio.h>
 
int func(int i);
 
int main(void)
{
  int i = 25;
  int v = func(i);
  printf("value is %d\n", v);
  getchar();
  return 0;
}
 
int func(int i)
{
  int a = 2;
  return a * i;
}
```
* 简单的makefile
```makefile
# marcros
CROSS_COMPILE := /home/edison/develop/arm/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-

CC		:= $(CROSS_COMPILE)gcc
LD		:= $(CC) -nostdlib
CPP		:= $(CC) -E

CCFLAGS := -Wall
DBGFLAG := -g
CCOBJFLAG := $(CCFLAG) -c

# Path

BIN_PATH := bin
OBJ_PATH := obj
SRC_PATH := src
DBG_PATH := debug

# compile 
TARGET_NAME := main

TARGET := $(BIN_PATH)/$(TARGET_NAME)
TARGET_DEBUG := $(DBG_PATH)/$(TARGET_NAME)

all: main.o
	$(CC) -o $@ $^

main.o: main.cpp
	$(CC) $(CCOBJFLAG) $(DBGFLAG) $^
    
clean:
	rm -rf *.o all
```


### 启动运行信息
```sh
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 4.19.11 (edison@aquarius) (gcc version 7.4.1 20181213 [linaro-7.4-2019.02 revision 56ec6f6b99cc167ff0c2f8e1a2eed33b1edc85d4] (Linaro GCC 7.4-2019.02)) #3 SMP PREEMPT Sat Jun 15 12:02:57 CST 2019
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] debug: ignoring loglevel setting.
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 32 MiB at 0x000000007e000000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000000000000-0x000000007fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x7dfea700-0x7dfebebf]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] On node 0 totalpages: 262144
[    0.000000]   DMA32 zone: 4096 pages used for memmap
[    0.000000]   DMA32 zone: 0 pages reserved
[    0.000000]   DMA32 zone: 262144 pages, LIFO batch:63
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] random: get_random_bytes called from start_kernel+0xa8/0x418 with crng_init=0
[    0.000000] percpu: Embedded 23 pages/cpu @(____ptrval____) s56984 r8192 d29032 u94208
[    0.000000] pcpu-alloc: s56984 r8192 d29032 u94208 alloc=23*4096
[    0.000000] pcpu-alloc: [0] 0 [0] 1 
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: enabling workaround for ARM erratum 843419
[    0.000000] CPU features: enabling workaround for ARM erratum 845719
[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 258048
[    0.000000] Policy zone: DMA32
[    0.000000] Kernel command line: root=/dev/ram0 rw rootfstype=ext4 console=ttyAMA0 init=/linuxrc ignore_loglevel
[    0.000000] Memory: 969596K/1048576K available (9020K kernel code, 610K rwdata, 3008K rodata, 768K init, 359K bss, 46212K reserved, 32768K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=2.
[    0.000000] 	Tasks RCU enabled.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv2m: range[mem 0x08020000-0x08020fff], SPI[80:143]
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000185] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.007286] Console: colour dummy device 80x25
[    0.009634] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.009828] pid_max: default: 32768 minimum: 301
[    0.011320] Security Framework initialized
[    0.013353] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.014631] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes)
[    0.014987] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes)
[    0.015139] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes)
[    0.072332] ASID allocator initialised with 32768 entries
[    0.079862] rcu: Hierarchical SRCU implementation.
[    0.102195] EFI services will not be available.
[    0.111945] smp: Bringing up secondary CPUs ...
[    0.150710] Detected VIPT I-cache on CPU1
[    0.152735] CPU1: Booted secondary processor 0x0000000001 [0x410fd034]
[    0.158057] smp: Brought up 1 node, 2 CPUs
[    0.158170] SMP: Total of 2 processors activated.
[    0.158288] CPU features: detected: 32-bit EL0 Support
[    0.185724] CPU: All CPU(s) started at EL1
[    0.186917] alternatives: patching kernel code
[    0.205598] devtmpfs: initialized
[    0.234248] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.234617] futex hash table entries: 512 (order: 3, 32768 bytes)
[    0.245880] pinctrl core: initialized pinctrl subsystem
[    0.275845] DMI not present or invalid.
[    0.285543] NET: Registered protocol family 16
[    0.289290] audit: initializing netlink subsys (disabled)
[    0.292277] audit: type=2000 audit(0.252:1): state=initialized audit_enabled=0 res=1
[    0.311872] cpuidle: using governor menu
[    0.314254] vdso: 2 pages (1 code @ (____ptrval____), 1 data @ (____ptrval____))
[    0.314476] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.325699] DMA: preallocated 256 KiB pool for atomic allocations
[    0.328282] Serial: AMBA PL011 UART driver
[    0.401940] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 39, base_baud = 0) is a PL011 rev1
[    0.433798] console [ttyAMA0] enabled
[    0.727257] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.733955] cryptd: max_cpu_qlen set to 1000
[    0.744142] ACPI: Interpreter disabled.
[    0.760164] vgaarb: loaded
[    0.765256] SCSI subsystem initialized
[    0.773399] libata version 3.00 loaded.
[    0.785663] usbcore: registered new interface driver usbfs
[    0.787906] usbcore: registered new interface driver hub
[    0.789752] usbcore: registered new device driver usb
[    0.794877] pps_core: LinuxPPS API ver. 1 registered
[    0.795307] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.796439] PTP clock support registered
[    0.806539] EDAC MC: Ver: 3.0.0
[    0.828166] Advanced Linux Sound Architecture Driver Initialized.
[    0.849084] clocksource: Switched to clocksource arch_sys_counter
[    0.851823] VFS: Disk quotas dquot_6.6.0
[    0.854846] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.858595] pnp: PnP ACPI: disabled
[    1.017342] NET: Registered protocol family 2
[    1.031887] tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes)
[    1.033022] TCP established hash table entries: 8192 (order: 4, 65536 bytes)
[    1.034055] TCP bind hash table entries: 8192 (order: 5, 131072 bytes)
[    1.034752] TCP: Hash tables configured (established 8192 bind 8192)
[    1.038780] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    1.039445] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    1.042094] NET: Registered protocol family 1
[    1.050677] RPC: Registered named UNIX socket transport module.
[    1.051236] RPC: Registered udp transport module.
[    1.051576] RPC: Registered tcp transport module.
[    1.051922] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    1.053121] PCI: CLS 0 bytes, default 64
[    1.058331] Trying to unpack rootfs image as initramfs...
[    1.071951] rootfs image is not initramfs (no cpio magic); looks like an initrd
[    1.219963] Freeing initrd memory: 15512K
[    1.225178] hw perfevents: enabled with armv8_pmuv3 PMU driver, 1 counters available
[    1.227220] kvm [1]: HYP mode not available
[    1.290935] Initialise system trusted keyrings
[    1.295592] workingset: timestamp_bits=44 max_order=18 bucket_order=0
[    1.563944] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    1.620068] NFS: Registering the id_resolver key type
[    1.626786] Key type id_resolver registered
[    1.627912] Key type id_legacy registered
[    1.630868] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    1.652401] 9p: Installing v9fs 9p2000 file system support
[    1.664508] pstore: using deflate compression
[    1.817988] Key type asymmetric registered
[    1.819643] Asymmetric key parser 'x509' registered
[    1.823133] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 246)
[    1.827632] io scheduler noop registered
[    1.828884] io scheduler deadline registered
[    1.834561] io scheduler cfq registered (default)
[    1.836114] io scheduler mq-deadline registered
[    1.837955] io scheduler kyber registered
[    1.926575] pl061_gpio 9030000.pl061: PL061 GPIO chip @0x0000000009030000 registered
[    1.944322] pci-host-generic 3f000000.pcie: host bridge /pcie@10000000 ranges:
[    1.950902] pci-host-generic 3f000000.pcie:    IO 0x3eff0000..0x3effffff -> 0x00000000
[    1.957916] pci-host-generic 3f000000.pcie:   MEM 0x10000000..0x3efeffff -> 0x10000000
[    1.962099] pci-host-generic 3f000000.pcie:   MEM 0x8000000000..0xffffffffff -> 0x8000000000
[    1.969611] pci-host-generic 3f000000.pcie: ECAM at [mem 0x3f000000-0x3fffffff] for [bus 00-0f]
[    1.983121] pci-host-generic 3f000000.pcie: PCI host bridge to bus 0000:00
[    1.987641] pci_bus 0000:00: root bus resource [bus 00-0f]
[    1.992250] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    1.995159] pci_bus 0000:00: root bus resource [mem 0x10000000-0x3efeffff]
[    1.998891] pci_bus 0000:00: root bus resource [mem 0x8000000000-0xffffffffff]
[    2.010065] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000
[    2.038555] pci 0000:00:01.0: [1af4:1000] type 00 class 0x020000
[    2.042423] pci 0000:00:01.0: reg 0x10: [io  0x0000-0x001f]
[    2.044329] pci 0000:00:01.0: reg 0x14: [mem 0x00000000-0x00000fff]
[    2.047344] pci 0000:00:01.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[    2.050395] pci 0000:00:01.0: reg 0x30: [mem 0x00000000-0x0007ffff pref]
[    2.066248] pci 0000:00:02.0: [1af4:1009] type 00 class 0x000200
[    2.069640] pci 0000:00:02.0: reg 0x10: [io  0x0000-0x003f]
[    2.072306] pci 0000:00:02.0: reg 0x14: [mem 0x00000000-0x00000fff]
[    2.075211] pci 0000:00:02.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[    2.103755] pci 0000:00:01.0: BAR 6: assigned [mem 0x10000000-0x1007ffff pref]
[    2.109717] pci 0000:00:01.0: BAR 4: assigned [mem 0x8000000000-0x8000003fff 64bit pref]
[    2.113851] pci 0000:00:02.0: BAR 4: assigned [mem 0x8000004000-0x8000007fff 64bit pref]
[    2.115820] pci 0000:00:01.0: BAR 1: assigned [mem 0x10080000-0x10080fff]
[    2.118111] pci 0000:00:02.0: BAR 1: assigned [mem 0x10081000-0x10081fff]
[    2.119817] pci 0000:00:02.0: BAR 0: assigned [io  0x1000-0x103f]
[    2.122333] pci 0000:00:01.0: BAR 0: assigned [io  0x1040-0x105f]
[    2.211197] EINJ: ACPI disabled.
[    2.330390] virtio-pci 0000:00:01.0: enabling device (0000 -> 0003)
[    2.354839] virtio-pci 0000:00:02.0: enabling device (0000 -> 0003)
[    2.512241] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    2.593580] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    2.638856] brd: module loaded
[    2.756131] loop: module loaded
[    2.834762] libphy: Fixed MDIO Bus: probed
[    2.844183] tun: Universal TUN/TAP device driver, 1.6
[    2.909715] thunder_xcv, ver 1.0
[    2.911181] thunder_bgx, ver 1.0
[    2.912558] nicpf, ver 1.0
[    2.921499] e1000e: Intel(R) PRO/1000 Network Driver - 3.2.6-k
[    2.922236] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    2.925385] igb: Intel(R) Gigabit Ethernet Network Driver - version 5.4.0-k
[    2.926237] igb: Copyright (c) 2007-2014 Intel Corporation.
[    2.928072] igbvf: Intel(R) Gigabit Virtual Function Network Driver - version 2.4.0-k
[    2.929604] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    2.932820] sky2: driver version 1.30
[    2.948916] VFIO - User Level meta-driver version: 0.3
[    2.954444] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    2.955462] ehci-pci: EHCI PCI platform driver
[    2.957773] ehci-platform: EHCI generic platform driver
[    2.961430] usbcore: registered new interface driver usb-storage
[    2.991082] rtc-pl031 9010000.pl031: rtc core: registered pl031 as rtc0
[    2.997556] i2c /dev entries driver
[    3.024361] sdhci: Secure Digital Host Controller Interface driver
[    3.030621] sdhci: Copyright(c) Pierre Ossman
[    3.035477] Synopsys Designware Multimedia Card Interface Driver
[    3.043428] sdhci-pltfm: SDHCI platform and OF driver helper
[    3.056220] ledtrig-cpu: registered to indicate activity on CPUs
[    3.086735] usbcore: registered new interface driver usbhid
[    3.087646] usbhid: USB HID core driver
[    3.115425] NET: Registered protocol family 17
[    3.121396] 9pnet: Installing 9P2000 support
[    3.127838] Key type dns_resolver registered
[    3.140496] registered taskstats version 1
[    3.141477] Loading compiled-in X.509 certificates
[    3.165868] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    3.174798] rtc-pl031 9010000.pl031: setting system clock to 2019-06-23 13:50:18 UTC (1561297818)
[    3.179007] ALSA device list:
[    3.179612]   No soundcards found.
[    3.190059] uart-pl011 9000000.pl011: no DMA platform data
[    3.197681] RAMDISK: gzip image found at block 0
[    8.860079] EXT4-fs (ram0): mounted filesystem with ordered data mode. Opts: (null)
[    8.861974] VFS: Mounted root (ext4 filesystem) on device 1:0.
[    8.870895] devtmpfs: mounted
[    8.997547] Freeing unused kernel memory: 768K
[    9.031224] Run /linuxrc as init process

Please press Enter to activate this console. 
[root@aarch64 ]# ls
bin         etc         linuxrc     mnt         root        sys         var
dev         lib         lost+found  proc        sbin        tmp
```
