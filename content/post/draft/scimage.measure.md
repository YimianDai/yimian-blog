---
title: scimage.measure
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

skimage 的全称是 scikit-image。但实际使用的时候，例如在导入一个 Python 包的时候，不能存在连字符，所以有缩略语 skimage。skimage 是原本 SciPy（Python 科学计算包）中图像处理相关算法独立出来的 Python 包。

[API Reference for skimage](https://scikit-image.org/docs/dev/api/api.html) 罗列了 `skimage` 所有 Submodules 及其所包含的函数.



| 模块                 | 功能         | 典型函数                                      |
| -------------------- | ------------ | --------------------------------------------- |
| `color`              | 颜色空间转换 | `rgb2hsv`, `rgb2lab`                          |
| `data`               | 返回常用图像 | `camera`, `moon`                              |
| `draw`               | 画常规图形   | `ellipse`, `line`                             |
| `exposure`           | 图像增强     | `adjust_gamma`, `equalize_hist`               |
| `feature`            | 特征提取     | `hog`, `blob_dog`                             |
| `filters`            | 常用的滤波器 | `gabor`, `gaussian`                           |
| `filters.rank`       | 返回特征数值 | `entropy`, `gradient`                         |
| `future`             |              |                                               |
| `future.graph`       |              | `ncut`, `cut_threshold`                       |
| `graph`              |              | `shortest_path`                               |
| `io`                 | IO 操作      | `imread`, `imshow`                            |
| `measure`            |              | `label`, `regionprops`, `moments_hu`          |
| `metric`             |              | `hausdorff_distance`, `structural_similarity` |
| `morphology`         | 形态学运算   | `dilation`, `remove_small_objects`            |
| `registration`       | 配准         | `optical_flow_tvl1`                           |
| `restoration`        | 图像复原     | `denoise_nl_means`, `denoise_tv_bregman`      |
| `segmentation`       | 图像分割     | `active_contour`, `chan_vese`, `watershed`    |
| `transform`          | 图像变换     | `hough_line`, `resize`                        |
| `util`               | 辅助函数     | `crop`, `pad`, `img_as_uint`                  |
| `viewer`             |              |                                               |
| `viewer.canvastools` |              |                                               |
| `viewer.plugins`     |              |                                               |
| `viewer.utils`       |              |                                               |
| `viewer.widgets`     |              |                                               |


[`skimage.measure.label`](https://scikit-image.org/docs/dev/api/skimage.measure.html#label) 的作用是把图像中每一个连通域都分配一个编号. 比如下面的代码就会输出 `[0 1 2 3 4]`, 即使属于同一个 label, 不是同一个连通域就会被标记成不同的序号. `background` 赋予的数值一定会被标记成 0.

```Python
import numpy as np
from skimage import measure as skm
from matplotlib import pyplot as plt

gt_semantic_seg = np.zeros((100, 100))
gt_semantic_seg[20:40, 20:40] = 1
gt_semantic_seg[20:40, 60:80] = 1
gt_semantic_seg[60:80, 20:40] = 2
gt_semantic_seg[60:80, 60:80] = 2

gt_regions = skm.label(gt_semantic_seg, background=0)
print(np.unique(gt_regions))

fig = plt.figure()
ax = fig.add_subplot(111)
ax.imshow(gt_semantic_seg)
plt.savefig("dummy_name.png")
```

`skimage.measure.regionprops` 会返回每个 labelled image region 相当多的 measure properties, 包括但不限于 area, bbox, centroid 等等. 如果要看全部的 measure properties, 可以在 Python 解释器中 help 一下.

```Python
from skimage import measure as skm
help(skm.regionprops)
```

代码示例:

```Python
import numpy as np
from skimage import measure as skm
from matplotlib import pyplot as plt

gt_semantic_seg = np.zeros((100, 100))
gt_semantic_seg[20:40, 20:40] = 1
gt_semantic_seg[20:40, 60:80] = 1
gt_semantic_seg[60:80, 20:40] = 2
gt_semantic_seg[60:80, 60:80] = 2

gt_labels = skm.label(gt_semantic_seg, background=0)
gt_regions = skm.regionprops(gt_labels)
for props in gt_regions:
    y0, x0 = props.centroid
    print("props.centroid: (%.1f, %.1f)" % (x0,y0))
    ymin, xmin, ymax, xmax = props.bbox
    print("BBox [(%.1f, %.1f), (%.1f, %.1f)]" % (xmin, ymin, xmax, ymax))
```