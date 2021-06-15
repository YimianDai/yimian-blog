---
title: Programming Style for Deep Learning
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

在 __init__ 中，如果传入的变量只在 _init__ 内会被用，在其他函数内不会被用，那么就不需要将这个变量变成 self.变量名。这里这个其他函数是指不会在 __init__ 下被调用的函数, 如果一个函数尽在 __init__ 下被调用，那还可以视作 __init__ 的一部分，对只会用于该函数而不会用于其他不被 __init__ 调用的函数的参数来说，是没有必要将其变成 self.变量的，仅需要作为形参/实参传入即可。

取名一开始就设计好，特别是 一些模型训练好后，如果后面 类名、函数名 改了，那训练好的模型参数就不能加载了，就白训练了, 所以取名一开始就要取好，否则事后还要重新训练