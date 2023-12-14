---
title: 搭建家庭简易IT实验室——PVE安装后
cover: https://cdn.miyunda.net//uPic/background.webp
coverWidth: 512
coverHeight: 320
date: 2022-06-05 05:32:54
categories:
  - 信息技术
tags:
  - 虚拟化
  - PVE
  - 数字化过家家
  - homelab
---
大家好，本文是《搭建家庭简易IT实验室》系列文章的第一篇，描述了我如何在安装完成后调整PVE服务器设置。系列文章还包括Terraform、Ansible、Github Action、Prometheus等内容，~~也有可能随时鸽~~。所有内容并非面向Linux大佬，而是面对像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。有什么不对或者可以做得更好的地方，还请大家不吝赐教。

<!-- more -->

阅读本文需要您知晓虚拟化的基本概念。动手实验需要您会基本的bash操作，并且有一台闲置电脑。配置不限，AMD/Intel都行，~~吃灰派请继续吃灰~~。最好还有一个自己控制的域名。

**您已知晓并同意，不恰当的操作可能会导致经济损失，由您自己承担，雨我无瓜。**

## C 缘起

有个说法。说是中年油腻男人有三宝：充电头、NAS和路由器。我省了省自己，觉得还好，并木有特别沉迷那三宝。其实IT呆子还有一宝：homelab，姑且称之为家庭IT实验室。IT实验环境不是什么陌生的概念，各企业的IT环境通常都有“Lab”，用来给员工做一些实验用，安全策略相对正式环境要宽松些，但是用来数字化过家家可能不太行，毕竟不是自己家的，不能随便折腾。所以我是一直在家里运行ESXi，感谢VMware提供的授权，闲暇时候拿出来把玩几下，不亦乐乎。

但是前几天，五月底吧，我收到一封邮件，VMware发的，美滋滋滴，说被博通（Broadcom）收购了，我一看标题好一似凉水浇头怀里抱着冰：博通那么抠门的公司，以后不送授权了那我岂不是占不到便宜了？再说vCenter的授权一直就是要钱买的，木有这玩意各种API都实现不了，想弄点自动化的东西都要靠SSH来搞，太不靠谱了。所以我在寻找一个替代品。

Proxmox Virtual Environment（又称Proxmox VE，下称PVE）我也是久仰了，基于Debian，支持LXC的容器虚拟化，和KVM的硬件全抽象层虚拟化，还提供 REST API，最重要的是免费！大户交订阅费就行了，小散就一分钱不花。

安装就不说了，去https://www.proxmox.com/en/downloads下载，然后找个闲置电脑跟着向导装就行了，[教程在这里](https://pve.proxmox.com/wiki/Installation)。

## D 更换软件源

