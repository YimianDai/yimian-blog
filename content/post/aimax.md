---
title: AI MAX 平台的使用
authors:
- admin
tags: []
categories: []
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
image:
  caption: ""
  focal_point: ""

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references 
#   `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

记录一下 AI MAX 平台的使用。

- [账号创建](#账号创建)
- [登录](#登录)
  - [连接 VPN](#连接-vpn)
  - [登录 AI MAX](#登录-ai-max)
  - [登录实例](#登录实例)
- [挂载硬盘](#挂载硬盘)

## 账号创建

首先，需要找到对应的管理员要求开通深信服云 IT 平台 <https://yun.njust.edu.cn/login> 和 AI MAX 平台 <http://172.25.100.125/index.html?#/login> 的账号，这两个账号是独立的，需要分别开通。

在创建 <https://yun.njust.edu.cn/login> 账号的同时，管理员还会一并给创建一块从属于该账户的 1T 存储空间。

## 登录

### 连接 VPN

外网是没有办法直接访问 AI MAX 平台的，所以需要通过特点的 VPN 才行。
使用浏览器打开网址：<https://yun.njust.edu.cn:7774/?src=connect>，一般都会出现提示不安全的情况，点击高级/忽略然后再前进即可。在 Mac 上，Chrome 即使点击了 advanced，也不会出现 proceed，但是换成 Safari 就可以了。Windows 机器貌似没有这个问题。

输入云平台用户名和密码，输入后点击登录后，可以看到 OpenVPN 在各个操作系统上的客户端以及你个人的 connection profile，分别下载到本地。
客户端安装并且打开后，点击该客户端的图标 -> Import -> from local file... 选中下载好的 user-locked profile，然后就可以连接上 VPN 了。

### 登录 AI MAX

连接好 VPN 后，便可以登录 AI MAX 平台 <http://172.25.100.125/index.html?#/login> 了。

点击「模型训练」->「交互式开发」->「创建」。
「工作空间」选择「私有数据」的「全部数据」，「执行器」选择「Terminal」。
在「选择镜像」中，点击「公共镜像」->「查看更多公共镜像」->「镜像名称 `cuda`，标签 `11.1-cudnn8-devel-ubuntu18.04`」。
「GPU 类型」选择相应的，比如「GeForce_RTX_3090」，**勾选掉**「使用比例模板」，内存：CPU：GPU 按照 256：32：8 的比例来，如果是 1 个 GPU，那就是 32：4：1。
最后，点击「提交」就可以耐心等待实例被创建了。

### 登录实例

在实例被创建后，可以在「模型训练」->「交互式开发」中看到相应的实例，并有形如 `ssh <ip address>:<port>` 这样的命令提示，用户名为 `root`。我们可以通过如下命令登录到该计算实例上：

```shell
ssh root@<ip address> -p <port>
```

输入 AI MAX 的密码即可登录成功。如果不想要每次都记住 `<ip address>` 和 `<port>`，可以在 `~/.ssh/config` 中添加如下设置：

```shell
Host AIMAX
    HostName <ip address>
    Port <port>
    User root
```

记得将 `<ip address>` 和 `<port>` 替换成真实分配到的 ip 地址和端口号。

## 挂载硬盘

在挂载硬盘前，需要先创建挂载点，比如

```shell
mkdir /<your name>
```

因为挂载的是 `cifs` 格式的硬盘，在挂载前还需要安装下相应的 package，这样 Ubuntu 才可以进行相应的读写：

```shell
apt update
apt install cifs-utils
```

如果不是以 `root` 登录的话，还需要在命令开头加上 `sudo`。

最后，进行挂载：

```shell
mount -t cifs -o username=<username>,password='<password>',vers=3.0 //pcalab.dynamic.oceanstor9000.com/pcalab_<username> /<your name>
```

这里的 `<username>` 就是 <https://yun.njust.edu.cn/login> 的账号，即学号或工号，`<password>` 则是 <https://yun.njust.edu.cn/login> 的密码，注意不是 AI Max 的密码。

最后，由于管理员的默认设置，`/<your name>` 下，我们是没有删除权限的，这对于我们写代码等实际操作很不方便，这个**需要联系管理员再赋予我们删除权限**，否则得话，只能用 `/opt/data/private`，这个是不需要专门联系管理员赋予我们删除权限的。

多说一句，机器实例里的 `~` 文件夹下的储存空间那么好用，我们为什么要费尽心思研究怎么挂载硬盘呢？因为机器实例里的储存空间会随着机器分配时间的到期而被收回，所以千万不能放代码之类的重要的东西，只有放在 `/opt/data/private` 和 `/<your name>` 下的内容才是永久的。
但是，机器实例里的 `~` 文件夹下也不是什么都不能放，可以在里面安装 `miniconda` 以及相应的 library，然后制作成镜像，这样即使被储存收回了，通过镜像创建实例的时候又都恢复了。

至此，对于 AI MAX 平台上的计算实例，其他操作就跟一般的 Linux 机器无异了。