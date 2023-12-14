---
title: Oracle Cloud运算实例的简易部署
featuredImage: https://cdn.miyunda.net/image/oci-tf-cover.webp
featuredImagePreview: https://cdn.miyunda.net/image/oci-terraform-cover-mini.jpg
coverWidth: 512
coverHeight: 320
date: 2021-11-11 19:56:26
categories:
  - 信息技术
tags:
  - 云计算
  - Terraform
  - Linux
  - 自动化
---
大家好，本文描述了如何使用Terraform自动部署Oracle Cloud的运算实例。本文并非面向Linux大佬，而是像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。

专业人士还请不吝赐教，谢谢。

<!-- more -->

*（封面来自世界名画《九品芝麻官》）*

阅读本文需要您理解VPS基本概念。动手实验需要您会基本的bash操作，并持有信用卡，卡组织为御三家（V/M/A）之一即可。

**您已知晓并同意，不恰当的操作可能会导致经济损失，由您自己承担，与我无瓜。**

## C 缘起

Oracle Cloud（下称甲骨云）前两年刚出来那阵喊着永久免费的口号，很是吸引了一帮~~穷逼撸sir~~互联网技术爱好者。一帮人呼啸卷过，弄得乌烟瘴气寸草不生，我玩了几下就放弃了。今年他家又推出了基于Arm的免费运算实例，最多4核24G内存（可以拆开成最多4台），比起同样免费的AMD实例（1G内存）香到不知道哪里去了，一波波勤劳的黄牛闻着味就来了。黄牛们使用脚本一秒钟一抢，抢到资源再拿去卖，而且还卖上百元之贵，还有黄牛弄一套亚洲北美欧洲账号，打包卖七、八百元，都是卖给那些拿去作引流站搞垃圾搜索的人吧，搁这开CDN呢？因为抠门不想花钱，我研究了下如何抢，这样不仅学到了知识，还相当于挣到了至少上百元之多，真是太睿智了。

注意：甲骨云免费的不仅有运算实例，还有数据库、负载均衡器、SSL证书、DNS服务器等等一堆IT基础架构的东西，拿来做正经事可能不太行，但是结合4个运算实例可以做出别家服务商的免费账号做不出来的实验，用来学习还是蛮好的。

## Dm 注册

注册很简单，一步一步跟着走就行了，https://www.oracle.com/cn/cloud/free/ 

需要注意的是：

  - 不要用不可描述的方式上网，会被判断为涉嫌欺诈。
  - 地址填信用卡账单地址，名字填信用卡卡面的持卡人名字，注意可能有空格。
  - 主地区一旦选定不可更改，您选春川或者首尔的话最佳，但是可能会捡不到漏，我选的圣何塞，虽然差了点但是胜在能捡漏。这个也不是绝对的，有时候春川有黄牛被ban了就腾退了一波资源。这个也不要一直死磕一个，觉得不行就换号去别的地区。云计算嘛，谁知道哪块云彩会下雨。

## Em 部署运算实例

### 选择镜像
注册完成之后、部署第一台虚机之前有一些可选的步骤，用默认的也不是不行，就是自定义一下显得讲究，比如
  - 建立区间（Compartment）
  - 建立虚拟网络（VCN）
  - VCN的网关（IGW）、默认路由
  - VCN的子网
  - VCN和子网的DNS标签（lable）
  - 安全组（防火墙）

然后就得给您的虚机选择操作系统镜像，可以用自定义的，也可以用平台提供的。
#### 制作自定义镜像

Arm的支持镜像比较少，我是选择的Ubuntu然后自己装了个Debian。

