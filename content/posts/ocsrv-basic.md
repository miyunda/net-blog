---
title: 家庭简易VPN服务器搭建（基础篇）
date: 2020-11-24 06:50:25
categories:
  - 信息技术
tags:
  - VPN
  - AnyConnect
---
有句话叫“宾至如归”，指的是作为主人，招待得很走心，客人来了就像回到自己家一样。但是有一件事，主人是不太可能做得到的。就是网络访问——走到哪都像在家一样联网。

<!-- more -->

阅读本文需要您理解VPN的基本概念，动手实验需要下列材料，并且会基本的shell操作。

- [x] 家庭宽带有公网IPv4地址
- [x] 家庭宽带NAT端口映射
- [x] 一个域名
- [x] DDNS服务
- [ ] 对应上面域名的数字证书（TLS）
- [x] 一台运行Linux的设备，Arm盒子或者虚拟机或者什么吃灰的淘汰电脑

**您已知晓并同意，不恰当的操作可能会导致损失，由您自己承担，与我无瓜。**

---

## C 缘起

上班连上公司Wi-Fi摸鱼？稍微正经点的组织都有能力监控统计员工的网络使用状况。

家里的网络配好了不可描述服务/广告过滤/DNS防污染，出门在外也想用？

家里有服务器/照片/视频/摄像头，出门也想随时访问？

以上有些可以用云服务满足，但是我在家里搭建VPN服务器来实现，需要的花费就是一个域名钱，数字证书就用免费的。也许有人就用群晖自带的VPN服务，但是我喜欢使用Cisco AnyConnect，因为这么弄比较简单，不需要太多配置工作。

## Dm 准备工作
### 家庭宽带
#### 公网IPv4地址

目前的趋势是御三家（联通电信移动）都在回收公网IPv4，不声不响滴给咱们们换成大内网地址，具体涉及到各省，策略又不见得一样，大趋势就这样。假如发现自己中招了需要打ISP的客服电话谈心，估计还会给您改回公网。不要用太复杂的话术，就打电话说要公网IP。某动的宽带用户可能会比较失望，他家PNAT套娃能套好几层。

有同学问：为什么不用IPv6？

~~因为我不会。~~

IPv6我感觉还不完善，各种蜜汁路由很任性绕来绕去，所以目前我还不想用。

### NAT端口映射

这个也是要打电话谈心，一般ISP都是给一个光猫（家庭~~傻缺~~智慧网关）自带拨号、WI-FI和路由，得打电话要求改成桥接模式用自己的设备拨号。

自己的设备都是有端口映射功能的，一般都是什么“端口转发”或者“防火墙配置”什么的，自行查阅随机带的手册即可，不在此赘述。

我们需要的是两条规则：

```
外部端口 TCP51443 <==> VPN服务器 TCP443
外部端口 UDP51443 <==> VPN服务器 UDP443
```
以上将家里的VPN服务器443端口以51443端口暴露在公网上，注意VPN服务器应使用(内网)固定IP地址。其中51443可以自行定义。

### DNS
#### DDNS

这个有很多家，我用的鹅厂的`DNSPod.cn`，需要一些配置，API密钥什么的，原理就是PPPoE接口的IP地址改变后去鹅厂的API写一下；华硕等厂家的固件自带DDNS功能，配置起来简单些，问题在于厂家的次级域名不可能委派（delegate）给您，您无法证明对域名的管理权，就弄不到数字证书。

#### 域名

所以还得买个域名，便宜点的十几块钱一年，也不需要备案，在哪家买都行。有了自己控制的域名就可以申请数字证书，很多家都有免费数字证书；用固件厂家的DDNS服务的可以用CNAME记录。

<table border="1">
    <tr>
        <td>/</td>
        <td>自定义脚本</td>
        <td>华硕等厂商功能</td>
    </tr>
    <tr>
        <td>DDNS A </td>
        <td>必须</td>
        <td>必须</td>
    </tr>
    <tr>
        <td>DNS CNAME</td>
        <td>可选</td>
        <td>必须</td>
    </tr>
</table>

#### 数字证书
要是不想买域名，就得不到公共认可的数字证书，那么就得自己签发一个，野生证书各操作系统全都不认，用是能用，比较难看/麻烦，不建议。

## Em VPN服务器
我用的是一台Debian的虚拟机，运行在家里的ESXi上；家里有闲置的吃灰盒子/电脑都可以用上，WSL不可以。因为只是给自己家提供服务，也不用追求性能，很随意，唯一需要的就是这台机器的IP地址不要变化。

它的地址是192.168.1.x/24，使用192.168.1.1作为默认网关和DNS服务器。

