---
title: 家庭简易VPN服务器搭建（高级篇）
date: 2020-12-05 05:23:27
categories:
  - 信息技术
tags:
  - VPN
  - AnyConnect
  - 数字证书
  - ocserv
---
本篇将继续折腾家庭简易VPN服务器，将客户端验证方式由密码改为数字证书。


<!-- more -->

**阅读本文需要您先读过[家庭简易VPN搭建（初级篇）](https://miyunda.com/ocsrv-basic/)，并完成动手试验。**

本次动手试验并不需要更多的材料。

**您已知晓并同意，不恰当的操作可能会导致损失，由您自己承担，与我无瓜。**

---

## C 前情提要
在上一集中，我们搭建了家里的VPN服务器，出门在外也可以访问家里的IT资源，当然也包括使用家里的互联网出口。那样的设置能用，但是不够优雅。主要问题在于验证身份的环节，每次都需要输入密码，一是麻烦，二是麻烦，三还是麻烦。

本集我们将改用客户端数字证书来验证，客户端的效果是这样的：

![iOS Anyconnect](https://cdn.miyunda.net/image/ios-anyconnect.gif)

## Dm 野生CA

### CA证书

与服务器端的证书需要一个被公开承认的CA不同，验证客户端用的数字证书只需要野生CA即可，因为这只需要VPN服务器信任该野生CA，并不麻烦。

我画了一个示意图，简单滴展示客户端如何与服务器相互验证握手。

![Mutual Authentication](https://cdn.miyunda.net/image/ocsrv-advanced-2021-11-29-19-27-54.png)

搭建野生CA并不需要太多资源，它可以是一台单独的Linux服务器，也可以和VPN服务器是同一台。本篇就以上一集的那个VPN服务器为例，先安装必要的工具：
```bash
sudo apt update
sudo apt install gnutls-bin
sudo mkdir /etc/ocserv/ssl/
cd /etc/ocserv/ssl/
```
给野生CA自己的证书生成密钥：
```bash
sudo certtool --generate-privkey --outfile ca-privkey.pem
```
接下来生成野生CA自己的证书的配置文件：
```bash
sudo nano ca-cert.cfg
```
其中内容类似这样的：（改不改无所谓，照我抄也行，反正就自己用。）
```
# X.509 证书

# 贵组织的名字，应为FQDN（不需要实际的DNS记录）
organization = "ca.foobar.com"

# CA名字
cn = "Foobar CA"

# 选号
serial = 001

# 过期时间，以天计，-1永不过期
expiration_days = -1

# 赋予CA身份
ca

# 可以签自己的密钥
signing_key

# 可以签（组织内）别人的密钥
cert_signing_key

# 可以撤销证书
crl_signing_key
```
生成CA证书
```bash
sudo certtool --generate-self-signed --load-privkey ca-privkey.pem --template ca-cert.cfg --outfile ca-cert.pem
```

### 客户端证书

生成客户端证书的密钥：
```bash
sudo certtool --generate-privkey --outfile client-privkey.pem
sudo nano client-cert.cfg
```
客户端证书配置文件内容类似这样的：
```
# X.509 证书
# 贵组织的名字 这里与上面CA证书的内容一致
organization = "ca.foobar.com"

# 这个会显示在客户端的管理界面中，建议使用上一篇文章创建的用户名
cn = "VPN User"

# 这里使用上一篇文章创建的用户名
uid = "<username>"

# 过期时间，以天计算
expiration_days = 3650

# 用途：可以作为客户端验证
tls_www_client

# 可以签自己的密钥
signing_key

# 可以在TLS会话中加密
encryption_key
```
生成客户端证书：
```bash
sudo certtool --generate-certificate --load-privkey client-privkey.pem \
              --load-ca-certificate ca-cert.pem --load-ca-privkey ca-privkey.pem \
              --template client-cert.cfg --outfile client-cert.pem
```
转为pkcs12（pfx）格式，两种都试试，哪个能用用哪个：
```bash
sudo certtool --to-p12 --load-privkey client-privkey.pem --load-certificate client-cert.pem \
                       --pkcs-cipher aes-256 --outfile client.p12 --outder
sudo certtool --to-p12 --load-privkey client-privkey.pem --load-certificate client-cert.pem \
                       --pkcs-cipher 3des-pkcs12 --outfile low-client.p12 --outder
```
查看证书信息：
```bash
certtool --p12-info --inder --infile=low-client.p12
```
以上转来的同一个证书的两种包装：`client.p12`给Windows、Linux使用； `low-client.p12`给macOS和iOS使用；至于安卓，两种都有可能，可以都试试；ChromeOS不能用这种方法发证书，需要提交CSR到企业自建服务，个人用户就别想了，不过我估计很少有人用ChromeOS。

为了举例方便，这篇文章里的客户端设备共用一个证书；真正使用的时候您得给每个客户端设备颁发各自的证书（每个客户端设备一对`<clientname>-privkey/cert.pem`，用户名都一样的话可以共用`.cfg`文件），这样发生不测的时候可以针对单个设备撤销证书：
```bash
nano crl.cfg
```
内容大概是这样的：
```
crl_next_update = 365
crl_number = 1
```

```bash
sudo cat <lostOrStolen>-cert.pem >> revoked.pem
sudo certtool --generate-crl --load-ca-privkey ca-privkey.pem \
              --load-ca-certificate ca-cert.pem --load-certificate revoked.pem \
              --template crl.cfg --outfile crl.pem
```

## Em VPN服务器配置

去编辑配置文件：
```bash
sudo nano /etc/ocserv/ocserv.conf
```
先找到上一集中我们写过的：
>auth = “plain[passwd=/etc/ocserv/ocpasswd]”

我们在下边加一行，把上面那行注释掉：
>auth = "certificate"

告诉ocserv服务器，VPN客户端证书的CA是哪个：
>ca-cert = /etc/ocserv/ssl/ca-cert.pem

指定使用 TLS1.2：
>tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-RSA:-VERS-SSL3.0:-ARCFOUR-128:-VERS-TLS1.3"

都改好了就重启服务：
```bash
sudo systemctl restart ocserv
```

## F VPN客户端

### Windows：
资源管理器双击`client.p12`证书就行了：

![Windows-import-p12](https://cdn.miyunda.net/image/ocsrv-advanced-2021-11-30-19-21-24.png)

导入之后可以用`mmc.exe`加载证书管理：
![mmc-certificate](https://cdn.miyunda.net/image/ocsrv-advanced-2021-11-30-20-44-42.png)

### Linux:
```bash
sudo openconnect -b https://ocs.foobar.com:51443 -c <path>/client.p12
``` 
### macOS

访达直接双击`low-client.p12`即可导入，开了钥匙串iCloud同步的话在任意一台导入会自动同步到其他电脑。

![macos-import-p12](https://cdn.miyunda.net/image/ocsrv-advanced-2021-11-30-21-46-04.png)

### 安卓：
随便用什么方法把文件复制过去，或者起个临时网页。

![Android Anyconnect import p12 cert](https://cdn.miyunda.net/image/android-import-p12.gif)

### iOS

给iOS传证书的时候**不要**用AirDrop，那样的话系统不予导入；要作为附件发给自己用内置的邮件App打开，或者起个临时网页用Safari打开，打开之后分享给Anyconnect。
![iOS Anyconnect import p12 cert](https://cdn.miyunda.net/image/ios-import-p12.gif)

---

好了，看过了就等于会了。极端恋家者也许可以考虑结合自动化App，判断当木有连接到家里Wi-Fi时自动连接家里VPN。
感谢观看。