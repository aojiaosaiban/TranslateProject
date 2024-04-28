---
status: translating
title: "Internals"
author: Syzkaller Community
collector: jxlpzqc
collected_date: 20240314
translator: aojiaosaiban
link: https://github.com/google/syzkaller/blob/master/docs/linux/external_fuzzing_usb.md
---

适用于Linux内核的USB外部模糊测试
=====================================

Syzkaller支持对Linux内核的USB子系统进行外部模糊测试 (例如，通过插入类似[Facedancer](https://github.com/usb-tools/Facedancer)这样的可编程USB设备). This allowed finding over   in the Linux kernel USB stack so far.到目前为止，这已经在Linux内核的USB堆栈发现了超过 [300 bugs](/docs/linux/found_bugs_usb.md) 

USB模糊测试支持由以下3个部分组成：

1. Syzkaller的更改；详细信息请参阅[Internals](/docs/linux/external_fuzzing_usb.md#Internals)部分。
2. 用于USB设备仿真的内核接口，称为[Raw Gadget](https://github.com/xairy/raw-gadget)，现已合并到主线内核。
3. 允许从后台内核线程和中断中收集覆盖率的KCOV更改，现已合并到主线内核中。

查看OffensiveCon 2019[Coverage-Guided USB Fuzzing with Syzkaller](https://docs.google.com/presentation/d/1z-giB9kom17Lk21YEjmceiNUVYeI6yIaG5_gZ3vKC-M/edit?usp=sharing) talk ([video](https://www.youtube.com/watch?v=1MD5JV6LfxA))，了解一些（部分已过时的）细节

由于USB模糊测试需要内核端支持，对于非主线内核，您需要所有涉及`drivers/usb/gadget/udc/dummy_hcd.c`, `drivers/usb/gadget/legacy/raw_gadget.c` 和 `kernel/kcov.c`的主线补丁。


## 内部结构

目前，syzkaller定义了6个USB伪系统调用（参见[syzlang descriptions](/sys/linux/vusb.txt)和[pseudo-syscalls](/executor/common_usb.h) [implementation](/executor/common_usb_linux.h)，它依赖于上面链接的Raw Gadget接口）：

1. `syz_usb_connect` - 连接一个USB设备。处理所有对控制端点的请求，直到收到`SET_CONFIGURATION`请求。
2. `syz_usb_connect_ath9k` - 连接一个`ath9k`USB设备。与`syz_usb_connect`相比，此系统调用还处理了在`ath9k`驱动程序的`SET_CONFIGURATION`之后发生的固件下载请求。
3. `syz_usb_disconnect` - 断开一个USB设备的连接。
4. `syz_usb_control_io` - 在端点0上发送或接收控制消息。
5. `syz_usb_ep_write` - 向非控制端点发送消息。
6. `syz_usb_ep_read` - 从非控制端点接收消息

这些伪系统调用针对几个不同的层次：

1. USB核心枚举过程被通用的`syz_usb_connect`变体所针对。由于这个伪系统调用的USB设备描述符字段被syzkaller运行时修补，因此`syz_usb_connect`还会简要地针对各种USB驱动程序的枚举过程。
2. 针对特定类驱动程序的枚举过程由`syz_usb_connect$hid`、`syz_usb_connect$cdc_ecm`等（提供给它们的设备描述符具有固定的标识USB ID，以始终匹配相同的USB类驱动程序）以及匹配的`syz_usb_control_io$*`伪系统调用来针对。
3. 对于支持的任何支持的类，目前还没有针对特定类驱动程序通过非控制端点进行的后续通信的现有描述。但是可以通过通用的`syz_usb_ep_write`和`syz_usb_ep_read`伪系统调用来触发。
4. 目前还没有现有描述覆盖的特定设备驱动程序的枚举过程。
5. 仅针对`ath9k`驱动程序部分描述了通过非控制端点进行的特定设备驱动程序的后续通信，通过`syz_usb_connect_ath9k`、`syz_usb_ep_write$ath9k_ep1`和`syz_usb_ep_write$ath9k_ep2`伪系统调用。

有关USB伪系统调用的运行测试[runtests](/sys/linux/test/)以`vusb`前缀命名，并且可以使用以下命令运行：

```
./bin/syz-runtest -config=usb-manager.cfg -tests=vusb
```


## 待改进方向

USB模糊测试的核心支持已经就位，但仍然有改善空间：

1. 在`syz_usb_disconnect`中移除`usb_devices`中的设备，并且在`syz_usb_connect`中不要多次调用`lookup_usb_index`。目前，这会导致一些生成器在不需要时设置`repeat`标志

2. 为更多相关的USB类别和驱动程序添加描述.

3. 解决[sys/linux/vusb.txt](/sys/linux/vusb.txt)中的待办事项.

4. 实现一种适当的方式来动态提取内核中相关的USB id（参考[discussion](https://www.spinics.net/lists/linux-usb/msg187915.html)）。

5. 添加一个用于独立模拟物理USB主机的模式（例如使用树莓派Zero）。
这至少包括：
a. 确保当前的USB模拟实现在不同的操作系统上正常工作（协议实现中存在一些差异）；
b. 使用来自主机的USB请求作为信号（如覆盖率）以启用“信号驱动”模糊测试，
c. 使UDC驱动程序名称可配置为`syz-execprog`和`syz-prog2c`。

6. 从由实际USB设备产生的usbmon跟踪中生成syzkaller程序（这应该使模糊测试器能够更深入地进入USB驱动程序代码）。


## 部署

1. 确保你使用的内核版本至少是 5.7。建议将所有涉及到 kcove、USB Raw Gadget 和 USB Dummy UDC/HCD 的内核补丁回溯到对应版本。

2. 配置内核：至少要启用 `CONFIG_USB_RAW_GADGET=y` 和 `CONFIG_USB_DUMMY_HCD=y`。

    最简单的方法是使用 syzbot USB 模糊测试实例中的[config](/dashboard/config/linux/upstream-usb.config)

3. 编译内核

4. 可选地，根据下面的说明[instructions](/docs/linux/external_fuzzing_usb.md#updating-syzkaller-usb-ids)提取 USB IDs，并更新 syzkaller 的描述。

5. 在管理器配置中启用 `syz_usb_connect`、`syz_usb_disconnect`、`syz_usb_control_io`、`syz_usb_ep_write` 和 `syz_usb_ep_read` 伪系统调用。

6. 在管理器配置中将 `sandbox` 设置为 `none`。

7. 在管理器配置中的内核命令行中传递 `dummy_hcd.num=8`（或者你用于 `procs` 的任何数字）。

8. 运行


## 更新 syzkaller USB IDs

Syzkaller uses a list of hardcoded [USB IDs](/sys/linux/init_vusb_ids.go) that are [patched](/sys/linux/init_vusb.go) into `syz_usb_connect` by syzkaller runtime. One of the ways to make syzkaller target only particular USB drivers is to alter that list. The instructions below describe a hackish way to generate syzkaller USB IDs for all USB drivers enabled in your `.config`.

1. Apply [this](/tools/syz-usbgen/usb_ids.patch) kernel patch.

2. Build and boot the kernel.

3. Connect a USB HID device. In case you're using a `CONFIG_USB_RAW_GADGET=y` kernel, use the
[keyboard emulation program](https://raw.githubusercontent.com/xairy/raw-gadget/master/examples/keyboard.c).

4. Use [syz-usbgen](/tools/syz-usbgen/usbgen.go) script to update [syzkaller descriptions](/sys/linux/init_vusb_ids.go):

    ```
    ./bin/syz-usbgen $KERNEL_LOG ./sys/linux/init_vusb_ids.go
    ```

5. Don't forget to revert the applied patch and rebuild the kernel before doing actual fuzzing.


## Running reproducers with Raspberry Pi Zero W

It's possible to run syzkaller USB reproducers by using a Linux board plugged into a physical USB host.
These instructions describe how to set this up on a Raspberry Pi Zero W, but any other board that has a working USB UDC driver can be used as well.

1. Download `raspbian-stretch-lite.img` from [here](https://www.raspberrypi.org/downloads/raspbian/).

2. Flash the image into an SD card as described [here](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

3. Enable UART as described [here](https://www.raspberrypi.org/documentation/configuration/uart.md).

4. Boot the board and get a shell over UART as described [here](https://learn.adafruit.com/raspberry-pi-zero-creation/give-it-life). You'll need a USB-UART module for that. The default login credentials are `pi` and `raspberry`.

5. Get the board connected to the internet (plug in a USB Ethernet adapter or follow [this](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)).

6. Update: `sudo apt-get update && sudo apt-get dist-upgrade && sudo rpi-update && sudo reboot`.

7. Install useful packages: `sudo apt-get install vim git`.

8. Download and install Go:

    ``` bash
    curl https://dl.google.com/go/go1.14.2.linux-armv6l.tar.gz -o go.linux-armv6l.tar.gz
    tar -xf go.linux-armv6l.tar.gz
    mv go goroot
    mkdir gopath
    export GOPATH=~/gopath
    export GOROOT=~/goroot
    export PATH=~/goroot/bin:$PATH
    export PATH=~/gopath/bin:$PATH
    ```

9. Download syzkaller, apply the patch below and build `syz-executor`:

``` c
diff --git a/executor/common_usb_linux.h b/executor/common_usb_linux.h
index 451b2a7b..64af45c7 100644
--- a/executor/common_usb_linux.h
+++ b/executor/common_usb_linux.h
@@ -292,9 +292,7 @@ static volatile long syz_usb_connect_impl(uint64 speed, uint64 dev_len, const ch

        // TODO: consider creating two dummy_udc's per proc to increace the chance of
        // triggering interaction between multiple USB devices within the same program.
-       char device[32];
-       sprintf(&device[0], "dummy_udc.%llu", procid);
-       int rv = usb_raw_init(fd, speed, "dummy_udc", &device[0]);
+       rv = usb_raw_init(fd, speed, "20980000.usb", "20980000.usb");
        if (rv < 0) {
                debug("syz_usb_connect: usb_raw_init failed with %d\n", rv);
                return rv;
```

``` bash
git clone https://github.com/google/syzkaller
cd syzkaller
# Put the patch above into ./syzkaller.patch
git apply ./syzkaller.patch
make executor
mkdir ~/syz-bin
cp bin/linux_arm/syz-executor ~/syz-bin/
```

10. Build `syz-execprog` on your host machine for arm32 with `make TARGETARCH=arm execprog` and copy to `~/syz-bin` onto the SD card. You may try building syz-execprog on the Raspberry Pi itself, but that worked poorly for me due to large memory consumption during the compilation process.

11. Make sure that you can now execute syzkaller programs:

    ``` bash
    cat socket.log
    r0 = socket$inet_tcp(0x2, 0x1, 0x0)
    sudo ./syz-bin/syz-execprog -executor ./syz-bin/syz-executor -threaded=0 -collide=0 -procs=1 -enable='' -debug socket.log
    ```

12. Setup the dwc2 USB gadget driver:

    ```
    echo "dtoverlay=dwc2" | sudo tee -a /boot/config.txt
    echo "dwc2" | sudo tee -a /etc/modules
    sudo reboot
    ```

13. Get Linux kernel headers following [this](https://github.com/notro/rpi-source/wiki).

14. Download and build the USB Raw Gadget module following [this](https://github.com/xairy/raw-gadget/tree/master/raw_gadget).

15. Insert the module with `sudo insmod raw_gadget.ko`.

16. [Download](https://raw.githubusercontent.com/xairy/raw-gadget/master/examples/keyboard.c), build, and run the [keyboard emulator program](https://github.com/xairy/raw-gadget/tree/master/examples):

    ``` bash
    # Get keyboard.c
    gcc keyboard.c -o keyboard
    sudo ./keyboard 20980000.usb 20980000.usb
    # Make sure you see the letter 'x' being entered on the host.
    ```

17. You should now be able to execute syzkaller USB programs:

    ``` bash
    $ cat usb.log
    r0 = syz_usb_connect(0x0, 0x24, &(0x7f00000001c0)={{0x12, 0x1, 0x0, 0x8e, 0x32, 0xf7, 0x20, 0xaf0, 0xd257, 0x4e87, 0x0, 0x0, 0x0, 0x1, [{{0x9, 0x2, 0x12, 0x1, 0x0, 0x0, 0x0, 0x0, [{{0x9, 0x4, 0xf, 0x0, 0x0, 0xff, 0xa5, 0x2c}}]}}]}}, 0x0)
    $ sudo ./syz-bin/syz-execprog -slowdown 3 -executor ./syz-bin/syz-executor -threaded=0 -collide=0 -procs=1 -enable='' -debug usb.log
    ```

    The `slowdown` parameter is a scaling factor which can be used for increasing the syscall timeouts.

18. Steps 19 through 21 are optional. You may use a UART console and a normal USB cable instead of ssh and Zero Stem.

19. Follow [this](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md) to set up a Wi-Fi hotspot.

20. Follow [this](https://www.raspberrypi.org/documentation/remote-access/ssh/) to enable ssh.

21. Optionally solder [Zero Stem](https://zerostem.io/) onto your Raspberry Pi Zero W.

21. You can now connect the board to an arbitrary USB port, wait for it to boot, join its Wi-Fi network, ssh onto it, and run arbitrary syzkaller USB programs.
