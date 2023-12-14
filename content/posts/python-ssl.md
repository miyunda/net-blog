---
title: Python遇到的证书无效问题
date: 2020-08-24 05:41:25
categories:
  - 信息技术
tags:
  - Python
  - SSL
---
本文解决了一个Python出错提示证书验证失败的问题。

<!-- more -->

阅读本文需要您理解SSL Cert CA基本概念，能运行简单的Python程序，并且会基本的shell操作。

#### 出错
当使用Python调用requests去访问REST/Web服务时
```python
r = requests.get('https://foo.bar/api/foo/bar')
```
出错，提示：

>requests.exceptions.SSLError: HTTPSConnectionPool(host='foo.bar.com', port=443): Max retries exceeded with url: /api/foo/bar/ (Caused by SSLError(SSLError("bad handshake: Error([('SSL routines', 'tls_process_server_certificate', 'certificate verify failed')])")))

#### 分析
看起来是服务器证书无法验证。但是使用浏览器可以正常验证证书。
所以能确定证书是有效的，于是我们将目光投向了信任链。
先来看一个正常的信任链
```bash
$ openssl s_client www.beijing2b.com:443
CONNECTED(00000005)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = CN, O = "TrustAsia Technologies, Inc.", OU = Domain Validated SSL, CN = TrustAsia TLS RSA CA
verify return:1
depth=0 CN = www.beijing2b.com
verify return:1
---
Certificate chain
 0 s:CN = www.beijing2b.com
   i:C = CN, O = "TrustAsia Technologies, Inc.", OU = Domain Validated SSL, CN = TrustAsia TLS RSA CA
 1 s:C = CN, O = "TrustAsia Technologies, Inc.", OU = Domain Validated SSL, CN = TrustAsia TLS RSA CA
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
---

```
数字证书的有效性，是由它的上级证书担保的，一直到根证书为止。一个典型的证书信任链包含了一个数字证书（上例的"depth=0"），至少一个Intermediate CA（上例的"depth=1"）证书，以及一个根CA证书（上例的"depth=2"），缺一不可。
经检查，出问题的这个服务器数字证书，它的根CA证书在Python的信任列表，而Intermediate CA证书则不在，验证时无法建立完整的信任链，访问自然不能；
而对于浏览器，这个证书的Intermediate CA证书和根证书都在信任列表中，访问毫无问题。

#### 解决
究其原因是服务器的管理员安装数字证书时将Intermediate CA证书漏了，只安装了服务器自己的数字证书。
所以最根本的办法是请管理员再装一次数字证书的完全体。但是很有可能联系不到管理员。

也有的人在Python代码里关闭验证数字证书——这太蠢了，除非是着急测试用。

本文的办法是将它的Intermediate CA证书手动加入证书列表。

首先确定Python使用的证书列表位置：
```python
>>> import certifi
>>> certifi.where()
'/Users/yu/miniconda3/envs/py37/lib/python3.7/site-packages/certifi/cacert.pem'
```
然后获得问题证书的Intermediate CA证书，建议用FireFox做这件事，界面简单友好。
接下来将导出的Intermediate CA证书的内容复制粘贴**附加**到cacert.pem里面保存就好了。可以用`#`加几句话注释下。

快行动起来吧。

#### PS
[Insomnia](https://Insomnia.rest)自带CA列表，而且Windows版不能选择不用。 Linux/macOS版可以强制Insomnia使用操作系统的CA列表。

感谢观看。