登入SSH，然后先安装证书，以识别TUNA的证书，此步骤可选，仅适用于TUNA的CA太新，以至于无法被PVE（Debian）识别的时候。这次安装的版本为PVE7.2，无此问题。
```bash
apt update
apt install apt-transport-https ca-certificates
```
引入TUNA的Debian源：
```bash
mv /etc/apt/sources.list /etc/apt/sources.list.bak
nano /etc/apt/sources.list
```
内容为：
```
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
```
删除PVE的商业仓库：
```bash
mv /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources.list.d/pve-enterprise.list.bak
```
引入TUNA的免费仓库：
```bash
wget https://enterprise.proxmox.com/debian/proxmox-release-bullseye.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bullseye.gpg
echo "deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian bullseye pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```
更多信息在[TUNA官网](https://mirrors.tuna.tsinghua.edu.cn/help/proxmox/)。

## E 账号与用户

在未与其他认证服务集成的时候，管理员可以指定账号使用PVE来认证身份，或者使用Linux自带的PAM（Pluggable Authentication Modules），二必选一。设我们的PVE服务器名字叫*Proxmox*, 那么可以同时或分别存在下列账号：
  - [x] miyunda@proxmox，密码由PVE验证
  - [x] miyunda@pam，密码由Debian验证，需要在Debian里面有个同名用户
>账号都是访问网页控制台和REST API用的，访问PVE所在的宿主操作系统与这些无关。比如我可以用foo@proxmox来访问网页控制台，用bar来访问SSH，操作系统里面不需要有foo用户，PVE里也不需要有bar账号。

### 先弄一个SSH用的用户：
肯定不能一直用root登录Linux的SSH，对于这一块不熟悉的可以参考[Linux服务器小白教程](https://miyunda.com/linux/)。
### 还得弄一个网页控制台的账号：
我在网页控制台禁用了root账号：
![pve-postinstall-disable-root](https://cdn.miyunda.net/uPic/pve-postinstall-disable-root.png)

并且创建了一个自己用的账号，具有一切权限：
![pve-postinstall-new-admin](https://cdn.miyunda.net/uPic/pve-postinstall-new-admin.png)

## F 性能调整

### swap太大了（可选）

这PVE向导装的时候我也木有特别注意，安装好了一看，好家伙。给我弄了8GB的swap。我这个环境木有什么吃内存的，就是玩玩。占那么多硬盘不合适。

用root身份（`sudo su -`）在终端里面看看：
```bash
lvdisplay | grep swap
```
如果您像我一样什么都木有改一路默认下来的话那么应该是这样的：
```bash
  LV Path                /dev/pve/swap
  LV Name                swap
```
先把它关了：
```bash
swapoff -v /dev/pve/swap
```
然后给它改成1GB，意思意思得了：
```bash
lvreduce /dev/pve/swap -L 1G
```
然后格式化并且装载：
```bash
mkswap /dev/pve/swap
swapon -va
free -h
```
别着急，挤出来的空间还浪费着呢，先看看有几个卷组（volume group）
```bash
vgs
```
一般就一个，再看看逻辑卷的信息（logical volume）
```bash
lvs
```
这回就多了，注意其中那个叫**data**的。

接下来看看设备名字：
```bash
fdisk -l
```
其中最大的就是我们需要的，它的类型是“Linux LVM”， 我这里是**nvme0n1p3**。

把多余的空间加给它：
```bash
pvresize /dev/nvme0n1p3
```
然后就可以把空间加给逻辑卷：
```bash
lvresize -l +100%FREE /dev/pve/data
```
之后不需要运行`resize2fs /dev/pve/data`这种了，因为PVE给虚拟机存储用的是“raw”格式。

### CPU自动降频 （可选）
我这个情景不需要什么性能，就是自己玩玩而已，所以CPU频率低点好，我要省电。注意CPU频率低也影响PCIE的性能，比如NVME硬盘读写也会变慢，不过这都不重要。

以root身份在SSH终端看看当前是运行在什么频率：
```bash
watch "lscpu | grep MHz"
```
看看当前系统支持什么策略：
```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_available_governors
```
从里面挑了一个：
```bash
echo "ondemand" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```
然后我发现不起作用，还是不会降频，于是去搞内核的开关：
```bash
nano /etc/default/grub
```
里面有一行是这样的：
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```
把它改成这样的：
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_pstate=disable“
```
然后重启：
```bash
update-grub
reboot
```
安装CPU频率工具:
```bash
apt install cpufrequtils
```
获取当前CPU频率调度信息：
```bash
cpufreq-info
```
可以看到当前起作用的是“**performance**”：
> The governor "performance" may decide which speed to use

改它：
```bash
nano /etc/init.d/cpufrequtils
```
里面有两行改成这样：
```
...
ENABLE="true"
GOVERNOR="ondemand"
...
```
然后令此服务载入新配置：
```bash
systemctl reload cpufrequtils.service
```
再次使用`cpufreq-info`观察，已经节能降频了。
```
#### 查看温度
可以看CPU、芯片组以及硬盘的温度：
```bash
sudo apt install lm-sensors
sensors
```
## F SSL证书（可选）
无论访问网页控制台还是REST API，作为http服务器（其实是个反向代理服务器），PVE是支持SSL证书的。SSL证书的好处就不用说了，肯定要用的。默认情况下，PVE是绑了自签名证书，可以令常用的客户端（浏览器等）信任其颁发者（issuer）即可；您也可以上传安装自行购买的证书；或者也可以利用内置的ACME脚本来申请并安装一个Let's Encrypt证书。 这里的ACME验证有两种：http和DNS，无非都是为了证明您对声明的域名有控制权。想要使用http验证，需要能被访问到TCP80和TCP443端口，就算了；我们将使用DNS验证。

整个过程差不多就是：

  1. PVE：我要一个SSL证书，名字要*proxmox.foobar.com*。
  2. Let's Encrypt：那你建个DNS text记录，名字是*_acme-challenge.proxmox.foobar.com*，内容是blabla。
  3. PVE：建完了，你解析下。
  4. Let's Encrypt：查到了，给你证书。

注意整个过程并不需要查询*proxmox.foobar.com*的A记录，是否存在，存在的话解析到什么IP地址，并不重要。
### DNS服务器
为了ACME脚本能在您控制的DNS服务器里管理记录，您得在DNS服务商那边准备一个凭证。我用的Cloudflare，这家服务商可以创建一个有区域（zone）编辑权限的API令牌（token）。

![cloudflare zone api token](https://cdn.miyunda.net//uPic/pve-postinstall05.png)

### ACME账号
之前我说把root账号禁用那里，有点说早了，ACME的功能只有系统原生root账号才能用，自建的管理员账号不给用ACME功能，所以可以等证书好了再禁用root账号。
![pve-postinstall-acme](https://cdn.miyunda.net/uPic/pve-postinstall-acme.png)

### ACME插件
这个支持很多DNS服务商的，用哪家的就加进来就好了，但是好像木有看到鹅厂。
![pve-postinstall-acme-plugin](https://cdn.miyunda.net/uPic/pve-postinstall-acme-plugin.png)
 
### 申请证书
甚至可以为多个FQDN申请证书，成品将以SANdns或者通配符的形式提供。
![pve-postinstall-order-cert](https://cdn.miyunda.net/uPic/pve-postinstall-order-cert.png)

## G 对时
时间同步很重要，但是这次就略过了，很简单的。

## Am 总结
这次我们在PVE安装完成之后进行了一番设置，虽然大多不是什么必须的设置，但是令我们使用PVE更方便更快捷。下一篇文章我们将开始创建虚拟机。

---
谢谢观看，看过就等于会了。