---
title: Linux下shadowsocks各种代理方式总结
date: 2018-04-13
tags: [Linux, socks, shadowsocks, 代理, 科学上网]
---

## 前言

本文算是一篇总结，把使用 shadowsocks无GUI客户端 的各种常见的代理方式梳理一遍，给有需要的人，避免走同样的弯路。以下是本文的预备条件：

- Linux 系统（本文用的是 Ubuntu 16.04）

- 已有 shadowsocks 帐号。获取免费帐号/自行搭建shadowsocks服务端 可以参考：

  > [寻找Shadowsocks账号/搭建Shadowsocks服务](https://doub.ws/ss-jc26/#%E4%BA%8C%E3%80%81%E5%AF%BB%E6%89%BEShadowsocks%E8%B4%A6%E5%8F%B7/%E6%90%AD%E5%BB%BAShadowsocks%E6%9C%8D%E5%8A%A1)
  >
  > [『原创』ShadowsocksR MudbJSON模式多用户一键脚本 支持流量限制等](https://doub.ws/ss-jc60/)
  >
  > [『视频』ShadowsocksR 服务端一键脚本安装 新手小白视频教程](https://doub.ws/ss-jc64/)


<!-- more -->

### 废话

如今，墙越来越高，科学上网越来越困难。Shadowsocks 可谓是科学上网的瑞士军刀，而Linux系统下却没有一个好用的 shadowsocks 或 shadowsocksR 的图形界面客户端（如果有，恳请发邮件告知，十分感谢！）。本来 Shadowsocks-Qt5 用得好好的，最近不知道怎么回事，帐号也连接上了，测试延迟也没有问题，但整个电脑的流量就是不走代理，又找不到哪里配置异常，卸了重装也没用。

本来也懒得折腾，可是当我 `sudo pip install tensorflow` ，看着十几 KB/s 的龟速下载一个 48M 的包，我比乌龟还难受。算了，放着先干其他事吧。然而过了几十分钟，提示 timeout，WTF! 伟大的墙让我心碎T_T

于是开始折腾Shadowsocks无GUI客户端：

1. 如果只用浏览器科学上网，那启动 shadowsocks 之后在浏览器配置插件即可。
2. 因为我想让 pip 也走代理，所以本能地以为开启系统(全局)代理就可以了，结果发现还是不行。什么原因呢？
3. 查资料发现，终端里执行的程序大多对socks代理支持不友好，所以就尝试用 proxychains 来实现在终端里的代理。果然，proxychains 的配合使用可以让很多命令走代理。可是，配合 pip 仍旧不行！难道 pip 要单独处理？
4. 果然发现 pip 可以单独[配置代理](#pip 使用代理)。至此，目标达到了，pip 下载包终于可以达到 1MB/s 以上。然而半天时间过去了，多么痛的领悟！



## 让shadowsocks跑起来

### 安装

**Debian/Ubuntu:** 

```bash
apt-get install python-pip  # 已经安装了pip的话跳过
pip install shadowsocks
```

**CentOS:** 

```bash
sudo yum install python-setuptools && easy_install pip
sudo pip install shadowsocks
```

### 配置

找个地方放shadowsocks的配置文件，一般放到 `/etc`下面：

```bash
sudo vim /etc/shadowsocks.json
```

比如我放在用户目录下，因为有时需要修改，放在这里方便些：

```bash
vim /home/your-username/shadowsocks.json
```

你可以根据自身情况考虑，文件名和路径都可以随意设置，只要自己能记住就行。

然后在`shadowsocks.json`里面添加配置信息，如：（注意双引号）

```json
{
  "server":"your_server_ip",
  "local_address": "127.0.0.2",
  "local_port":1080,
  "server_port":your_server_port,
  "password":"your_server_password",
  "timeout":600,
  "method":"aes-128-ctr",
  "fast_open":false
}
```

把

- `your_server_ip`改为自己的服务器IP
- `your_server_port`改为自己的服务器端口
- `your_server_password`改为自己的密码
- `method`的值改为自己的加密方式，一般是`aes-256-cfb`或者`rc4-md5`

详细配置说明：

| Name          | 说明                                       |
| ------------- | ---------------------------------------- |
| server        | 服务器地址，填ip或域名                             |
| local_address | 本地地址                                     |
| local_port    | 本地端口，一般1080，可任意                          |
| server_port   | 服务器对外开的端口                                |
| password      | 密码，可以每个服务器端口设置不同密码                       |
| port_password | server_port + password ，服务器端口加密码的组合      |
| timeout       | 超时重连                                     |
| method        | 默认: “aes-256-cfb”，见 [Encryption](https://github.com/shadowsocks/shadowsocks/wiki/Encryption) |
| fast_open     | 开启或关闭 [TCP_FASTOPEN](https://github.com/shadowsocks/shadowsocks/wiki/TCP-Fast-Open), 填true / false，需要服务端支持 |

保存退出就配置好啦！

### 启动

配置文件的路径改成自己的，如：`/etc/shadowsocks.json`

- 前端启动：`sslocal -c /home/your-username/shadowsocks.json`；
- 后端启动：`sslocal -c /home/your-username/shadowsocks.json -d start`；
- 后端停止：`sslocal -c /home/your-username/shadowsocks.json -d stop`；
- 重启(修改配置要重启才生效)：`sslocal -c /home/your-username/shadowsocks.json -d restart`

### 开机自启

以下使用Systemd来实现shadowsocks开机自启。

先检查一下`sslocal` 的具体目录，然后配置文件：

```bash
$ where sslocal
/usr/local/bin/sslocal
$ sudo vim /etc/systemd/system/shadowsocks.service
```

在里面填写如下内容：

```
[Unit]
Description=Shadowsocks Client Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/local/sslocal -c /home/your-username/shadowsocks.json

[Install]
WantedBy=multi-user.target
```

把 `/usr/bin/local/sslocal` 修改为你的 `sslocal` 所在路径，如 `/usr/bin/sslocal` 

把`/home/your-username/shadowsocks.json`修改为你的`shadowsocks.json`所在路径，如：`/etc/shadowsocks.json`

> 小提示：复制上面的内容后，在 vim 中标准(Normal)模式下，输入 `"+p` (输入英文引号、加号、p)可以实现粘贴。
>
> 小知识：如果发现粘贴不了，那是因为只有vim.gtk或vim.gnome才能使用系统全局粘贴板，默认的vim.basic看不到`+`号寄存器。安装vim.gnome使用`apt-get install vim-gnome`，然后vim自动会链接到vim.gnome。

配置生效：

```bash
systemctl enable /etc/systemd/system/shadowsocks.service
```

输入管理员密码就可以了。

现在你可以马上重启试试，或先在后台启动，等下次重启再看看！

---

正常来说，我们主要是通过浏览器去访问一些被墙的网站，而且大多数人也仅仅只用浏览器科学上网，所以浏览器的代理就成为重中之重。下面具体阐述如何实现浏览器代理。

## 浏览器代理

### 安装 SwitchyOmega 插件

以 Chrome 为例，安装 SwitchyOmega 插件代理。

Github 下载 SwitchyOmega：<https://github.com/FelisCatus/SwitchyOmega/releases/>

Chrome 打开<chrome://extensions/>，把插件托进去安装。

### 配置 Proxy Profile

打开插件配置页面，点击左下角的 “+New profile...” ，在弹出的窗口中选择“Proxy Profile”，名字随便起(比如ss)，创建完成后配置一下：

- `Server`填写`shadowsocks.json`配置中的`local_address`
- `Port`填写`shadowsocks.json`配置中的`local_port`
- 下面的 Bypass List 填上不需要走代理的地址，比如：`shadowsocks.json`配置中的`local_address`应该填进去。
- 好了，点击左边绿色的`Apply changes`保存。

![](http://p1nwamyah.bkt.clouddn.com/18-4-13/63345407.jpg)

### 配置 Switch Profile

同样点击左下角的 “+New profile...” ，在弹出的窗口中选择“Switch Profile”，名字随便起，创建完成后如图：

![img](http://p1nwamyah.bkt.clouddn.com/18-4-13/18099298.jpg)



先点击“+Add a rule list”，然后开始配置：

![](http://p1nwamyah.bkt.clouddn.com/18-4-13/84531209.jpg)



- 勾上`Rule list rules`，右边的`Profile`列在下拉框中选择刚才创建的`ss`

- `Default`的`Profile`列，选择直连模式`[Direct]`

- `Rule List Format`选择`AutoProxy`

- 下面的`Rule List URL`填写`gfwlist`地址:

  ```bash
  https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt
  ```

- 下载规则文件，点击`Download Profile Now`

- 点击左边绿色的`Apply changes`保存

### 启用 SwitchyOmega

启用 SwitchyOmega 插件，选择 Auto Switch 模式就可以了。



## 系统代理

如有你有系统全局代理的需要，那么就往下看，否则只需要参考上文的 [浏览器代理](#浏览器代理) 即可。

找到“网络”配置项：

![](http://p1nwamyah.bkt.clouddn.com/18-4-13/53938885.jpg)

设置成如下：

![](http://p1nwamyah.bkt.clouddn.com/18-4-13/62362422.jpg)

设置完成点击"Apply system wide"，需要输入验证权限，输入密码即可，这里开启的默认版本是socks5。

Tips: 如果要关闭系统代理，上图中的那个 Method 选择 "None"，然后同样点击"Apply system wide"即可。

系统代理是全局走代理的，访问国内网站就不太方便了，相当于绕了个弯，速度变慢，甚至是访问反而受限了，如无必要，还是推荐浏览器方式按规则选择性代理。

---

这样设置后，浏览器一般可以正常科学上网了，因为浏览器基本上都能完美支持http代理和socks代理。然而终端里执行的程序基本上都不能友好的支持socks代理，在终端要使用socks代理，常见的有两种方法：使用 polipo 或 proxychains，以下分别介绍。



## 终端代理

### 方法一：proxychains

Tips: 使用 proxychains 不需要开启系统代理，只要启动着 shadowsocks 就可以了。

#### 安装

```bash
sudo apt-get install proxychains   # 安装 proxychains
```

#### 配置

```bash
sudo chmod 777 /etc/proxychains.conf # 原始权限不足，需要修改权限
vim /etc/proxychains.conf
sudo chmod 644 /etc/proxychains.conf # 保存好配置后，权限修改回去
```

编辑配置文件时，将最下面类似 `socks4 127.0.0.1 9050` 的一行注释掉，添加一行 `socks5 127.0.0.2 1080` 这里的IP地址和端口就是上面刚刚设置的系统代理的 `Socks Host` 和端口。

#### 测试

```bash
proxychains curl www.google.com
```

#### 使用

在正常使用的命令行前面，加上 `proxychains`（如上面测试的命令）， 该命令就会使用代理。

也可以通过输入`proxychains bash`建立一个新的bash，基于这个bash运行的所有命令都将使用代理。



### 方法二：polipo

Tips: 该部分以 http_proxy 为例，socks5 是否可行未知，可自行google。

#### 安装polipo

```bash
$ sudo apt-get install polipo
```

#### 修改配置文件

config文件是只读的，要想修改里面的数据，需要获得最高权限。

```bash
$ cd /etc/polipo/
$ sudo chmod 777 config # 为config文件申请最高权限
$ vim /etc/polipo/config # 打开进行编辑
```

原文件中已经有了两句话，那么需要新加入3句话：

```bash
socksParentProxy = "localhost:1080"
socksProxyType = socks5
logLevel=4
```

ps：这里建议修改文件后恢复其本来的权限，这算是个好习惯。

#### 关闭和启动polipo

关闭软件，让配置生效，然后重启。

```bash
$ sudo service polipo stop
$ sudo service polipo start
```

#### 验证和使用

安装完成后使用下面代码验证效果：

```bash
$ curl ip.gs #查询你的IP地址和地理信息
```

```bash
$ http_proxy=http://localhost:8123 curl ip.gs
```

第二条语句得到的ip地址已经不是中国的了：“当前 IP：103.204.172.117 来自：日本大阪府大阪 starrydns.com”，说明安装成功。

上面实验说明了想要为某个命令加上代理，就在前面使用：http_proxy=[http://localhost:8123](http://localhost:8123/)

ps：8123是polipo的默认端口，如有需要，可以修改成其他有效端口。

#### 设置别名

每一次都输入这么一串命令实在太不人性化，解决方法就是给这个命令一个缩写的别名，比如“hp”。

```bash
$ vim .bashrc
```

打开配置文件，在最后面加上一句：

```bash
alias hp="http_proxy=http://localhost:8123" 
```

关闭文件，执行下面代码：

```bash
$ source ~/.bashrc
```

这样，hp就可以代表之前很长的命令，试验一下：

```bash
$ hp curl ip.gs
```

当前 IP：103.204.172.117 来自：日本大阪府大阪 starrydns.com ，bingo！

#### 为当前会话设置全局代理

难道要在每条联网命令前面都加上“hp”？当然不会，以下操作可以让**当前终端**窗口的所有联网命令都经过代理，一条命令，接管所有：

```bash
$ export http_proxy=http://localhost:8123 # 当前终端使用代理
$ unset http_proxy # 当前终端取消代理
```

#### 更为长久的代理设置

如果我想Ubuntu终端一直处于代理状态，能不能做到呢？这也是可以的，以下设置可以让本机的终端一直拥有代理能力，设置好后就完全不用操心了，类似于写入环境变量的操作。

方法很简单，将以下语句：

```bash
export http_proxy=http://localhost:8123
```

加入.bashrc文件末尾，再执行`source ~/.bashrc`即可；或者如果你使用了zsh，那么就是 ~/.zshrc。

现在有很多网站使用了更安全的https协议，因此，上一句若为`export https_proxy=http://localhost:8123` 则能对htttps协议起作用，但是这两句在bashrc中共存，貌似会出问题，这里博主使用别名“hps”来调用，确保不出错。那下文的例子就很好解释了。

**ps：实际使用中，某些命令貌似还是需要单独加hp，比如我用wget命令下载文件的时候，加上了别名hp ，下载速度才快得起来，例如：**

```bash
# 加上hp，才能达到1M/s以上的下载速度
wget hp https://sourceforge.net/projects/opencvlibrary/files/opencv-unix/2.4.9/opencv-2.4.9.zip/download 
```



## 其他代理(pip|git)

### git 使用代理

刚才的一大堆设置对git命令没有作用，为此我们要单独设置。

事实上在git命令最后加参数可以实现代理：

```bash
--config http.proxy=localhost:8123
```

但我们仍然觉得不方便，还是起个别名吧，比如就叫“gp”。

在.bashrc文件末尾加入这一句：

```bash
gp=" --config http.proxy=localhost:8123"
```

执行`source ~/.bashrc`

以后，在git clone命令后面加入`$gp`就可以加快克隆速度，比如：

```bash
$ git clone https://github.com/gmarik/Vundle.vim.git $gp
```

### pip 使用代理

在配置完系统代理后，不需要配置 polipo 或 proxychains (配了似乎也没用？参见“后话”)，直接使用以下方法就可以实现代理，举例：

```bash
sudo pip --proxy socks5://127.0.0.2:1080 install tensorflow
```

pip 虽然能够管理包，却没有切换镜像源的功能，而我们下载的包，大多数都在国外大型的代码托管服务器上，这就导致了往往几M的包要下载一个小时。

如果你已经给电脑部署了 HTTPS 以及 HTTP 代理，甚至给终端也配置了代理，但 pip 不吃这一套，所以就需要单独给 pip 设置代理。

所幸，pip 有个选项，如下：

```bash
--proxy <proxy>        Specify a proxy in the form [user:passwd@]proxy.server:port.
```

我们直接用这个命令就好了，不过这个命令需要每次在你下载包时附加，比如这样：

```bash
pip3 --proxy 127.0.0.1:6152 install snowlp
```

这太麻烦了，编辑 ~/.bashrc 文件，或者如果你使用了zsh，那么就是 ~/.zshrc ,在文末添加 `alias pip3="pip3 --proxy 127.0.0.1:6152"`  (像我使用socks代理，则添加 `alias pip3="pip3 --proxy socks5://127.0.0.2:1080"` )，下次打开终端后就不用每次都输入这么长的代理选项了。

后话：

如果使用 `sudo proxychains pip install 包名` 的方式实现 pip 代理，可能会[报错](https://stackoverflow.com/questions/38794015/pythons-requests-missing-dependencies-for-socks-support-when-using-socks5-fro)，这时候可以尝试先把系统代理关了(不关可能也行，只是速度慢)，然后 `sudo pip install pysocks` ，装完后再开启系统代理，再使用 `sudo proxychains pip install 包名` 的方式。很大可能仍然不行，比如我的会报错（无法识别socks版本）：

```
ValueError: Unable to determine SOCKS version from socks://127.0.0.2:1080/
```

可是我在 proxychains.conf 里都已经配置了 socks5 了，为什么识别不了呢？欢迎留言或发邮件讨论。



## Reference

1. [Linux安装配置Shadowsocks客户端及开机自动启动](https://blog.huihut.com/2017/08/25/LinuxInstallConfigShadowsocksClient/)

2. [ubuntu16.04LTS+shadowsocks+proxychains全局访问](http://zheng-zy.github.io/posts/30918/)

3. [Ubuntu系统下浏览器和终端的SS代理配置](https://blog.csdn.net/Jesse_Mx/article/details/52863204)

4. [让 pip 走代理](https://www.logcg.com/archives/1914.html)

5. [ubuntu开启http和socks全局代理测试与检验](http://govfate.iteye.com/blog/2069022)


