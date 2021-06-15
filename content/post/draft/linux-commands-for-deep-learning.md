---
title: Linux Commands for Deep Learning
authors:
- admin
tags: []
categories: []
featured: false
draft: true

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

linux drwxr-xr-x 
第一位表示文件类型。d是目录文件，l是链接文件，-是普通文件，p是管道
第2-4位表示这个文件的属主拥有的权限，r是读，w是写，x是执行。
第5-7位表示和这个文件属主所在同一个组的用户所具有的权限。
第8-10位表示其他用户所具有的权限。




### `Tmux`

参考资料：[Tmux 使用教程](http://www.ruanyifeng.com/blog/2019/10/tmux.html)

#### Tmux 新建会话

```shell
$ tmux new -s <session-name>
```

#### 查看当前所有的 Tmux 会话

```shell
$ tmux ls
```

#### 重新接入某个已存在的会话

```shell
$ tmux attach -t <session-name>
```

#### 窗格快捷键

`Ctrl+b x`：关闭当前窗格。