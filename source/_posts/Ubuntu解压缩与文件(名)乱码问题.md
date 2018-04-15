---
title: Ubuntu解压缩与文件(名/内容)乱码解决方案
date: 2018-01-31
tags: 
- Linux
- 解压
- 乱码
---

我的另一篇笔记：<a href="https://godsing.top/2018/01/21/%E4%B8%AD%E6%96%87%E5%AD%97%E7%AC%A6%E9%9B%86%E7%BC%96%E7%A0%81GB2312%E3%80%81GBK(CP936)%E3%80%81GB18030/">中文编码知识</a>

## 1. 分析

### 1.1 zip解压时文件乱码

本质问题还是zip格式的缺陷，没有字段标志出文件名的编码格式。
ZIP在压缩与解压缩的时候默认使用了系统的本地编码，如windows中文环境下的编码多为gbk，gb2312，日文环境下是JIS，linux默认编码为UTF8等；那么在不同系统环境下，只要压缩与解压缩的编码不一致，就会出现乱码。[^1] 

<!-- more -->

### 1.2 tar.xz 文件

tar.xz结尾的压缩文件，解决方法如下： 

```shell
$xz -d file.tar.xz # 会得到 file.tar 文件
$tar -xvf  file.tar -C your-directory
```

这个压缩包也是打包后再压缩，外面是xz压缩方式，里层是tar打包方式。

### 1.3 tar.gz(即.tgz)解压

```shell
tar -zxvf File.tgz -C Your-Directory
```



## 2. 文件名|乱码解决方案

### 2.1 unzip方案

```shell
# 先试试
unzip -O GB18030 file.zip -d directory
# 不行再试试 GBK也可写成CP936，这是gbk编码在windows里的别称
unzip -O GBK file.zip -d directory 
# 再不行继续试试
unzip -O GB2312 file.zip -d directory
```

这么试的原因是，编码技术的演进方向为：GB2312 ⇒ GBK(=CP936) ⇒ GB18030，最新的一般能兼容旧的编码技术，遇到不兼容的情况再用旧的编码去尝试。如果无法使用 `-O` 参数，参考以下链接打补丁。

#### 缺参数，则给unzip打补丁

参考 https://github.com/ikohara/dpkg-unzip-iconv ；按步骤给unzip打补丁，打完即可使用`-O`参数了。



### 2.2 unar方案

如果是Debian，已经默认安装了unar，这个工具会自动检测文件的编码，也可以通过-e来指定：

```shell
sudo apt-get install unar
unar file.zip
```

即可解压出中文文件。

**1. 安装**

```shell
sudo apt-get install unar
```

**2.列出压缩包内容**

```shell
lsar test.zip
```

**3.解压压缩包**

```shell
unar test.zip
```

**4.unar常用选项解释**

-o

> 解释：指定解压结果保存的位置 
> `unar test.zip -o /home/dir/`

-e

> 解释：指定编码 
> `unar -e GBK test.zip`

-p

> 解释：指定解压密码 
> `unar -p 123456 test.zip`

**3.解决linux解压压缩包中文文件名乱码问题**

```shell
lsar test.zip

###若发现乱码，可指定压缩包文件名使用的编码格式##
lsar -e GB18030 test.zip

###若能正常列出文件名，可解压###
unar -e GB18030 test.zip
```



### 2.3 7z和convmv结合方案

在ubuntu下的安装命令是

```shell
sudo apt-get install p7zip-full convmv
```

安装完之后，就可以用`7z`和`convmv`两个命令完成解压缩任务。

```shell
LANG=C 7z x zip-file.zip
convmv -f GBK -t utf8 --notest -r your_unzipped_file_floder/
# 或者先cd到解压好的地方
convmv -f gbk -t utf8 --notest  ./* （其实应该一个点 . 就行了，而不用 ./*）
```

第一条命令用于解压缩，而LANG=C表示以US-ASCII这样的编码输出文件名，如果没有这个语言设置，它同样会输出乱码，只不过是UTF8格式的乱码(`convmv`会忽略这样的乱码)。

