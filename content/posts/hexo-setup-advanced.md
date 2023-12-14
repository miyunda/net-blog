---
title: 利用CI/CD部署个人网站
date: 2019-10-28 15:18:36
categories:
  - 信息技术
tags:
  - CI/CD
  - Travis
  - GitHub
  - Hexo
---
>初闻征雁已无蝉
>百尺楼南水接天
>青女素娥俱耐冷
>月中霜里斗婵娟

大家好，CI/CD是个热门话题。

<!-- more -->

> 更新于2023年10月：由于Travis开始收费了（还高达64美元每月！），我已经搬到Cloudflare Pages了；Hexo也换成了Hugo。本文章只具有考古价值。

阅读本文需要您了解基本的网络概念，但是不需要您会编写代码。文中涉及到的绝大多数命令行操作都可以用[VSCode](https://code.visualstudio.com/)完成。

建议阅读本文**之前**阅读[我上一篇Hexo文章](https://miyunda.com/hexo-setup/)
前情提要：已经有一个本地工作站，它把作者写的markdown格式文件通过nodejs渲染成HTML文件，并推送到远端的自建git服务上，然后git服务会将收到的网页发布到web服务器上，这是一套完整的Hexo流程。

今天我们将这个流程变得削微复杂一丢丢，利用~~同性交友网站~~GitHub管理源码以及利用CI/CD渲染网页并推送到自建仓库。

![hexo-setup-advanced](https://cdn.miyunda.net/image/hexo-setup-advanced-2019102921210.jpg)

---

## 远程仓库

1. 到GitHub去注册个账号，为网站创建一个源码仓库，这里将存放写作者提交的markdown文章。大概是[这样的](https://github.com/miyunda/hexo-blog.git)
。另外建立一个分支`blog-src`并且设置为默认分支。`master`分支不动，留给GitHub Pages用。

2. 去把[theme Next](https://github.com/theme-next/hexo-theme-next)岔回来，这将是源码仓库的子模块仓库。暂且叫它主题仓库，看起来是[这样的](https://github.com/miyunda/hexo-theme-next.git)。这个仓库就用`master`分支就行了。

<details>
<summary>新手点开看图</summary>

![gif](https://cdn.miyunda.net/image/hexo-setup-advanced-20191028215026.gif)

![Fork](https://cdn.miyunda.net/image/hexo-setup-advanced-20191028215522.jpg)

</details>

3. 生成**SSH**密钥，每个仓库都有一对，这两个密钥是GitHub推/拉代码验证身份用的。

<details>
<summary>点开看命令</summary>

```bash
ssh-keygen -t rsa -b 4096 -C "404@mbeijing2b.com" #邮件地址写自己的
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/yu/.ssh/id_rsa): /Users/yu/.ssh/hexo-blog
```
```bash
ssh-keygen -t rsa -b 4096 -C "404@beijing2b.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/yu/.ssh/id_rsa): /Users/yu/.ssh/hexo-next
```
</details>

4. 把`~/.ssh/hexo-blog.pub`的内容复制粘贴到源码仓库的密钥设置中；主题仓库的密钥在`~/.ssh/hexo-next.pub`，也是一样要粘贴在主题仓库的密钥设置中。
​ 
- [x] Allow write access

5. 配置本地工作站的SSH客户端

编辑`~/.ssh/config`，内容大概是这样的：

```
  Host next-github
​    HostName github.com
​    AddKeysToAgent no
​    UseKeychain yes
​    IdentityFile ~/.ssh/hexo-next

  Host blog-github
​    HostName github.com
​    AddKeysToAgent no
​    UseKeychain yes
​    IdentityFile ~/.ssh/hexo-blog
```
以上为同一个域名设置了两个别名，这是因为它们的用户名都是`git`。您要是嫌酱紫烦的话可以去[GitHub申请一个token](https://github.com/settings/tokens)，用token可以用Https的方式读写仓库，这里不多介绍了。

6. 试试上面的配置，效果大概是这样的：
   
```bash
   ssh -T git@blog-github
   Hi miyunda/hexo-blog! You've successfully authenticated, but GitHub does not provide shell access.
```
```bash
   ssh -T git@next-github
   Hi miyunda/next-blog! You've successfully authenticated, but GitHub does not provide shell access.
  ```
---
## 本地文件夹

1. 复制源码仓库到本地
```bash
git clone -b blog-src git@blog-github:miyunda/hexo-blog.git
```
2. 进入`hexo-blog`——我们称呼它为根文件夹， 编辑`.gitignore`，必须包含下面几行，其他内容因各人操作系统而异。
```
.DS_Store
.AppleDouble
.LSOverride
._*
Thumbs.db
db.json
*.log
*~
node_modules/
public/
.deploy*/
```
3. 把主题仓库下载回来
```bash
git fetch
git submodule  add -f git@next-github:miyunda/hexo-theme-next.git themes/next
```

4. 完成初始设置
   
 把`themes/next/_config.yml`里面的**全部**内容 复制粘贴到根文件夹的`_config.yml`,并缩进**两个**空格，在VSCode里面是<kbd>Ctrl</kbd>+<kbd>]</kbd>或者<kbd>Cmd</kbd>+<kbd>]</kbd>
粘贴进去的内容上面加一行`theme_config:`，酱紫主题的设置文件就与其他文件分开了。以后升级主题痛苦小很多。

![SampleofConfig](https://cdn.miyunda.net/image/hexo-setup-advanced-20191029221726.jpg)

1. 设置GitHub用户信息
```bash
git config user.name "blog updater"
git config user.email "hexo-blog-updater@miyunda.net/image"
```
  进入到主题文件夹`themes\next`同样设置
  ```bash
git config user.name "theme updater"
git config user.email "hexo-theme-updater@miyunda.net/image"
```

---

## Travis
接下来我们需要一个CI/CD的服务，这种服务有好几家，咱们这种最简单的任务，哪家都能做。这次用的是Travis
1. 别着急，先去Slack里面添加一个`Travis CI APP`，酱紫就可以及时得知最新消息了。
![SlackToken](https://cdn.miyunda.net/image/hexo-setup-advanced-20191028222628.jpg)

2. 打开[Travis网站](https://travis-ci.com)
用GitHub账号登录

选择我们的源码仓库

![TravisandGitHub](https://cdn.miyunda.net/image/hexo-setup-advanced-20191028222944.jpg)

3. 在源码根文件夹生成空的配置文件 `.travis.yml`
```bash
touch .travis.yml
```

4. 在本地安装命令行工具
已经有`nvm`的话可以跳过这步.
```bash
curl -sSL https://get.rvm.io | bash -s stable --ruby
source ~/.rvm/scripts/rvm
```

安装
```bash
gem install travis
```
登录
```bash
travis login --pro # 用GitHub账号登录
```
把我们自建git服务的服务器密钥加密，在源码根文件夹执行
```bash
travis encrypt-file ~/.ssh/web-bj-01_id_rsa --add --com
mv web-bj-01_id_rsa.enc .travis
```
把Slack给的token也加密
```bash
travis encrypt "beijing2b:YOURSLACKTOKEN" --add notifications.slack
```
然后编辑`.travis/ssh_config`，内容参照本文开头提到的[上一篇Hexo文章](https://miyunda.net/image/hexo-setup/)。

5. 编辑`.travis.yml`。这个文件是最核心的，它描述了当代码更新之后`Travis`的行为。这时您应该已经注意到前面两条命令生成的内容被自动加到这个文件里了。目前可能有个bug，它在添加的时候会xjb加多余的转义，需要您自行纠正。

```yaml
notifications:
  slack:
    secure: MF1R4Da7eIrZuHmp6EMQHcXiqnKrETxQBUYNj92XA3mQS7pl5H14K1APv9Tajmefj5SnIldJGyyFyvIu+aH6HMzvqyiRvSkitR81epZXNLb1PRRxtzAgo7aVejRFCHpPIm1dWD6zsXlX7rdcqYvJ1NRb/OpKuwatF3MF0Fxu2p6Z6VjYmZJhdd5FBrCTmLXzDRrJOkZk2AlMOZkvgQKvEEMaCCwt1PqcoXbf9fCov4jmEVj66imJW5V2SvY6omfG6LFTugA4DhOyRq/YWWdLkiWMFibOpk0zt4jrb5gu7dnKV2Jwi7/K3QRZhdiA29+c7nh4kQTPWvcr1WAwEIFqqw6ERSFVWcaUCjNenLO1AG41SQizg5EymkYQNhXJf9WuigHmQpcQViz0O0Hvy0i3s/iKMt5IX+aWUdrhSHYM2MITptdtInGiRSN7kik1DLaZIuztnliV9wisWqNRDf18LSfDNvU5OtBIr9RvEaWNUJYyoELul1aXcdqCR4dXynLdc4FnhLCCa6EayoeWxq77oqUTBAmxlbHkTUD7O1vj3VvfOj0CgPP05cO5rJhj1DNiy69D3Zk8T2N8ZdP6sV10Bdy7AB8jtRkz3xb4Qdei2e2BmS+2Fx3a9XHTxG8Q04Z828DCLGnQZCVfb645EQR6RuTFwhsAjR/q4rG7aWdYHzQ=
# 不克隆子模块仓库
git:
  submodules: false
# 设置钩子只检测blog-src分支的push变动
branches:
  only:
  - blog-src
# 设置缓存文件夹，缓存可能导致网页渲染出现蜜汁错误，有问题的可以把这节注释掉。
cache:
  directories:
  - node_modules
# 在构建之前配置ssh环境，hexo环境和主题。
before_install:
- openssl aes-256-cbc -K $encrypted_f1eb9b6ab60a_key -iv $encrypted_f1eb9b6ab60a_iv -in .travis/web-bj-01_id_rsa.enc -out ~/.ssh/web-bj-01_id_rsa -d
- chmod 600 ~/.ssh/web-bj-01_id_rsa
- eval $(ssh-agent)
- ssh-add ~/.ssh/web-bj-01_id_rsa
- cp .travis/ssh_config ~/.ssh/config
- npm install -g hexo-cli
- rm -rf themes/next
- git clone  https://github.com/miyunda/hexo-theme-next.git themes/next
# 安装需要的插件
install:
- npm install
- npm install hexo-deployer-git --save
- npm install hexo-generator-searchdb --save
- npm install hexo-tag-dplayer --save
# 生成html静态文件
script:
- hexo generate
# 部署到自建git服务器
after_script:
- hexo deploy
```

### 加密储存
由于API Key什么的都是不能被别人知道的，所以我们不能把他们直接存在`_config.yml`里面。
Travis提供了环境变量，仓库范围内有效——各分支可以共享或不共享。

![TravisEnv](https://cdn.miyunda.net/image/hexo-setup-advanced-20191030213941.jpg)

Travis会在每次构建的时候帮我们传递到虚拟机里。

以上图为例，我的`_config.yml`里面有
```yaml
  valine:
    appId: valineappId
    appKey: valineappKey
 ```
使用了两个环境变量。我就得提前把这两个换成API服务商给的值。在`.travis.yml`里面就是
```yaml
before_install:
- sed -i "s/valineappId/${ValineId}/g" _config.yml
- sed -i "s/valineappKey/${ValineKey}/g" _config.yml
```
在GitHub Pages展示网页的小伙伴就得仿照上面的样子自己把GitHub Token存在里面了。

6. 随便写点什么文章，保存。
   
7. 提交变更
   ```bash
   git add .
   git commit -am "初始源码和主题"
   ```

8. 送到GitHub仓库去
```bash
git push origin blog-src
```
这时Travis就会检测到仓库有更新，它就开始工作了。完成之后还会通知。
![SlackBot](https://cdn.miyunda.net/image/hexo-setup-advanced-20191028223633.jpg)

#### 用法
1. 写完了文章保存提交推到远端源码仓库
```bash
git commit -m "改动内容简介" # 在源码根文件夹执行
git push
```
2. 改了主题里面的文件的话呢（一般不会改的），要让我们源码仓库更新一次。才会触发。
```bash
git commit -m "改动内容简介" # 在themes/next主题文件夹执行
git push
cd ../..
git submodule update --remote
git add .
commit -m "track master SHA1 gitlink"
git push
```
3. 假如主题作者更新了主题，我们还是到主题文件夹去执行
先把人家的原版引入
```bash
git remote add upstream https://github.com/theme-next/hexo-theme-next.git
```
然后查看它的分支，注意观察这种提示 `* [new branch]      master       -> upstream/master`
```bash
git fetch upstream
```
下载
```bash
git checkout master
git merge upstream/msater
```
4. 换了新电脑
参照上面[本地文件夹](https://miyunda/hexo-setup-advanced/#本地文件夹)的步骤
---
完，感谢观看。

### 总结
WordPress真香!
![所以WordPress真香~](https://cdn.miyunda.net/image/hexo-setup-advanced-2019103182044.gif)

[点击围观本项目](https://travis-ci.com/miyunda/hexo-blog)
