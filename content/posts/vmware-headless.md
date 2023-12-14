---
title: VMware Fusion 无头模式
date: 2020-07-24 06:01:34
categories:
  - 消费电子
tags:
  - MacOS
  - VMware
---
本文分享了一点VMware Fusion无头模式的心得

<!-- more -->

出于种种原因，我们在个人电脑上使用虚拟机来运行额外的操作系统：比如我需要运行一些宿主（host）不能运行的软件；比如我需要一个linux测试环境；又比如我需要一个随便折腾的环境，等等。

阅读本文需要您会基本的shell操作。

### 缘起
之前的两年我一直使用Parallels Desktop（下称PD），无奈这公司吃相太难看，变着法er地割韭菜；而Virtual Box（下称VBox）总赶脚在稳定性和性能上不太舒服，~~其实是UI太丑~~；VMware Fusion就成了虚拟机御三家中最后的选择——其他产品太小众就不考虑了。

迁移到VMware还是比较简单的，VMware Fusion提供了向导，直接导入PD虚拟机即可。

然后用着用着我就想VMware Fusion有木有VBox那样的无头模式，不启动图形界面，多少能省点资源。

#### vmrun
感谢VMware提供了`vmrun`命令，这命令有很多用处，参见[官方网站的PDF文档](https://www.vmware.com/pdf/vix162_vmrun_command.pdf)

我只用无头模式那几个。

先把命令建个链接方便在任何路径下使用，此处假设`/usr/local/bin/` 在`PATH=`里面：
```bash
ln -s "$(which vmrun)"  /usr/local/bin/vmrun
```
在虚拟机里设置好网络，打开RDP或者SSH，并且测试下能登录。
然后给我的`<虚拟机名>.vmx`文件建个环境变量，也是为了方便使用，我使用的是ZSH，用BASH的同学自行替换下：
```bash
echo 'export ltsc2019="/Users/yu/Virtual Machines.localized/ltsc2019.vmwarevm/ltsc2019.vmx"' >> ~/.zshrc
source ~/.zshrc
```
以上有多个需要经常使用的虚拟机就多设置几个环境变量。

接下来就可以用了：
```bash
vmrun start "$ltsc2019" nogui
vmrun suspend "$ltsc2019" nogui
```
其中最经常使用的那个虚拟机可以有特别的快捷方式：将以下内容保存为`/usr/local/bin/vm` 并且给它 `chmod +x`
```bash
#!/bin/sh

if [ -z "$ltsc2019" ]
  then
    echo "您还木有设置环境变量呢"
    exit 1
fi

case "$1" in
  start)
    vmrun start "$ltsc2019" nogui
    ;;
  stop)
    vmrun stop "$ltsc2019" nogui
    ;;
  suspend)
    vmrun suspend "$ltsc2019" nogui
    ;;
  pause)
    vmrun pause "$ltsc2019" nogui
    ;;
  unpause)
    vmrun unpause "$ltsc2019" nogui
    ;;
  reset)
    vmrun reset "$ltsc2019" nogui
    ;;
  status)
    vmrun list
    ;;
  *)
    echo "用法: start | stop | suspend | pause | unpause | reset | status"
    exit 1
esac
```
酱紫就可以把最常用的那台省略了，命令行直接输入 `vm start` 或者 `vm suspend`即可.

#### PS
Windows宿主也有这个功能，就是环境变量赋值还有脚本的细节不同，不赘述了。

即使不用无头模式也可以用`vmrun`命令，比如假设您用笔记本电脑，想实现拔了电源自动暂停虚拟机，插上电源自动恢复虚拟机的效果，就可以参考本文结合其他自动化工具实现。

快行动起来吧。

感谢观看。