### 安装ocserv
我就用Debian系举例了，DNF系的也是基本一样。
```bash
sudo apt update
sudo apt install ocserv
```
安装完了看看状态：
```bash
systemctl status ocserv
```
大概是这样的：
```
● ocserv.service - OpenConnect SSL VPN server
   Loaded: loaded (/lib/systemd/system/ocserv.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-10-21 11:29:03 CST; 24min ago
     Docs: man:ocserv(8)
 Main PID: 616 (ocserv-main)
    Tasks: 2 (limit: 1121)
   Memory: 6.2M
   CGroup: /system.slice/ocserv.service
           ├─616 ocserv-main
           └─630 ocserv-sm
```
加入开机启动：
```bash
sudo systemctl enable ocserv
```
重启动服务
```bash
sudo systemctl restart ocserv
```
### 配置ocserv
先把数字证书放到一个合适的位置，比如：
```bash
sudo cp path/1_ocs.foobar.com_bundle.crt /etc/ssl/certs
sudo cp path/2_ocs.foobar.com.key /etc/ssl/private
```
然后编辑配置文件：
```bash
sudo nano /etc/ocserv/ocserv.conf
```
寻找其中这样的字：
>auth = "pam[gid-min=1000]"

这个是使用操作系统自带的账号系统验证身份，某些较老的系统可能需要将**1000**改成**500**

或者使用ocserv内置的账号系统验证身份（推荐）
>auth = "plain[passwd=/etc/ocserv/ocpasswd]"

侦听端口，默认是443：
>tcp-port = 443

>udp-port = 443

告诉ocserv数字证书存在哪里：
>server-cert = /etc/ssl/certs/1_ocs.foobar.com_bundle.crt

>server-key = /etc/ssl/private/2_ocs.foobar.com.key

自动优化IP报文大小，有问题的话改回`false`：
>try-mtu-discovery = true

VPN服务的FQDN，应该与证书一致：
>default-domain = ocs.foobar.com

客户端IP地址池，注意我的地址池`192.168.1.160/28` ⊂ VPN服务器所在子网`192.168.1.0/24`，这种配置比较简单，路由啊防火墙啊什么的都不用多配，在外边看家里群晖上的流媒体方便，在外边串流家里的游戏机玩游戏也方便，需要在VPN服务器代理ARP；或者也可以使用不相干的地址池，比如`192.168.2.1/24`：
>ipv4-network = 192.168.1.160

>ipv4-netmask = 255.255.255.248

DNS服务器，指示客户端使用家里的DNS服务器：
>tunnel-all-dns = true

>dns = 192.168.1.1

以上都改好，保存即可。
### 账号
建立验证身份的账号，密码复杂点，也不要复用其他账号的密码，要知道这是暴露在公网上的：
```bash
sudo ocpasswd -c ocpasswd <username>
```
---
### 配置ocserv的操作系统

#### 操作系统的防火墙

UFW：
```bash
sudo ufw allow https
sudo ufw allow 443/udp
```
或者CentOS：
```bash
sudo firewall-cmd --zone=public --permanent --add-port=443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=443/udp
sudo firewall-cmd --reload
```
又或者iptable：
```bash
sudo iptables -A INPUT -p udp -m udp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
```


### IP转发
这里VPN服务器扮演路由器的角色，在客户端网段和家里网段之间转发IP包。
```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.conf > /dev/null
sudo sysctl -p
```
#### 代理ARP
代理ARP：VPN服务器使用自己的MAC地址和VPN客户端的IP地址来应答寻求VPN客户端IP地址的ARP广播包。当VPN客户端地址池与VPN服务器所在网段不相干时不要使用。另外注意VPN客户端地址池不应该与家里的DHCP池冲突，在我的例子中家里的DHCP不应该分配`192.168.1.160~168`。
```bash
echo "net.ipv4.conf.all.proxy_arp = 1" | sudo tee /etc/sysctl.conf > /dev/null
sudo sysctl -p
```
## F VPN客户端
按理说呢，桌面客户端应该使用openconnect，使用思科的客户端貌似不占理。移动端可以~~理直气壮滴~~用。安装的时候注意不要默认选项，默认全家桶会安装一大堆用不上的模块，那些是给企业用户的。

![macOS 客户端安装](https://cdn.miyunda.net/image/ocsrv-basic-2021-11-29-15-47-57.png)

[ios在App Store下载](https://apps.apple.com/cn/app/cisco-anyconnect/id1135064690)

[安卓在Google Play](https://play.google.com/store/apps/details?id=com.cisco.anyconnect.vpn.android.avf)

[MacOS和Win10我放网盘里面](https://pan.miyunda.net/s/Vwu7) Arm和x86_64都有。

Linux我忘记放哪了，不过用openconnect就挺好：
```bash
sudo apt install openconnect
sudo opennect -b https://ocs.foobar.com:51443 -u <username>
```

安装好了直接联就行了。
---
好了，看过了就等于会了。享受肉身在外，精神在家的旅行吧。我好像出门了，但又木有完全出。
感谢观看。