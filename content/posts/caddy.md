---
title: 用Caddy搭建简易的PHP运行环境
cover: https://cdn.miyunda.net/image/background.jpg
coverWidth: 512
coverHeight: 320
date: 2022-05-13 04:30:21
categories:
  - 信息技术
tags:
  - Caddy
  - MySQL
  - PHP
  - 游戏
---
还记得07年初那个Travian吗？自从有了高仿之后我也搭了一个，自己玩，现在我邀请各位一起来玩，欢迎大家来我的公益网站免费玩，无需下载。进入游戏以后给我发游戏内邮件，我看到后会送一年VIP账号和9999金币，用完了再找我要。。。

访问 https://t.miyunda.net 即可游玩。

我在西南（-，-）~~是兄弟就来砍我~~。

<!-- more -->
> 2023年夏天我不得不关闭了服务器——熬了整整一个赛季，只有我一个玩家。
{.note}
---
##  缘起
本文介绍了如何搭建一个PHP与SQL数据库的动态网站。
写这篇主要是最近看到一些小伙伴上了鹅厂的~~贼船~~车，入手了云服务器，又因为木有用处在那吃灰而想提供~~垃圾~~HTML5游戏服务甚至手游服务，既能给别人娱乐自己又能在搭建过程中体会折腾的乐趣。但是纯小白往往不知道从哪动手开始，跟着我做就可以了。

当然了，不只是游戏，搭建个博客或者CMS类也是可以的，反正都是PHP的动态网站。

