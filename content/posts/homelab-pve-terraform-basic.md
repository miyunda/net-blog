---
title: 搭建简易家庭IT实验室——Terraform（初阶）
cover: https://cdn.miyunda.net//uPic/background.webp
coverWidth: 512
coverHeight: 320
date: 2022-06-17 2:59:54
categories:
  - 信息技术
tags:
  - 虚拟化
  - PVE
  - 数字化过家家
  - homelab
---
大家好，本文是《搭建简易家庭IT实验室》系列文章的第三篇，描述了如何使用一个在本地搭建一个简易的Terraform来创建及销毁我的PVE中的虚拟机。所有内容并非面向Linux大佬，而是面对像我一样不会Linux但是又喜欢玩数字化过家家的朋友。本文记录了我学习相关知识的过程，也供感兴趣的朋友参考。有什么不对或者可以做得更好的地方，还请大家不吝赐教。

<!-- more -->

阅读本文需要您知晓虚拟化的基本概念，建议先阅读上一篇：[《搭建家庭简易IT实验室——cloud-init》](https://miyunda.com/homelab-pve-cloud-init/)。

## C 前情提要

在上一篇中，我们使用PVE自带的命令`qm`以及红帽的工具`cloud-init`做到了命令行创建并销毁虚拟机。我们可以很快速滴得到若干虚拟机，它们都具有相同的设置，主机名等个性化设置除外。为了管理虚拟机，我们要么是使用web控制台鼠标点点点，要么是shell命令行敲敲敲。想象下需要大量虚拟机的时候，我们这样重复劳动，肝疼不疼？而且是人工操作就有可能出错，有的错误后果不堪设想；还有就是开发人员测试App的时候想要退回“上月那个环境”，您就：“？？？上哪找去？”

这一篇我们将学到如何使用Terraform（以下简称tf）达成以上目标，tf是[HashiCorp公司](https://hashicorp.com)的一个自动化工具，可以管理各种公有云私有云，得益于网上社区的力量，tf几乎可以管理各种IT基础架构。

“基础架构即代码”（IaC）使用代码来管理和配置IT基础架构。相对于古法手作，IaC省去了很多步骤和时间，可以与代码版本控制集成，还能减少对工作人员的技能需求，确保工作质量，堪称CI/CD必玩必会的工具。注：我们在这里是用tf管理PVE的虚拟机，重要的是思路，我相信看完了这文章您也可以用tf创建一个AWS的VPC。

![Terraform](https://cdn.miyunda.net/uPic/homelab-terraform_01.png)

IaC有两种实施方法：声明式（declarative）和命令式（imperative）。tf正是使用声明式IaC的一种工具软件。tf一次部署/配置通常需要三个阶段：
  - Write——用户需要用HCL (HashiCorp Configuration Language)编写声明文件。
  - Plan——用户在正式制备/改变配置之前有机会预览效果
  - Apply——用户梦想成真

以上看着眼熟吧？[我抄袭我自己](https://miyunda.com/oci-terraform/)。

## Dm 安装
tf运行在Linux/macOS/Windows上，也不需要什么服务什么进程守护。以Debian为例：
```bash
sudo apt install wget gpg -y
wget -O- https://apt.releases.hashicorp.com/gpg  | gpg --dearmor | sudo tee /usr/share/keyrings/terraform-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/terraform-archive-keyring.gpg] https://apt.releases.hashicorp.com bullseye main" | sudo tee /etc/apt/sources.list.d/terraform.list
sudo apt update
sudo apt install terraform -y
terraform -v
```
我自己用的是Ansible，对应上面步骤写了个简单的playbook，分享在这里：
<details>
  <summary>感兴趣的点开复制保存，第一个冒号后面是要运行tf的机器的名字或组名</summary>
  
 ```yaml
 - hosts: <要运行TF的机器名字>
  become: true
  become_method: sudo
  tasks:
    - name: Install gpg
      apt:
        name: gpg
        state: latest
        update_cache: yes

    - name: Remove gpg keys
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /usr/share/keyrings/hashicorp-archive-keyring.gpg_armored
        - /usr/share/keyrings/hashicorp-archive-keyring.gpg

    - name: Download HashiCorp gpg key
      get_url:
        url: https://apt.releases.hashicorp.com/gpg
        dest: /usr/share/keyrings/hashicorp-archive-keyring.gpg_armored
        checksum: sha256:ecc3a34eca4ba12166b58820fd8a71e8f6cc0166d7ed7598a63453648f49c4c5

    - name: De-Armor HashiCorp gpg key
      shell: gpg --dearmor < /usr/share/keyrings/hashicorp-archive-keyring.gpg_armored > /usr/share/keyrings/hashicorp-archive-keyring.gpg
      no_log: true
      args:
        creates: /usr/share/keyrings/hashicorp-archive-keyring.gpg

    - name: Add HashiCorp's repository to APT sources list
      apt_repository:
        repo: "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com {{ ansible_distribution_release }} stable"
        filename: /etc/apt/sources.list.d/hashicorp.list
        state: present
        update_cache: true

    - name: Install TF
      apt:
        name: terraform
        state: latest
        update_cache: no
 ```
 </details>

## Em PVE端的准备
为了让tf能与PVE交互，我们得先在PVE上创建一个用户，这个用户应该有能完成工作所需的最小权限。

先加个角色并给予权限：
```bash
sudo pveum role add terraform -privs "VM.Allocate VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit"
```
我也不知道需要多少权限，看名字瞎猜的。~~先点这么多，不够再加。~~
建个用户：
```bash
sudo pveum user add terraform-svc@pam
```
给这个用户赋予角色：
```bash
sudo pveum aclmod / -user terraform-svc@pam -role terraform
```
给这个用户创建访问REST API的凭据：
```bash
sudo pveum user token add terraform-svc@pam terraform-token --privsep=0
```
以上会返回这样的信息，记下来，并且保密，~~至少不能像我这样公开发出来~~：
```
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ terraform-svc@pam!terraform-token    │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ c2cf68f4-eadd-41c8-a743-375bc1a3042b │
└──────────────┴──────────────────────────────────────┘
```
然后可以试试这套凭据，不一定要从pve服务器上运行：
```bash
curl "https://<pve服务器以及端口>/api2/json/access/permissions" \
     -H 'Authorization: PVEAPIToken=terraform-svc@pam!terraform-token=c2cf68f4-eadd-41c8-a743-375bc1a3042b'
```
注：使用自签名证书的也许需要`–insecure`选项。

## F Terraform的文件
tf运行的时候默认会在当前文件夹寻找几个文件，作为信息的输入。默认情况下它还会在当前文件夹用文件来保存状态和锁。

以下是一个典型的文件夹（不含状态和锁）：
```
.
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── ...
├── modules/
│   ├── foo/
│   │   ├── README.md
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   ├── bar/
│   ├── .../
```
我们从空白开始：
```bash
mkdir ~/terraform-pve && cd ~/terraform-pve
```
### main.tf
这是最主要的入口，甚至可以是唯一的入口，不过不推荐这样做。`main.tf`可以包含描述管理资源的代码，在复杂一点的项目中，后者应该与`main.tf`分割，独立存放。我的`main.tf`是这样的：
```hcl
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
    }
    dns = {
      source = "hashicorp/dns"
    }
  }
}
```
其中描述了我们将使用两个插件，具体内容为双引号包围的部分。注意我不用DNS插件，这里只是为了演示调用超过一个插件的写法。
### provider.tf
这个也可以不要，内容写在上面的文件里也行：
```hcl
provider "proxmox" {
  # pm_tls_insecure     = true
  pm_api_url          = var.pm_api_url
  pm_api_token_id     = var.pm_api_token_id
  pm_api_token_secret = var.pm_api_token_secret
}
provider "dns" {
  update {
    server        = var.dns_ip
    key_name      = var.dns_key
    key_algorithm = "hmac-md5"
    key_secret    = var.dns_key_secret
  }
}
```
里面内容可以用变量，也可以写死。
### variables.tf
各位当然知道，大多数情况下写死并不明智，于是我们用一个`variables.tf`来定义变量：
```hcl
...
variable "pm_agent_enabled" {
  description = "Enable Proxmox agent (qemu-quest-agent daemon must be installed/running on the guest)"
  type        = number
  default     = 1
}
variable "pm_api_url" {
  description = "Proxmox API endpoint URL"
  type        = string
  sensitive   = true
}
variable "pm_api_token_id" {
  description = "Proxmox API token ID"
  sensitive   = true
}
variable "pm_api_token_secret" {
  description = "Proxmox API token secret"
  sensitive   = true
}
...
```
我这个情景中，上面的`type`可以不写，而`sensitive`则必须写，tf看到这种变量不会将其值输出。

### .tfvars或者环境变量

用了变量就得有地方给变量赋值。


可以写在`.tfvars`文件里，但是强烈不建议！
---

可以写在`.tfvars`文件里，但是强烈不建议！
---

可以写在`.tfvars`文件里，但是强烈不建议！
---

一个例子：`k8s-prod.tfvars`:
```hcl
dns_ip = "192.168.1.103"
dns_key = "labDomainKey"
dns_key_secret = "bGFiRG9tYWluS2V5Cg=="
pm_api_url = "https://<pve服务器以及端口>/api2/json"
pm_api_token_id = "terraform-svc@pam!terraform-token"
pm_api_token_secret = "c2cf68f4-eadd-41c8-a743-375bc1a3042b"
```
想必各位一眼就能看出这么写的缺点，凑合用用还行，稍微正经点的地方都不会这么写。稍微好点的方法是弄成环境变量。在运行tf的工作站上运行：
```bash
export TF_VAR_pm_api_url=https://<pve服务器以及端口>/api2/json
export TF_VAR_pm_api_token_id=terraform-svc@pam!terraform-token
export TF_VAR_pm_api_token_secret=c2cf68f4-eadd-41c8-a743-375bc1a3042b
```
不过这样做仍然有泄露的风险，比如`history`命令或者`printenv`就能出卖一切。不过比前面那种写在文件里好多了。追求更安全的同学可以去搜索“key vault”关键词，HashiCorp/Azure/Oracle都有解决方案。我这里不涉及——~~因为我不会~~——这边最终运行tf是在用过即销毁的工作站 （CI/CD runner）上。

### <module.tf>
前面铺垫那么多，终于要做事了。我们建个文件，比如`k8s-prod.tf`：
```hcl
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
    }
  }
}

resource "proxmox_vm_qemu" "pve-tf" {
  count       = <要几台就写几>
  agent       = var.pm_agent_enabled
  vmid        = 201 + count.index
  name        = "k8s-prod-0${1 + count.index}"
  target_node = var.pm_target_node
  clone       = var.pm_vm_template
  full_clone  = var.pm_vm_full_clone
  os_type     = var.pm_vm_os_type
  cores       = var.pm_vm_cores
  sockets     = var.pm_vm_sockets
  cpu         = var.pm_vm_cpu_type
  memory      = var.pm_vm_memory
  scsihw      = var.pm_vm_scsihw
  bootdisk    = var.pm_vm_bootdisk
  disk {
    size    = var.pm_vm_disk_size
    type    = var.pm_vm_disk_type
    storage = var.pm_vm_disk_storage
    #storage_type = "lvmthin"
    iothread = var.pm_vm_disk_iothread
  }
  network {
    model  = var.pm_vm_network_model
    bridge = var.pm_vm_network_bridge
  }
  lifecycle {
    ignore_changes = [
      network,
    ]
  }
  # Create Ansible user, introduce its SSH key pub.
  ciuser  = var.pm_vm_ciuser
  sshkeys = var.pm_vm_sshkeys
}
```
以上文件有些不用写，因为默认值就正好可以用，这里为了演示所以把大部分都写出来了。[具体见文档](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs/resources/vm_qemu)。我们约定：虚拟机的id从*201*开始，名字从*k8s-prod-01*开始，IP地址用DHCP。其实这里应该用个IPAM/DNS的API来搞，比如[Infoblox](https://www.infoblox.com/products/bloxone-ddi/)，因为这个要钱，我又找不到免费的平替，就先偷懒用DHCP了。
>注意每一个变量都要在上上节的`variables.tf`里面定义，上上节我只写了4个作为示例。

tf还会有输出文件（默认无）和状态文件，状态文件主要是能让tf知道工作进度及状况，默认存在当前文件夹。
## G 运行
先初始化一下，让tf把插件什么的准备好，此时仍然可以介绍变量给tf：
```bash
terraform init
或者
terraform init -k="v"
```
从这时起，状态文件和锁就生成了。

然后可以预览下，同样可以加变量：
```bash
terraform plan
```
确定可以做了就执行，还是可以加变量：
```bash
terraform apply
```
虚拟机等亿小会就可以用了。
不想用的时候就
```bash
terraform destroy
```
## Am 总结
这篇文章我们使用tf去访问PVE的REST API以管理虚拟机，设置好变量等输入文件后，即可高效滴制备或者摧毁虚拟机，但是仍然有可以改进的地方。比如状态文件`terraform.tfstate`存放在本地磁盘，多台工作站的情景中就不太方便了。下一篇我们将把它挪到云存储去。

---

好了，看过就等于会了。谢谢观看。