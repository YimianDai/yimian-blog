---
title: 
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

Keypoint Based Object Detection: A Survey

Part II: Keypoint Detection

上次围绕着基于关键点的目标检测开了个头, 今天讲一下计划中的第二部分, 也就是关键点检测本身.
由于关键点检测是一个非常基础性的内容, 往往是作为目标检测, 人体姿态估计, 图像匹配和 3 维重建等应用的中间结果, 所以其 pipeline 也随着后端任务的不同而差异较大.

#### Concept Anatonmy

在看论文的过程中, 我发现 Keypoint Detection 大致可以分为**几何特征点** (Geometric Keypoint) 和**语义特征点** (Semantic Keypoint) 两大类.
前者主要是为图像匹配和 3 维重建所采用, 如图所示, **检测的关键点主要为具有几何意义的角点和 blob, 在图像上分布密集**.

![image-matching-van](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/image-matching-van.jpg)

![geometric-keypoints](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/geometric-keypoints.jpg)

后者主要是人体姿态估计中所采用的, 如图所示, 这些特征点主要是语义层面的, 并不一定具有几何特征, 数量也是稀疏且固定的, 如果不考虑遮挡和残疾等情况. 我今天的整理也主要围绕着这一类关键点检测.

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/human-pose-estimation.jpg)

#### 本次主旨

上一讲我没讲好, 上一讲其实想讲的重点其实是 在独立地检测到若干个关键点以后, 如何将这些关键点组装成一个物体, 而其中的重点是如何抑制虚警.

这一讲则是更前一个阶段, 即如何更好的检测关键点. 因为我是从能否照搬到目标检测的视角来看搜集资料的, 所以主要整理的是一些能够对目标检测有




1. Data
  + Label Assignment
  + Data Augmentation
2. Backbone
3. **Neck**
  + **Feature Sampling / Pooling** (4)
  + Multi-scale Feature Fusion
4. Head
5. **Loss** (2)


## Neck

### Feature Sampling / Pooling

#### ECCV-18-CornerNet: Detecting objects as paired keypoints

CornerNet 这篇文章的主旨是一个物体的边界框可以由左上和右下的两个角点来表示, 那么只要检测出角点, 然后将他们配对组合成一个物体就完成了物体检测.

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/CornerNet-Fig-2.jpg)

但采用角点存在一个问题在于, 边界框的角点往往在物体外面, 仅仅采用这一处的特征来预测, 语义信息是不够的.

为此, 作者提出了一个 corner pooling 的操作, 以左上角的角点为例, 分别向右和向下做 max pooling, 本质上是采样边界框边缘上特征相应最强的点, 然后再相加. 将本来不属于物体的点的特征 替换为 至少跟物体相切的边界框边缘上的特征, 也就是拉近了预测点特征和真实物体之间语义上的鸿沟.

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/CornerNet-Fig-3.jpg)

#### ICCV-19-CenterNet - Keypoint Triplets for Object Detection

假如我们卷积网络的有效感受野想象的很小, 极端情况退化到点, 那么上一篇 CornerNet 其实就是解决了一个角点特征语义有无的问题, 用的是边缘特征. 但物体边缘又能有多少语义呢? 真正语义丰富的, 还是要在物体内部的特征点.

在这个思想指导下, 这篇文章就往物体里面多看了一眼, 提出了 Cascade corner pooling. 还是以左上角的角点为例, 具体的流程就是

1. 先向右和向下找到响应最大的点;
2. 再从以各自的响应最大点出发, 向下和向右找到相应最大的点
3. 再把这两个点的特征相加

本质上, CenterNet 相比于 CornerNet, 是将对最大特征响应点的采样从边缘拓展到了物体内部.

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/CenterNet-Fig-4.jpg)

#### CVPR-20-CentripetalNet - Pursuing High-quality Keypoint Pairs for Object Detection



#### CVPR-21-Bottom-Up Human Pose Estimation Via Disentangled Keypoint Regression

这篇文章和前面 3 篇文章一样的地方在于, 也是要解决提供特征的关键点与需要预测的关键点之间不匹配的问题, 但方向刚好跟前

## Loss

最后是 Loss. 前面讲的这些工作其实往往采用的都是 L2 范数或者交叉熵之类的比较一般性的损失函数, 这里介绍两篇稍微不同的.

#### arXiv-20-CoKe: Localized Contrastive Learning for Robust Keypoint Detection

这篇论文, 看题目, 顾名思义就是把对比学习用到了关键点检测里面, 而且是有监督的对比学习.

对比学习, 大家听硕哥的报告应该很熟悉了, 主要是把跟自己同类的拉近, 跟自己不同类的推远.
具体到特征点检测上, 可以分为 3 点:

1. 最小化同类关键点的特征表示之间的距离
2. 最大化不同类关键点的特征表示之间的距离
3. 最大化关键点与背景点的特征表示之间的距离

