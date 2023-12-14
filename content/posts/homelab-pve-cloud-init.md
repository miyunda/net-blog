---
title: 搭建家庭简易IT实验室——cloud-init
cover: https://cdn.miyunda.net//uPic/background.webp
coverWidth: 512
coverHeight: 320
date: 2022-06-10 18:30:45
categories:
  - 信息技术
tags:
  - 虚拟化
  - cloud-init
  - 数字化过家家
  - homelab
---
大家好，本文是《搭建简易家庭IT实验室》系列文章的第二篇，描述了如何创建及销毁我的PVE中的虚拟机。所有内容并非面向Linux大佬，而是面对像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。有什么不对或者可以做得更好的地方，还请大家不吝赐教。

<!-- more -->

阅读本文需要您知晓虚拟化的基本概念，建议先阅读上一篇[《PVE安装后》](https://miyunda.com/homelab-pve-postinstall/)

**您已知晓并同意，不恰当的操作可能会导致经济损失，由您自己承担，雨我无瓜。**

## C 前情提要
上一篇文章我无比事逼滴调整了几处设置，得到的是一个单节点的PVE，无高可用性（HA），无集群。存储方面就是系统默认的。
![pve defaultstorage](https://cdn.miyunda.net//uPic/pve-vm-01.png)
其中“*local*”将用来存放ISO文件，*LVM-Thin*用来存放虚拟机的磁盘。
也可以使用SSH登入终端查看：
```bash
sudo pvesm status
```
列出里面存放的文件信息：
```bash
sudo pvesm list <上个命令输出存储名字>
```

## Dm 准备镜像文件

要先安装虚拟机，一般得先有个“安装盘”的ISO。
![pve guest iso](https://cdn.miyunda.net//uPic/pve-vm-02.png)
可以直接上传，也可以从指定的网址下载。之后就可以去右上角那里创建虚拟机了，后面步骤跟着向导走就好了，哪里不会点哪里。

## Em 云计算优化的镜像
上一节是传统的安装虚拟机方法，能用，装几个Linux服务器不成问题。但是我想要云服务厂商那种镜像。各位在御三家（AWS/Azure/GCP）里面创建运算实例的时候都见过那种，内核裁剪过，吃更少的资源。还能配合各种自动化方案快速制备/销毁。红帽系和乌班图系都有相应的镜像，其中Debian官方的云镜像在https://cloud.debian.org/images/cloud/ ，我们登入PVE，下载一个通用版：
```bash
wget https://cdimage.debian.org/cdimage/cloud/bullseye/latest/debian-11-genericcloud-amd64.qcow2
```
小白请注意：这不是什么ISO文件，这是一个Linux服务器的硬盘镜像。

## F 创建一个模版机
以下命令创建一个 编号为200的虚拟机，它有一个桥接网卡，使用DHCP获取IP地址。
>例子中的CPU可能要在BIOS里允许CPU虚拟化，不确定的话可以省略这个选项，它会使用默认的*kvm64*作为CPU类型.
```bash
sudo qm create 200 --name bullseye-vm-tmpl --memory 4096 --net0 virtio,bridge=vmbr0 --cores=2 --cpu host
```
然后复制刚才下载的镜像并导入到虚拟机*200*，存储使用*local-lvm*：
```bash
sudo qm importdisk 200 debian-11-genericcloud-amd64.qcow2 local-lvm
```
给新来的硬盘一个硬件路径：
```bash
sudo qm set 200 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-200-disk-0
```
它原来是2GB的硬盘，给它扩大点
```bash
sudo qm resize 200 scsi0 8G
```
告诉新的虚拟机从且仅从这硬盘引导
```bash
sudo qm set 200 --boot order=scsi0
```
给它装个显示输出，串口的
```bash
sudo qm set 200 --serial0 socket --vga serial0
```
## G cloud-init
在继续下一步之前，我们有必要了解一下`cloud-init`。想象一下，当您克隆出多台Linux服务器然后它们都叫一个名字那有多烦，更别说各种id，什么SSH身份的一堆东西都重复的。`cloud-init`在Linux引导之后重新初始化相关信息。不过与各大云服务厂商不同，他们的`cloud-init`会去`169.254.169.254`取元数据（metadata），咱们的就相对简陋一点，元数据由PVE写在虚拟光盘里，不过用起来效果是一样的。老朋友们会记得去年我在[Oracle Cloud运算实例的简易部署](https://miyunda.com/oci-terraform/)中使用过`cloud-init`。想要更多信息的朋友请自行去[红帽官网](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_cloud-init_for_rhel_8/introduction-to-cloud-init_cloud-content)学习。

要打开`could-init`选项，PVE使用的方法是挂载一个虚拟光驱：
```bash
sudo qm set 200 --ide2 local-lvm:cloudinit
```
还能建用户及导入对应的SSH公钥，比如这样就在虚拟光盘里面放置了一个文件，告诉cloud-init去做哪些事：
```bash
sudo qm set 200 --ciuser miyunda --sshkey  ~/.ssh/id_ed25519.pub
```
>官方教程说搞好之后不要开机，要不然它会再初始化一次，这个是错的，不要理他们，按我的做就行。

然后就可以开机了，SSH登进去做一番设置/安装，反正就是作为模版机那些共同需要的东西都搞好。比如在**模版机**上安装软件：
```bash
sudo apt-get install qemu-guest-agent
```
确定都妥了之后模版机就可以关机了，关机之后令PVE将其转换为模版：
```bash
sudo qm template 200
```
### 自定义cloud-init文件
假如需要用cloud-init管理的项目太多，那么把他们写在一个文件里会更方便，pve会把这个文件写到虚拟光盘里面，我们的虚拟机开机之后会从这个文件得到指示。

得先开启“片段（snippet）”功能，默认未开启：

![enable snippets](https://cdn.miyunda.net/uPic/Downloads/homelab-cloudinit-enbalesinppets.png)
或者去SSH去pve：
```bash
sudo pvesm set local --content vztmpl,backup,iso,snippets
```

## Am 一键克隆
之后就可以利用模版机~~无性繁殖生三胎了~~：
```bash
sudo qm clone 200 201 --full --name k8s-prod-01
```
稍等片刻之后，克隆出来的虚拟机就可以用了。

新的虚拟机不想要了就可以销毁：
```bash
sudo qm stop 201
sudo qm destroy 201
```
## Bdim 总结
本次大家学会了如何安装一台适合用于云计算的Debian模版机，并利用模版机克隆出我们需要的多个Linux服务器。在下一篇文章中，我们将尝试使用Terraform来获得更高效率的虚拟机生命周期管理。

---
好了，看过就等于会了，谢谢观看～