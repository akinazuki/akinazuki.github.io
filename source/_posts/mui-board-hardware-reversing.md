---
title: mui Board 折腾记
date: 2026-04-08 00:00:00
tags: reverse-engineering
---

![Clipboard 2026-04-08 AM 10.15.16.png](https://i.see.you/2026/04/08/tBj7/Clipboard-2026-04-08-AM-101516.png)

去年搬家的时候买了一块 [muiBoard](https://muilab.com/ja/products_and_services/muiboard/)——一块日本 mui Lab 做的"木头智能家居面板"，理念是"不打扰的计算"（Calm Technology），不用的时候就是一块普普通通的木板，touch 一下表面才会亮起 LED 矩阵显示时间、天气、家里的智能设备状态。颜值确实高，放在玄关很有格调。但配套 APP 的功能实在过于残废，连个正经的自动化都配不明白，并且触摸交互体验也很糟糕...，于是买来没多久就被我扔进柜子吃灰了。

<!-- more -->

这段时间放假在家闲得无聊，想起柜子里还躺着这么个东西，干脆拆开看看里面是什么料。

## 拆机

![Screenshot_2026-04-08_09-44-32.jpg](https://i.see.you/2026/04/08/S7qs/Screenshot_2026-04-08_09-44-32.jpg)

拧开背板螺丝，里面赫然躺着一块 RPi CM4。好家伙，拆开之前我还担心是 ESP32 之类的低功耗 MCUs，没想到直接上了个树莓派，性能完全够用，甚至有点过剩了。

## I/O 板

但是问题来了：CM4 是 SoM 模组，自己不带 UART 引脚，看了一下它接入的板子也没有明显的 UART Pin，只能配合 IO 扩展板才能引出串口。但是手头上没有现成的，Amazon 上搜了一圈，最后斥巨资 4300 円下单了一块微雪的 CM4 I/O Board。

![Clipboard 2026-04-08 AM 9.02.24.png](https://i.see.you/2026/04/08/3bvZ/Clipboard-2026-04-08-AM-90224.png)


插上成功进入 U-Boot 终端。接下来只要在 bootargs 后面加个 `init=/bin/bash`，就能拿到 root shell 了......吗？

![IMG_8435.jpg](https://i.see.you/2026/04/08/ciV5/IMG_8435.jpg)


## 第一次尝试：setenv bootargs，失败

```
U-Boot> setenv bootargs "... init=/bin/bash"
U-Boot> boot
```

系统……正常启动了。systemd 欢快地拉起一堆服务，完全无视了我设的 bootargs。

`printenv` 看了下，`bootcmd=bootflow scan`。`bootflow scan` 会扫 FAT 分区里的 `boot.scr`，而 boot.scr 的第一行就是：

```
fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
```

它从 DTB 的 `/chosen/bootargs` 节点重新读值覆盖 `$bootargs`，我 setenv 设的东西直接被冲掉了。

这其实是 RPi 的设计习惯：cmdline 放在 FAT 分区的 `cmdline.txt` 里，GPU 固件（start4.elf）在启动时把它写进 DTB 的 `/chosen` 节点。U-Boot 在 RPi 上只是二级接力，不信任自己 env 里的 bootargs，而是信任固件写好的那份。boot.scr 于是固定从 DTB 读，保证跟原生行为一致。

## 第二次尝试：绕过 bootflow，失败

既然 boot.scr 会覆盖，那我不走 bootflow，自己手动 `load + booti` 总行了吧。

```
U-Boot> load mmc 0:1 ${kernel_addr_r} Image
U-Boot> load mmc 0:1 ${fdt_addr_r} bcm2711-rpi-cm4.dtb
U-Boot> setenv bootargs "... console=ttyAMA0,115200 root=/dev/mmcblk0p2 init=/bin/bash"
U-Boot> booti ${kernel_addr_r} - ${fdt_addr_r}

Starting kernel ...
```

然后……就没有然后了。串口再也没有任何输出。

想了想才反应过来，我从 FAT 分区 load 出来的是一份裸 DTB，没有经过固件层面的 overlay 合并，更关键的是，console 设备树节点的配置在 overlay 里，裸 DTB 里根本没正确映射 ttyAMA0。内核起来了，但 console 输出到了一个不存在的地方，活活憋死在黑屏里。

## 手搓 boot.scr 流程，成功

反复分析 boot.scr 的逻辑之后，决定干脆完全复刻它的行为，但自己控制顺序：

```
U-Boot> fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
U-Boot> printenv bootargs
bootargs=coherent_pool=1M 8250.nr_uarts=1 ... console=ttyAMA0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait net.ifnames=0 init=/bin/bash

U-Boot> load mmc 0:3 ${kernel_addr_r} boot/Image
U-Boot> setenv bootargs "${bootargs} root=/dev/mmcblk0p3 init=/bin/bash"
U-Boot> booti ${kernel_addr_r} - ${fdt_addr}
```

关键在于最后一行用的是 `${fdt_addr}` 固件传进来的那份 DTB 地址。

然后 `setenv` 在 boot.scr 逻辑之外追加 `init=/bin/bash`，确保它排在 cmdline 末尾且不会被 boot.scr 二次覆盖。

```
[    3.317547] Freeing unused kernel memory: 4544K
[    3.322216] Run /bin/bash as init process
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.2#
```

这次成功了！

## Dump

拿到 shell 之后的第一件事当然是全盘备份。但它系统自带的 nc 有点问题，管道一接就卡死。手动把网卡拉起来：

```
bash-5.2# mount -t proc proc /proc && mount -t sysfs sys /sys
bash-5.2# ip link set eth0 up
bash-5.2# ip addr add 10.0.0.2/24 dev eth0
```

Mac 这边用 USB 网卡配上 `10.0.0.1/24`

然后用系统自带的 Python 跑个小服务，把 eMMC 的内容走 TCP 发出来：

```python
import socket, subprocess
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 9000))
s.listen(1)
c, _ = s.accept()
p = subprocess.Popen(['dd', 'if=/dev/mmcblk0', 'bs=4M'], stdout=subprocess.PIPE)
while True:
    d = p.stdout.read(65536)
    if not d: break
    c.sendall(d)
```

Mac 端 `nc 10.0.0.2 9000 > mui_dump.img`，等几分钟完整镜像就到手了。

## 分区与意料之中的 LUKS

`gpt -r show mui_dump.img` 看了下分区表：

```
       8192    150128      1  FAT16 boot
     163840   5452594      2  ext4 rootfs A
    5619712   5452594      3  ext4 rootfs B
   11075584    266240      4  LUKS
```

典型的 A/B 双根分区 + 一块 LUKS 数据区。第四个 LUKS1 用了 `aes-xts-plain64, sha256` 加密，没有 key 就只能干瞪眼。

但是！这种开机自动挂载的 LUKS 分区，解密脚本肯定在 rootfs 的某个地方。

```
grep -rl luks ./dump/p2_rootfs/usr/bin/
```

`auto-mount.sh`，bingo：

```bash
ENCRYPTED_DATA=$(fw_printenv store_k | cut -d= -f2)

SERIAL=$(grep Serial /proc/cpuinfo | awk '{print $3}')
KEY="${SERIAL}${SERIAL}${SERIAL}${SERIAL}"

IV=$(echo "$ENCRYPTED_DATA" | cut -d: -f1)
CT=$(echo "$ENCRYPTED_DATA" | cut -d: -f2)
PASS=$(echo "$CT" | openssl enc -aes-256-cbc -d -K "$KEY" -iv "$IV" -base64)

echo "$PASS" | cryptsetup luksOpen $DEVICE luks_partition
```

整个"安全设计"是这样的：

1. LUKS passphrase 用 AES-256-CBC 加密后存在 u-boot 环境变量 `store_k` 里
2. AES key 由 CPU serial 重复 4 次拼出来（serial 是 16 hex = 8 字节，重复 4 次正好 32 字节）
3. CPU serial 从 `/proc/cpuinfo` 直接读

换句话说：**只要能拿到 CPU serial + u-boot env，就能解开 LUKS**。而 CPU serial 根本不是什么秘密，任何能进 U-Boot 或跑起系统的人都能读到；u-boot env 就在 eMMC 上，dump 出来就有。整套"加密"只防住了那种"把 eMMC 吹下来读 LUKS 分区但完全不看 rootfs"的抽象攻击者。


## 顺手看一眼 DTB

看看 `config.txt` 加载的 overlay 列表里：

```
dtoverlay=mpm1-wacom-hid2.dtbo
dtoverlay=hifimems-soundcard
dtoverlay=uart5
dtoverlay=uart3
dtoverlay=disable-bt
```

后面几个是 stock 自带的，`mpm1-wacom-hid2.dtbo` 是 mui 自研的——把它解出来看：

### `mpm1-wacom-hid2.dtbo` — 触摸面板

```dts
i2c-hid-dev@a {
    compatible = "hid-over-i2c";
    reg = <0x0a>;
    hid-descr-addr = <0x01>;
    interrupts = <0x1a 0x08>;   // GPIO 26
};
```

看起来 muiBoard 表面那块木头底下贴的不是普通电容触摸，而是一片完整的 Wacom 数位板模组。难怪触摸和书写的体验跟普通智能家居面板完全不在一个级别（甚至魔改一下能当 OSU 手台？）。

整套硬件定制全部走 RPi 标准的 overlay 机制——一个 Wacom dtbo + 两条 UART overlay 就搞定了。基础 dtb 跟官方完全一致，将来升级 RPi firmware 不会冲突。

这其实是相当克制的工程做法，比某些厂商把整个 dts fork 出来自己魔改、然后永远停留在某个内核版本的做法漂亮多了。残念的还是只有上面那套残废的 APP……

## 收获

muiBoard 硬件是真的不错，木头的质感、点阵触摸屏效果，都做得相当用心。残废的只是官方那套软件栈。现在把软件层彻底掀翻，反而可以当一块"长得很好看的 CM4 开发板"来用，性价比瞬间回正（？）

理论上换一块自己的 CM4 板子上去也是完全可行的，毕竟它的 bootloader 就是标准的 RPi U-Boot
