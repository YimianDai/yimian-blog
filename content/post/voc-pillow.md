---
title: Pascal VOC Dataset 与 Pillow 的调色盘
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

这篇文章记录一下我想在 MMSegmentation 的 `PascalVOCDataset` 类的基础上定制自己的数据集类的踩坑经历, 主要是回答我昨天看代码时的两个困惑:

1. `PascalVOCDataset` 是如何在没有进行显式 `colormap2label` 映射的情况下, 完成从 RGB 的 label image 向填充 0, 1, 2, 3 这种 class label 的 `gt_semantic_seg` 转换的.
2. 为什么 `PascalVOCDataset` 要设置一个数值为 255 的 `ignore_index`.

这是一张 Pascal VOC Dataset 的标记图片, 可以看到图中有 4 种颜色:

+ rgb(0, 0, 0) 代表 background 类
+ rgb(192, 128, 128) 代表 person 类
+ rgb(128, 0, 128) 代表 bottle 类
+ rgb(224, 224, 192) 代表各个物体的边缘, 并没有被分配具体的类别

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/voc-pillow/2007_000346.png)

`PascalVOCDataset` 的 `CLASSES` 与其调色盘 `PALETTE` 具体如下:

```Python
  CLASSES = ('background', 'aeroplane', 'bicycle', 'bird', 'boat', 'bottle',
              'bus', 'car', 'cat', 'chair', 'cow', 'diningtable', 'dog',
              'horse', 'motorbike', 'person', 'pottedplant', 'sheep', 'sofa',
              'train', 'tvmonitor')

  PALETTE = [[0, 0, 0], [128, 0, 0], [0, 128, 0], [128, 128, 0], [0, 0, 128],
              [128, 0, 128], [0, 128, 128], [128, 128, 128], [64, 0, 0],
              [192, 0, 0], [64, 128, 0], [192, 128, 0], [64, 0, 128],
              [192, 0, 128], [64, 128, 128], [192, 128, 128], [0, 64, 0],
              [128, 64, 0], [0, 192, 0], [128, 192, 0], [0, 64, 128]]
```

为了将 label 喂给网络, 需要做一个 `colormap2label` 映射, 将 `'background', 'aeroplane', 'bicycle', ...` 对应的 `[0, 0, 0], [128, 0, 0], [0, 128, 0], ...` 转化为 `0, 1, 2, ...` 这样的.
从 `PascalVOCDataset` (其实是其父类 `CustomDataset`) 的形参 `ignore_index` 默认值 `255` 可以知道, rgb(224, 224, 192) 代表各个物体的边缘应该被映射成 `255`, 对应的像素在计算 loss 和 metric 的时候都不会参与计算.

对于构建语义分割数据集而言, 将物体边缘设为 ignore 是非常合理的, 因为成像的关系, 物体边缘的过渡区域往往是模糊的. 如果不设置 ignore 像素, 强行将其划分为背景或者前景类, 会导致 label 中存在有大量由人为因素造成的噪音, 即不同的人对同一张图上物体边缘区域的划分结果不同, 同一个人对不同图像的物体边缘区域划分标准也充满随机性, 甚至同一个人对同一图像不同时间的划分结果也会不一致.
在训练中后期, 网络很可能会将相当多的精力投入于去拟合这些毫无意义的人为噪音.
因此, 标注 ignore 像素对于语义分割来说是相当有意义的, 这也是之前在构建数据集过程中被我所忽略的事情.

## 笨人笨办法

首先说一下我想到的 `colormap2label` 的思路, 大体就是读取彩色的 label image, 然后遍历每个像素, 将每个 RGB pair 与其在 PALETTE 中的位置索引对应起来, 知道了其 RGB 也就知道了其在 PALETTE 中的位置索引, 也就是 class label.
然后想一个可以向量化的实现方式, 避免采用循环来实现.

