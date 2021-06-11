---
title: OpenMMLab 环境的配置
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

记录一下 OpenMMLab 环境的配置的流程，系统是 Ubuntu 18.04，CUDA 版本是 11.1.

- [准备工作](#准备工作)
    - [更新系统并安装必需的 package](#更新系统并安装必需的-package)
    - [安装 zsh 和 oh-my-zsh](#安装-zsh-和-oh-my-zsh)
    - [安装 Tmux](#安装-tmux)
- [安装 `CUDA Toolkit` 与 `miniconda`](#安装-cuda-toolkit-与-miniconda)
    - [安装 CUDA Toolkit](#安装-cuda-toolkit)
    - [安装 `miniconda`](#安装-miniconda)
- [安装 `PyTorch` 与 `OpenMMLab` 库](#安装-pytorch-与-openmmlab-库)
    - [创建虚拟环境](#创建虚拟环境)
    - [安装 `PyTorch`](#安装-pytorch)
    - [安装 `mmcv`](#安装-mmcv)
    - [安装 `mmcls`](#安装-mmcls)
    - [安装 `mmdet`](#安装-mmdet)
    - [安装 `mmseg`](#安装-mmseg)

## 准备工作

#### 更新系统并安装必需的 package

```shell
sudo apt-get update && sudo apt-get install -y build-essential git
```

其中，`build-essential` 包含了 GCC 等 GNU 编译器，GNU 调试器和其他编译软件所必需的开发库和工具；`git` 的话，安装其他 toolkit 和后期开发自己的代码会用上。

#### 安装 zsh 和 oh-my-zsh

```shell
sudo apt install zsh
wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh
sh install.sh
```

安装 oh-my-zsh 的过程会自动将默认 shell 设置成 zsh。

#### 安装 Tmux

Tmux 的全称是 Terminal MUtipleXer，即终端复用软件。顾名思义，它的主要功能就是在你关闭终端窗口之后保持进程的运行，这样的话，即使 SSH 连接断开了程序也不会被终止。

```shell
apt-get install tmux
```

## 安装 `CUDA Toolkit` 与 `miniconda` 

#### 安装 CUDA Toolkit

需要注意的是，从官网 [CUDA Toolkit Archive](https://developer.nvidia.com/cuda-toolkit-archive) 下载的 CUDA Toolkit 其实是同时带有 GPU Driver 和 CUDA Toolkit 的。

```shell
wget https://developer.download.nvidia.com/compute/cuda/11.1.1/local_installers/cuda_11.1.1_455.32.00_linux.run
sudo sh cuda_11.1.1_455.32.00_linux.run
```

#### 安装 `miniconda`

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sudo sh Miniconda3-latest-Linux-x86_64.sh
```

让 `CUDA` 和 `conda` 生效

```shell
source ~/.zshrc
conda --version
nvidia-smi
```

## 安装 `PyTorch` 与 `OpenMMLab` 库

#### 创建虚拟环境

```shell
conda create -n mmlab python=3.7 -y
conda activate mmlab
```

虚拟环境名为 `mmlab`

#### 安装 `PyTorch`

根据 [PyTorch 官网](https://pytorch.org/) 给的 command

```shell
conda install pytorch torchvision torchaudio cudatoolkit=11.1 -c pytorch -c nvidia
```

打开 `python`，测试一下能否正常调用 GPU

```python
import torch
torch.cuda.is_available()
```

#### 安装 `mmcv`

按照 [`mmcv` 官方文档](https://github.com/open-mmlab/mmcv)：

```shell
pip install mmcv-full -f https://download.openmmlab.com/mmcv/dist/cu111/torch1.8.0/index.html
```

需要注意的是，官方文档的更新不一定跟实际进度完全同步，比如此时此刻官方文档上说目前 `mmcv-full` 最新的版本是支持 `CUDA 11` 和 `PyTorch 1.7.0`。然而，在 <https://download.openmmlab.com/mmcv/dist/> 可以查到，目前已经有支持 CUDA 11.1 和 `PyTorch 1.8.0` 了。

打开 `python`，测试一下能否正常 `import mmcv`，会遇到如下报错 `ImportError: libGL.so.1: cannot open shared object file: No such file or directory`. 按照下缺少的 `libgl1-mesa-glx` 库即可。

```shell
sudo apt install libgl1-mesa-glx
sudo apt-get install libglib2.0-0
```

再次打开 `python`，测试 `import mmcv` 不再报错了。

#### 安装 `mmcls`

按照 [`mmcls` 官方文档](https://mmclassification.readthedocs.io/en/latest/install.html#install-mmclassification)：

```shell
git clone https://github.com/open-mmlab/mmclassification.git --depth 1
cd mmclassification
pip install -e .  # or "python setup.py develop"
```

打开 `python`，测试 `import mmcls` 正常。

#### 安装 `mmdet`

按照 [`mmdet` 官方文档](https://mmdetection.readthedocs.io/en/latest/get_started.html#install-mmdetection)：

```shell
git clone https://github.com/open-mmlab/mmdetection.git --depth 1
cd mmdetection
pip install -r requirements/build.txt
pip install -v -e .
```

打开 `python`，测试 `import mmdet` 正常。

#### 安装 `mmseg`

按照 [`mmseg` 官方文档](https://mmsegmentation.readthedocs.io/en/latest/get_started.html#installation)：

```shell
pip install mmsegmentation # install the latest release
```

打开 `python`，测试 `import mmseg` 正常。

最后，用 `pip list` 查看一下相应的版本：

```shell
(mmlab) ➜  ~ pip list
Package            Version
------------------ -------------------
mmcls              0.12.0
mmcv-full          1.3.5
mmdet              2.13.0
mmsegmentation     0.14.0
torch              1.8.1
torchvision        0.9.1
```