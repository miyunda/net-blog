---
title: 简易备份Fly.io容器里的SQLite的数据
featuredImage: https://cdn.miyunda.net/uPic/fly-sqlite-backup-s3-banner_20231106.png
date: 2023-11-01 11:11:11
categories:
  - 信息技术
tags:
  - Fly.io
  - SQlite
  - S3
  - CI/CD
  
---
>免为富贵浮生计
>费力功名报主恩
>真是太平常乐事
>香从和气蔼天阍

<!-- more -->
大家好，本文介绍了如何在Fly.io容器中运行自己的App，以及如何将容器中的数据备份到S3。阅读本文需要您有入门级别的Linux容器知识，动手练习则需要以下材料：

- [x] 个人电脑，运行Windows(WSL)或Linux或macOS
- [x] Fly.io账号
- [ ] 域名
- [ ] S3存储
- [ ] Github账号
- [ ] Grafana服务

笔者不会Linux，本文所写都是本人在学习过程中的心得，愿与各位分享，以求互相促进，共同提高。假如您有任何问题或建议，请在评论中告诉我。

>请注意，不适当的操作可能导致您直接或间接的损失，对于您的任何损失，我概不负责。如果您不同意，请立即退出，继续阅读则表示您同意我对您的行为不承担任何责任。
---
## C 缘起
<details>
<summary> Fly.io 简介 </summary>
Fly.io 是一个[平台即服务]^(PaaS)提供商，专注于帮助开发者部署和运行容器化应用程序。它提供了一种简单的方式来将应用程序部署到全球范围的数据中心，以实现更快的性能和更高的可用性。Fly.io 具有内置的负载均衡、自动缩放和全球网络加速功能，使开发者能够轻松地将他们的应用程序扩展到不同的地理位置，以满足不同用户的需求。这个平台还提供了强大的开发工具和监控功能，以帮助开发者更好地管理和优化他们的应用程序。总的来说，Fly.io 旨在简化容器化应用程序的部署和管理，使开发者能够更轻松地构建全球性能卓越的应用程序。
</details>

说那么多，就是在容器里面跑APP，这APP可以是您自己写的，也可以是来自第三方的。对我们赛博乞丐来说，关键是全免费！我是看上它不要钱~~吗~~，我看上的是它部署APP简单方便。

免费账号的资源如下：
- 最多三个虚拟机（分别内存256mb）
- 最多3GB永久存储 （总共）
- 最多160GB每月流量 （100北美欧洲、30印度、30其他）

>请在动手练习之前先注册账号，注册账号需要验证信用卡，即使免费账号也需要信用卡，预授权稍后退回，维萨万事达银联都可以。

结合以上的特性，我决定部署一个密码（以及其他安全凭据）的云同步服务，这将不需要太多的存储也不需要太多的流量。

## D Vaultwarden

我用过一些密码同步服务，Lasspass已弃坑，KeePassXC和Bitwarden在观望中，目前在用1Password。用1Password的理由是好看。

Bitwarden官方的收费是10USD每年，还是可以接受的；也可以自己部署服务器，需要.NET和MSSQL。能租得起这种服务器还不如缴纳10USD每年。

这次我将部署的是Vaultwarden，它是兼容Bitwarden客户端的第三方Docker项目，极大地降低了服务器硬件需求。即使是Fly.io的乞丐版服务器也毫无压力。

也就是说我将使用Bitwarden官方客户端，但是连接到我自己的Vaultwarden。

本文不讨论自建服务与购买商业订阅哪个好哪个不好，只讨论如何自建服务。

>注：目前Vaultwarden不支持Passkey，这是一个特别爽的功能——免密码登录——丝般顺滑。Bitwarden自己的开源服务器也不支持，只能买商业订阅。


## E 部署app

### 安装flyctl
先安装Fly.io的命令行工具flyctl，以Linux/macOS/WSL为例：
```bash
curl -L https://fly.io/install.sh | sh
```
PowerShell为例：

```powershell
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"
```
>注，运行老版本PowerShell可以将`pwsh`替换为`powershell`。