第二条命令是将GBK编码的文件名转化为UTF8编码，`-r`表示递归访问目录，即对当前目录中所有文件进行转换。

- convmv支持的部分参数如下：

 `-f`  源编码

 `-t` 目标编码

 `--notest`  convmv默认只会显示文件名转换后的结果而不会实际进行转换。使用这个参数使convmv对文件名进行实际的编码转换。

 `--list`  列出convmv支持的所有编码

 `-r`  递归转换所有子目录的文件名编码

### 2.4 7z方案

```
apt-get install p7zip-full
```

7zip命令有7z和7za，7za是精简版部分格式不支持，7z是全功能版的，建议使用7z。

```
7z {a|d|l|e|u|x} 压缩包文件名 {文件列表或目录，可选}
```

- a 向压缩包里添加文件或创建压缩包，如向deepvps.7z添加deepvps001.jpg，执行：7z a deepvps.7z deepvps001.jpg；将deepvps目录打包执行：7z a deepvps.7z deepvps；
- d 从压缩里删除文件，如将deepvps.7z里的test.jpg删除，执行：7z d deepvps.7z test.jpg
- l 列出压缩包里的文件，如列出deepvps.7z里的文件，执行：7z l deepvps.7z
- e 解压到当前目录，目录结构会被破坏，如deepvps.rar内有如下目录及文件123/456/789.html，执行：7z e deepvps.rar，目录123和456及文件789.html都会存放在当前目录下。
- x 以完整路径解压。

由于zip文件中没有声明其编码，所以在Linux上使用unzip解压以默认编码解压，中文文件名会出现乱码。

使用7z解压即可解决：`7z x deepvps.zip`

也可以使用：`jar xvf deepvps.zip`



## 3 文件内容|乱码解决方案

### 2.1 iconv工具

```
iconv -f gbk -t utf-8 file1 -o file2  # gbk编码转换为utf-8
```

命令很简单，可以man出手册或者`--help`看一下。

### 2.2 enca工具

```shell
# -L指明文件语言，一般可以省略
enca -L zh_CN file # 检查文件的编码
enca -L zh_CN -x UTF-8 file # 将文件编码转换为"UTF-8"编码
enca -L zh_CN -x UTF-8 file1 file2 # 如果不想覆盖原文件可以这样
```

说说遇到的坑：非得让我指定语言，不指定还不行，说可以用 `-L none`，然而根本识别不了。然后又试了 zh_CN，一样不行。用 `enca --list language` 查看所支持的语言列表，然而写那么多一堆，居然没有English，逗我？！而且，这列表里的项不能直接当作 `-L`的参数，不知道参数去哪里查！然后就懒得继续查了，毕竟问题已解决。

### 2.3 查看文件编码

#### 2.3.1 用vim查看

用vim打开文件，输入：

```shell
:set fileencoding
```

回车便可看到编码，但是**不一定可信**！

我的一个本地文件 besttrace4linux.txt ，明明是GB18030编码，但用vim查询显示是latin1编码，搞死我了！

用`iconv`试着把该文件从latin1, ascii 转成 utf-8, iso88592, gb2312 各种错！最后试着转到 GB18030，发现居然能转过去，只是转过去后仍然是乱码。此时我灵机一动，该不会原本就是这个编码吧？然后回到我的zsh，发现果然在zsh打印的信息里显示了正确的中文！我屮艸芔茻，花了一个小时终于见到曙光了。这时候，一句：

```shell
iconv -f gb18030 -t utf-8 besttrace4linux.txt -o readme.txt
```

哇！世界从未如此美好，成功！至此，知道**用vim查看编码是会骗人的**！！！



## Reference

http://marshal-r.iteye.com/blog/2161903

https://linuxtoy.org/archives/wrong-handling-of-chinese-coded-filename-in-fileroller-unzip.html

https://www.findhao.net/easycoding/1605

[^1]: 作者：天白才痴 链接：https://www.zhihu.com/question/20523036/answer/75186086

