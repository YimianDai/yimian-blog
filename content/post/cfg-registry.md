---
title: OpenMMLab 的 cfg 模式和 Registry 机制
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

- [cfg 模式](#cfg-模式)
- [Registry 机制](#registry-机制)
  - [模块类的全局注册](#模块类的全局注册)
  - [字符串到模块类实例的映射](#字符串到模块类实例的映射)

[OpenMMLab 的 Hook 机制](https://yimiandai.me/post/mmlab-hook/) 一文已经阐述过 Hook 机制, 但是对于 OpenMMLab 框架本身来说, 采用 Hook 机制其实最主要是为了减少代码的重复性, 提升整体框架的模块化水平.
而居于 OpenMMLab 框架核心地位的则是本文将要阐述的 cfg 模式和 Registry 机制, 其分别对应了 MMCV 中两个非常核心的基础类： `Registry` 和 `Config`, 外加两者之间的桥梁 `build_from_cfg` 函数.

## cfg 模式

在相当多的开源代码和工具箱中, 是以传入**命令行参数**的形式来实现对训练过程的不同配置. 具体而言, 是采用 `argparse` 模块在 `train.py` 中实现对命令行参数的定义, 赋默认值和解析, 大体的流程如下:

```Python
# 导入模块
import argparse

# 创建解析对象
parser = argparse.ArgumentParser()

# 添加命令行参数及默认值
parser.add_argument('--model', type=str, default='fcn',
                    help='model name (default: fcn)')

# 完成解析
args = parser.parse_args()
```

这种做法存在一些不足.
1. 首先, 是所有的可选参数罗列位于同一层, 对于训练过程中会同时用到的不同模块类, 如果具有相同名称的形参, 则需要在各自的可选参数前面加上相应的前缀以示区分.
2. 此外, 这种扁平化的参数罗列的可读性并不是很好, 很难将参数和具体的类对应起来, 需要看 help 或者看被赋予了某类的哪个形参.
3. 最后, 由于所有命令行参数的定义 (包括赋默认值) 和解析都是在 `train.py` 中，导致 `train.py` 与模型高度绑定，传入参数定义不同的模型就需要不同的 `train.py`. 例如, 以 GluonCV 为例, 在 `gluoncv/scripts/detection` 中，CenterNet, Faster R-CNN, SSD, YOLO 等各有自己的 `train.py`, 相当多的代码存在重复.

与上面相对的, OpenMMLab 系列的一大特点是其所采用的 cfg 模式, 即所有参数设置都是写在一个**配置文件**中, 如 `mmseg` 中的 `configs/fcn/fcn_r50-d8_512x512_20k_voc12aug.py`.
对于默认采用的 `py` 格式, 其具体的参数设置是以一个个字典的形式存在，而且字典还可以嵌套, 如下所示:

```Python
runner = dict(type='IterBasedRunner', max_iters=40000)
model = dict(
    type='EncoderDecoder',
    backbone=dict(
        type='ResNetV1c',
        # ... 省略
        num_stages=4),
    neck=dict(
        type='FPN',
        # ... 省略
        num_outs=4),
    # ... 省略
)
```

此外, 这个配置文件可以通过继承更加基础的配置文件而来, 并通过对部分参数设置的重写 (override) 来扩展功能, 如:

```Python
_base_ = [
    '../_base_/models/fcn_r50-d8.py', '../_base_/datasets/pascal_voc12_aug.py',
    '../_base_/default_runtime.py', '../_base_/schedules/schedule_20k.py'
]
model = dict(
    decode_head=dict(num_classes=21), auxiliary_head=dict(num_classes=21))
```

与之配合, 在 `train.py` 中采用 `Config` 类的 `fromfile` 来**解析配置文件**, 得到 Config 类实例 `cfg`:

```Python
cfg = Config.fromfile(args.config)
```

采用这种 cfg 模式的好处在于:
1. 可读性好, 同属一个字典的所有参数都是 key 为 `type` 的 value 的那个类的参数, 比如在 `backbone` 这个 `dict` 中, `depth`, `num_stages`, `out_indices` 等都是 `type` 的 value 即 `ResNetV1c` 的参数.
2. 字典的嵌套结构, 使得分属不同类的参数可以拥有相同的参数名, 如 `decode_head` 和 `auxiliary_head` 都可以有名为 `in_channels` 的参数.
3. 由于将参数定义在配置文件中，`train.py` 只需负责解析, 即将其转化为 `Config` 类实例，解耦了模型参数的定义和解析，使得 `train.py` 与具体模型的究竟会包含哪些参数无关，从而使得 `train.py` 可以为多个模型所复用.
4. 如果比较 GluonCV `scripts/segmentation/train.py` 和 MMSegmentation 中的配置文件, 可以看到后者的可选参数远远多于前者, 暴露出更多的可选项, 这也体现了 OpenMMLab 更加灵活, 定制化水平更高的原因. 最典型的就是 `IterBasedRunner` 还是 `EpochBasedRunner` 这样都可以选择, 而这些在 GluonCV 中都是写死在 `train.py` 中的, 并不是可选项.

最后, 得到的 `cfg` 为 `Config` 类实例, 支持以**属性**的方式获取参数值. 原本字典的 key 变成了 Config 实例的属性名称, 而原本字典的 value 变成了 Config 实例的属性值, 且支持逐层嵌套访问属性值, 如

```Python
cfg = Config(dict(a=1, b=dict(b1=[0, 1])))

# 可以通过 .属性方式访问，支持嵌套访问
cfg.b.b1 # [0, 1]
```

## Registry 机制

`mmcv` 中 `Registry` 类的定义很简单, 在此基础上, 我又浓缩了一下, 大体如下:

```Python
class Registry:

    def __init__():
        self._module_dict = dict()

    def build():
        build_from_cfg(cfg, registry)

    def register_module(self, name=None, force=False, module=None):
        self._register_module(
            module_class=module, module_name=name, force=force)
        return module

    def _register_module(self, module_class, module_name=None, force=False):
        self._module_dict[name] = module_class

```

在我的理解里, `Registry` 类 = 模块类的全局注册器 + 字符串到对应模块类实例的映射器.
因此, Registry 机制就是通过 `Registry` 类实例实现**模块类的全局注册**以及**字符串到对应模块类实例的映射**.

### 模块类的全局注册

除非具体使用哪个模块都一一写在了 `train.py` 中, 否则都需要一个**将传入的字符串参数映射成对应模块类实例**的过程.
这种映射一般都是由字典完成, 传入的字符串参数作为字典的 key, 而对应的模块类作为 value.

在 GluonCV 中, 这类字典位于 `gluoncv/data/__init__.py`, `gluoncv/model_zoo/model_zoo.py` 等文件中, 是一个开发者**手动注册**的大字典。手动注册的意思是, 一旦某个类实现好后, 比如 `resnet18_v1`, 需要先在 `model_zoo.py` 中, 导入该类, 然后再在字典中手动添加该条映射:

```Python
from .resnet import *

_models = {
    # ... 省略
    'resnet18_v1': resnet18_v1,
    # ... 省略
}
```

借助 `Registry` 类和 Python 的**装饰器**, OpenMMLab 实现了模块类的**自动注册**.
不同于上面的做法, OpenMMLab 是建立了 `DATASETS`, `BACKBONES`, `NECKS`, `HEADS`, `LOSSES` 等 `Registry` 类实例, 而在每个实例中, 都有一个名为 `self._module_dict` 的字典, 用于存储字符串到模块类之间的映射.
此外, 如之前的代码所示, `Registry` 类的 `register_module` 函数是一个装饰器 (decorator). 用法如下所示, 功能是将定义好的模块类, 如 `FCOSHead`, 添加到相应的 Registry 类实例中, 如 `HEADS`.

```Python
@HEADS.register_module()
class FCOSHead(AnchorFreeHead):
    # ... 省略
```

一般而言, 作为一个接受另一个函数 (被装饰的函数) 为参数的可调用对象, 装饰器通常是会处理被装饰的函数，然后把它返回, 抑或是将其替换成另一个函数或可调用对象。
但是, 在 OpenMMLab 中, `Registry` 类的 `register_module` 函数这个装饰器, 是会原封不动地返回被装饰的模块类 (如 `FCOSHead`), 唯一所做的便是将该类注册到 `Registry` 类的 `self._module_dict` 字典中.
其实, 这种做法也不算稀有, 很多 Python 的 Web 框架也会使用这样的装饰器把函数添加到某种中央注册处，例如把 URL 模式映射到生成 HTTP 响应的函数上的注册处[^fluent]。

此外, 装饰器的一个关键特性是，它们**在被装饰的函数定义之后立即运行**。通常, 在 `import` 相应模块时, 都会过一遍相应的定义被装饰对象的代码, 此时装饰器就已经运行了.
例如, 对于 `mmdet` 的 `AnchorGenerator`, `SSDAnchorGenerator` 这些类, 他们是在何时注册到 `ANCHOR_GENERATORS` 这个 `Registry` 类实例中的? 在 `train.py` 执行 `from mmdet.core import DistEvalHook, EvalHook` 时, 会调用并执行 `mmdet/core/__init__.py` 中的 `from .anchor import *`, 进而会调用并执行 `mmdet/core/anchor/__init__.py` 中的 `from .anchor_generator import (AnchorGenerator, LegacyAnchorGenerator, YOLOAnchorGenerator)`, 从而完成 `AnchorGenerator`, `SSDAnchorGenerator` 这些类的注册.

### 字符串到模块类实例的映射

通过 `Registry` 类, 程序在运行时可以实现从配置文件中的字符串到具体模块类实例的映射, 即在构建模型的过程中，通过配置文件字典的 key (`'type'`) 去查找对应的类（在注册器实例的 `self._module_dict` 字典中），然后将其实例化。

具体负责实例化的函数是 `build_from_cfg`, 以 `Config` 类实例 `cfg` 和 `Registry` 类实例 (如 `ANCHOR_GENERATORS`)为输入, 返回一个具体的模块类实例, 如下所示:

```Python
from mmcv.utils import Registry, build_from_cfg

ANCHOR_GENERATORS = Registry('Anchor generator')

def build_anchor_generator(cfg, default_args=None):
    return build_from_cfg(cfg, ANCHOR_GENERATORS, default_args)
```

当然也会有部分 `build_xxx` 函数并不直接调用 `build_from_cfg`, 而是调用 `Registry` 类的 `build` 方法, 如下所示. 

```Python
def build_head(cfg):
    """Build head."""
    return HEADS.build(cfg)
```

但看过上面脱水版 `Registry` 类定义的我们很清楚, 其实 `build` 方法的底层还是调用了 `build_from_cfg` 函数, 这是 **cfg 模式与 Registry 机制之间的桥梁**, 也是将字符串映射成模块类实例的关键.
为此, 我们再来个脱水版的 `build_from_cfg` 函数定义, 主要是两步.

第一步, 完成从字符串 (string, Config 类实例的 'type') 到模块类的映射:

```Python
obj_type = args.pop('type')
obj_cls = registry.get(obj_type)
```

第二步, 将该类实例化:

```Python
obj_cls(**args)
```

从 `build_from_cfg` 函数实例化模块类的 `obj_cls(**args)` 这句代码也可以看出来, 其实**配置文件中的那些选项和参数其实就是对应的模块类的参数**.
所以, 通过看配置文件, 基本也就可以了解整个模型的细节和训练流程了.

至此, cfg 模式和 Registry 机制在 OpenMMLab 系列中的工作机理也讲得差不多了, 大致是如下流程:

1. 构建 `Registry` 类实例, 如 `DATASETS`, `BACKBONES`, `NECKS`, `HEADS`, `LOSSES` (`mmdet`, `mmseg` 已为我们做好)
2. 定义具体的模块类 + 装饰器语法糖完成注册 (自己的定制过程从这一步才开始)
3. 根据相应的模块类参数, 写好具体的 Config 配置
4. 在训练程序周期中, `build_segmentor`, `build_backbone` 等函数通过底层调用 `build_from_cfg` 函数会根据 `cfg` 生成相应的模块类实例

最后, 再重复一次, 如果我们写了一个类, 并在类的声明上方添加了装饰器代码, 一定要保证该类定义会在训练程序中被 `import`, 即别忘了在相关的 `__init__.py` 文件中 `import` 该类, 这样才能保证在使用前, 自己定义的类注册成功了.

[^fluent]: 流畅的 Python, 第七章 函数装饰器与闭包