---
title: matplotlib
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

![](https://zhuanlan.zhihu.com/p/139052035)

画图流程

![](https://raw.githubusercontent.com/YimianDai/imgbed/master/blog/matplotlib/pyplot-flow.jpeg)

figure 画布 -> axes / subplot 子图

matplotlib cheatsheets
![](https://github.com/matplotlib/cheatsheets)

matplotlib绘图库的操作是通过API实现的，一种操作方法是类似MATLAB的函数接口的API；另一种操作方法是面向对象的API。这两种API可以并行使用，不过函数接口的API的易用性明显好于面向对象的API; 入门篇主要使用函数接口的API，精进和演练篇主要使用面向对象的API

`matplotlib` 库的图表组成元素

`matploblib` 库的图形内容的基本操作

NumPy是matplotlib库的基础，也就是说，matplotlib库是建立在NumPy基础之上的Python绘图库

* figure 画布
  + Axes 实例
    - 坐标轴 Axes.xaxis, Axes.yaxis
      * 刻度线
      * 刻度标签
      * 刻度线定位器
      * 刻度标签格式器
    - 刻度
    - 标签
    - 线
    - 标记

快速绘图模块 `pyplot`, 提供了绘图的 API, 类似 MATLAB 函数接口的 API

##### 函数 `plot()` 展现变量的趋势变化

`plt.plot(x,y,ls="-",lw=2,label="plot figure")`

+ `x`: x 轴上的数值。
+ `y`: y轴上的数值。
+ `ls`：折线图的线条风格。
+ `lw`：折线图的线条宽度。
+ `label`：标记图形内容的标签文本。

##### 函数 `scatter()`: 寻找变量之间的关系

`plt.scatter(x,y1,c="b",label="scatter figure")`

+ `x`: x 轴上的数值。
+ `y`: y轴上的数值。
+ `c`：散点图中的标记的颜色。
+ `label`：标记图形内容的标签文本。

##### 函数 `xlim()` 设置x轴的数值显示范围

调用签名：`plt.xlim(xmin, xmax)`

+ `xmin`: x轴上的最小值。
+ `xmax`: x轴上的最大值。

##### 函数 `xlabel()`——设置x轴的标签文本

调用签名：`plt.xlabel(string)`

+ `string`：标签文本内容。

##### 函数 `grid()`——绘制刻度线的网格线

调用签名：`plt.grid(linestyle=":",color="r")`

+ `linestyle`：网格线的线条风格。
+ `color`：网格线的线条颜色。

##### 函数 `axhline()`——绘制平行于x轴的水平参考线

`plt.axhline(y=0.0,c="r",ls="--",lw=2)`

+ `y`：水平参考线的出发点。
+ `c`：参考线的线条颜色。
+ `ls`：参考线的线条风格。
+ `lw`：参考线的线条宽度。

相应的, 平行于 y 轴的是 `axvline()`

##### 函数 `axvspan()`——绘制垂直于x轴的参考区域

`plt.axvspan(xmin=1.0, xmax=2.0, facecolor="y", alpha=0.3)`

+ `xmin`：参考区域的起始位置
+ `xmax`：参考区域的终止位置
+ `facecolor`：参考区域的填充颜色
+ `alpha`：参考区域的填充颜色的透明度

相应的, 垂直于 y 轴的是 `axhspan()`

##### 函数 `annotate()`——添加图形内容细节的指向型注释文本

`plt.annotate(string, xy=(np.pi/2, 1.0), xytext=((np.pi/2)+0.15,1.5),weight="bold", color="b", arrowprops=dict(arrowstyle="->",connectionstyle="arc3", color="b"))`

+ `string`：图形内容的注释文本。
+ `xy`：被注释图形内容的位置坐标
+ `xytext`：注释文本的位置坐标
+ `weight`：注释文本的字体粗细风格
+ `color`：注释文本的字体颜色
+ `arrowprops`：指示被注释内容的箭头的属性字典

##### 函数 `text()`——添加图形内容细节的无指向型注释文本

`plt.text(x,y,string,weight="bold",color="b")`

+ `x`：注释文本内容所在位置的横坐标
+ `y`：注释文本内容所在位置的纵坐标
+ `string`：注释文本内容
+ `weight`：注释文本内容的粗细风格
+ `color`：注释文本内容的字体颜色

##### 函数 `title()`——添加图形内容的标题

`plt.title(string)`

`string`：图形内容的标题文本

##### 函数 `legend()`——标示不同图形的文本标签图例

`plt.legend(loc="lower left")`

`loc`：图例在图中的地理位置。