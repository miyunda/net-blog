---
title: 搭建简易家庭IT实验室——Terraform（后端云存储）
cover: https://cdn.miyunda.net//uPic/background.webp
coverWidth: 512
coverHeight: 320
date: 2022-06-24 18:06:06
categories:
  - 信息技术
tags:
  - 虚拟化
  - PVE
  - 数字化过家家
  - homelab
---
大家好，本文是《搭建简易家庭IT实验室》系列文章的第四篇，描述了如何让Terraform的后端使用云存储管理状态文件。内容并非面向Linux大佬，而是面对像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。有什么不对或者可以做得更好的地方，还请大家不吝赐教。

<!-- more -->

## C 前情提要
上一篇我们使用一个本地运行的Terraform（下称tf）访问PVE的REST API，管理我们的虚拟机。但是当我们在多台工作站运行tf的时候就需要tf将状态存储在云端了。阅读本文需要您先完成上一篇[搭建简易家庭IT实验室——Terraform（初阶）](https://miyunda.com/homelab-pve-terraform-basic/)。

## Dm S3存储
[tf后端状态存储支持一大堆厂商](https://www.terraform.io/language/settings/backends/configuration)，您可以自建，也可以用现成的云对象存储服务，甚至[Hashicorp自己也提供云服务](https://cloud.hashicorp.com/products/terraform/pricing)，要是您刚开始开荒，一张白纸什么都木有的话，建议使用。注：免费版本不支持运行任务（CI/CD），问题不大。

但是这次我们不讨论他家的云服务，我们将令tf后端将状态存储至一个S3桶，其实各家的使用方法都差不多。 

又但是我木有S3服务，因为AWS要钱，我抠门。所以我用Oracle Cloud Infrastructure（以下简称OCI）的对象存储模拟一个S3，其实tf直接就支持OCI的对象存储，我这种折腾只是为了演示。

## Em 建桶
### 租户与区间
先得有个OCI账号，哪个地区的都行。免费额度好像是最多20GB，下载流量10T/月。这个足够了，我们的数据量很少的。

然后搞清楚OCI的概念：
- 租户（Tennacy）

  租户是OCI自动创建的，包含了一个组织中所有的OCI资源，也就是Root Compartment（根区间）。用户、组、区间和某些策略等IAM实体将直接在租户中创建，也可以将策略放入租户中的（子）区间。其他类型的云资源（例如：实例、虚拟网络、块存储卷等）将放置在我们创建的区间中。

- 区间（Compartments）

    区间是OCI的基本组成部分，用于组织和隔离OCI上的云资源，它是一个逻辑上的概念，是一组相关资源的集合。区间为我们带来很多管理上的便利，例如：可以使用区间来衡量资源的使用情况，并方便计费核算；**可以通过使用策略来控制对一组资源的访问**；可以通过将一个项目或业务单位的资源与另一个项目或业务单位分开来实现资源的隔离，一种常见的方法是按照组织架构创建区间，比如人事部门、IT部门、财务部门等。

    举例来说，我们可以将一个部门所使用的所有资源（计算实例、存储、网络等等）放在一个区间里面，而另外一个部门的资源放在另外一个区间里面，以此来方便管理和计费操作。

在本例中，我的OCI账号有一个租户，其下有两个区间，所以我一共有三个区间：
```
root
├── k8s-sjc
├── ManagedCompartmentForPaaS
```
我将在*k8s-sjc*中建桶。
### 使用网页控制台或者命令行
要建桶需要网页控制台，哪里不会点哪里。

或者也可以命令行工具（其实和网页控制台一样都是调用REST API），需要一个OCI-CLI，OCI的命令行工具，安装和配置方法见：[Oracle Cloud运算实例的简易部署](https://miyunda.com/oci-terraform/)。懒得装的话也可以从网页控制台启动一个云工作站：
![Luanch your Cloud Shell](https://cdn.miyunda.net//uPic/terraform-backend01.png)
假如您就木有建过区间：
```bash
oci iam compartment list --query 'data[0]."compartment-id"'
```
那么以上命令就返回您的租户同时也是区间的id。
假如您像我一样戏精爱演，创建了多个区间，那么这样显示他们：
```bash
oci iam compartment list --query 'data[*].{name:"name",id:"id"}' --output table
```
以上将返回除租户之外的区间id。
挑一个吧：
```bash
export COMPARTMENT_ID="<区间的id在此>"
```
### 创建一个用户账号
肯定不可能把我们的root账号给tf用，先建个用户：
```bash
oci iam user create --name terraform-user --description "terraform service account" 
```
把它的id记下来：
```bash
export TF_USER_ID="<新用户的id在上一步输出里面找>"
```
~~事逼就是事逼~~，弄点安全加固
```bash
oci iam user update-user-capabilities --user-id $TF_USER_ID \
--can-use-api-keys 0 \
--can-use-auth-tokens 0 \
--can-use-console-password 0 \
--can-use-db-credentials 0 \
--can-use-o-auth2-client-credentials 0 \
--can-use-smtp-credentials 0
```
给它建个密钥（secret key）：
```bash
oci iam customer-secret-key create --display-name "terraform user SigV4" --user-id $TF_USER_ID
```
输出是这样的（注意保密）：
```json
{
  "data": {
    "display-name": "terraform user SigV4",
    "id": "47cd0283c马赛克马赛克马赛克ecf6838d",
    "inactive-status": null,
    "key": "MUjwafE5tdnP8马赛克马赛克马赛克mFIQrrJVFj/Sg=",
    "lifecycle-state": "ACTIVE",
    "time-created": "2022-06-24T15:42:06.098000+00:00",
    "time-expires": null,
    "user-id": "ocid1.user.oc1..aaaaaaaag2pxue6orf马赛克马赛克马赛克hppxobh6q"
  }
}
```
把里面的**id**和**key**的值记下来，**注意您只有这一次机会看到他们**，要是木有记下来就得再建一个。 更多内容在[OCI官方文档](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcredentials.htm#create-secret-key)。
### 桶（bucket）
建个桶
```bash
oci os bucket create --compartment-id $COMPARTMENT_ID --name <桶的名字>
```

得知您的对象存储命名空间:
```bash
oci os ns get --query data
```
于是我们就得到的了对象存储的访问地址：
  >https://<您的对象存储命名空间>.compat.objectstorage.<地区>.oraclecloud.com

比如：
  >https://ay马赛克马赛克2j.compat.objectstorage.us-sanjose-1.oraclecloud.com

记下来。

存储桶的的AWS风格为：`s3://<桶的名字>`

更多内容在[OCI官方文档](https://docs.oracle.com/en-us/iaas/api/)。
## F 权限
有桶了，也有用户了，得给用户读写桶的权限。
### 组
先建个组：
```bash
oci iam group create --description "terraform backend group" --name terraform-group
```
把组的id记下来：
```bash
export TF_GRP_ID="<组的id在上面的输出里找>"
```
把用户加到组里：
```bash
oci iam group add-user --group-id $TF_GRP_ID --user-id $TF_USER_ID
```
### 策略（policy）
创建个策略，描述了哪个组对哪些对象有什么样的权限。

先准备一个描述文件：
```bash
nano statements.json
```
内容是这样的，第一行令指定的组可以读桶，第二行令指定的组可以读写指定的桶里的对象：
```json
[
"Allow group terraform-group to read buckets in compartment k8s-sjc",
"Allow group terraform-group to manage objects in compartment k8s-sjc where any {target.bucket.name='<桶名>', request.permission='OBJECT_CREATE', request.permission='OBJECT_INSPECT'}"
]
```
然后就可以创建策略了：
```bash
oci iam policy create --compartment-id $COMPARTMENT_ID \
--description "Enable terraform backend storage r/w access" \
--name "terraform-policy" \
--statements file://~/statements.json
```

有兴趣的话请移步[更多内容](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies)和[更更多内容](https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/objectstoragepolicyreference.htm) 。
## G 测试
先随便找个工作站。可以是运行tf的（建议），也可以不是。

测试S3桶的访问那当然是AWS CLI最权威啦。
### 安装AWS CLI
以Debian x64为例：
```bash
sudo apt update
sudo apt install curl unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
其他系统请看[官方文档](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)。
### 导入key
前面那**id**和**key**还记得吧？
```bash
$ aws configure
AWS Access Key ID [None]: 0150b1eb3马赛克马赛克马赛克0d3e7c9e136cc
AWS Secret Access Key [None]: 8y5b2KkR/jbI马赛克马赛克马赛克o0l9A8imhDFsYo=
Default region name [None]:
Default output format [None]:
```
也可以直接编辑 `~/.aws/credentials`同时保持`~/.aws/configure`内容为空。同时检查有木有重复的环境变量，假如有，用`unset`干掉：
```bash
printenv | grep AWS
```
感兴趣的话去看[官方文档](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)
### 读写
```bash
export ENDPOINT_URL=https://<您的对象存储命名空间>.compat.objectstorage.<地区>.oraclecloud.com
touch ~/test.txt
aws s3api list-buckets  --endpoint-url $ENDPOINT_URL
aws s3 ls s3://<桶名> --endpoint-url $ENDPOINT_URL
aws s3 cp ~/test.txt s3://<桶名>/ --endpoint-url $ENDPOINT_URL
aws s3 cp s3://<桶名>/test.txt . --endpoint-url $ENDPOINT_URL
```

### GUI
日常生活中为了查看这个桶肯定还是图形界面方便点，我用的是[CyberDuck](https://cyberduck.io/)，不过就是开箱不带OCI模拟的S3支持，需要自己装一下。![cyberduck-oci-s3](https://cdn.miyunda.net/uPic/terraform-backend02.png)
## Am 配置Terraform
在运行tf的工作站上，使用环境变量确保访问OCI的S3桶的凭据已经设定好：
```bash
export AWS_ACCESS_KEY_ID=<id>
export AWS_SECRET_ACCESS_KEY=<key>
export AWS_S3_ENDPOINT=<https://<您的对象存储命名空间>.compat.objectstorage.<地区>.oraclecloud.com>
```
注：以上环境变量当SSH会话断开即失效，想一劳永逸的话可以写在`~/.bashrc`或者` ~/.zshrc`里面。或者写在``~/.aws/credentials`里面——不需要安装AWS CLI，就有这么个文件就行。

然后进到项目文件夹，初始化：
```bash
terraform init -backend-config="bucket=<桶名>" -backend-config="key=k8s/prod/terraform.tfstate"
```
注：无需在`variables.tf`里面定义这三个变量。

之后tf将把状态文件存放到我们指定的存储桶。在每一个运行tf的工作站上都设置一遍然后它们就共享这个桶了。
## Bdim 总结
这次我们在OCI建了存储桶，设置了相应的用户/组/策略，并且令tf使用存储桶来存取状态文件。这给我们的下一篇文章打好了基础，下一篇文章我们将tf与代码版本控制系统集成，tf任务也将不再是本地工作站上手动运行，而是基于工作流的全自动化。