建议纯新手先完成[Linux服务器小白教程](https://miyunda.com/caddy/)。

### LNMP LAMP LAMPA
何为LNMP？
* Linux——操作系统
* Nginx**或/和**Apache——网页服务以及反向代理服务（可选）
* MySQL——数据库服务
* PHP——不解释，~~世界上最好的语言~~

LNMP是一个经典的新手套装，在一台Linux服务器上提供动态的网页服务（前台）和数据库服务（后台）。这个组合人气很高的，像AWS啊GCP啊各种云计算服务商的社区镜像有不少都是LNMP，力求“开箱即用”。也有一些“一键安装”的脚本做得很好用，以及小白最喜欢的某塔面板，提供了图形化安装界面。~~为了展现优越感~~，我这里是纯手动安装，问就是折腾的乐趣，唉，就是玩。

### Caddy
虽然但是，Nginx很好很强大，配置繁琐，我现在更喜欢使用Caddy，配置简洁，真香。性能、安全性什么的谁比谁强我都不知道，因为只是自己玩，木有什么连接数。

## 跟我动手做

### C 准备
您需要：
- [x] 一台Linux服务器，我用的Debian
- [ ] 域名
- [ ] 数字证书
- [x] 一个公网IP地址
- [ ] 开放的TCP80，443端口

请给操作系统做好必要的安全加固，继续阅读代表您同意您自己负责任何恰当不恰当的操作，您造成的任何直接损失间接损失与本文作者无瓜。
### D 安装Caddy
安装Caddy前先看看**有木有同行在运行：**
```bash
sudo netstat -lp
```
以上命令列出正在侦听的程序，有发现其他同行的话请自行关闭处理。

然后使用以下命令导入Caddy的官方源并安装：
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo tee /etc/apt/trusted.gpg.d/caddy-stable.asc
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```
（[以上命令详细解释参见《apt-key不安全?》](https://miyunda.com/apt-key/)）

装好了以后可以看看状态：
```bash
systemctl status caddy
```
这样的就挺好：
```bash
● caddy.service - Caddy
     Loaded: loaded (/lib/systemd/system/caddy.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2022-04-07 05:38:16 PDT; 4min 6s ago
       Docs: https://caddyserver.com/docs/
   Main PID: 1062904 (caddy)
      Tasks: 7 (limit: 4152)
     Memory: 6.7M
        CPU: 107ms
     CGroup: /system.slice/caddy.service
             └─1062904 /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
```
或者这样的：
```bash
● caddy.service - Caddy
   Loaded: loaded (/lib/systemd/system/caddy.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: https://caddyserver.com/docs/
```
很可能是有别的进程在侦听端口，比如Apache，就得把这种进程停了并禁止开机自启动，然后把Caddy启动起来并加入开机自启动：
```bash
sudo systemctl start caddy
sudo systemctl enable caddy
```
更多详细信息在日志里：
```bash
sudo journalctl -u caddy --no-pager | less +G
```
### E MySQL
#### 安装
这个安装起来更简单，官方源自带：
```bash
sudo apt update
sudo apt install mariadb-server
```
#### 安装后
不过装好之后还是要配置亿点点：
```bash
sudo mysql_secure_installation
```
以上脚本是一个向导，帮助我们完成安装后的配置。针对向导的问题，回答如下：
>Enter current password for root (enter for none):
    
    直接回车（Y）

>Set root password? [Y/n]
    
    输入n
    
>Remove anonymous users? [Y/n]
    
    直接回车（Y）
    
>Disallow root login remotely? [Y/n]
    
    直接回车（Y）
    
>Remove test database and access to it? [Y/n]  
    
    直接回车（Y）
    
>Reload privilege tables now? [Y/n]
    
    直接回车（Y）

以上并木有给数据自带的管理员（root）设置密码，接下来我们自己创建一个管理员账号：

```bash
sudo mysql
```
以上就以root身份进到了数据库命令提示符，接下来输入SQL语句：
```sql
GRANT ALL ON *.* TO 'foobaradmin'@'localhost' IDENTIFIED BY '这里是密码' WITH GRANT OPTION;
```
然后可以刷新下：
```sql
FLUSH PRIVILEGES;
exit
```
接下来试试这个新建的管理员账号：
```bash
mysqladmin -u foobaradmin -p version
```
顺利的话应该会打印出数据库服务的信息，到此，MySQL也安装好了。

### F phpMyadmin
MySQL装好了，它所有配置和日常操作都可以通过上面的命令提示符完成，但是这对咱们小白来说太不友好了，俩眼一抹黑。我还需要一个phpMyadmin。


本次教程中，phpMyadmin是一个可选的组件。它提供网页界面——对于大多数人来说更友好——它将咱们输入的操作转给MySQL，再将MySQL返回的结果输出在网页上。
#### phpMyadmin安装
用`apt`命令安装的话很可能会遇到包依赖的问题而陷入臭名昭著的新手死循环，咱们用手动的方式来试试：

先安装可能有用的组件
```bash
sudo apt install php-bz2 php-tcpdf
```
再去[官网](https://www.phpmyadmin.net/downloads/)找找下载链接。
```bash
wget https://files.phpmyadmin.net/phpMyAdmin/5.1.3/phpMyAdmin-5.1.3-all-languages.zip
# 或者
wget https://files.phpmyadmin.net/phpMyAdmin/5.1.3/phpMyAdmin-5.1.3-all-languages.tar.gz
```
就地解压缩：
```bash
unzip phpMyAdmin-5.1.3-all-languages.zip
# 或者
tar xvf phpMyAdmin-5.1.3-all-languages.tar.gz
```
移动到一个看起来很专业的地方：
```bash
sudo mv phpMyAdmin-5.1.3-all-languages/ /usr/share/phpmyadmin
```
#### 配置
先建个临时文件夹：
```bash
sudo mkdir /usr/share/phpmyadmin/tmp
```
修改文件夹权限：
```bash
sudo chown -R caddy:caddy /usr/share/phpmyadmin/
```
将自带的配置文件示例模版复制为配置文件：
```bash
sudo cp /usr/share/phpmyadmin/config.sample.inc.php /usr/share/phpmyadmin/config.inc.php
```
还需要生成一个**32位**的密码，记下来，稍后会用到；也可以用随便什么随机密码生成器：
```bash
openssl rand -base64 24
fTwPuOsQdUjC8r9SeD+TCMxNtIlZG4ex
```
开始编辑配置文件：
```bash
sudo nano /usr/share/phpmyadmin/config.inc.php
```
原本有块这样的：
```
. . .
$cfg['blowfish_secret'] = ''; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
. . .
```
咱们把密码填进去，改成这样的：
```
. . .
$cfg['blowfish_secret'] = '刚才生成的密码'; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
. . .
```
后面有两行得取消注释（删除`// `），还要填入一个有MySQL权限的用户：
```
. . .
/* User used to manipulate with storage */
// $cfg['Servers'][$i]['controlhost'] = '';
// $cfg['Servers'][$i]['controlport'] = '';
$cfg['Servers'][$i]['controluser'] = 'foobarpma';
$cfg['Servers'][$i]['controlpass'] = '上面用户的密码';
. . .
```
这段可以照我抄：
```
. . .
/* Storage database and tables */
$cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
$cfg['Servers'][$i]['bookmarktable'] = 'pma__bookmark';
$cfg['Servers'][$i]['relation'] = 'pma__relation';
$cfg['Servers'][$i]['table_info'] = 'pma__table_info';
$cfg['Servers'][$i]['table_coords'] = 'pma__table_coords';
$cfg['Servers'][$i]['pdf_pages'] = 'pma__pdf_pages';
$cfg['Servers'][$i]['column_info'] = 'pma__column_info';
$cfg['Servers'][$i]['history'] = 'pma__history';
$cfg['Servers'][$i]['table_uiprefs'] = 'pma__table_uiprefs';
$cfg['Servers'][$i]['tracking'] = 'pma__tracking';
$cfg['Servers'][$i]['userconfig'] = 'pma__userconfig';
$cfg['Servers'][$i]['recent'] = 'pma__recent';
$cfg['Servers'][$i]['favorite'] = 'pma__favorite';
$cfg['Servers'][$i]['users'] = 'pma__users';
$cfg['Servers'][$i]['usergroups'] = 'pma__usergroups';
$cfg['Servers'][$i]['navigationhiding'] = 'pma__navigationhiding';
$cfg['Servers'][$i]['savedsearches'] = 'pma__savedsearches';
$cfg['Servers'][$i]['central_columns'] = 'pma__central_columns';
$cfg['Servers'][$i]['designer_settings'] = 'pma__designer_settings';
$cfg['Servers'][$i]['export_templates'] = 'pma__export_templates';
. . .
```
最后自己加一行：
```
. . .
$cfg['TempDir'] = '/usr/share/phpmyadmin/tmp';
```
保存退出之后创建表：
```bash
sudo mariadb < /usr/share/phpmyadmin/sql/create_tables.sql
```
刚才填入的MySQL服务账号还不存在，得去`sudo mysql`创建下：

```mysql
GRANT SELECT, INSERT, UPDATE, DELETE ON phpmyadmin.* TO 'foobarpma'@'localhost' IDENTIFIED BY '上面用户的密码';
```
然后得知道php当前在哪个端口侦听：
```bash
sudo netstat -lp | grep php
```
结果大概是这样的（您可能是7.4，和我不一样，木有关系，记下来就好）：
```bash
Proto RefCnt Flags       Type       State         I-Node   PID/Program name     Path
unix  2      [ ACC ]     STREAM     LISTENING     4069648  9182/php-fpm: maste  /run/php/php7.3-fpm.sock
```
然后去Caddy给phpMyadmin建个网站：
```bash
sudo nano /etc/caddy/Caddyfile
```
其中内容为：
```
pma.foobar.com {
  root * /usr/share/phpmyadmin
  php_fastcgi unix/run/php/php7.3-fpm.sock
  file_server
  tls ssladmin@foobar.com
  encode gzip
  log {
    output file /var/log/caddy/phpmyadmin/access.log {
    #roll_size 100mb
    roll_keep 10
    roll_keep_for 168h
       }
    }
}
```
然后就可以用浏览器打开上面第一行的网址了，这里要注意的是网址不要真的用pma什么的容易被人猜到，不用的时候最好把phpMyadmin的网站关了，用的时候手动起来；或者把域名的DNS解析停了，在自己常用的环境里弄私有解析。

### G 应用
至此，无论是WordPress或者Typecho就都可以安装了，小白的博客就开张啦。

不过一开始说好了本次示例不玩博客，而是教小白装游戏的。跟着我一步一步安装Travian——一款曾经在2007年初风靡一时的网页游戏。
#### 获取源码
去大型同性交友网站可以得到：
```bash
git clone https://github.com/Shadowss/TravianZ.git
```
假如您的服务器在境内那么可能需要代理：
```bash
git clone https://ghproxy.com/github.com/Shadowss/TravianZ.git
```
接下来移动走：
```bash
sudo mkdir /var/wwwroot
sudo mv TravianZ/ /var/wwwroot/travian
sudo chown -R caddy:caddy /var/wwwroot/travian
sudo nano /etc/caddy/Caddyfile
sudo systemctl reload caddy
```
以上`Caddyfile`照phpMyadmin的抄就好了，域名别抄，比如我的：
```
t.miyunda.net {
  root * /var/wwwroot/travian
  php_fastcgi unix/run/php/php7.3-fpm.sock
  file_server
  tls ssladmin@miyunda.com
  encode gzip
  log {
    output file /var/log/caddy/t_miyunda_net/access.log {
    #roll_size 100mb
    roll_keep 10
    roll_keep_for 168h
       }
    }
}
```
然后给Travian创建数据库用户：
```bash
sudo mysql
```
```sql
CREATE DATABASE IF NOT EXISTS `travian`;  
GRANT ALL PRIVILEGES ON `travian`.* TO 'travian_db_user'@'localhost' IDENTIFIED BY '密码' WITH GRANT OPTION;
exit
```
修改权限以允许Travian安装向导生成/删除文件：
```bash
cd /var/wwwroot/travian
sudo chmod -R 777 install
sudo chmod -R 777 GameEngine
```
接下来访问  `https://域名/` 即可启动Travian安装向导.

安装向导运行完毕后再回来改权限：
```bash
sudo rm -r install
sudo chmod -R 755 GameEngine
sudo chmod -R 777 GameEngine/Prevention
sudo chmod -R 777 GameEngine/Notes
sudo chmod -R 777 var/log
```
到这里安装就全结束了，叫上朋友一起来玩就行了，这就是开头的网站的幕后故事。

感谢观看，看过就等于会了，再见～～
 
