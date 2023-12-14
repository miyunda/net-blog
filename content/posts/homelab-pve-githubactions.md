---
title: 搭建简易家庭IT实验室——Terraform（工作流）
cover: https://cdn.miyunda.net//uPic/background.webp
coverWidth: 512
coverHeight: 320
date: 2022-07-01 21:16:50
categories:
  - 信息技术
tags:
  - 虚拟化
  - PVE
  - 数字化过家家
  - homelab
---
大家好，本文是《搭建简易家庭IT实验室》系列文章的第五篇，描述了如何将Terraform与代码版本控制系统集成，并完成自动化的工作流。内容并非面向Linux大佬，而是面对像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。有什么不对或者可以做得更好的地方，还请大家不吝赐教。

<!-- more -->

## C 前情提要
在上一篇文章中，我们为tf建立了S3存储，tf将后端的数据写入S3存储，这就为今天的内容打好了基础。阅读本文需要您先完成上一篇[搭建简易家庭IT实验室——Terraform（后端云存储）](https://miyunda.com/homelab-terraform-s3/)。

## Dm 仓库
代码版本控制的好处就不用说了，我选择GitHub作本次演示。

注：这样操作需要PVE的REST API可以被公网访问，爱折腾的可以起个反向代理用来限制源IP地址CIDR，[只允许Azure的IP地址段](https://docs.github.com/en/rest/meta#get-github-meta-information)（不建议）。完全不想暴露的可以自建GitLab什么的。

各位可以在Github建立仓库，然后将之前跟我一路学下来积攒的文件上传。

我已建好了一个用来演示的仓库，地址是https://github.com/miyunda/terraform-pve 。懒得自己写的同学照我克隆然后改改就行了。其中至少得有：
  - [x] `k8s-prod.tf` 项目专属文件
  - [x] `main.tf` 各项目共用文件
  - [x] `provider.tf` 定义安装哪些插件
  - [x] `variables.tf` 定义变量
  
## Em 环境变量

之前我们说为了安全，有些变量的值不能泄露，而各位一上来就用个vault类的服务的话，难度有点高。使用Github的环境变量就很简单，将那些敏感的变量存入Github，然后调用它们。至于这么做的安全性呢，木有什么是绝对安全的，只看您在费用/精力各方面能做到妥协多少。

先建个环境，这个环境的名字要记住，我的叫*prod*。
![manage gitlab envs](https://cdn.miyunda.net/uPic/homelab-github-env.png)
然后在环境里面给变量赋值。
![manage github env vars](https://cdn.miyunda.net/uPic/homelab-github-envvar.png)
注：tf会自动搜索并使用`TF_VAR_`开头的环境变量。
## F 工作流
当仓库发生变化时，GitHub会感知到（废话），然后根据我们事先写好的指示去启动一个Azure虚拟机执行一些任务，这个叫“Actions”，用免费的就行。
我们在仓库里写个文件 `.github/workflows/terraform.yml`。

内容大概是这样的：
```yaml
name: "Create or Destroy" #第一行是名字，起个中二点的名字

#能被何种条件触发，不写分支的就是所有分支。
on:
  push:
    branches:
      - main
  pull_request:

#这边给它派了一个活，叫terraform，分了八个步骤
jobs:
  terraform:
    name: "Terraform-pve"
    runs-on: ubuntu-latest
    #之前上一节定义的环境的名字
    environment: prod
    #环境变量赋值
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_S3_ENDPOINT: ${{ secrets.AWS_S3_ENDPOINT }}
      TF_VAR_pm_api_token_id: ${{ secrets.TF_VAR_PM_API_TOKEN_ID }}
      TF_VAR_pm_api_token_secret: ${{ secrets.TF_VAR_PM_API_TOKEN_SECRET }}
      TF_VAR_pm_api_url: ${{ secrets.TF_VAR_PM_API_URL }}
      TF_VAR_github_actor: ${{ github.actor }}
      TF_VAR_github_updated_at: ${{ github.event.repository.updated_at }}
    
    steps:
      #告诉虚拟机克隆本仓库
      - name: Checkout
        uses: actions/checkout@v3

      #安装tf的二进制文件
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
      
      #格式检查
      - name: Terraform Format
        id: fmt
        run: terraform fmt -check -diff

      #初始化，并附有关于后端配置的命令行
      - name: Terraform Init
        id: init
        run: >-
          terraform init 
          -backend-config="bucket=${{secrets.BACKEND_BUCKET}}" 
          -backend-config="key=k8s/prod/terraform.tfstate"
      
      #验证代码是否合理
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color

      #预览，只有PR时才会发生，目的是给审批人员看看效果
      - name: Terraform Plan
        id: plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -input=false
        continue-on-error: true

      - uses: actions/github-script@v6
        if: github.event_name == 'pull_request'
        env:
          PLAN: "terraform\n${{ steps.plan.outputs.stdout }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`

            </details>

            *Pushed by: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
        
      
      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1
        
      #实施
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -input=false
```

## G 总结
本篇文章中我们建立了仓库，给仓库设置了环境与环境变量，然后编写工作流文件使得GitHub Actions自动执行tf任务。关于玩耍Terraform的文章就到这里了。下一篇文章我们尝试监控PVE的性能。

---
好了，看过就是会了，感谢观看～ 