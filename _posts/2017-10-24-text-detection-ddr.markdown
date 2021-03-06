---
layout:     post
title:      "论文阅读与实现--DDR"
subtitle:   "Deep direct regression for multi-oriented scene text detection"
description:	"场景文本检测，ddr，Deep direct regression for multi-oriented scene text detection，scene text detection"
date:       2017-10-24 10:00:00
author:     "slade"
header-img: "img/post-bg-001.jpg"
catalog: true
tags:
    - paper
    - deep learning
---

# 简介

论文地址：[arxiv](https://arxiv.org/pdf/1703.08289.pdf)  
这篇论文首创性的提出了一种直接回归的方法进行场景文本检测。相较于之前的方法，比如那些基于Faster RCNN的方法，一般来说都只能检测水平文本，比如说之前博客里介绍过的[CTPN](http://slade-ruan.me/2017/10/22/text-detection-ctpn/)。当然也有不少算法进行了改进，用RRPN，RROI之类的技巧来实现倾斜文本的检测。但是这些方法的pipeline都很复杂，效果也一般。  
为了解决倾斜场景文本的检测，作者提出了将现有检测方法分类间接回归和直接回归两大类。下图中左边的图就是间接回归的示意图，所谓间接回归，意思就是网络预测的不是直接的bounding box，而是首先需要提proposal，然后预测的是和这个proposal之间的偏移距离。右图是直接回归的示意图，直接回归直接从每一个像素点回归出这个文本的四个角点。他们的方法在ICDAR15竞赛上取得了83.75%的F-measure，在目前都还算是非常不错的结果。  
![img1](http://slade-ruan.me/img/in-post/ddr/in-post-1024-01.png)

------


# pipeline

整个算法的整体框架如下图所示：  
![img2](http://slade-ruan.me/img/in-post/ddr/in-post-1024-02.png)
整个算法由四个部分组成：特征提取、特征融合、multi-task learning和后处理。
- 特征提取部分，作者认为文本的特征不如通用物体那样复杂，所以不需要太多的参数。因此他们在VGG16的基础上进行了改造，删减了几个卷积层，同时为了增加感受野，增加了一层max pool。
- 特征融合部分的主要目的有两个，首先是高层的感受野大，可以检测大文本，底层的可以检测小文本，所以需要进行融合。第二点也是最重要的，这个模型的multi-task learning部分的classification可以看做是一个语义分割的问题，而回归的时候文本边缘的纹理信息之类的底层信息对于精确的回归出文本的bounding box又是非常重要的，因此非常有必要融合高低层的特征，这样整个网络既知道哪一块是文本，又能精确的回归出文本的bounding box。
- multi-task learning主要有两个任务，分类和回归，分类是为了确认图像中的哪些区域是文本区域。然后回归部分就在这个基础上，从这些文本区域的像素上直接回归出当前文本的bounding box的四个角点。
- 后处理部分很简单，可以使用标准的非极大值抑制（NMS），文章中为了提高召回率，提出了一个改进的NMS，叫做recalled nms。想法很简单，但是提升也是比较明显的，性价比很高。  
整个网络的结构见下图：
![img3](http://slade-ruan.me/img/in-post/ddr/in-post-1024-03.png)

------

# 一些细节

## multi-task learning

在分类任务中，和一般的语义分割问题不同，文本的边缘可能是不明显的。因此标注的时候不能和通常的语义分割问题一样标注。为了不让网络confuse，作者没有把text region中的所有像素点都作为正样本，而是多加了一个dont care的过渡区域。只有和text center line的距离小于一定值的那些像素点才会被认为是正样本，而text region中的其他像素就认为是dont care，最后对分类不起作用。除此之外，作者还对训练集中文本的大小做了一个限制，只有文本行的高度落在一个范围内的那些样本才认为是正样本，因为太大或者太小的文本网络都是学不到的。太小的文本在经过几次max pool之后，在高层可能完全看不到了。而太大的文本，网络学习到的可能就是笔画之类的结构特征，而不是完整文本的特征。具体的示意可以看下图：  
![img4](http://slade-ruan.me/img/in-post/ddr/in-post-1024-04.png)
左图是分类的真值，右图是回归的真值。回归的真值就是当前像素点离四个角点的x，y距离，所以一共回归8个值。

## loss function

总的loss如下：  
![img5](http://slade-ruan.me/img/in-post/ddr/in-post-1024-05.png)
分类使用的loss是hinge loss，这个没什么好介绍的  
![img6](http://slade-ruan.me/img/in-post/ddr/in-post-1024-06.png)
回归的时候作者用了一个scale&shift的模块。想法很简单，就是做一下归一化，因为要直接回归出当前像素点到四个角点的真实距离的话，这个值的变化是非常恐怖的，所以要进行一下归一化。具体来说就是这样  
![img7](http://slade-ruan.me/img/in-post/ddr/in-post-1024-07.png)
然后回归用的loss是L1 loss，这个对离群点不敏感，很faster rcnn中用的是一样的，具体如下：  
![img8](http://slade-ruan.me/img/in-post/ddr/in-post-1024-08.png)

## Recalled NMS

Recalled nms有三步：  
- 首先做一次标准的nms
- 然后把第一步中得到的框映射到未做nms之前和这个框IoU最大的那个框。
- 最后合并的奥的所有框。
下图分别是三步的结果：  
![img9](http://slade-ruan.me/img/in-post/ddr/in-post-1024-09.png)

------

# 总结

作者创新性的提出了直接回归的这个方法，将文字检测变成一个语义分割+回归的方法。（也就是fcn based）这种方法对于倾斜文本的检测是有效的。除此之外整体pipeline非常简洁。不过有一点就是文章中test的阶段，用了一个multi-scale的sliding window的策略，计算量大大增加。当总体来说，这个方法还是非常优雅的，结果很很赞。


