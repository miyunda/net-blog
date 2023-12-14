---
title: 99%的影楼师傅都不会的技巧，我手把手教你。

categories:
  - 消费电子
date: 2017-01-19 22:42:35
---

真标题：根据给定的文件列表，快速复制文件。

各位有木有遇到过要从一大堆文件里按照名单复制文件？

前几天我们带娃去了一个儿童摄影工作室，给娃拍了一些照片。

拍摄过程不表。拍完以后，工作人员当场把所有照片复制给了我，叮嘱我回家挑选好要制作的那些，发到她邮箱。

她复制给我的是酱紫的，拍了一大堆：

![copy-filelist-corresponding-files-007](https://cdn.miyunda.net/uPic/copy-filelist-corresponding-files-007_20231030.jpg)

我挑好了之后写了一个文本文件：

![copy-filelist-corresponding-files-009](https://cdn.miyunda.net/uPic/copy-filelist-corresponding-files-009_20231030.png)

就是上面那样，发给了她。 

然后她给我打电话，说我发的不行，要把文件发给她，而不是列表。我电话里告诉她，这要求不合理：“我的文件都来自您那里，您应该有所有的文件，按照我发的列表挑出即可；假如我发送一遍实际文件，那是浪费时间与带宽资源。”

她呢，很坦诚：“你说的我都明白，但是我不会。” 

有一个文件列表是文本文件 *wildcard.txt* ，里面每行存有一个文件名（的局部），要求我们把相应的文件复制出来，比如我的文本文件内容有*6666*，那么就一定会去复制文件名是*IMG6666.JPG*的文件。

macOS，打开`Terminal`：

```bash
while read filename
do
 find /yourpath/source/ -maxdepth 1 -type f \
 -name "*$filename*" -exec cp {} /yourpath/dest/ \\;
done</yourpath/wildcard.txt
```

Windows，打开`命令提示符（Dos窗口）(Powershell)`将以下文本存为BAT文件并运行
：
```powershell
 for /f "delims=" %%i in (wildcard.txt) do (
 copy /y "c:\\source\\*%%~ni*" "C:\\dest\\"
 )
 PAUSE
```

注意可能犯的错误是，每行结尾的符号Windows与Unix是不同的，建议在各自的环境生成*wildcard.txt*。下图酱紫就是Windows的格式，不能直接在macOS使用。

![copy-filelist-corresponding-files-010](https://cdn.miyunda.net/uPic/copy-filelist-corresponding-files-010_20231030.png)

Windows运行起来效果类似下面，macOS无提示。

![copy-filelist-corresponding-files-008_](https://cdn.miyunda.net/uPic/copy-filelist-corresponding-files-008_20231030.png)

学会了吗？你也可以去开影楼啦，而且能打败99%的竞争者呢。

感谢观赏，完。

* * *

~~你的那些*.mp4 也能组织得井井有条了。~~