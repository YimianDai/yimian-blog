---
title: Install zsh without root
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

记录一下在一台没有 `root` 权限的 Linux 机器上安装 `zsh` 的过程，由于`zsh` 和 `oh-my-zsh` 的安装默认都需要 `root` 权限，所以本文算是一个怎么绕开不必要的 `root` 权限的一个例子。

- [安装 `zsh`](#安装-zsh)
- [配置 `zsh`](#配置-zsh)
- [安装 `oh-my-zsh`](#安装-oh-my-zsh)

## 安装 `zsh`

安装 `zsh` 依据 <https://stackoverflow.com/questions/15293406/install-zsh-without-root-access>

第一步，下载 `zsh`

```Shell
wget -O zsh.tar.xz https://sourceforge.net/projects/zsh/files/latest/download
mkdir zsh && unxz zsh.tar.xz && tar -xvf zsh.tar -C zsh --strip-components 1
cd zsh
```

第二步，编译安装 `zsh`，所需时间有点长，耐心等待即可。虽然这个答案是 13 年的（后来在 19 年做了小幅修改），但我在 21 年年中测试还是完全 OK 的。

```Shell
./configure --prefix=$HOME
make
make install
```

结束后，`zsh` 就安装好了，`exec $HOME/bin/zsh -l` 即可进入 `zsh`。

## 配置 `zsh`

每次 `exec $HOME/bin/zsh -l` 只能手动的进入 `zsh`，那么有什么方法可以在登录的时候就自动进入 `zsh` 呢？
可以将 `exec ~/bin/zsh` 写入 `~/.bashrc` 就可以在登录 `bash` 的时候自动跳转到 `zsh` 了, 当然这个代价是以后只要运行 `bash`，就会自动跳转到 `zsh` 。需要 `source ~/.bashrc` 一下使其生效。

另外，初始的 `zsh` 啥都没有，安装好的 `conda` 环境之类的也都不知道，只需要将 `~/.bashrc` 中的关于 `conda` 的设置复制粘贴进 `~/.zshrc` 即可。
`source ~/.zshrc` 一下就生效了。

此外，在 `zsh` 下，可能还会遇到 `source activate <env_name>` 失效，报错 `source: no such file or directory: activate`. 只需要将 `export PATH="/root/miniconda3/bin:$PATH"` 写入 `~/.zshrc` 即可，注意 `/root/miniconda3/bin` 是我的 `miniconda` 的路径，需要将其替换成你本地的路径。在 `zsh` 中跑一句，`conda init zsh` 则可以使得 `conda activate <env_name>` 生效。最后，都需要 `source ~/.zshrc` 一下。

## 安装 `oh-my-zsh`

根据 <https://gist.github.com/mgbckr/b8dc6d7d228e25325b6dfaa1c4018e78>

```Shell
# clone repository into local dotfiles
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh

# copy template file into home directory
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

`source ~/.zshrc` 一下就生效了。

后记：在文章写完的时候，我才知道原来还有一个功能更加强大的 shell `fish`。