[DIVE INTO DEEP LEARNING](https://zh.gluon.ai/chapter_computer-vision/semantic-segmentation-and-dataset.html#%E5%9B%BE%E5%83%8F%E5%88%86%E5%89%B2%E5%92%8C%E5%AE%9E%E4%BE%8B%E5%88%86%E5%89%B2) 一书中提供了这种思路的一种实现方式. 我整理了一下代码如下所示:

1. 这种方式其实是 [128, 128, 128] 这样的 RGB 值当成了一个 256 进制的数字三位.
2. `label_img`, 也就是 `colormap`, 是一个 H x W x 3 的 RGB 图像;
3. `colormap2label` 是一个 256 ** 3 的 list, RGB 转化来的 256 进制数字的 10 进制数值是 list 的 index, 对应的 value 是 class label, 也就是 `VOC_COLORMAP` 里面的顺序.
4. `idx` 就是将 `label_img` 的 RGB 这个三元数值转成 256 进制的单个数, idx 是一个 H x W 的 2D np.array
5. `colormap2label[idx]` 这个是 np.array 的特有性质 (语法糖), 直接返回一个跟 `idx` 一样大小的 np.array, 然后里面的数值是具体的 `idx` 里面数值作为索引取出的 `colormap2label` 中的值. 


```Python
import numpy as np
import mmcv

def voc_label_indices(colormap, colormap2label):
    colormap = colormap.astype('int32')
    idx = ((colormap[:, :, 0] * 256 + colormap[:, :, 1]) * 256
           + colormap[:, :, 2])
    print("idx:", idx)
    print("len(idx):", len(idx))
    return colormap2label[idx]

def main():
    VOC_COLORMAP = [[0, 0, 0], [128, 0, 0], [0, 128, 0], [128, 128, 0],
                    [0, 0, 128], [128, 0, 128], [0, 128, 128], [128, 128, 128],
                    [64, 0, 0], [192, 0, 0], [64, 128, 0], [192, 128, 0],
                    [64, 0, 128], [192, 0, 128], [64, 128, 128], [192, 128, 128],
                    [0, 64, 0], [128, 64, 0], [0, 192, 0], [128, 192, 0],
                    [0, 64, 128]]

    colormap2label = np.zeros(256 ** 3)
    for i, colormap in enumerate(VOC_COLORMAP):
        colormap2label[(colormap[0] * 256 + colormap[1]) * 256 + colormap[2]] = i

    filename = '/data/yimian/VOCdevkit/VOC2012/SegmentationClass/2010_001577.png'
    label_img = mmcv.imread(filename)

    label = voc_label_indices(label_img, colormap2label)
    print("label:", label)
    print("label.max():", label.max())

if __name__ == '__main__':
    main()
```

不过这个代码很毛糙, 没有处理物体边缘这些 ignore 像素. 当然要做也不难, 就是将 ignore 区域的 RGB 数值加入到 `VOC_COLORMAP` 中, 那么其就是第 22 类 (索引是 21), 然后再将 label 中的 21 全部都赋值成 255 就好了.

在过 `LoadAnnotations` 这个生成 label 的 `PIPELINE` 类的时候, 我一直期盼这其有类似的代码, 但很神奇的是 MMSegmentation 的实现中并没有类似代码, 但的确把 label 给生成了. 为此我把所有相关的 `PIPELINE` 类的代码都检查了几遍, 都没有发现, 最后还是在 print 大法的帮助下, 发现 MMSegmentation 在 `LoadAnnotations` 中用了一种特别聪明的实现方式.

## 聪明人聪明办法

在 LoadAnnotations 类中, 读取 RGB 的 label 图像, 然后将其转化成 class label 的 label (也就是 `colormap2label`) 的核心代码就只有下面这一句代码, 没错就这一句代码完成了 `imread` + `colormap2label` 两个任务. 

```Python
gt_semantic_seg = mmcv.imfrombytes(
    img_bytes, flag='unchanged',
    backend=self.imdecode_backend).squeeze().astype(np.uint8)
```

其中, `self.imdecode_backend` 的值为 `'pillow'`. 如果将其值改为 'cv2', 那得到的 `gt_semantic_seg` 就是 RGB 图像对应的 H x W x 3 的 np.array 了, 而不是原来的其数值是 0, 1, 2, 3 这样的 H x W 的 label 了. 所以, 奥妙都出在 pillow 上. 如果去看 `mmcv.imfrombytes` 的底层代码, 可以发现, 在 `flag='unchanged', backend='pillow'` 的选项下, `mmcv.imfrombytes` 函数本质就是调用了 PIL (也就是 pillow) 库的 `Image.open` 函数读取了图像. 我们可以试一下直接用 `Image.open` 函数读图像, 然后打印出来, 就已经是 class label 了.

```Python
from PIL import Image
filename = '/data/yimian/VOCdevkit/VOC2012/SegmentationClass/2010_001577.png'
img = Image.open(filename)
print(list(img.getdata()))
```

为什么这么神奇呢? 其实并不神奇, 我之所以感觉神奇只不过因为我少了一个先验知识, 那就是 Pascal VOC Dataset 的 label image 的颜色并不是它自己随便指定了, 而是其选用了 pillow 的 palette (调色盘) 中的前 21 种颜色. 可以把 PIL 调色盘的前 21 种颜色打印出来看一下, 刚好跟 `PascalVOCDataset` 的 `PALETTE` 一模一样, 而 ignore 的颜色刚好是最后一个颜色, 所以 `ignore_index` 才是 255.

```Python
np.array(img.getpalette()).reshape(-1, 3)[:20,:]
# array([[  0,   0,   0],
#        [128,   0,   0],
#        [  0, 128,   0],
#        [128, 128,   0],
#        [  0,   0, 128],
#        [128,   0, 128],
#        [  0, 128, 128],
#        [128, 128, 128],
#        [ 64,   0,   0],
#        [192,   0,   0],
#        [ 64, 128,   0],
#        [192, 128,   0],
#        [ 64,   0, 128],
#        [192,   0, 128],
#        [ 64, 128, 128],
#        [192, 128, 128],
#        [  0,  64,   0],
#        [128,  64,   0],
#        [  0, 192,   0],
#        [128, 192,   0],
#        [  0,  64, 128]])
np.array(img.getpalette()).reshape(-1, 3)[-1,:]
# [224, 224, 192]
```

对于用 `Image.open` 函数读入的 label image, 其 mode 是 'P',  代表 8-bit pixels, mapped to any other mode using a color palette. 所以读入的 label 就是 color palette 上面的索引值, 也就刚好对应着 class label. 

但是要注意, 并不是所有图像用 `Image.open` 函数读取时其 mode 就是 'P', 这应该是 VOC 数据集的制作者在生成彩色的 label image 标记的时候, 将 color palette 内嵌到图像中去了, 所以 `Image.open` 函数才知道将其读取成 'P' mode. 比如我们对等待的原图读取, 其 mode 就是 'RGB', 且不存在 color palette. 实例代码如下:

```Python
filename = '/data/yimian/VOCdevkit/VOC2012/JPEGImages/2010_001577.jpg'
img = Image.open(filename)
img.mode # 输出为 'RGB'
img.getpalette() # 输出为空
```

至此, `PascalVOCDataset` 是如何在没有进行显式 `colormap2label` 映射的情况下, 实现 `colormap2label` 功能的, 以及为什么 `PascalVOCDataset` 要设置一个数值为 255 的 `ignore_index` 这两个问题已经解决了.

同时, 这也给我们后面制作自己的语义分割数据集提供了启示:

1. 背景和前景各类的颜色可以依赖于 PIL 模块的 color palette 来制定;
2. 基于 `PascalVOCDataset` 想定制自己的 Dataset 类的时候要当心, 如果你的 label image 不是如同 VOC 那样制作的, 那就需要重写一下 `LoadAnnotations` 这个类. 