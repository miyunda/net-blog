---
title: 谷歌浏览器播放视频失败一例
date: 2020-04-19 05:46:32
categories:
  - 消费电子
tags:
  - 浏览器
  - Chrome
  - Chromium
---
大家好！疫情期间，大多数学生都是在家通过互联网学习的。我们在使用谷歌浏览器Chrome访问某些学习网站的时候播放视频失败。这里分享下解决方法。

<!-- more -->

现象是页面可以打开，但是视频无法播放。

按F12打开控制台，可以看到下图：

![mixedcontent-01](https://cdn.miyunda.net/image/mixedcontent-2020419225524.jpg)
其原因在于，谷歌浏览器正在强制推广全站SSL加密，当前网站页面使用了SSL加密技术，且数字证书有效，其镶嵌的视频并未采用加密传输，浏览器在渲染页面的时候“替用户做主”，直接把`http://` 替换为 `https://` ，但是此网站开发者尚未提供相应的SSL加密，于是造成视频无法找到，也就播放失败了。

在相信网站开发者的前提下，目前我们可以（针对单一站点）暂时关闭谷歌浏览器的这一功能。

![mixedcontent-202041923239](https://cdn.miyunda.net/image/mixedcontent-202041923239.jpg)
![mixedcontent-20204192333](https://cdn.miyunda.net/image/mixedcontent-20204192333.jpg)
将默认的选项改为“允许”，然后重新加载页面，视频就可以正常播放了。

此方法对微软新Edge等基于Chromium的其他浏览器也有相同效果。

感谢观看，希望能帮到更多学生。

相关链接 https://blog.chromium.org/2019/10/no-more-mixed-messages-about-https.html?m=1