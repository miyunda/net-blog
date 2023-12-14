---
title: 移除macOS预装应用
date: 2019-11-24 05:02:18
categories:
  - 消费电子
tags:
  - 预装
  - 精简
  - macOS
---
>我花钱买的软件，凭什么要用？！

<!-- more -->

macOS有一些预装的应用，有的还挺好的，有的不需要，特别是某台计算机只做工作用途，我就想删除那些用不上的应用。

阅读本文并不需要信息技术背景。

### 温馨提示
**想好了再动手，删错了东西与我无瓜。聪明人先备份**

### 步骤

#### 关闭SIP
macOS默认是有系统完整性保护（SIP）的。这个功能打开时有一些安全性增强，其中之一就是禁止删除预装应用
1. 按住Cmd+R启动计算机到恢复模式
2. 然后启动终端
![在恢复模式中启动终端](https://cdn.miyunda.net/image/remove-macos-presinstalled-app-20191124142739.jpg)
3. 输入以下命令（每次输入一行然后按回车）
```bash
csrutil disable
reboot
```
#### 删除
1. 按Cmd+空格，然后输入`terminal.app`，这将打开一个终端窗口。
![terminal.app](https://cdn.miyunda.net/image/remove-macos-presinstalled-app-20191124145458.jpg)
2. Catalina用户需要输入以下命令，Mojave和更早的版本可以跳过。
```bash
sudo mount -uw /
```
3. 在终端窗口中输入`sudo rm -rf `，**不要着急按回车,看清楚里面有三个空格**。
4. 打开一个访达，选中看着不顺眼的App，拖到终端里，回车。
![sudo rm -rf ](https://cdn.miyunda.net/uPic/remove-macos-presinstalled-app_20231029.gif)
熟练了之后可以直接像下面这样:
```bash
cd /System/Applications # Catalina
cd /Applications # Mojave以及更早
sudo rm -rf Maps.app
sudo rm -rf TV.app
sudo rm -rf News.app
sudo rm -rf FaceTime.app
sudo rm -rf Contacts.app
sudo rm -rf Chess.app
sudo rm -rf Books.app
sudo rm -rf Messages.app
sudo rm -rf Photos.app
sudo rm -rf VoiceMemos.app
sudo rm -rf Time\ Machine.app
sudo rm -rf Siri.app
sudo rm -rf Music.app
sudo rm -rf Podcasts.app
sudo rm -rf Stocks.app
sudo rm -rf Image\ Capture.app
sudo rm -rf FindMy.app
sudo rm -rf Home.app
```
##### 打开SIP
在**恢复模式**的终端里运行
```bash
csrutil enable
reboot
```
快行动起来吧。
感谢观看。