它将安装在`~.fly/bin/`文件夹里。需要指定路径才能用，或者可以将其添加到环境变量中，请自行编辑`~/.bashrc`或者`~/.zshrc`。PowerShell也可以用图形界面添加环境变量。

```bash
export FLYCTL_INSTALL="/home/<username>/.fly" # macOS是"/Users/<username>/.fly"
export PATH="$FLYCTL_INSTALL/bin:$PATH"
```
### 部署App
Vaultwarden提供了一个[Docker镜像](https://hub.docker.com/r/vaultwarden/server)。传统的做法是找个虚机安装Docker然后拉取这个镜像，运行起来，虚机上面再弄个nginx反向代理这个容器暴露的端口。在Fly.io也是类似的思路，不过简单地多。

#### 生成配置文件
建个文件夹然后进入之后：
```bash
fly launch
```
然后它会在当前文件夹查找部署App用的文件，既然是新建的文件夹那当然什么都木有，它就向导式地创建一个。需要您提供App名字以及部署地区，哪个地区快取决于您的ISP，部署之后觉得慢可以摧毁了换一个。我建议从以下四选一，排名不分先后。
- 香港（hkg）
- 东京（nrt）
- 新加坡 （sin）
- 洛杉矶 （lax）
  
>我演示用的这个位于香港，大家可以访问下试试速度。https://meropide.miyunda.com 用户名和密码都是`vvvdemo@miyunda.com`。
  
向导生成了一个文件`fly.toml`，还显示了一个URL，是App的访问地址。接下来我们要让它知道我们要运行Vaultwarden。

#### 修改配置文件

以上文件已经把https的部分写好了，我们就不用操心了，把注意力放在App上面就好了，加入一些内容：
```toml
[env]
  SIGNUPS_ALLOWED = "true"
[build]
  image = "vaultwarden/server:latest"
[mounts]
  source = "meropide_data"
  destination = "/data"
```
第一段`env`是传给Docker的环境变量，我这里设置的是允许注册用户，待我们自己注册完成后将把这里的“true”改为“false”并重新部署；第二段`build`是Docker的镜像；第三段`mounts`是Docker的挂载配置，我们把名为“meropide_data”的（稍后创建）虚拟磁盘挂载到（容器里面的）”/data“上。

还要改这里：
```yaml
[[services]]
  http_checks = []
  internal_port = 80
```
因为Vaultwarden的Docker就是侦听在TCP 80的。

到这里，配置文件就改好了，看起来是这样的：

```toml {title="fly.toml"}
# fly.toml app configuration file generated for meropide on 2023-09-21T10:23:35+08:00
#
# See https://fly.io/docs/reference/configuration/ for information about how to use this file.

#不要照我抄啊，看个思路就行

# 这里是您自己的App名字
app = "meropide"
# 这里是您的部署地区
primary_region = "hkg"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]
  # 这里以后改成 “false”
  SIGNUPS_ALLOWED = "true"
[build]
  image = "vaultwarden/server:latest"
[mounts]
  source = "meropide_data"
  destination = "/data"

[experimental]
  auto_rollback = true

[[services]]
  http_checks = []
  # 这里从8080改成80
  internal_port = 80
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```

#### 挂载磁盘
别猴急猴急地去部署，磁盘还木有建好，之前说了免费用户有3GB空间，先慷慨地拿出1GB来：
```bash
fly volumes create meropide_data --size 1
```
建这个的的时候它吧啦吧啦警告了一堆，大意就是只建一个磁盘“不稳”，磁盘挂了App就宕机了，甚至数据也丢了。文档里说应该建两个App，每个App再挂载一个磁盘，多个App一起工作。我想了想觉得很麻烦，还得弄同步什么的，不会。就一个磁盘算了。反正离线状态下Bitwarden客户端本地也有数据，导出就好了。Vaultwarden也支持外置数据库，这就躲过了磁盘坏的风险，Fly.io提供Postgres，下次一定。

按“y”跳过警告，然后就可以上线了：
```bash
fly deploy
```
以上命令运行后会打印App的URL，复制到浏览器即可😺。

 ![iShot_2023-10-21_11.34.51_20231022](https://cdn.miyunda.net/uPic/iShot_2023-10-21_11.34.51_20231022.png)

#### 禁止注册
先把自己的账号注册好。

编辑我们前面用到的文件：
```bash
nano fly.yaml
```

把这里改下：
```yaml
[env]
  SIGNUPS_ALLOWED = "false"
```
然后再次部署即可：
```bash
fly deploy
```
这之后野人注册就会失败了，您的家人朋友想用的话您可以发送注册邀请，你们甚至可以同享密码～

#### 版本升级
当Vaultwarden有了新版本的时候，就还是在项目文件夹里直接部署：
```bash
fly deploy
```

#### 自己的域名（可选）
Fly.io提供的域名 “<App名字>.fly.dev”看起来还不错，但是假如自己有域名就可以弄得更Coooool～

到您的域名服务商那里去建个CNAME记录就行了。

不同的域名服务商界面不一样，我用的是Cloudflare：

![给我的App建立CNAME记录](https://cdn.miyunda.net/uPic/fly-sqlite-backup-cname_20231106.png)

需要注意的是最后面有个点“.”，别疏忽了。

##### 证书

然后还得让Fly.io监听我们自己的域名，并安装数字证书：
```bash
fly certs add <自己的域名>
```

之后可以正常用了，浏览器扩展、ios、安卓等Bitwarden客户端第一次用的时候选择[自建]^(self-hosted)即可。

## F 监控（可选）

Fly.io提供了一个Prometheus服务，我们可以用它监控我们的App：

### Prometheus

由于一个账号可能是多租户的，得先找到我们的组织别名（Slug）：
```bash
flyctl orgs list
```
对于我们小散，这个值是固定的，就是“personal”。
```bash
ORG_SLUG=personal
```
当然还需要验证身份，在项目文件夹中运行：
```bash
TOKEN=$(flyctl auth token)
```
然后就可以缝合出一条查询命令了：
```bash
curl https://api.fly.io/prometheus/$ORG_SLUG/api/v1/query \
  --data-urlencode 'query=sum(fly_instance_load_average) by (app, status)' \
  -H "Authorization: Bearer $TOKEN"
```
### Grafana
这个环节需要您自己有Grafana，木有的话可以用Fly.io搭建一个。
1. 在Grafana上新建一个数据源，`https://api.fly.io/prometheus/<org-slug>/`
2. 添加定义HTTP头部，Grafana可以在Fly.io的Prometheus上验证身份：
   - Header: Authorization, Value: Bearer <令牌>
      ![添加数据源](https://cdn.miyunda.net/uPic/fly-sqlite-backup-grafana_20231107.png) 
3. 导入[Fly.io的仪表板](https://grafana.com/grafana/dashboards/14741)
成品效果：

![fly的仪表板](https://cdn.miyunda.net/uPic/fly-sqlite-backup-dashbd_20231107.png)

效果不满意的可以自己定制，慢慢玩吧。

### StatusCake
[StatusCake](https://www.statuscake.com/)是一个免费的网站监控工具，当我们的App宕机时会发送警告。

## G 玩腻了
确定不要了可以把数据导出然后在项目文件夹里运行：

```bash
fly scale show
fly scale count 0
```
以上摧毁行为可逆，将“0”改为“1”即可恢复。

以下行为**不可逆**，删完了就完了。
```bash
fly vol list
```
以上将会列出我们用到的磁盘，类似这样的输出：
```bash
fly vol list
ID                  	STATE  	NAME         	SIZE	REGION	ZONE	ENCRYPTED	ATTACHED VM   	CREATED AT
vol_d427jgl8lm615e0v	created	meropide_data	1GB 	hkg 
```
把它删除就结束了。
```bash
fly vol destroy vol_d427jgl8lm615e0v
Warning! Individual volumes are pinned to individual hosts. You should create two or more volumes per application. Deleting this volume will leave you with 0 volume(s) for this application, and it is not reversible.  Learn more at https://fly.io/docs/reference/volumes/
? Are you sure you want to destroy this volume? No
```

## A 备份与恢复

### 备份

我们用的是默认的数据库，SQlite，容器里面的`/data`文件夹看起来是这样的：
```
data
├── attachments          # 每一个附件都作为单独的文件存储在此目录下。
│   └── <uuid>           # （假如未创建过附件，则无子目录）
│       └── <random_id>
├── config.json          # 存储管理页面配置；之前未已启用管理页面则无。
├── db.sqlite3           # 主 SQLite 数据库文件。
├── db.sqlite3-shm       # SQLite 共享内存文件（并非始终存在）。
├── db.sqlite3-wal       # SQLite 预写日志文件（并非始终存在）。
├── icon_cache           # 站点图标 (favicon) 缓存在此目录下。
│   ├── <domain>.png
│   ├── foobar.com.png
│   ├── foobar.net.png
│   └── foobar.org.png
├── rsa_key.der          # ‘rsa_key.*’ 文件用于签署验证令牌。
├── rsa_key.pem
├── rsa_key.pub.der
└── sends                # 每一个 Send 的附件都作为单独的文件存储在此目录下。
    └── <uuid>           # （假如未创建过 Send 附件，则无子目录）
        └── <random_id>
```
我不上传附件也不发送附件，只是存储用户名密码及一些API凭据，站点图片我也不需要，丢了它会自己再缓存的，所以我只备份`db.sqlite3`文件。具体方法是使用Fly.io提供的SSH工具进入容器执行Shell命令，之后步骤大概是：
1. 安装`sqlite3`工具，这是备份数据库用的。
2. 备份到文件。
3. 压缩备份出来的文件。
4. 清理以前的备份文件。
5. 将压缩的备份文件复制回本地（自己的电脑）。
我准备了两个脚本用来执行以上任务。请根据自己的操作系统选择一个，并保存在项目文件夹里。

- 一条适用于Bash：
```bash {title="backup.sh"}
  #!/bin/bash
DATE=$(date +%F)
INSTALL_SQLITE="apt-get update && apt-get install sqlite3 -y"
BACKUP_DB="sqlite3 /data/db.sqlite3 '.backup /data/db.bak'"
CREATE_ARCHIVE="gzip -cv /data/db.bak > /opt/$DATE.db.bak.gz"
CLEAN_UP="find /opt -type f -name '*.db.bak.gz' -mtime +30 -exec rm '{}' +"

fly ssh console  -C 'bash -c "'"$INSTALL_SQLITE"' && '"$BACKUP_DB"' && '"$CREATE_ARCHIVE"' && '"$CLEAN_UP"'"'

fly sftp get "db.bak.$DATE.gz"
```
别忘记给它加上执行权限：`chmod +x backup.sh`

- 另一条适用于PowerShell：
```powershell {title="backup.ps1"}
$DATE = get-date -Format "yyyy-MM-dd"

$INSTALL_SQLITE = "apt-get update && apt-get install sqlite3 -y"
$BACKUP_DB = "sqlite3 /data/db.sqlite3 '.backup /data/db.bak'"
$CREATE_ARCHIVE = "gzip -cv /data/db.bak > /opt/$DATE.db.bak.gz"
$CLEAN_UP="find /opt -type f -name '*.db.bak.gz' -mtime +30 -exec rm '{}' +"

fly ssh console -q -C "bash -c ""$INSTALL_SQLITE && $BACKUP_DB && $CREATE_ARCHIVE && $CLEAN_UP"" "

fly sftp get "db.bak.$DATE.gz"
```
以上二选一，运行一下就会得到一个备份文件，建议定期运行并遵循“两地三中心”原则妥善保管。

### 恢复

恢复就比较简单了，直接把备份的文件复制替换正在运行的数据库文件即可。

假如是超过30天的，需要先复制本地文件进去:

```bash
fly sftp shell (-a <App名字>) # 不在项目文件夹内运行需要添加-a
» put <日期>.db.bak.gz /opt/<日期>.db.bak.gz
```
按CTRL+C退出即可。

SSH进入容器：
```bash
fly ssh console (-a <App名字>)
```
然后解压缩并恢复数据库：
```bash
rm /data/db.sqlite3
rm /data/db.sqlite3-wal
gzip -dk /opt/<日期>.db.bak.gz
cp /opt/<日期>.db.bak /data/db.sqlite3
```
强烈建议在正式投入使用之前先演练至少一次数据恢复，填入一些数据然后备份然后做一些修改再恢复并观察效果。

## B 正文开始
>本节不止针对自建Vaultwarden，任何在容器里使用SQlite的App都可以这样备份。

每天备份的话需要一个电脑，一般都是用个虚机来做这件事，还得设置一堆环境变量升级flyctl什么的，要是这虚机被我玩坏了还得从头来。

我把上一节的备份脚本搬到了Github Actions（下称gha），这样就不用操心机器的事情了。gha本意不是干这个的，那是人家用来做大事的，我这用橙弓打野猪了属于是，主要的目的是给未接触过gha的同学一点知识普及。不知道我这样做是否属于滥用资源，还请互联网伦理学的专家给讲讲。

### 桶
先准备一个S3兼容存储，需要以下信息：
- [x] AWS_ACCESS_KEY_ID
- [x] AWS_S3_ENDPOINT
- [x] AWS_S3_REGION
- [x] AWS_SECRET_ACCESS_KEY
- [x] BUCKET_NAME

以上缺一不可，具体可以从存储服务提供商获得。

您说您木有？那就去[B2](https://www.backblaze.com/cloud-storage)注册一个免费账号，不要信用卡。

注册好了以后建个桶，名字随意，桶名的S3格式为`s3://<桶名>/`或者`s3://<桶名>/<文件夹>/` （文件夹需手动创建）比如桶名为“backup”，它的S3格式是`s3://backup/`，这时您得到了2、3、5（上面列表从上到下，下同）。
![B2创建桶](https://cdn.miyunda.net/uPic/fly-sqlite-backup-b2-bucket_20231108.png)

![B2的桶的信息](https://cdn.miyunda.net/uPic/fly-sqlite-backup-b2-bucketinfo_20231108.png)
然后给它设置访问凭据：

在B2的**My Account**页面，打开**Application Keys**，点击**Add a New Application Key**：
![B2创建key](https://cdn.miyunda.net/uPic/fly-sqlite-backup-b2-newkey_20231108.png)

> 按说这里应该设置为“只写”权限，可以在我本地安装的2.13.31版本aws-cli测试成功上传，但是无法在gha自带的aws-cli使用，不想折腾安装/升级了，就用“读写”算了。

这时您就得到了1、4：
![B2的Key的信息](https://cdn.miyunda.net/uPic/fly-sqlite-backup-b2-newkeyinfo_20231108.png)

其中4是昙花一现，必须立刻记录。可以存在您的Bitwarden里😇。

### Fly.io 访问凭据
Fly.io 的API有两种访问凭据：一种是作用于整个账号的；另一种是每App自己的。前者范围太广，后者比较合适，不过我在试用后者的时候感觉它时灵时不灵，我也不想研究了，反正我这项目就一个App，凑合用吧。

![Fly.io token](https://cdn.miyunda.net/uPic/fly-sqlite-backup-token_20231109.png)

生成API访问令牌之后也是要立刻保存起来。

### Github Actions

#### 仓库设置

先建一个仓库，公开的，名字随意。可以Fork[我的仓库](https://github.com/miyunda/fly-sqlite-backup-s3)。

这仓库需要先设置一些环境变量，他们将用于与S3，Fly.io和Github交互，前两种我们已经准备好了，现在是最后一种。

点自己头像，然后点**Settings**，然后点**Developer settings**，然后点**Personal access tokens**，最后点**Generate new token**。

![新建Github令牌](https://cdn.miyunda.net/uPic/fly-sqlite-backup-ghtoken-new_20231109.png)

然后给新令牌一个看得懂的名字，选择有效期，选择刚建好的仓库，选择权限**Repository permissions**，列表里面第一项：**Actions** -> Access: Read and write；其它项目空着就行，不需要。

![新建Github令牌权限](https://cdn.miyunda.net/uPic/fly-sqlite-backup-ghtoken-acl_20231109.png)

得到的令牌也是要立刻保存起来。

然后到新仓库的设置页面，把我们的环境变量全都定义：

![Github Actions定义环境变量](https://cdn.miyunda.net/uPic/fly-sqlite-backup-ghrepo-secrets_20231109.png)

| 名字                  | 描述               | 值（示例）                  |
| --------------------- | ------------------ | --------------------------- |
| APP_NAME              | 在Fly.io里的应用名 | meropide                    |
| AWS_ACCESS_KEY_ID     | S3访问凭据ID       | 随机字母数字（和序号）      |
| AWS_S3_ENDPOINT       | S3访问域名         | https://“域名”，最后不要“/” |
| AWS_S3_REGION         | S3所在区域         | 地区-方位-数字              |
| AWS_SECRET_ACCESS_KEY | S3访问凭据         | 随机字母数字                |
| BUCKET_NAME           | S3桶名             | s3://backup/meropide/       |
| FLY_API_TOKEN         | Fly.io访问凭据     | fo1_随机字母数字            |
| REPO_TOKEN            | Github访问凭据     | github_pat_随机字母数字     |

#### 仓库内容
一共就两个文件：

- backup.sh
- .github/workflows/backup.yml

“backup.sh”不说了，上面有。

“.github/workflows/backup.yml”是核心文件，gha会启动虚机（runner），从这个文件得知该怎样运行上面的脚本，内容为：

```yaml {title=".github/workflows/backup.yml"}  
name: Fly Sqlite Backup to S3
on:
  # 可以手动运行
  workflow_dispatch:
   # 当仓库有更新时自动运行
    push:
    branches:
      - main
  pull_request:
    branches:
      - main
  # 自动定时运行，北京时间每天凌晨2:53启动，这行自己改改哈，别大家都一个时间启动 😹
  schedule:
    - cron: '53 18 * * *'
# 引入环境变量
env:
    APP_NAME: ${{ secrets.APP_NAME }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_S3_ENDPOINT: ${{ secrets.AWS_S3_ENDPOINT }}
    AWS_S3_REGION: ${{ secrets.AWS_S3_REGION }}
    BUCKET_NAME: ${{ secrets.BUCKET_NAME }}
    FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
    RUNNER_TZ: Asia/Shanghai
    REPO_TOKEN: ${{ secrets.REPO_TOKEN }}
jobs:
  # 任务名字
  backup:
    runs-on: ubuntu-latest
    steps:
    # 步骤1，随便检查下
    - name: Greeting
      run: |
        cd ${GITHUB_WORKSPACE}
        if [[ -z "${{ secrets.APP_NAME }}" ]]; then
          echo "Kindly provide an App name within Actions secrets."
          exit 1
        fi
        if [[ -z "${{ secrets.FLY_API_TOKEN }}" ]]; then
        echo "Please create an API token within Actions secrets to enable Fly access."
        exit 1
        fi
        sudo timedatectl set-timezone "$RUNNER_TZ"
        
    # 步骤2，把这个仓库克隆到gha的虚机（runner）
    - name: Checkout
      uses: actions/checkout@main
    # 步骤3，安装Flyctl
    - name: Install Flyctl
      run: |
        curl -L https://fly.io/install.sh | sh
        echo "FLYCTL_INSTALL=$HOME/.fly" >> $GITHUB_ENV
        echo "$HOME/.fly/bin" >> $GITHUB_PATH
    # 步骤4，运行脚本 
    - name: SSH into my Fly VM  
      run: |
        chmod +x $GITHUB_WORKSPACE/backup.sh
        ./backup.sh
    # 步骤5，上传到S3
    - name: Upload backup data to S3
      run: |  
        aws s3 cp *.gz $BUCKET_NAME \
          --endpoint-url $AWS_S3_ENDPOINT \
          --region $AWS_S3_REGION \
          > result.txt
        sed -i "s|$BUCKET_NAME|secreteBucket/|g" result.txt
        rm -f *.gz
    # 步骤6，上传结果
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: backup-result
        path: result.txt
        if-no-files-found: error
    # 步骤7，清理
    - name: Clean up Actions
      if: env.REPO_TOKEN
      uses: Mattraks/delete-workflow-runs@main
      with:
        token: ${{ secrets.REPO_TOKEN }}
        repository: ${{ github.repository }}
        keep_minimum_runs: 30
        retain_days: 30
```
好了，看过了就算学会了。感谢观看。🫶