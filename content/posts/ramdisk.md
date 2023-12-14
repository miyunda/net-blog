---
title: 使用RamDisk
date: 2021-01-21 22:00:45
categories:
  - 消费电子
tags:
  - RamDisk
  - Win10
  - MacOS
---
大家好，本文描述了使用在个人电脑上开启并使用RamDisk（内存磁盘）的基本步骤

<!-- more -->

## C 段子

先来个段子：
2019年4月的时候，有“朋友的朋友”让我帮着给参谋参谋她新装电脑的配置，问我多大内存够用，问我是因为我比较“懂电脑”。其实这个问题的答案因人而异，人和人的需求是不同的。我当时正好入手了新的电脑，就~~绝非炫耀性地绝不显摆地~~以我的64GB内存为例给她简单地说了说。后来她不再理我了，她跟我们共同的朋友反馈说：

>这人（指我）不懂装懂，64哪够啊，起码得128的！最好上256的！

~~您自己[攒]^(cuán)手机去吧~~

---

本文尽量针对命令行操作零基础的同学，高手请自行跳过过低内容。

**阅读本文代表您同意：不恰当的操作可能会引起数据丢失，造成的任何损失由您自己负责。**

## Dm 反向swap

感谢政府，大家纷纷用上了性能更好的电脑，其中就包括更大的内存容量，内存大了干什么用呢？
RamDisk的技术古已有之，就是用内存来虚拟硬盘驱动器，容量不得超过物理内存的容量（废话）。
日常使用电脑时，有一些内容是不需要长期保存的，只是一些临时的文件，我们可以将临时文件存在RamDisk里，关机/断电即消失；缓存的文件夹啦；P2P的文件夹啦；爬虫爬回来的数据啦；另外性能也比M.2 PCI-E要好——虽然不太能感觉到——起码能节省写入次数，我们发展中人民就是这么节省的啦。

## Em 开启 RamDisk

### Win10 开启RamDisk

