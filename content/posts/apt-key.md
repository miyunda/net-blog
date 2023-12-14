---
title: apt-key不安全?
date: 2021-03-17 06:30:29
FeaturedImage: https://cdn.miyunda.net/uPic/apt-key-cover_20231027.jpeg
categories:
  - 信息技术
tags:
  - Debian
  - APT
  - Ubuntu
  - Mint
---
大家好，本文描述了如何不使用`apt-key add`给Debian系Linux添加第三方的源。

<!-- more -->

阅读本文需要您理解apt包管理的基本概念，会基本的bash操作。

**你已知晓并同意，不恰当的操作可能会导致损失，任何损失由您自己承担，与我无瓜**

## C 缘起

在使用Linux操作系统时，往往需要添加第三方的源，源使用OpenPGP格式的签名来确保安全，在Debian系的系统中，这可以通过apt-key add命令来添加。
```bash
sudo apt-key add <3rdparty-repo.gpg>
```
使用的时候它提示

{{< admonition warning >}}
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
{{< /admonition >}}

然后我看了下 `man apt-key`，总之就是虽然能用，但是建议不用，过几年连用都用不了了。

## Dm 发生肾么事了
究其原因，我们把OpenPGP格式的密钥添加到`/etc/apt/trusted.gpg`或者`/etc/apt/trusted.gpg.d`里面，本意为了认证特定的源；但是这个密钥被APT无条件滴信任，可以用来认证其它木有签名的源，甚至可以替换Ubuntu/Debian官方的源提供的包。所以这是一个不安全的操作。

[Debian维基](https://wiki.debian.org/DebianRepository/UseThirdParty)说的是：

{{< admonition quote >}}
The key MUST be downloaded over a secure mechanism like HTTPS to a location only writable by root, which SHOULD be /usr/share/keyrings.

The key MUST NOT be placed in /etc/apt/trusted.gpg.d or loaded by apt-key add.
{{< /admonition >}}



## Em 密钥

因为不会Linux，所以上面Debian维基我也看不懂。
先把那个密钥下载下来看看

```bash
file <3rdparty-repo>.gpg
```

这样的就是个套了马甲：
```bash
<3rdparty-repo>gpg: PGP public key block Public-Key (old)
```

`man apt-key`里面说了需要二进制的密钥：

{{< admonition quote >}}
apt-key supports only the binary OpenPGP format (also known as "GPG key public ring") in files with the "gpg" extension
{{< /admonition >}}

那么就给它二进制：
```bash
gpg --dearmor <3rdparty-repo>.gpg
```
立刻蹦出来一堆不是人话的内容。

然后把这个密钥加到指定文件夹就好了：
```bash
wget -O- <https://foobar.com/key/<3rdparty-repo>.gpg> | gpg --dearmor | sudo tee /usr/share/keyrings/<3rdparty-repo>-archive-keyring.gpg
```
以上就是下载密钥，转格式，然后加入系统

## F 添加源
第三方的源不要加到`/etc/apt/sources.list`，而是加到`/etc/apt/sources.list.d`里面。
```bash
sudo nano /etc/apt/sources.list.d/<3rdparty-repo>.list
```
这个文件里面的内容就是：
```
deb [arch=amd64 signed-by=/usr/share/keyrings/<3rdparty-repo>-archive-keyring.gpg] <3rdparty-repo URL> <lsb-release> main
```
以上内容自行替换哈，其中`<lsb-release>`是指系统的版本，比如`xenial`、`bionic`、`focal`、`groovy`、`buster`、`bullseye`等等。不知道的可以用`lsb_release -cs`查看。

弄好了就可以运行`sudo apt update`了。

以后不想要这个源的话，可以删除它，别忘记把`/usr/share/keyrings/<3rdparty-repo>-archive-keyring.gpg`也删除。

## G 删除不安全的密钥
有同学问了，你怎么不早说啊？我以前用不安全的办法添加过源，我怎么办啊？

![IT支持之三连](https://cdn.miyunda.net/uPic/apt-key-3_20231027.png)

~~IT技术支持三大招数：重启、重装、买新的。（排名分先后，不分也行）~~ 重启这次是不行了，重装吧。

```bash
 sudo apt-key list
 ```
 以上列出密钥，我们自行使用肉眼搜索里面的可疑密钥，比如：
 ```
 pub   rsa4096 2020-11-10 [SC] [expires: 2028-11-11]
      1234 5678 001A BCDE FFED  CBA9 876F 5432 1098
uid           [ unknown] Foo Bar Signing Key (11/bullseye) <unsecure@foobar.com>
sub   rsa4096 2020-11-10 [S] [expires: 2028-11-11]
```
我们就可以把它删除：
```bash
sudo apt-key del 54321098
```
---
感谢观看，看过了就等于会了。