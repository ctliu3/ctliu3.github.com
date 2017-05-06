---
title: Implement style transfer in Pytorch
date: 2017-04-16 09:19:22
categories: ML
tags: [neural-style, pytorch, deep-learning]
---

Prisma 在去年异常火爆，它能将艺术图片（如梵高的「星空」）与任意图片融合，生成一张同时带有两者风格的图片。专业点的说法，这叫 Style transfer（风格迁移）。其实第一篇风格化相关的文章 2015 年就有了，但由于它把训练和预测放一起了，导致无法做到实时。当然，这个问题后面几个月就被解决了，也就有了再后来的 Prisma。在这篇文章里，只介绍 style transfer 的主要思路，Pytorch 实现的代码可以查看[这里](https://github.com/ctliu3/neural-style)。

<!--more-->

### Style Transfer
对于一个图像分类网络，输入是图像，输出是该图像属于每个类别的概率。而训练风格迁移，输入是艺术图和待风格化的图，输出是两者的融合，即风格化后的图。虽然我拿它跟图像分类作对比，但除了都使用 CNN 来提取特征，它们并不具有可比性。具体流程可以看下面的流程图


{% img full-image /Implement-style-transfer-in-Pytorch/neural-style-flow.jpg 600 150 '""' '"风格迁移流程图"'%}

* 风络迁移网络[1]使用中间层的 feature map 来计算 loss。如果想让保留更多原图的信息，靠近输入的层可以设置较高的权重。论文中使用的是 `vgg19`，全连接层不需要用到。
* 使用的网络是在 imagenet 等数据集上预训练好的，网络的参数在风格化训练过程中并不更新，而是更新上图中的输入 `X`. `X` 是一幅初始图片，每一次反向传播都会更改 `X` 的像素值。为了让训练更快、更好地逼进解，可以使用待风格化的图片作为初始图片。这种利用 image graident 来更新原图的方式也可以用来生成 [fool image](http://karpathy.github.io/2015/03/30/breaking-convnets/)。
* Loss 的计算上，`X` 与艺术图和原图的 loss 上有些区别。我没实验过，但只与原图的高层特征求 loss，应该是为了让结果更接近艺术图。加上如果使用原始图片作为初始 `X` 的话，只对高层特征求 loss 确实是合理的。 

### Make it faster
每个图像都要经过多轮迭代才能得到风格化的图片，这种方法显然实用性不强。为了解决这个问题，[2] 修改了原来的训练流程：使用两个网络，一个网络用来学习风格化的参数，我们把它称为 `image transform net`，一个网络用来计算 loss，我们称之为 `loss net`。`loss net` 主要用来计算损失，因此不更新参数。`image transform net` 的输出即是最后风格化后的图。[2] 使用了如下的网络结构

``` python
class ImageTransformNet(nn.Module):

    def forward(self, x):
        x = F.relu(self.bn1(self.conv1(x)))
        x = F.relu(self.bn2(self.conv2(x)))
        x = F.relu(self.bn3(self.conv3(x)))
        x = self.res1(x) # residual block
        x = self.res2(x)
        x = self.res3(x)
        x = self.res4(x)
        x = self.res5(x)
        x = F.relu(self.bn4(self.conv4(x)))
        x = F.relu(self.bn5(self.conv5(x)))
        x = self.conv6(x)
        return x
```

### Experience with Pytorch
其实写这个项目的主要一个原因是为了熟悉 Pytorch。这很大程度源于看了 Andrej Karpathy 的一句[推文](https://twitter.com/PyTorch/status/829539819709624321/photo/1)。
为了吸引更多的人使用，不管是 pytorch 还是 Caffe, MXNet 等框架都在极力降低入坑的门槛，更简洁的 API，更丰富的预训练好的模型。以致发展到现在，各家的接口，在某种程度上说，长得都差不多，特别是 python 的接口。因此到最后，还是回归到架构的设计上，哪个更为灵活，使用的显存更少，更好的支持分布式训练等等。从这段时间的使用情况来看，pytorch

* 社区并不活跃，[discuss](https://discuss.pytorch.org/) 上的帖子并不多，回复得比较积极的似乎总是那几个人。
* 接口非常易用，文档相比于 Caffe, MXNet 也算完善，但离 Tensorflow 还有些距离。不过一般的问题跟跟源码或是在论坛上基本都能找到解答。
* 官方主打的动态图模型。我没在其它框架上写过定制化特别强的复杂网络，因此不好比较。在使用 pytorch 过程中，我确实感觉到了一些便利：定义新的 layer/loss，在 forward 过程中随意的把中间结果拿出来，放在其他网络中。

### Reference
[[1]](http://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Gatys_Image_Style_Transfer_CVPR_2016_paper.pdf): Gatys L A, Ecker A S, Bethge M, Image Style Transfer Using Convolutional Neural Networks, CVPR 2016

[[2]](https://arxiv.org/abs/1603.08155): Johnson J, Alahi A, Fei-Fei L., Perceptual Losses for Real-Time Style Transfer and Super-Resolution, ECCV 2016
