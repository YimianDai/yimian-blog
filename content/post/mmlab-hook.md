---
title: OpenMMLab 的 Hook 机制
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

- [为什么要设计 Hook 机制？](#为什么要设计-hook-机制)
- [Hook 机制的工作流程](#hook-机制的工作流程)
- [Hook 机制的底层实现](#hook-机制的底层实现)
- [示例：`mmseg` 中的 Hooks](#示例mmseg-中的-hooks)

## 为什么要设计 Hook 机制？

<!-- 使用 Hook 进行开发的最大好处是可以无侵入的扩展功能, 从而实现对算法训练、测试进行了抽象和解耦 -->

OpenMMLab 系列的一大特色是其所采用的 Hook 机制。
同样作为计算机视觉算法框架, 相比于简单易懂、对新手友好的 Gluon-CV, Hook 机制无疑提高了初学者入门 OpenMMLab 系列工具箱的难度。
一个很自然的问题就是，为什么要引入 Hook 机制？

事实上，OpenMMLab 中的 Hook 机制其实是**面向切面编程 (Aspect Oriented Program, AOP)** 编程思想的一种体现。在抽象层面上，问为什么要引入 Hook 机制，其实就是在问在软件开发中，为什么要采用面向切面编程这种设计模式？

我个人粗浅的理解，之所以提出面向切面编程，是为了解决面向对象编程 (Object Oriented Programming，OOP) 代码重复性的问题。
面向对象编程的思想是职责分配，将功能分散到不同的对象类中，在不同的类里设计不同的方法。
如果两个类 A、B 都需要用到同一种方法，那么可以将该方法写在一个独立的类 C 中，然后那两个类各自继承这个类 C。
然而这么做有两个问题。
第一个问题在于，对于不具有多继承特性的语言，比如 Java，继承了类 C 就不能继承其他类了，如果类 C 中的功能并不是类 A、B 的主要功能，那通过继承类 C 来获取其方法就行不通了。简单粗暴的解决方案是各自在类 A、B 中实现类 C 中的那个子函数，这样的话，一模一样的代码就存在于两处，代码的重复性大大增加。如果你要修改该方法，也必须两处都修改。两处还可以 handle，如果这种情况有 m 个，每个重复 n 处呢？
第二个问题在于，即使能够继承，类 A、B 就和类 C 耦合在一起了，如果存在一个跟类 C 具有相似但不同子函数的类 D，我希望能够让类 A、B 通过用户配置选项动态地选择是调用类 C 还是类 D 中的子函数，那么这种直接继承的方案也没法提供这种动态选择的灵活性。
本质上，除了继承之外，面向对象编程所追求的封装特性斩断了类与类之间的联系和共享，而为了降低代码的重复性、**提升软件的模块化水平**，需要将分散在各个类内的重复代码统一起来，两者之间就存在了矛盾。

这种**在程序运行时，动态地将所需要的代码切入到类的指定方法、指定位置上的编程思想**就是**面向切面编程**。
其中，几个类共同需要调用的、为此被抽取出来的代码片段叫作**切面**，其会在程序运行时被切入到指定类的指定方法中，而被切入的那些类、那些方法叫作切入点。
面向切面编程，使得我们可以把和当前的业务逻辑无关的部分抽到单独的一层中去，实现无侵入式的功能扩展。

正是借由 Hook 机制，OpenMMLab 系列能够对网络实现以及算法训练、测试流程进行抽象和解耦，从而达到了相当高度的模块化水平。

## Hook 机制的工作流程

<!-- Hook 机制是怎么结合 Runner 类实现对训练过程的整个生命周期进行管理的? -->

Hook 机制, 其实并不是 OpenMMLab 的特例，只是由于我代码经验太少，第一次见而已。
钩子编程 (hooking) ，是计算机程序设计术语，指通过拦截软件模块间的函数调用、消息传递、事件传递来修改或扩展操作系统、应用程序或其他软件组件的程序执行流程。
其中，处理被拦截的函数调用、事件、消息的代码，被称为钩子 (hook) ，应该也就是前文 AOP 编程里面的切面。

在 OpenMMLab 中，Hook 机制是由 Runner 类 (比如 `IterBasedRunner`, `EpochBasedRunner`) 和 HOOK 类 (比如 `EvalHook`) 配合完成的, 共同构成一套训练框架的架构规范. 

首先, 在 OpenMMLab 中, 定义好了 Hook 类, 如下所示, 其定义了一堆对应训练流程中各个步骤/时刻/节点的钩子函数, 等待被 Runner 类实例调用, 用于完成特定的功能. 
比如, 在网络训练流程中每个或者隔几个 `after_epoch` 时刻, 可以调用想用的钩子函数从而完成对 validation set 的 evaluation.
当然, 下面这个 Hook 类是最最原始的实现, 也就是基本什么功能都没有实现. 
如果想定义一些操作, 实现一些功能，可以继承这个类并定制我们需要的功能, 比如 `mmcv.runner.hooks.evaluation` 模块中的 `EvalHook` 类继承了最最原始的 `Hook` 类, 将里面的子函数基本都具体实现了一下;
而 `mmseg.core.evaluation` 模块中的 `EvalHook` 类则进一步继承了前一个 `EvalHook` 类, 重写了 `after_train_iter` 和 `after_train_epoch` 两个子函数.

```Python
from mmcv.utils import Registry

HOOKS = Registry('hook')

class Hook:

    def before_run(self, runner):
        pass

    def after_run(self, runner):
        pass

    def before_epoch(self, runner):
        pass

    def after_epoch(self, runner):
        pass

    def before_iter(self, runner):
        pass

    def after_iter(self, runner):
        pass

    # ... 省略
```

其次, 在 OpenMMLab 的定义好的各个 Runner 类中, 事先在其运行函数 `run`, `train` 中的各个节点, 比如 `before_run`, `before_epoch`, `before_iter`, `after_iter`, `after_epoch`, `after_run` 这些, call 指定的钩子函数, 比如 `self.call_hook('after_train_iter')`. 
下面 `EpochBasedRunner` 的 `train` 函数的一部分 call hook 函数的代码:

```Python
# 省略 ...
self.call_hook('before_train_epoch')
for i, data_batch in enumerate(self.data_loader):
    # 省略 ...
    self.call_hook('before_train_iter')
    # 省略 ...
    self.call_hook('after_train_iter')
    # 省略 ...
self.call_hook('after_train_epoch')  
```
这些被 call 的钩子函数, 比如 EvalHook 类中的 `after_train_iter` 函数会进一步会调用 `mmseg.apis.test` 模块中的 `single_gpu_test` 函数计算 validation set 上的结果, 再调用 `dataset` 的 `evaluate` 函数计算出 mIoU, mDice, mFscore 等 metric 数值. 


个人感觉, 这套 Hook 机制很像通信系统里面的**轮流询问**机制.
其之所以起作用，是因为在 Runner 类的被调用方法中, 每一个节点都规定了 call 相应 hook 函数的操作.
Runner 类在训练过程中会依次轮流询问端口, 也就是依次 call 下每个节点的 hook 函数, 如果对应钩子函数有被专门定制过, 那就执行下该功能. 如果没有, 那就是个空函数, 直接 pass 了, 继续执行下一步，从而实现了拦截模块间的函数调用、消息传递、事件传递，从而修改或扩展组件的行为.


## Hook 机制的底层实现

在清楚了 Runner 类与 Hook 类配合实现 Hook 机制的工作流程后, 还剩下的问题两个问题.
第一个问题是, 怎么让 Runner 类实例知道去调用某个具体的 Hook 类实例的子函数, 也就是怎么将 Runner 类实例和 Hook 类实例关联起来?
第二个问题是, Runner 类实例可能会调用多个 Hook 对象, 每个 Hook 对象都会有各自同名的子函数, 比如 `after_train_iter`, 这种情况是如何处理的?

对于第一个问题, 是通过 Runner 类的 `register_hook` 函数将 `HOOK` 类实例注册进 Runner 类实例的. 
我们以 MMSegmentation 为例, 在训练模型的时候, 会调用 `mmseg.apis` 模块的 `train_segmentor` 函数. 
其中有两步是给 `IterBasedRunner` 类实例 `runner` 注册 training hooks 和 validation hooks:

```Python
runner.register_training_hooks(cfg)
runner.register_hook(eval_hook(val_dataloader, eval_cfg))
```

Runner 类提供了两种注册 hook 的方法: 

1. `register_hook` 方法是直接传入一个实例化的 `HOOK` 对象，并将它插入到 Runner 类实例的 `self._hooks` 列表中;
2. `register_hook_from_cfg` 方法是传入一个配置项 `cfg`，根据配置项来实例化 `HOOK` 对象, 然后再将其插入到 `self._hooks` 列表中.

其实, 第二种方法就是先调用 `mmcv.build_from_cfg` 方法生成一个实例化的 `HOOK` 对象，然后再调用第一种 `register_hook` 方法将实例化后的 `HOOK` 对象插入到 `self._hooks` 列表中。

有了存有注册了的 Hook 类实例的 `self._hooks` 列表, Runner 类在运行中调用注册了的 Hook 类实例的子函数也就顺理成章了.
看一下 `BaseRunner` 类中 `call_hook` 函数的定义, 其中 `fn_name` 就是 `self.call_hook('after_train_iter')` 传入的 `after_train_iter`. 
`getattr(hook, fn_name)(self)` 其实就是在调用 `self._hooks` 列表中的 hook 对象的名为 `fn_name` 的函数, 比如 `EvalHook` 类实例的 `after_train_iter` 方法. 
至此, 第一个问题, 如何动态地将想要的 Hook 类实例的某个方法切入到 Runner 类实例的运行过程中已经实现了.

```Python
    def call_hook(self, fn_name):
        """Call all hooks.

        Args:
            fn_name (str): The function name in each hook to be called, such as
                "before_train_epoch".
        """
        for hook in self._hooks:
            getattr(hook, fn_name)(self)
```

对于第二个问题, 从上面 `call_hook` 函数的定义也可以看出, 在 Runner 实例的 `run` 函数运行过程中, 在每一个设置 `call_hook` 函数的节点, 都会就轮流执行一遍 `self._hooks` 列表中所有 hook 实例中该时刻对应的方法.
比如, 对于 `after_train_iter` 这个时刻, 就是遍历一遍所有 hook 实例的 `after_train_iter` 方法. 
如果只有一个 Hook 实例重写了该方法, 而其他实例的该方法都是 `pass`, 那也无所谓.
但如果有两个及以上实例的该方法实现不是 `pass`, 那这就涉及到一个哪个实例的方法该先被调用的问题, 具体到程序中, 则是每个 Hook 了实例被插入到 `self._hooks` 列表的位置的前后, 因为 `call_hook` 函数是依次调用的. 

优先级这点, 在注册 hook 的时候就已经实现了, `priority` 是默认变量. 
从下面 `register_hook` 函数的定义就可以看出, 对于新注册的一个 Hook 实例, 按照其指定的优先级, 没有指定就默认 `'NORMAL'` 优先级, 插入到 `self._hooks` 中, 优先级越高的, 越靠前.
如果新注册的 Hook 实例与就有的 Hook 实例优先级相同, 那就按照先来后到, 先来的排在更前面.
至此, 第二个问题也解决了.

```Python
def register_hook(self, hook, priority='NORMAL'):
    """Register a hook into the hook list.

    The hook will be inserted into a priority queue, with the specified
    priority (See :class:`Priority` for details of priorities).
    For hooks with the same priority, they will be triggered in the same
    order as they are registered.

    Args:
        hook (:obj:`Hook`): The hook to be registered.
        priority (int or str or :obj:`Priority`): Hook priority.
            Lower value means higher priority.
    """
    assert isinstance(hook, Hook)
    if hasattr(hook, 'priority'):
        raise ValueError('"priority" is a reserved attribute for hooks')
    priority = get_priority(priority)
    hook.priority = priority
    # insert the hook to a sorted list
    inserted = False
    for i in range(len(self._hooks) - 1, -1, -1):
        if priority >= self._hooks[i].priority:
            self._hooks.insert(i + 1, hook)
            inserted = True
            break
    if not inserted:
        self._hooks.insert(0, hook)
```

## 示例：`mmseg` 中的 Hooks

在下图中，我整理了 `mmseg` 的 `tools/train.py` 整个运行周期中会用到的所有 hooks 对应的具体的 Hook 类以及相应被调用的时刻。

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/mmlab-hook/Hooks.png)

另外，以 `IterBasedRunner` 为例，整理了这些 Hooks 被调用的时刻以及相应的优先级（先后顺序）。

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/mmlab-hook/mmseg-hooks-priority.png)