去 https://cloud.debian.org/images/cloud/OpenStack/ 下载然后上传到一个存储桶再导入镜像。
这样的：
![OCI import custom image](https://cdn.miyunda.net/image/oci-tf-custom-import.jpg)

或者也可以：

先捡到一个Ubuntu实例作为模版机，用SSH登入然后去 https://github.com/bohanyang/debi 用脚本直接装。

TL;DR 觉得上面脚本太复杂的话，我提供个简易方法：

**装完赶紧改密码！**

**装完赶紧改密码！**

**装完赶紧改密码！**

```bash
wget https://raw.githubusercontent.com/bohanyang/debi/master/debi.sh && chmod a+rx debi.sh
sudo ./debi.sh --architecture arm64 --user root --password 密码
```

在模版机上建立自己的用户还有SSH公钥、环境变量之类的、该安装的软件包、时间同步还有禁止root远程登录等等各种要设置的东西**全都弄好了**之后。最后安装[cloud-init](https://cloud-init.io/)。这个软件在操作系统部署之后读取来自`169.254.169.254`的元数据，帮您修改操作系统——我主要是需要它帮我改机器名字（hostname）——以免四台虚机都叫同一个名字。在模版机上运行：
```bash
sudo apt update
sudo apt install cloud-init
sudo systemctl enable cloud-init
```

然后简单滴编辑下，生成一个文件，名字随意，我这样命名~~逼格比较高~~显得专业。
```bash
sudo nano /etc/cloud/cloud.cfg.d/99-oracle-compute-infra-datasource.cfg
```
其内容为令cloud-init修改`/etc/hosts`。
```yaml
# CLOUD_IMG: This file was created by miyunda for OCI build process
manage_etc_hosts: true
```
准备好就关机哦。

然后在甲骨云的控制台以此模版机制作一个自定义镜像，这样的：
![Create an OCI custom image](https://cdn.miyunda.net/image/oci-tf-custom-image.jpg)
这时可以去玩会别的，大概二十分钟到三十分钟能完成。

那么有同学要问了，这模版机哪来的呢？这里我使用了插叙，一种很高级的文学手法。

#### 使用平台提供的镜像
嫌自定义镜像麻烦或者是正在建立模版机就用甲骨云提供的现成镜像也行，有Oracle Linux，CentOS以及Ubuntu可选，都一样，反正哪个我也不会用。

## F 古法手作
不管是自定义还是平台提供的镜像，部署都是一样的，用网页控制台去点：
![create an instance from custom image](https://cdn.miyunda.net/image/oci-tf-custom-image-apply.jpg)
或者点这里：
![create an instance from platform image](https://cdn.miyunda.net/image/oci-tf-stock-image.jpg)

我第一次就成功滴捡到了一个漏，于是我就以为这里很好捡，~~以为有新手福利呢~~。
后来才知道这里的日常是这样的：
![out of capacity](https://cdn.miyunda.net/image/oci-tf-out-of-capacity.jpg)
~~接下来您就可以用按键精灵持续滴点下去。~~

## G 自动化
从上面的截图您也看出来了，甲骨云的运算实例CI/CD是用Terraform（下称tf）做的，网页控制台是个壳，实际做事的是tf，那么咱们就可以直接找tf做事，木有中间商赚差价。

### 甲骨云的命令行工具
#### 安装
甲骨云有两个CLI（命令行工具），咱们使用OCI-CLI：先找一个运行Linux/MacOS的电脑，Windows10/11的话用WSL也行，用PowerShell也行，总之木有什么太高要求，就是不要关机。既然是用来抢机的机器，就叫它抢机机吧，捡机机也行。我是在家里的ESXi起的虚机，您有闲置那vps更好。是个狠人的话去找个CI/CD服务，比如Travis或者Github的Workflow（运行都有最长时间限制，扯淡扯太远了），实在不行也可以用模版机兼任抢机机 ~~无性繁殖生三胎。~~ 

我以Debian为例，其它操作系统用包管理`DNF`或者`brew`什么的，更简单：
```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
```
一路回车即可，假如您不用bash用而是使用其它sh的需要输入自己的rc文件路径，以创建正确的环境变量（路径）。

测试下
```bash
oci -v
```
#### 配置
需要如下信息：
- [x] 用户（user）的API key 或者 Token；
- [x] 用户的OCID
- [x] 租户（tenancy）的OCID
- [x] 地区

其中 API key在这里生成：（注意这不是SSH登录运算实例用的）
![generate an OCI API kay](https://cdn.miyunda.net/image/oci-tf-apikey.jpg)
下载后把私钥放到运行抢机机上存起来，顺便改下文件权限，比如：
```bash
chmod 600 ~/.oci/oraclecloud-date-time.pem
```
私钥的路径要记住，配置向导会问。

用户和租户的OCID在这里找：
![find user ocid](https://cdn.miyunda.net/image/oci-tf-ocid.jpg)
![find tenacy ocid](https://cdn.miyunda.net/image/oci-tf-tenacy.jpg)

信息收集齐了就可以启动配置向导：
```bash
oci setup config
```
然后就可以试试：
```bash
oci iam user list
```
不出意外的话，您的用户信息会被展示出来，OCI-CLI就配置好了。

假如提示下面这样的那么是私钥/用户/租户弄错了，回到上面步骤去检查。
```json
ServiceError:
{
    "code": "NotAuthenticated",
    "message": "The required information to complete authentication was not provided.",
    "opc-request-id": "sjc-1:马赛克马赛克马赛克马赛克马赛克",
    "status": 401
}
```

 ### Terraform
“基础架构即代码”（IaC）使用代码来管理和配置IT基础架构。相对于古法手作，IaC省去了很多步骤和时间，可以与源代码版本控制集成，还能减少对工作人员的技能需求，确保工作质量，堪称CI/CD自动化必备。在1202年的现在，~~还有人不会使用么？不会吧不会吧不会吧~~已经有很广泛的应用了。
IaC有两种实施方法：声明式（declarative）和命令式（imperative）。tf正是使用声明式IaC的一种工具软件。tf一次部署/配置通常需要三个阶段：
  - Write——用户需要用HCL (HashiCorp Configuration Language)编写声明文件。
  - Plan——用户在正式制备/改变配置之前有机会预览、检查效果
  - Apply——用户梦想成真

第一步甲骨云已经替咱们做好了，直接下载他家做好的声明文件即可。

第二步在咱们的场景中可以跳过，反正也不是什么正经的环境。

第三步咱们自己动手。

#### 安装Terraform
[tf的官方文档已经很全了，DNF/Brew什么的都有，照着抄就好了。](https://learn.hashicorp.com/tutorials/terraform/install-cli) 装完了用`terraform -v`测试下。

不过其中apt系的方法不敢苟同——他家用的方法不太好看。

我在我的Debian上是这么装的：
```bash
wget -O- https://apt.releases.hashicorp.com/gpg  | gpg --dearmor | sudo tee /usr/share/keyrings/terraform-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/terraform-archive-keyring.gpg] https://apt.releases.hashicorp.com bullseye main" | sudo tee /etc/apt/sources.list.d/terraform.list
sudo apt update
sudo apt install terraform
```
具体可以参考[apt-key不安全?](https://miyunda.com/apt-key/)

#### 配置Terraform文件夹
这个环节咱们给抢机活动准备一个文件夹，里面存放项目需要的文件。
```bash
mkdir ~/gimme
```
然后把甲骨云生成的`main.tf`放进去，这个文件在web控制台下载，如图中红框：
![Download Terraform config](https://cdn.miyunda.net/image/oci-tf-dl-config.jpg)

文件放好了之后就可以初始化这个文件夹：
```bash
cd ~/gimme
terraform init
```
初始化完成可以做事：
```bash
terraform apply
```
以上需要输入`yes`并回车才会启动。运气好的话一发入魂。运气不好就再把`terraform apply`运行一次。
### 自动化
这个抢机的事情谁也不知道什么时候有资源，一遍遍运行上面命令还得人看着，好麻烦。所以我想自动化抢机。
#### Terraform output
首先咱们需要加一个声明文件，这个声明文件告诉tf在做事之后应该把哪些信息输出：
```bash
nano ~/gimme/outputs.tf
```
里面的内容如下：
```hcl
# Query outputs from OCI compute instance provisioning.
output "instance-state" {
  value = oci_core_instance.generated_oci_core_instance.state
}
```
这样，tf将在每次运行之后输出实例状态，成功的是“RUNNING”，失败则无。
#### 脚本
我写了一个简单的脚本来代替人工。把脚本内容复制保存并赋予它执行权限：
```bash
nano ~/gimme.sh
chmod +x ~/gimme.sh
```
脚本内容：
```bash
#!/usr/bin/bash
#
# Use oci-cli and terraform to provision an instance. (c) 2021 Miyunda

# Where the terraform project folder is.
tfFolder=/home/<username>/gimme
cd $tfFolder

# Do some clean
tfState=``
count=0
rm terraform.tfstate*

# Looplooploop
while [[ "$tfState" != 'RUNNING' ]]
do
    # Start
    echo 'yes' | terraform apply
    #Read output to determine (possible) provisioned instance state
    tfState=`terraform output -raw instance-state`
    
    ((count ++))
    echo "$count"
    # Wait for 5~10 minutes
    sleep $((RANDOM % 300 + 300))
done
echo -e "\033[32m Provisioned with $count) \033[0m"
```
保存好之后 运行`./gimme.sh`即可自动抢机了。

可以使用`screen`来防止SSH人走茶凉，以Debian为例：

先安装工具：
```bash
sudo apt install screen
```
然后开始抢机：
```bash
screen -S gimme
~/.gimme.sh
```
看烦了想走就依次**按住**<kbd>Ctrl</kbd> + <kbd>a</kbd> + <kbd>d</kbd>，
看到如下信息：
>[detached from 386386.gimme]

即可放心走开。

什么时候回来了想看一看，就SSH到抢机机上：
```bash
screen -ls
screen -r 386386
```
---
## Am 总结
到这里，抢机，或者说是捡机就结束了，想起来就去网页控制台收货吧。要是再部署一个相同的实例就把`main.tf`里面的机器名字改改就行了。有了四台机器之后就好好利用它们学习吧。别忘记把自定义镜像删除——我觉得这个东西可能是收费的：https://www.oracle.com/cloud/price-list.html#storage

谢谢观看，看过就等于会了。