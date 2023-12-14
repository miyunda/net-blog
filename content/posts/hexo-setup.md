---
title: Hexo 安装（初级）
date: 2019-05-27 15:18:36
categories:
  - 信息技术
tags:
  - Hexo
  - nginx
  - git
  - markdown
---
---

>莫言行路难
>夷狄如中国
>谓言骨肉亲
>中门如异域
>出处全在人
>路亦无通塞
>门前两条辙
>何处去不得

<!-- more -->

大家好，强行加了个定场诗，我也不懂是什么意思。

> 更新于2023年10月：由于Travis开始收费了（还高达64美元每月！），我已经搬到Cloudflare Pages了；Hexo也换成了Hugo。本文章只具有考古价值。

## Hexo简介

>快速、简洁且高效的博客框架

静态网站框架，有Hexo也有[Hugo](https://gohugo.io/)，后者多少需要些动手能力；而前者全傻瓜化，比较适合我这种动手能力极差的人。~~虽然Hexo渲染效率低下，我也忍了~~。

本文旨在帮助想要搭建个人网站的朋友，阅读本文要求您了解基础的网络概念，使用基本的命令行操作，但是不需要您会编写代码。

这里不讨论Hexo的具体设置，[官网文档](https://hexo.io/zh-cn/docs/)有介绍。

关于Hexo的工作流程，我画了个简单的示意图。

![Hexo-setup-dataflow](https://cdn.miyunda.net/image/Hexo-setup-20191027112743.jpg)
​这里把运行web服务与git服务的服务器称为**远端**， 运行Hexo与git客户端的工作站称为**本地**。

需要材料如下
 - [x] 随便什么PC —— MacOS、 Windows、Linux
 - [x] 一个Linux 服务器，配置最低的VPS就行 或/和 [GitHub Pages 下称GP](https://pages.github.com/)
 - [ ] 一个DNS域名
 - [ ] 一个SSL证书
 - [ ] 文本编辑器[VSCode](https://code.visualstudio.com/)，您要是就用记事本也不是不可以。
  
---
## web服务器端配置（远端）
使用GP的可以跳过
<details>
<summary>CentOS</summary>

**安装nginx**

引入源
```bash
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```
安装
```bash
yum install nginx -y
```
启动
```bash
systemctl start nginx.service
```
自动启动
```bash
systemctl enable nginx.service
```
**配置firewalld**
安装防火墙
```bash
yum install firewalld firewalld-config -y
```
​启动
```bash
systemctl start firewalld
```
自动启动
```bash
systemctl enable firewalld
```
​添加规则（这里使用`public`区域）
```bash
 firewall-cmd --permanent --add-service=https
 firewall-cmd --permanent --add-service=http
```
​使新规则生效
```bsh
firewall-cmd --reload
```
​	查看规则
```bash
firewall-cmd --zone=public --list-all | grep services
services: ssh https http
```
</details>

<details>
<summary>Debian</summary>

**安装nginx**

​	安装
```bash
apt install nginx -y
```
**配置ufw**
​	安装防火墙
```bash
apt install ufw -y
```
​	添加规则
```bash
ufw allow 'Nginx HTTP'
ufw allow 'Nginx HTTPS'
ufw allow 'SSH'
```
​	使新规则生效
```bsh
ufw active
ufw enable
```
  看看状态
```bash
systemctl status nginx
```
​</details>
---
#### 配置虚拟主机
​	为站点生成配置文件
```bash
touch /etc/nginx/sites-available/hexo
ln -s /etc/nginx/sites-available/hexo /etc/nginx/sites-enabled/
nano /etc/nginx/sites-available/hexo
```
​	配置文件内容
```json
server {
  #侦听443端口，这个是https访问端口，不想用证书的设置为80
  listen    443;
  #域名
  server_name  www.beijing2b.com;
  #网站根目录位置
  root /var/www/hexo/;  

  #设定本虚拟主机的访问日志
  access_log  logs/nginx.access.log  main;

  # 证书设置，不想用证书的可以 设置为 off
  ssl on;
  ssl_certificate /etc/1_www.beijing2b.com_bundle.crt;
  ssl_certificate_key 2_www.beijing2b.com.key;
  ssl_session_timeout 5m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
  ssl_prefer_server_ciphers on;

  #默认请求
  location / {     
  root /var/www/hexo/;      
      #定义首页索引文件的名称
      index index.html;
  }

  #静态文件，nginx自己处理
  location ~ ^/(images|javascript|js|css|flash|media|static)/ {
      #过期30天
       expires 30d;
  }

  #禁止访问 .htxxx 文件
      location ~ /.ht {
      deny all;
  }

}
server
{
  # 80端口是http默认接口
  listen 80;
  server_name www.beijing2b.com;
  # 在这里我做了跳转，木有证书的别这么做
  rewrite ^(.*) https://$host$1 permanent;
}
```
---
### 建立网站

​创建文件夹及主页
```bash
mkdir -p /var/www/hexo
cd /var/www
mkdir hexo
nano index.html
```
​`index.html` 内容
```html
Hello nerd
```
​	检查权限，其中文件夹权限是 _755_ ，文件权限是 _644_ 。
```bash
ls -al
total 12
drwxr-xr-x 2 root root 4096 May 20 21:24 .
drwxr-xr-x 3 root root 4096 May 20 21:24 ..
-rw-r--r-- 1 root root   21 May 20 21:24 index.html
```
​	<!---->若需修改权限
```bash
chmod -R 755 /var/www/hexo
chmod 644 /var/www/hexo/index.html
```
​	测试配置文件
```bash
nginx -t
nginx: the configuration file /etc/nginx/sites-available/hexo
nginx: configuration file /etc/nginx/sites-available/hexo test is successful
```
​	测试成功就可以重新载入配置文件了
```bash
nginx -s reload
```
---
### 安装git服务
​	安装git
```
yum install git -y #debian用apt
```
​	创建git用户

```bash
adduser git
```
​	在工作站（本地）上生成验证用的钥匙对

```bash
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/yu/.ssh/web-bj-01_id_rsa): ~/.ssh/web-bj-01_id_rsa
# 密码设为空
```

​	在远端操作。将生成的公钥传到服务器上，此时不能使用`ssh-copy-id`

```bash
su git
cd
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
nano ~/.ssh/authorized_keys #将上一步生成的.pub文件所有内容粘贴进来
```

​	然后可以从工作站（本地）尝试下登录

```bash
 ssh -i ~/.ssh/web-bj-01_id_rsa git@web-bj-01.beijing2b.com
```

​	回到服务器上限制git用户的登录
```bash
nano /etc/passwd
# 找到其中git用户的那一行，将 /bin/bash 改成 /usr/bin/git-shell
cat /etc/shells # 发现'git-shell'不在
which git-shell # 找到它的路径
nano /etc/shells # 把找到的路径附加进来
echo /usr/bin/git-shell |  sudo tee -a /etc/shells # 木有权限的话可以试试这行
chsh git # 默认提示即是上面找到的路径，直接回车即可。
```
回到工作站（本地）再次尝试下登录，看到下面这种就对了
```bash
fatal: Interactive git shell is not enabled.
hint: ~/git-shell-commands should exist and have read and execute access.
Connection to web-bj-01.beijing2b.com closed.
```
生成网页仓库

初始化仓库文件夹
```bash
mkdir /home/git/repo && cd /home/git/repo
git init --bare hexo.git
```
生成webhook钩子，让服务器知道收到新文件后该做什么
```bash
cd hexo.git/hooks/
nano post-receive
```
`post-receive`内容
```bash
git --work-tree=/var/www/hexo --git-dir=/home/git/repo/hexo.git checkout -f
```
赋予相应权限
```bash
chmod +x post-receive
chown -R git:git /home/git/repo/hexo.git
chown -R git:git /var/www/hexo
```
---
## 本地Hexo
​ 用包管理安装git，以下以MacOS为例。
```bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install git
brew link --force git
git --version
```

​本地安装node.js
去[官网下载就好了](https://nodejs.org/zh-cn/download/)，推荐使用**LTS**版本。
​	本地安装hexo

```bash
sudo npm install -g hexo
```
​	新建hexo站点文件夹，并且进入
```bash
hexo init hexo-blog
cd hexo-blog
npm install
npm install hexo-deployer-git --save
```

​	修改本地ssh客户端配置，编辑`~/.ssh/config`加入以下内容

	Host web-bj-01 # 一个力求方便的别名
		HostName web-bj-01.beijing2b.com # 远端的FQDN
		User git
		IdentityFile ~/.ssh/web-bj-01_id_rsa

## 部署
在hexo的文件夹找到`_config.yml`， 编辑以下段落。使用[GitHub Pages](https://pages.github.com/)的将仓库地址改下

```yaml
deploy:
  type: git
  repo: web-bj-01:/home/git/repo/hexo.git
  branch: master
```

然后按照[官网文档](https://hexo.io/zh-cn/docs/commands)的指导开始配置，写个新文章或者改了什么配置的话就可以部署到您的远端了。
```bash
hexo g #渲染生成静态html页面
```
```bash
hexo s #在本地的TCP4000端口开启web服务器预览
```
满意之后
```bash
hexo d #通过git push放到远端仓库
```
然后webhook会把仓库的内容部署到web服务器文件夹，一个网站就建好了。
感谢观赏。