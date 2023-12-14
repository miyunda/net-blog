---
title: 搭建家庭简易DNS服务器
date: 2021-10-24 20:58:13
FeaturedImage: https://cdn.miyunda.net/image/dual-adgh-cover.jpg
categories:
  - 消费电子
tags:
  - OpenWrt
  - LEDE
  - Docker
  - 宽带
  - DNS
---
大家好，本文记录了我搭建家庭简易DNS服务器的思路和手法，假如您需要快速、准确（尽量少污染）、相对少广告，尽量不被画像（profiling）的互联网，也许这篇文章能帮到您。
假如您不知道DNS服务器是什么，那么这篇文章不适合您；假如您肉身在境外的话，也不用看了。

<!-- more -->

**更新于2022年1月9日：我家已经改用[mosdns v3](https://github.com/IrineSistiana/mosdns/wiki)，此软件的单一实例已经代替了多实例ADGuardHome的功能，以下文章已过时。**

<details>
<summary>低调做事，不要作死</summary>

翻墙上网被处罚是怎么回事呢？翻墙上网相信大家都很熟悉，但是翻墙上网被处罚是怎么回事呢？下面就让小编带大家一起了解吧。 翻墙上网被处罚，其实就是翻墙上网被处罚了。大家可能会感到很惊讶，翻墙上网怎么会被处罚呢？... 但事实就是这样，小编也感到非常惊讶。那么这就是关于翻墙上网被处罚的事情了，大家有什么想法呢？欢迎在评论区告诉小编一起讨论哦。


</details>


阅读本文需要了解如下基本概念：
- [x] DNS
- [x] Docker

动手操作需要：
- [x] 宽带接入
- [x] 订阅或者自建的不可描述服务
- [x] 一台能运行Docker的设备，比如能~~刷入~~安装OpenWrt的设备，ARM或者x86_64都可以
- [x] 一台有SSH客户端的电脑（是个电脑就行）

**阅读本文代表您同意：不恰当的操作可能会导致家里无法访问互联网，引发家庭~~暴力~~矛盾。造成的任何损失由您自己负责。**

## C 缘起
我之前是用[RouterOS](https://mikrotik.com/aboutus)拨号并且提供NAT和DNS/DHCP服务，配合一个[OpenWrt](https://openwrt.org/)上运行的[SmartDNS](https://github.com/pymumu/smartdns) （提供DNS服务）和[AdGuardHome](https://github.com/AdguardTeam/AdGuardHome)（下称ADGH，提供DNS服务用来去广告/反跟踪）以及$SR+/PA$SWALL提供不可描述服务。

我原来这种做法需要在RouterOS上维护"分流列表"，这个不支持域名，只支持CIDR，初期操作还挺繁琐的，前几天这个RouterOS还正好被我玩坏了又木有备份设置，干脆关机丢一边专心鼓捣OpenWrt. 又因为我~~生活在发达的大城市~~这边的互联网建设还算不错，多数时候SmartDNS的加速作用不是很明显，所以我干脆也不要SmartDNS了，用ADGH提供DNS服务。

ADGH基本的原理是它作为一个DNS服务器，拥有着亿点点过滤规则，当DNS客户端查询的域名命中时则不予解析（黑名单），否则转发到您指定的上游DNS服务器。当然了，通过域名的方式过滤广告就图个乐，追求极致效果的还得搭配HTTP层面的手段。

![AdGuardHome查询简单示意图](https://cdn.miyunda.net/image/dual-adgh-diagram.jpg)

需要注意的是并非您打开一个网页浏览器就只查询您输入的域名，还有无数个域名镶嵌在页面里，它们也都被发送到DNS服务器去查询了。ADGH正是分析这些域名来决定是否允许查询。

---

## Dm DNS条件转发

DNS条件转发（conditional forwarding）指的是管理员为DNS服务器先设定若干规则，规定了对于不同的域名应该转发到指定的不同上游DNS服务器。
比如这样的设置(DNSMASQ):

```
server=/foo.com/12.34.56.7
server=/bar.com/98.76.54.32
server=/foo.bb/bar.bb/12.34.43.12
```

或者这样的(BIND):
```
zone "foo.com" {
        type forward;
        forward only;
        forwarders {12.34.56.7; 12.34.56.8;};
                };
zone "bar.com" {
        type forward;
        forward only;
        forwarders {98.76.54.32; 98.76.54.31;};
                };
```
本次实验操作时不用操心以上列表，自动完成，以上仅为示意。

![DNS条件转发示意图](https://cdn.miyunda.net/image/dual-adgh-conditional-fowarding.jpg)

---
## Em 双ADGH
把以上两节的流程缝合起来，我们就有了一个既能（有限度）去广告防跟踪，又能快速查询到正确DNS记录的系统。理想的状况是先判断有木有广告然后再按照目的地进行条件转发，但是ADGH不具备DNS**条件**转发的功能，所以我先条件转发再用两个ADGH分别为长城内外提供过滤服务。

![双ADGuardHome流程示意图](https://cdn.miyunda.net/image/dual-adgh-final.jpg)

当有DNS查询时先“分流”，确定是长城外还是长城内，内则转发到内用ADGH；外则指向外用ADGH。再由ADGH判断是否需要拦截，需要拦截的，直接失败；不需要拦截的，转发到上游DNS服务器，内的转发去本地ISP服务器/国内公共DNS服务器，外的去机场的ISP的DNS服务器或者国外公共DNS服务器。

---

## F 动手操作
### 环境简介：
一台运行OpenWrt的虚拟机，它有两个网卡，eth1拨号上网，eth0接家里的交换机，设置固定IP地址192.168.1.1并且用DNSMASQ担任DHCP/DNS服务器，我相信这是大多数人家里的设置，姑且就叫路由器吧。

### 准备文件夹
首先SSH登入路由器并查看Docker的信息:
```bash
ssh root@192.168.1.1
```

```bash
docker info
```

查找其中的`Docker Root Dir: /opt/docker`字样，我这里是`/opt/docker`（硬盘），也可能是`mmcblk`（闪存）什么的，记下来。

接下来为我们的ADGH建立文件夹：数据文件夹x2；配置文件夹x2，其中gw是内，~~不是“国外”的拼音~~ oversea是外。

```bash
mkdir -p /opt/docker/ADGuardHome/gwconf
mkdir  /opt/docker/ADGuardHome/gwwork
mkdir  /opt/docker/ADGuardHome/overseaconf
mkdir  /opt/docker/ADGuardHome/overseawork
```
### 去内的容器

然后启动去内的容器：

```bash
docker run --name adguardhomegw \
    -v /opt/docker/ADGuardHome/gwwork:/opt/adguardhome/work \
    -v /opt/docker/ADGuardHome/gwconf:/opt/adguardhome/conf \
    --net=host \
    --restart=always \
    -d adguard/adguardhome:latest
```
以上将刚才的文件夹与容器内部文件夹映射，还会拉取需要的Docker镜像。

#### 去内的ADGH初始化

然后用浏览器打开192.168.1.1:3000，这是初始安装向导，我们给web界面绑定一个非3000的侦听端口，比如192.168.1.1:5080；给DNS服务绑定一个非53的侦听端口，比如127.0.0.1#5553。

#### 去内的DNS服务设置

完成后登入192.168.1.1:5080，找到**设置**-**DNS设置**

**上游服务器**：填入ISP分给您的那两个DNS服务器，这个通常是PPPoE拨号的时候"对端通告”来的。不知道的话去路由器管理页面的状态-概览-系统日志里面搜索“primary   DNS address”和”secondary DNS address“。

选中**并行请求**。（注：Docker版本的负载均衡（RoundRobin）在我这有BUG，具体就是两个上游DNS服务器会有一个随机超时，所以虽然用不上我也还是用并行请求）

**Bootstrap DNS 服务器**：这个不用管它，这个是为了解析DOH/DOT的FQDN，本次咱们不用DoH/DoT。

其它DNS设定就是丰俭由人了，我是禁用了IPv6，TTL最小变为1800，最大变为3600

#### 过滤设置

**过滤器**-**DNS封锁清单**
**添加阻止列表**
**添加一个自定义列表**

对于刚上手的用户我推荐以下两个二选一即可：

- https://github.com/Cats-Team/AdRules

- https://github.com/neodevpro/neodevhost

比如第一个，输入URL是 `https://cdn.jsdelivr.net/gh/Cats-Team/AdRules@latest/adguard.txt`

#### DNSMASQ设置

然后到路由器的管理网页，**选择网络-接口-DHCP/DNS**，改：

**常规设置**：**DNS转发**：127.0.0.1#5553

**高级设置**：**DNS** 查询缓存的大小：0

另外禁用IPv6查询

去内的设置基本就完成了，接着弄去外的：

### 去外的ADGH

回到路由器的SSH会话，再弄一个容器：

```bash
docker run --name adguardhomeoversea \
    -v /opt/docker/ADGuardHome/overseawork:/opt/adguardhome/work \
    -v /opt/docker/ADGuardHome/overseaconf:/opt/adguardhome/conf \
    --net=host \
    --restart=always \
    -d adguard/adguardhome:latest
```

#### 初始化去外的ADGH

然后还是去开192.168.1.1:3000，这是另一个初始安装向导，我们给web界面绑定一个随便的侦听端口，比如192.168.1.1:6080；给DNS服务绑定一个非53的侦听端口，比如127.0.0.1#5335。

完成后登入192.168.1.1:6080，找到设置-DNS设置

#### 去外的DNS服务器设置

上游服务器；填入海外ISP的两个DNS服务器的IP地址，格式为`TCP://12.34.56.7`，具体地址的话这个自建机场的应该会比较容易知道信息，就用VPS的就行。

订阅服务您可能就不太清楚，可以去https://www.apnic.net/about-apnic/whois_search/your-ip/ 然后看结果里面的netname，然后就知道是机场用的哪家ISP了，尝试下ISP的可能存在的DNS服务器FQDN。比如netname是foobar，那么我可以大胆滴拼装出**ns3**.foobar.com和**ns4**.foobar.com，尝试解析他们，通常能得到IP地址。或者您运气不好找不到的话也可以用公共的，比如填入`TCP://8.8.8.8`和`TCP://1.1.1.1` 这样解析到的结果可能不是离海外ISP最近的地址，但是也能用。

用ISP的DNS服务器请选择**并行请求**。

**Bootstrap DNS 服务器**：这个还是不用管它，本次咱们不用DoH/DoT。
最后别忘记禁用IPv6。

##### 过滤设置

同去内的，也可以不同，多样的搭配组合自己尝试效果满意即可

##### 不可描述的设置

我一般使用$SR+ ：

![¥SR去外ADGH](https://cdn.miyunda.net/image/dual-adgh-ssr.jpg)

喜欢用Pa$swall的看这里：

![PaS$wall](https://cdn.miyunda.net/image/dual-adgh-passwall.jpg)

## G 后续修改

以后想改怎么办？SSH登入路由器:

```bash
vi /opt/docker/ADGuardHome/overseaconf/AdGuardHome.yaml
```
或者
```bash
vi /opt/docker/ADGuardHome/gwconf/AdGuardHome.yaml
```

用不惯`vi`的同学可以用`opkg install nano`

里面的内容大概是这样的：

```yaml
bind_host: 192.168.1.1
bind_port: 6080
beta_bind_port: 0
users:
- name: root
  password: 马赛克马赛克马赛克马赛克马赛克马赛克
auth_attempts: 5
block_auth_min: 15
http_proxy: ""
language: ""
rlimit_nofile: 0
debug_pprof: false
web_session_ttl: 720
dns:
  bind_hosts:
  - 127.0.0.1
  port: 5335
```

前两行改控制台的页面IP/端口，密码忘记了就把马赛克那里都删掉，最后两行是DNS服务绑定IP/端口。
改完了以后重启容器生效。先看清要重启哪一个，别弄混了：

```bash
docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED        STATUS        PORTS     NAMES
477253e95893   adguard/adguardhome:latest   "/opt/adguardhome/Ad…"   37 hours ago   Up 37 hours             adguardhomegw
2c44c3950de1   adguard/adguardhome:latest   "/opt/adguardhome/Ad…"   37 hours ago   Up 37 hours             adguardhomeoversea
```

重启：

```bash
docker restart 2c44 #容器ID前4位
```

---

可能有人嫌管理的时候输入端口号麻烦，我建议您可以在路由器上或家里别的地方用一个nginx作反向代理，这就超出本文范畴，就不讨论了。

好了，看完了就等于会了。最后祝大家1024快乐！好人一生平安。