这篇文章要解决的关键问题是对比学习需要计算样本与样本之间的距离, 是平方关系, 怎么把这个巨大的计算量降下来, 比如降成线性的.

这篇文章用的手段, 文章中叫作 Feature Bank, 其实跟之前周成伟报告里提到的 memory bank 很像, 是一个每次迭代都会更新的每一类 keypoint 特征表示的模板, 用一类的模板代替该类所有的实例.
我们以最小化同类关键点的特征表示之间的距离为例, 对于同属于第 k 类的所有关键点, 彼此之间的距离为

$$D_{\text {within }} = \sum_{i=1}^{N} D_{\text {within }}\left(\mathbf{f}_{k}^{i}\right)=\sum_{i=1}^{N} \sum_{j=1}^{N} d\left(\mathbf{f}_{k}^{i}, \mathbf{f}_{k}^{j}\right)$$

而用了 Feature Bank 之后, 就可以退化成线性

$$D_{\text {within }} = \sum_{i=1}^{N} \bar{D}_{\text {within }}\left(\mathbf{f}_{k}^{i}\right)=\sum_{i=1}^{N} d\left(\mathbf{f}_{k}^{i}, \theta_{k}\right)$$

其他两类距离计算量的线性化也类似.

Feature Bank 的更新如图所示:

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/Coke-Fig-2.jpg)

#### CVPR-19-Locating Objects Without Bounding Boxes

这篇文章讨论的是当物体标记完全退化为只有一个点的时候, Loss 应该如何设计. 前面介绍的所有工作虽然也是关键点检测, 但还是有实例 mask, 或者多个点构成的. 这篇文章所面对的任务的特殊性在于, 当物体标记完全退化为只有一个点, 也就没法采用 IoU 之类的指标, 预测和标记之间也就没有了是否 match 的概念, 也就是没有是否检测到物体的概念消失了.

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/keypoint-detection/WHD-Fig-1.jpg)

衡量两个点集之间距离的常用度量是 Hausdorff 距离

$d_{\mathrm{H}}(X, Y)=\max \left\{\sup _{x \in X} \inf _{y \in Y} d(x, y), \sup _{y \in Y} \inf _{x \in X} d(x, y)\right\}$

为了克服其对 outlier 敏感的问题, 之前的工作常用 average Hausdorff distance

$d_{\mathrm{AH}}(X, Y)=\frac{1}{|X|} \sum_{x \in X} \min _{y \in Y} d(x, y)+\frac{1}{|Y|} \sum_{y \in Y} \min _{x \in X} d(x, y)$

基于 average Hausdorff distance 的改进工作在深度学习流行以前还是很多的.

这篇文章要解决的关键问题是, 关键点检测网络得到的是 score map, 而非坐标点集, 即如何度量 score map 与 Groundtruth 点集之间的距离作为 Loss.

具体的技术手段其实也不新鲜, 就是用加权平均和广义均值代替 hard selection 和 min 运算符, 特别是前者, 这些年成为主流的 Soft Attention, 以及将 IoU 软化得到的 Soft-IoU Loss 都是例子.

软化后的 Weighted Hausdorff Distance

$\begin{aligned} d_{\mathrm{WH}}(p, Y)=& \frac{1}{\mathcal{S}+\epsilon} \sum_{x \in \Omega} p_{x} \min _{y \in Y} d(x, y)+\\ & \frac{1}{|Y|} \sum_{y \in Y} M_{\alpha}\left[p_{x} d(x, y)+\left(1-p_{x}\right) d_{\max }\right] \end{aligned}$

其中,
$\mathcal{S}=\sum_{x \in \Omega} p_{x}$

$\underset{a \in A}{M_{\alpha}}[f(a)]=\left(\frac{1}{|A|} \sum_{a \in A} f^{\alpha}(a)\right)^{\frac{1}{\alpha}}$

对比下 average Hausdorff distance 和 Weighted Hausdorff Distance, 软化就体现在 $\sum_{x \in X} 1 \cdot$ 转化为 $\sum_{x \in \Omega} p_{x} \cdot$ 上面. 
还有一个就是后面的用 广义均值 来代替 min, 这个 alpha 负无穷大的时候就是 min 了.

另外, 我想分享这篇文章的另一个目的是, 我在读论文的时候发现 Hausdorff 距离其实提供了一种在模型训练之前, 就可以解耦虚警和漏检, 也就是调整虚警和漏检侧重的方向.
这也是我之前提过的, 比如同样是检测红外小目标, 由于制导和预警用途的不同, 对虚警和漏检的偏好是完全相反的, 而我们现有的目标检测方法的损失函数并没有这方面的解耦.

Weighted Hausdorff Distance 的第一项其实反映的是在虚警上面的损失, 要想这一项小, 对于所有远离 y 的 x, 其概率必须小, 否则 loss 降不下来.
如果加大第一项的权重, 虚警就会下降, 当然也有代价, 就是第二项控制的漏检现象会更严重.
第二项反映的是漏检, 对于所以靠近 y 的 x, 其概率必须大, 否则 d max 这一项就小不下去, 整个 loss 也就小不下去.
如果加大第二项的权重, 漏检现象会改善就会下降, 但虚警会上升.


