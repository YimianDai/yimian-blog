---
title: 深度学习环境配置中的 CUDA
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

CUDA 是配置深度学习环境过程中必备的一环。
因为没有系统性地了解过 CUDA 相关的知识，一直以来，我都将其当作一个麻烦、令人害怕的黑箱，深度学习环境的维护也一直处于一种浑浑噩噩、得过且过的状态，对于 CUDA 相关的 Toolkit、cudnn、Driver 之类的关系也一头雾水。
这篇文章的目的在于结束这种糊里糊涂的现状，厘清与 CUDA 相关的诸多概念，并且给出具体的 CUDA 配置步骤。

## GPU 与 GPGPU

显卡驱动就是用来驱动显卡的程序，它是硬件所对应的软件。驱动程序即添加到操作系统中的一小块代码，其中包含有关硬件设备的信息。

有了此信息，计算机就可以与设备进行通信。驱动程序是硬件厂商根据操作系统编写的配置文件，可以说没有驱动程序，计算机中的硬件就无法工作。操作系统不同，硬件的驱动程序也不同，各个硬件厂商为了保证硬件的兼容性及增强硬件的功能会不断地升级驱动程序。





## CUDA Driver 与 CUDA Toolkit

NVIDIA CUDA Toolkit 所包含的具体内容可以看 <https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#major-components>, 里面有 NVIDIA Linux Driver，还有 cudart 等各种. 仔细读一下这个页面的内容，可以对 CUDA 相关的关系了解的更清楚。

从 <https://developer.nvidia.com/cuda-zone> 可知 The CUDA Toolkit from NVIDIA provides everything you need to develop GPU-accelerated applications. The CUDA Toolkit includes GPU-accelerated libraries, a compiler, development tools and the CUDA runtime. 也就是说尽管官方提供的 CUDA Toolkit 包含了 GPU Driver（也就是 CUDA Driver），但是这只是为了我们方便，在官方严格的定义里面，CUDA Toolkit 这个概念是不含有 GPU Driver（也就是 CUDA Driver）的。

[CUDA Toolkit Documentation v11.3.1](https://docs.nvidia.com/cuda/)

[Difference between the driver and runtime APIs](https://docs.nvidia.com/cuda/cuda-driver-api/driver-vs-runtime-api.html#driver-vs-runtime-api)

[How to Check CUDA Version on Ubuntu 18.04](https://varhowto.com/check-cuda-version-ubuntu-18-04/)

作者：AIChiNiurou
链接：https://www.zhihu.com/question/378419173/answer/1153666140
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

答案如下，这里其实是 driver 里面带的 cuda，也就是你图形驱动程序 441.87，你安装的 pytorch 基于这个 drive cuda 使用的所以为 TrueCUDA 有两个主要的 API：runtime (运行时) API 和 driver API。这两个 API 都有对应的 CUDA 版本（如 9.2 和 10.0 等）。用于支持 driver API 的必要文件 (如 libcuda.so) 是由 GPU driver installer 安装的。nvidia-smi 就属于这一类 API。用于支持 runtime API 的必要文件 (如 libcudart.so 以及 nvcc) 是由 CUDA Toolkit installer 安装的。（CUDA Toolkit Installer 有时可能会集成了 GPU driver Installer）。nvcc 是与 CUDA Toolkit 一起安装的 CUDA compiler-driver tool，它只知道它自身构建时的 CUDA runtime 版本。它不知道安装了什么版本的 GPU driver，甚至不知道是否安装了 GPU driver。综上，如果 driver API 和 runtime API 的 CUDA 版本不一致可能是因为你使用的是单独的 GPU driver installer，而不是 CUDA Toolkit installer 里的 GPU driver installer

为了防止冲突，pytorch 安装 自带 cuda runtime。

nvidia-smi 能显示就表明 GPU driver 已经安装好了

CUDA Driver 是跟硬件打交道的最底层，CUDA Toolkit 是依赖于 Driver 的，所以 CUDA Toolkit 的版本必须与 CUDA Driver 的版本相适配。
说得再详细点，那就是一旦 CUDA Driver 版本定了，比如我电脑上目前是 460.80, 那么根据 <https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#major-components> 中的 Table 3. CUDA Toolkit and Corresponding Driver Versions, 我的 CUDA Toolkit 最多只能安装到 CUDA 11.2.2 Update 2, 再往上 CUDA 11.3 之类的就不行了，如果要装更新版本的 CUDA Toolkit 必须安装更高版本的 CUDA Driver。

最新的 CUDA Toolkit 也许没法安装在老的 CUDA Driver 上面，但是最新的 CUDA Driver 一定能够支持老的 CUDA Toolkit。因为我们的代码是依赖于 CUDA Toolkit 的，所以新的 GPU Driver 是没问题的。


```shell
$ /usr/local/cuda/bin/nvcc --version
```
返回的版本是 11.1

为什么 nvidia-smi 和 nvcc 看到的 CUDA Version 会不一样，比如我目前 nvidia-smi 看到的是 11.2，而 nvcc 看到的是 11.1

NVCC 的全称是 Nvidia CUDA Compiler，所以它是 CUDA Toolkit 的一部分

为什么 nvidia-smi 和 nvcc 看到的 CUDA Version 会不一样?
[stackoverflow](https://stackoverflow.com/questions/53422407/different-cuda-versions-shown-by-nvcc-and-nvidia-smi) 上的一个解释如下：

CUDA 有两个主要的 API：runtime (运行时) API 和 driver API。这两个 API 都有对应的 CUDA 版本（如 9.2 和 10.0 等）。

用于支持 driver API 的必要文件 (如 libcuda.so) 是由 GPU driver installer 安装的。nvidia-smi 就属于这一类 API。
用于支持 runtime API 的必要文件 (如 libcudart.so 以及 nvcc) 是由 CUDA Toolkit installer 安装的。（官方的 CUDA Toolkit Installer 会集成 GPU driver Installer）。nvcc 是与 CUDA Toolkit 一起安装的 CUDA compiler-driver tool，它只知道它自身构建时的 CUDA runtime 版本。它不知道安装了什么版本的 GPU driver，甚至不知道是否安装了 GPU driver。
综上，如果 driver API 和 runtime API 的 CUDA 版本不一致可能是因为你使用的是单独的 GPU driver installer，而不是 CUDA Toolkit installer 里的 GPU driver installer。

所以我感觉看 driver 版本以 nvidia-smi 为准，看 toolkit 版本以 nvcc 为准

下图很清楚的展示前面提到的各种概念之间的关系，其中 runtime 和 driver API 在很多情况非常相似，也就是说用起来的效果是等价的，但是你不能混合使用这两个 API，因为二者是互斥的。也就是说在开发过程中，你只能选择其中一种 API。简单理解二者的区别就是：runtime 是更高级的封装，开发人员用起来更方便ß，而 driver API 更接近底层，速度可能会更快。

要看你說的是哪個 CUDA 版本。nvcc --version 和 nvidia-smi 的版本可能會不一樣，前者是 runtime api 對應的版本，後者是 driver api 的版本。通常需要和 runtime 的版本匹配，driver 的版本可以向下兼容。
