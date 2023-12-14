---
title: 搭建Web自动化测试环境
date: 2022-01-01 00:00:01
FeaturedImage: https://cdn.miyunda.net/uPic/seleniumMeme_20231026.jpg
categories:
  - 信息技术
tags:
  - Python
  - Selenium
  - 自动化测试
---
本文记录了我如何使用Selenium和Python搭建Web自动化测试环境。


<!-- more -->

阅读本文需要您会简单的Shell操作。完成动手实验需要有一台运行Linux/Windows/macOS之一的电脑。

---

# C 缘起
日常工作、生活和学习的时候，我们经常会有与Web页面交互的任务。有些耗费大量时间与精力，通过编写简单的程序可以使得某些任务自动化，将人力解放出来。

有些自动化任务可以调用API，还有的使用发送/接收http包的形式完成，方便快捷；但是也有网站不得不使用浏览器访问，于是我就需要一个能自动化点击页面元素、输入数据、提交表单、获取数据的自动化程序。

> 第二百八十五条    违反国家规定，侵入国家事务、国防建设、尖端科学技术领域的计算机信息系统的，处三年以下有期徒刑或者拘役。

以上摘自[刑法](http://www.npc.gov.cn/wxzl/wxzl/2000-12/17/content_4680.htm)


![非法获取计算机信息系统数据](https://cdn.miyunda.net/image/selenium-2021-12-30-21-48-45.png)
以上为使用“非法获取计算机信息系统数据”为关键字在北京法院网站搜索结果的截图。（时间为2020年12月30日）

2016年至2017年间，被告人张某某、宋某、侯某某作为被告单位上海晟品网络科技有限公司主管人员，在上海市共谋采用网络爬虫技术抓取被害单位北京字节跳动网络技术有限公司服务器中存储的视频数据，并由侯某某指使被告人郭某破解北京字节跳动网络技术有限公司的反爬虫措施、实施视频数据抓取行为，造成被害单位损失技术服务费人民币2万元。2017年11月，北京市海淀区人民法院判决本案被告单位上海晟品网络科技有限公司、被告人张某某、宋某、侯某某、郭某构成非法获取计算机信息系统数据罪。以上摘录自北京市海淀区人民法院〔2017〕京0108刑初2384号刑事判决书。

我也不知道为什么去某音爬视频会被判这个罪，尖端科学技术领域了属于是。

# Dm Selenium 简介

Selenium（硒）正是这样的一种工具，一方面它提供了API，另一方面它通过WebDriver（简称driver，下同）去控制浏览器。
![selenium-webdriver-architect](https://cdn.miyunda.net/image/selenium-2021-12-30-22-25-47.png)
因为我的画图软件只有上面三个图标，所以就只画了三个，事实上Selenium（简称Se，下同）支持各种主流的语言和浏览器。每一种浏览器都要有对应的driver，不同版本不同CPU架构平台那么driver也不同。

本文选择的操作系统为CentOS 7 x86_64 和 Debian 11 arm64。

本文为x86_64平台选择了Chrome，不解释，管理器会自动帮我们安装Chrome需要的driver，并随着Chrome升级而升级。

然后因为我找不到Chrome在arm64 Linux的driver，管理器并不会识别arm64，它会安装一个x86_64的版本，导致无法运行。（注：2020年12月30日仍未改正）所以我给arm64 Linux安装Chromium以及对应的driver，因为Debian 11的官方仓库就有，比较方便。


# Em 安装浏览器
## x86_64 安装Chrome

注：这个仓库是谷歌的，但是无需不可描述服务。

CentOS 7我用的`yum`，但是`dnf`我猜测应该也能用：
```bash
sudo nano /etc/yum.repos.d/google-chrome.repo
```
其内容为：
```
[google-chrome]
name=google-chrome
baseurl=http://dl.google.com/linux/chrome/rpm/stable/x86_64
enabled=1
gpgcheck=1
gpgkey=https://dl.google.com/linux/linux_signing_key.pub
```
安装：
```bash
sudo yum install google-chrome-stable
```
Debian 11呢，我提供一个简便方法，可以不用手动编辑仓库，假如您愿意使用不简便方法可以参考[apt-key不安全？](https://miyunda.com/apt-key/)：
```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt update
sudo apt install ./google-chrome-stable_current_amd64.deb
```
安装好了可以试试：
```bash
google-chrome --version
```
## arm64 安装Chromium
Debian 11 官方源就有Chromium的driver，安装它会把Chromium作为依赖也装好：
```bash
sudo apt update
sudo apt install chromium-driver
```
安装好之后可以假装自己安装的是Chrome：
```bash
sudo ln -s $(which chromium) /usr/bin/google-chrome
google-chrome --version
```

`dnf`系的arm64的driver我嫌麻烦，所以就略过了。

## driver
x86_64的Linux无需手动安装，可以使用Python自动安装、更新。

arm64的Linux上面说过了，它目前无法自动安装，只能手动安装然后在自动化测试的时候调用。

目前支持最完善的是macOS，Intel和M1的都可以自动安装，丝般顺滑。

# F 准备Selenium环境
我使用Conda来准备一个虚拟环境，可以参考[利用Conda管理Python环境](https://miyunda.com/setup-python/)
```bash
conda create --name webtest python=3.9
conda activate webtest
```
安装 Se：
```bash
conda install selenium
```
安装driver管理器，arm64的Linux无需这一步，因为目前用不上：
```bash
conda config --add channels conda-forge
conda config --set channel_priority strict
conda install webdriver-manager
```
# G 测试
我准备了两个文件，一个是有管理器自动安装driver的；另一个是需要在操作系统中事先装好的的。 结合自己实际情况选一个就行了。

先看看自动的，适用于x86_64的Linux、Windows、所有macOS:
```bash
nano test.py
```

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.options import Options

chrome_options = Options()
chrome_options.add_argument("disable-extensions")
chrome_options.add_argument("disable-gpu")
chrome_options.add_argument("no-sandbox")
chrome_options.add_argument("headless")

driver = webdriver.Chrome(ChromeDriverManager().install(), options=chrome_options)
home_url = "https://miyunda.com"
driver.get(home_url)
print(driver.title)
driver.quit()
```

把上面保存下运行:
```bash
python test.py
```
第一次运行的时候会安装webdriver：
```
====== WebDriver manager ======
Current google-chrome version is 96.0.4664
Get LATEST chromedriver version for 96.0.4664 google-chrome
There is no [mac64] chromedriver for browser  in cache
Trying to download new driver from https://chromedriver.storage.googleapis.com/96.0.4664.45/chromedriver_mac64.zip
Driver has been saved in cache [/yourhomefolder/.wdm/drivers/chromedriver/mac64/96.0.4664.45]
```

---
或者是自行指定driver路径的，适用于arm64的Linux：
```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

chrome_options = Options()
chrome_options.add_argument("disable-extensions")
chrome_options.add_argument("disable-gpu")
chrome_options.add_argument("no-sandbox")
chrome_options.add_argument("headless")

driver = webdriver.Chrome('/usr/bin/chromedriver', options=chrome_options)
home_url = "https://miyunda.com"
driver.get(home_url)
print(driver.title)
driver.quit()
```
---
运行结果应该是这样的：

![selenium test result](https://cdn.miyunda.net/image/selenium-2021-12-31-12-01-05.png)

好了。看过了就等于会了，自己动手写一写，享受您的爬虫吧。

感谢观看。
