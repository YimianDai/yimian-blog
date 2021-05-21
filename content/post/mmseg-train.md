---
title: MMSegmentation 训练流程
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

1. Level 0: 运行 Shell 命令: `python tools/train.py ${CONFIG_FILE [optional arguments]()`
2. Level 1: 在 tools/train.py 内: 
   1. 读取各种 config: `cfg = Config.fromfile(args.config)`
   2. 创建 model: `model = build_segmentor(cfg.model, train_cfg, test_cfg)`
   3. 创建 training dataset: `datasets = [build_dataset(cfg.data.train)]()`
   4. 创建 validation dataset: `datasets.append(build_dataset(val_dataset))`
   5. 将 model, data, config 喂给训练函数: `train_segmentor(model, datasets, cfg)`
3. Level 2: 转进到 `mmseg.apis` 模块的 `train_segmentor` 函数内:   
   1. 创建 dataloader: `data_loaders = [build]()_dataloader(dataset, config)]`
   2. 将 model 搬到 GPU 上去: `model = MMDataParallel(model.cuda(), cfg)`
   3. 创建 optimizer: `optimizer = build_optimizer(model, cfg)`
   4. 创建 runner: `runner = build_runner(model, cfg, optimizer)`
   5. 给 runner 注册 training hooks: `runner.register_training_hooks(cfg)`
   6. 给 runner 注册 validation hooks: `runner.register_hook(eval_hook(val_dataloader, eval_cfg))`
      + 这个 `eval_hook` 是 `EvalHook` 类实例, 其重写了 `after_train_iter` 和 `after_train_epoch` 两个方法, 在 `IterBasedRunner` 中用的是 `after_train_iter`。
   7. 开始训练 `runner.run(data_loaders, cfg.workflow)`
4. Level 3: 转进到 `mmcv/runner/iter_based_runner.py` 内的 `IterBasedRunner` 类的 `run` 函数内部:  
   1. Training 模式, `mode = 'train', i = 0`, 运行 `iter_runner(iter_loaders[i](), **kwargs)`
      + 实质上是在运行 `IterBasedRunner` 类的 `train` 函数: `train(iter_loaders[0](), **kwargs)`
      + 从 `while self.iter < self._max_iters:` 可以看到, 这个 `train` 函数一共会被调用 `self._max_iters` 次
      + 从中也可以看到这个 `train` 函数其实只负责做一个 batch 数据的 forward 计算
    2. Validation 模式, 此处其实没有运行
      + `mmseg` 的所有 setting 都是 `workflow = [('train', 1)]`
      + 实际上的 validation 是通过在 `after_train_epoch` 节点调用 `EvalHook` 对象的 `after_train_iter` 方法实现的。
5. Level 4: 转进到 `IterBasedRunner` 类的 `train` 函数内部
   1. 读取一个 batch 的数据: `data_batch = next(data_loader)`
   2. 调用 model 的 `train_step` 函数计算 `loss: outputs = self.model.train_step(data_batch)`
   3. 尝试选择性进行 validation：`self.call_hook('after_train_iter')` 
      + 实质上是调用 `EvalHook` 类实例的 `after_train_iter` 函数;
6. Level 5: 转进到 `EvalHook` 类实例的 `after_train_iter` 函数内部: 
   1. 如果当前迭代数不能够被 `interval` 整除, 就不做 validation: `if not self.every_n_iters(runner, self.interval): return`
   2. 如果能被整除, 计算一下 validation set 上的结果: `results = single_gpu_test(model, dataloader)`
      + 这一步就是 `enumerate` 一下 `data_loader`, 对于每个 batch 都用 model forward 一下, 把 result 都 append 起来得到一个 list `results`, 就不再展开了
   3. 对于分割结果再调用 dataset 的 `evaluate` 函数计算一下 mIoU, mDice, mFscore 等 metric 数值    
      + 其实就是通过调用下 `mmseg.core` 里面的 `eval_metrics` 函数调用 `total_intersect_and_union` 函数计算下上述数值