Win10 有一些免费的App可以实现这个功能。我用的是[Radeon™ RAMDisk](http://www.radeonramdisk.com/software_downloads.php)，对就是 AMD Yes! 的那个（别走，Intel CPU的电脑也能用）。免费版最大4GB，支持多个虚拟硬盘但是总和不得超过授权容量；收费的有几个档次，我一开始买的12GB，2017年初又加钱升级到24GB。这家服务还挺好的，买的时候说好了授权只能转移一次（redeem），后来我又找客服给我转了好几次，人家都挺痛快地给转了。

### Linux 开启RamDisk

Linux用户不需要看这篇文章，自己去搞吧

### MacOS 开启RamDisk

和Linux一样，MacOS自带RamDisk的功能，免费。
先打开脚本编辑器
![ramdisk-script.png](https://cdn.miyunda.net/image/ramdisk-script.png)

然后粘贴下列内容：

```bash
do shell script "
 
if ! test -e /Volumes/\"RamDisk\" ; then
 
diskutil erasevolume HFS+ \"RamDisk\" `hdiutil attach -nomount ram://33554432`
 
fi
 
 
mkdir -p /Volumes/RamDisk/Library/Caches/Google

mkdir -p /Volumes/RamDisk/Library/Caches/Firefox
 
mkdir -p /Volumes/RamDisk/Library/Containers/com.apple.Safari/Data/Library/Caches/com.apple.Safari/


mkdir -p /Volumes/RamDisk/Downloads
 
"
```

并保存为`/Applications/RamDisk.app`

![ramdisk-save.png](https://cdn.miyunda.net/image/ramdisk-save.png)

其中`33554432`的单位是扇区（sector），每个扇区512 bytes，上面脚本将创建一个16GB的RamDisk，要是只需要4GB的内存磁盘，就可以改成`83888608`；它还建立四个文件夹，其中Safari缓存的位置是Big Sur的，Mojave用户可以换成：

```bash
mkdir -p /Volumes/RamDisk/Library/Caches/com.apple.Safari/fsCachedData
```

注：四个文件夹只是示例，请根据自己的需求添加删改，一般都在 `~/Library/Caches/`
然后让它自动运行：
![ramdisk-login.gif](https://cdn.miyunda.net/image/ramdisk-login.gif)

## F 使用

### Win10 使用RamDisk

安装好AMD RamDisk之后先启动

![ramdisk-win-1.PNG](https://cdn.miyunda.net/image/ramdisk-win-1.PNG)

然后用资源管理器到R:盘去建立需要的文件夹。比如"R:\\Downloads" 之后回到AMD RamDisk点下保存。

![ramdisk-win-2.PNG](https://cdn.miyunda.net/image/ramdisk-win-2.PNG)

之后把下载文件夹搬走，在下载文件夹点右键，选好新地方之后点击移动：

![ramdisk-move-downloads.PNG](https://cdn.miyunda.net/image/ramdisk-move-downloads.PNG)

把临时文件夹搬走：打开系统高级属性的环境变量，把临时文件夹都设置为"R:\\Temp"：

![ramdisk-temp.png](https://cdn.miyunda.net/image/ramdisk-temp.png)

改新Edge的缓存位置：
去把“C:\Users\\%Username%\\AppData\\Local\\Microsoft\\Edge\\User Data\\Default” 里面的 “Cache” 删除
然后右键点开始菜单，在管理员权限的命令行输入：

![ramdisk-powershell.PNG](https://cdn.miyunda.net/image/ramdisk-powershell.PNG)

```powershell
mklink /d "C:\Users\%USERNAME%\AppData\Local\Microsoft\Edge\User Data\Default\Cache" "R:\EdgeCache"
```

Chrome同理，Firefox可以参考MacOS的

### MacOS 使用RamDisk

重启电脑，或者手动运行一下`RamDisk.app`。
然后启动一个终端，按Cmd+空格然后输入`terminal`回车即可。

![ramdisk-terminal.png](https://cdn.miyunda.net/image/ramdisk-terminal.png)

输入命令看看内存磁盘是不是建好了:

```bash
diskutil info /Volumes/RamDisk
```

先把下载文件夹改到新地方：

```bash
sudo rm -rf ~/Downloads/
ln -s /Volumes/RamDisk/Downloads/ ~/Downloads
```

注：以上操作可能会导致访达不在侧边显示下载文件夹，自己去改下就好了。

![ramdisk-sidebar.png](https://cdn.miyunda.net/image/ramdisk-sidebar.png)
把Chrome的也改咯：

```bash
rm -r  ~/Library/Caches/Google
ln -s /Volumes/RamDisk/Library/Caches/Google ~/Library/Caches/Google
```

还有Safari

```bash
rm -r ~/Library/Containers/com.apple.Safari/Data/Library/Caches/com.apple.Safari
ln -s /Volumes/RamDisk/Library/Containers/com.apple.Safari/Data/Library/Caches/com.apple.Safari ~/Library/Containers/com.apple.Safari/Data/Library/Caches/com.apple.Safari
```

注：最好在关闭了浏览器的时候弄。
Firefox的不是这么弄。得打开浏览器，在地址栏输入`about:config`
然后新建一个字符串（String），名字为`browser.cache.disk.parent_directory`，值为`/Volumes/RamDisk/Library/Caches/Firefox`

![ramdisk-firefox.gif](https://cdn.miyunda.net/image/ramdisk-firefox.gif)

新版Firefox的界面有些变化，起手略有区别，实际上是一样的

![ramdisk-firefox.png](https://cdn.miyunda.net/image/ramdisk-firefox.png)

其他App需要改的基本上就这两种思路，要么创建文件系统的链接；要么在App的设置界面里改。

好啦，经过了一番操作之后，您的电脑有木有更好用了呢。感谢观看～～