Keypoint Detection: From Geometric to Learnable Features

Keypoint Detection 跟哪些 application 相关?
+ Image Matching
+ Object Detection (Anchor-free Detector)
+ 3D Reconstruction (Structure From Motion)
+ Human pose estimation (2D and 3D)


points (a.k.a. keypoints or interest points)

points can be roughly classified into corner and blob

A blob feature is commonly indicated as a local closed region (e.g., with a regular shape of circle or ellipse), inside which the pixels are considered similar to one another and are distinct from the surrounding neighborhoods.

A Coarse-Fine Network for Keypoint Localization(2017,ICCV)
Joint Learning of Semantic Alignment and Object Landmark Detection(2019,ICCV)
Attentive Fashion Grammar Network for Fashion Landmark Detection and Clothing Category Classification(2018,CVPR)
Efficient grouping for keypoint detection(2020)
Key.Net: Keypoint Detection by Handcrafted and Learned CNN Filters(2019,ICCV)
Improving Landmark Localization with Semi-Supervised Learning(2018,CVPR)
Unsupervised Learning of Object Landmarks through Conditional Image Generation(2018)

仔细啃透 Yan Junchi IJCV 综述的 应该就够了
2.4.2 Deep Learning-Based Detectors
3.3.2 Deep Learning-Based Descriptors

相比单人姿态检测，由于不知道图像中每个人的位置和总人数，多人姿态检测技术在预测图片中每个人的不同关键点所在的位置时更加困难。其困难在于：不仅要定位不同种类的关键点，还要确定哪些关键点属于同一个人。(这也是 keypoint-based Object Detection 的难点)

针对这一困难，学术界有两种解决方案，一种是自顶向下的方法，先检测出人体目标框，再对框内的人体完成单人姿态检测，这种方法的优点是更准确，但开销花费也更大；另一种则是自底向上的方法，常常先用热度图检测关键点，然后再进行组合，该方法的优点是其运行效率比较高，但需要繁琐的后处理过程。

而微软亚洲研究院的研究员们认为，**回归关键点坐标的特征必须集中注意到关键点周围的区域，才能够精确回归出关键点坐标**。基于此，微软亚洲研究院提出了一种基于密集关键点坐标回归的方法：解构式关键点回归（Disentangled Keypoint Regression, DEKR）。

对于 2D 姿态估计，当下研究的多为多人姿态估计，即每张图片可能包含多个人。解决该类问题的思路通常有两种：top-down 和 bottom-up：

top-down 的思路是首先对图片进行目标检测，找出所有的人；然后将人从原图中 crop 出来，resize 后输入到网络中进行姿态估计。换言之，**top-down 是将多人姿态估计的问题转化为多个单人姿态估计的问题**。
bottom-up 的思路是首先找出图片中所有关键点，然后对关键点进行分组，从而得到一个个人。

top-down 的方法将多人姿态估计转换为单人姿态估计，那么网络的输入就是包含一个人的 bounding box，网络预测的是人的 k 个关键点坐标。对于关键点的 ground truth（对应网络的输出）如何表示有两种思路：

+ 直接对坐标进行回归，网络的输出是经过 fc 层输出的 2k 个数字
+ k 个 heatmap，即为每个关键点预测一个 heatmap 作为关键点的中间表示，heatmap 上的最大值处即对应关键点的坐标。对于改种方法，heatmap 的 ground truth 是 以关键点为中心的二维高斯分布（高斯核大小为超参）

早期的工作如 DeepPose 多为直接回归坐标，当下的工作多数以 heatmap 作为网络的输出，这种中间表示形式使得回归结果更加精确。

AE (Associative Embedding) 用于自底向上的多人姿态估计和语义分割

自底向上的多人姿态估计要解决两个问题：

预测关键点的位置
预测关键点属于哪个人

在 AE 之前，最出名的工作是 OpenPose, 可以简述为，它让网络对每个像素输出一个 offset，根据它我们计算某个关键点应该连向哪里，比如手腕 A 连接哪个手肘是正确的，也就是比较著名的 PAFmap。后续工作比如 PersonLab, 也都是在 offset 上变着花样。那么 AE 的核心观点是：不需要规定某个关节点需要输出一个固定的值，来决定它属于某个人，只需规定属于不同人的关节点，输出的值有差异即可。

作者：外婆家的橘猫
链接：https://www.zhihu.com/question/440729199/answer/1700604201
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

总结来说对于损失函数：

属于同一个人的关节点，输出的值应该相同 -> 损失函数为：当前值 - 该人的平均值。
属于不同人的关节点，输出应该不同 -> 损失函数为：（当前值 - 其他人的平均值）取负指数函数来改变单调性。