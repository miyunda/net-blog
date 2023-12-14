---
title: 解决 Big Sur 休眠后自动唤醒的问题
date: 2021-02-03 20:14:34
FeaturedImage: https://cdn.miyunda.net/uPic/bigsur-wake-cover_20231027.png
categories:
  - 消费电子
tags:
  - macOS
---
大家好，本文记录了我解决Big Sur休眠后自动唤醒的问题的手法，假如您也被这个问题困扰，也许这篇文章能帮到您。

<!-- more -->

## C 缘起

我最近使用Big Sur 11.1，用了一段时间，有个问题很烦，就是休眠后电脑会自己唤醒，一晚上好多次。
与是否iCloud同步，连接蓝牙，网络唤醒等等均不相关。
我相信包括我在内的大多数人是不关电脑的（关机），而是休眠。

---

**阅读本文代表您同意：不恰当的操作可能会引起数据丢失，造成的任何损失由您自己负责。**

## Dm 范围

- [x] 硬件：苹果台式机或笔记本电脑
- [x] 软件：运行Big Sur 11.1

## Em 线索

在终端里输入如下命令检查唤醒事件

```bash
$ pmset -g log | grep DarkWake
```

检查其中最频繁的事件，其中有两种：

第一种类似这样的：

{{< admonition quote >}}
DarkWake DarkWake from Deep Idle [CDNP] : due to SMC.OutboxNotEmpty smc.70070000 wifibt wlan/ Using AC (Charge:100%) 6 secs
{{< /admonition >}}

这种是由TCPKeepAlive引起的。

---

第二种类似这样的：

{{< admonition quote >}}
DarkWake DarkWake from Deep Idle [CDNPB] : due to NUB.SPMISw3IRQ nub-spmi.0x02 rtc/Maintenance Using AC (Charge:92%) 45 secs
{{< /admonition >}}

这种是由Power Nap引起的。

## F 解决

### 先升级到Big Sur 11.2

苹果昨天发布了Big Sur 11.2，虽然版本说明里面木有提自动唤醒的事，但是暗中解决了。这个是前提。

### 关闭相应选项

升级完成后

针对第一种，在终端输入：

```bash
sudo pmset -a tcpkeepalive 0
```
---
针对第二种，可以在系统偏好设置里的节能选项里关闭Power Nap（限Intel电脑）；M1的电脑还是在终端输入：

```bash
sudo pmset -a powernap 0
```

好啦，经过了一番操作之后，您的电脑应该不会再失眠啦。谢谢观看。
