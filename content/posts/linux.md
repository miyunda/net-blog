---
title: Linux服务器小白教程
FeaturedImage: https://cdn.miyunda.net/uPic/linuxmeme_20231026.jpg
coverWidth: 512
coverHeight: 320
date: 2022-04-17 21:30:03
categories:
  - 信息技术
tags:
  - Linux
---
大家好，最近有些小白买到了很便宜的Linux服务器，几年只要一、二百块钱那种，但是苦于不会基础的设置，就很烦恼。跟着我做就好了。高手就不用看了。

<!-- more -->

首先小白需要一个终端，我假设小白日常用的是Win10/11，建议用[Windows Terminal](https://www.microsoft.com/store/apps/9n0dx20hk701)。服务器端我假设小白用的是Debian，其他也差不多。
通常，一个小散户拿到手的是一个Linux服务器，只有个root账号密码。
在Windows Terminal里面输入 `ssh root@yourIP` 然后按照提示输入密码就行了，这里是不显示密码的，摸黑粘贴即可。 
## 账号
首先要做的事情是创建一个自己使用的账号而不是一直使用root账号。
### 创建账号
``` bash
useradd -m foobar
passwd foobar
```
以上需要输入密码两次，密码不显示在终端里，摸黑粘贴即可。

### 把日常账号加入sudo组

```bash
apt update
apt install sudo -y
usermod -aG sudo foobar
```
以新用户身份查看组成员：
```bash
su - foobar
bash
id foobar
```
（可选）咱们就是说运行sudo命令时不想输入密码：
编辑以下文件
```bash
sudo nano /etc/sudoers.d/foobar
```
内容为：
```
foobar    ALL=(ALL) NOPASSWD:ALL
```
按Ctrl+o保存，然后按Ctrl+x退出nano编辑器。

### 设置SSH服务： 
SSH服务几乎是我们管理服务器的唯一渠道，当然要把它设置得兼具效率与安全。
#### 生成SSH密钥
除了用密码验证身份，SSH可以使用密钥验证，相比密码验证要安全得多。
在日常使用的电脑上——可以是Linux/macOS/Windows（WSL）——生成：
```bash
ssh-keygen -t ed25519 -C "foobar@workstation1"
ssh-keygen -t ed25519 -C "foobar@workstation2"
```
以上引号内建议填写能分清不同日常用机的字符。

以上单行命令会生成一对文件，一个是`id_ed25519.pub`，这个文件（公钥）的内容将会被放到SSH服务器端；另一个是`id_ed25519`，这个文件（私钥）的内容要保护好，谁有这个文件谁就能登录有公钥的服务器。

#### 在服务器端配置
创建新用户的文件夹及SSH公钥：
```bash
su foobar #这是为了确保当前命令运行在刚刚创建的新用户身份下
bash
mkdir ~/.ssh
touch ~/.ssh/authorized_keys
chmod -R 700  ~/.ssh
chmod 644  ~/.ssh/authorized_keys
sudo chown -R foobar:foobar /home/foobar
sudo nano /etc/ssh/sshd_config
```

我们使用`/etc/ssh/sshd_config`来做到：
- [x] 禁止root账号通过SSH登录 
- [x] 所有账号使用SSH公钥验证
- [x] 禁止任何账号SSH登录使用密码验证

找到以下内容并更改选项：
```
PermitRootLogin no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
AuthorizedKeysFile     .ssh/authorized_keys .ssh/authorized_keys2
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
RSAAuthentication yes
```
#### 服务器上导入个人SSH公钥
`nano ~/.ssh/authorized_keys`
粘贴公钥进来，每行一个：
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINuSy63hAnD/NtY6yi8Ovl5g9/VSb5yYW4g3enwx foobar@bworkstation1
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJciaQzj8Ms9VdOQae6S0rEx8vW1E/3fFAEHfAgjrBbVTJKL foobar@workstation2
```
#### 令新配置生效
```bash
sudo systemctl reload ssh
```

然后在日常用的电脑用新身份建立SSH回话：
```bash
ssh foobar@yourIP
```
注：日常用机是类Unix系统的话对私钥文件的权限有要求：
```bash
chmod 600 ~/.ssh/id_ed25519
```
#### （可选）安装fail2ban
Fail2ban可以一定程度上驱赶讨厌的小苍蝇，具体就是把登录验证错误几次的客户端IP地址暂时加入防火墙拒绝列表，有兴趣的自己搜索吧。

### （可选）安装zsh及oy-my-zsh
这个不是必须的，只是为了美观与方便。
安装zsh：
```bash
sudo apt install zsh wget git unzip -y
chsh foobar -s $(which zsh)
```
安装oy-my-zsh及自动补全、语法高亮插件：
```bash
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/plugins/zsh-syntax-highlighting
nano ~/.zshrc
```

`~/.zshrc`的内容为：
```
···
zstyle ':omz:update' frequency 30 #检查更新间隔时间，单位是天

plugins=(git zsh-syntax-highlighting zsh-autosuggestions)

# Setting for the new UTF-8 terminal support in MacOS
LC_CTYPE=en_US.UTF-8
LC_ALL=en_US.UTF-8
```
可以注销再登录或者这样：
```bash
source ~/.zshrc
```

### 改主机名
得用root身份运行。使用`sudo -s`或者`sudo su -`
```bash
hostnamectl set-hostname newname
```
编辑`/etc/hosts`： 
```bash
nano /etc/hosts
```

### 更新系统
```bash
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
```

好了，看了就等于会了，小白去动手配置自己的Linux服务器吧，谢谢观看。