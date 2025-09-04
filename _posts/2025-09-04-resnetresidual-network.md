---
layout: post
title: "ResNet——Residual Network"
subtitle: ""
date: 2025-09-04
author: "Can"
header-img: "img/sunrise.jpg"
mathjax: true
tags: ["Paper-Reading", "ResNet", "CNN", "Computer-Vision"]
---

论文：[Deep Residual Learning for Image Recognition（CVPR 2016）](https://arxiv.org/abs/1512.03385)

## Background
![background](/img/in-post/image-amfd.png)
先讲下Resnet出现的背景，CNN卷积神经网络在图像识别任务中取得了显著的成功，在CNN中通过卷积激活池化的叠加运算可以使模型提取更深层次更高级的特征，但问题接踵而至，学习更好的网络并不像堆叠更多的层一样容易。深层网络面临着梯度消失和梯度爆炸的问题，当然这个问题可以通过适当的权重初始化 + Batch Normalization加速网络收敛。

当网络能够收敛时，新的问题出现了，那就是网络退化问题，最明显的现象就是网络的性能在增加层数时并没有显著提升，反而可能出现性能下降。在正常情况下随着网络的加深性能应该是逐渐变好的。在图中我们可以看到56层的性能比20层的差，这是不符合逻辑的。

这里还需要注意的一点是，退化问题并不是由于过拟合现象导致的。过拟合是指模型在训练数据上表现良好，但在未见过的数据上表现不佳，通常是由于模型过于复杂，学习到了训练数据中的噪声。而在图中我们可以看到在CIFAR-10的训练集和测试集上，模型通过不断迭代学习，训练错误率和测试错误率是均有下降的。

## Motivation
基于以上提到的背景，作者尝试构建一种非常深的神经网络，同时又能保持良好的训练效果和准确率。他们提出了残差学习的概念，旨在通过引入跳跃连接（shortcut connections）来缓解深度网络训练中的问题，使得网络能够更容易地学习到有效的特征表示。

## Method
![method](/img/in-post/image-wryn.png)
如图所示，可以看到该图中最显眼的便是这个identity mapping的short connection，这条路直接将输入传递给了输出，没有经过任何处理，这也是残差网络的重要之处，通过这种映射确保了在网络的某些层中，输入可以直接影响输出，帮助缓解梯度消失的问题，加快网络训练速度。

这里我们将残差定义为F（x），自身输入定义为x，函数H(x) 也就是y = F(x) + x

这里对残差块的执行流程做进一步解释：
1. 首先，输入 x 进入残差块
2. 输入x经过第一个卷积层，通常使用小卷积核（如3x3），并应用激活函数（如ReLU）
3. 输出再次经过第二个卷积层，通常也是3x3卷积，得到新的feature map
4. 在这个过程中，输入 x 会通过一个shortcut connection（快捷连接）直接与第二个卷积层的输出相加。
5. 最后，输出 y也就是H(x) 经过激活函数（如ReLU）得到最终的feature map

模型从直接拟合函数H(x)转变为拟合残差F(x)，显然拟合残差F(x)比拟合函数H(x)成本更低，模型的目标是学习输入与目标输出之间的残差，而不是直接学习目标函数本身，使得模型可以更容易地捕捉到数据中的重要特征。

通过叠加多个残差块，ResNet能够构建非常深的网络，同时保持较好的梯度流动。

最后我们对核心思想做一个解释：
传统多层网络难以拟合恒等映射，加了恒等映射后深层网络至少不会比浅层差，如果恒等映射已经最优，残差模块只需拟合零映射，即F(x)无限接近于0，使得即使加入了残差学习，输入与输出也能保持一致，即使网络学习得不够好，至少整体性能也不会显著下降。
![method-1](/img/in-post/image-uvym.png)
shortcut connections没有引入额外的参数和计算复杂度，加法操作实际并不会消耗多少时间。

这里有一个要注意的点是最后的加法运算只能在F(x)与x同维时使用,当残差分支出现下采样时，使用对short connection多出来的channel补0填充或用 1x1卷积升维来确保输入输出能够同维，但不管采取哪种匹配维度的方案，shortcut分支的第一个卷积层步长均为2。
![method-2](/img/in-post/image-oxfw.png)
接下来我们以ResNet-50模型为例介绍一下执行过程：
* 首先是对输入进行维度调整，确保输入在经过多个卷积层后仍能保持合理的尺寸和信息完整性。
* 之后是stage1 做了卷积、BN、激活、最大池化的操作提取输入特征
* 后面四个stage其实就是我们之前分析的残差块模型，
* 完成残差学习后，通过GAP全局池化将feature map的空间维度减少到1，得到一个固定长度的特征向量，将特征向量展开后执行softmax激活函数输出预测概率进行分类处理。

## Experiment
![experiment](/img/in-post/image-qgto.png)
接下来讲实验部分

如图所示,给出了VGG-19、普通的34层的CNN，34层的残差网络，可以看到最大的不同是残差网络存在实线和虚线的shortcut connection,虚线的shortcut connection意味着这里有维度调整1x1卷积操作，以保证输入输出的维度匹配。而实线的shortcut connection是identity mapping。这里存在的下采样pool使得feature map size减半 channel个数翻倍。

通过减小feature map的空间维度，网络能够更有效地聚焦于重要的特征，同时减少冗余信息 channel数的增加使得网络能够捕捉到更多的特征，增强了模型的学习能力。
![experiment-1](/img/in-post/image-mvcq.png)
上图是普通网络与残差网络的训练效果图。
![experiment-2](/img/in-post/image-uorp.png)
通过在ImageNet 2012 classiﬁcation dataset上的验证，得到了Table2的实验结果，实验也表明带残差学习的模型层数增加错误率下降，普通网络层数高的错误率反而升高，出现了网络退化问题。
![experiment-3](/img/in-post/image-wuuy.png)
以上实验表明：
* 网络退化问题不是由梯度消失导致的，实验中模型的前向/反向传播良好
* 使用更多迭代轮次并不能解决退化问题
* 数据本身决定了该类问题的上限，而模型只是逼近这个上限
* 模型结构本身决定了模型上限而训练调参只是在逼近这个上限
![experiment-4](/img/in-post/image-zkcl.png)
这里作者对shortcut connection 用恒等映射Identity mapping还是进行其他的投影处理进行了实验，如图所示。

A所有的shortcut无额外参数 升维用padding补零

B平常的shortcut用identity mapping 升维时用1x1卷积

C所有的shortcut都使用1x1卷积

显然ABC三个方案都比不带残差的plain net好

B比A好，A在升维时用padding补0 相当于丢失了shortcut分支的信息 没有进行残差学习

C比B好，C的13个非下采样残差模块的shortcut都有参数 模型表示能力强

ABC差不了太多说明identity mapping的shortcut足以解决退化问题
![experiment-5](/img/in-post/image-wqly.png)
接下来作者介绍了更深层次的网络，考虑到训练的时间，作者引入了bottleneck残差模块，相比于普通的残差模块，bottleneck使用1x1卷积先降维后升维，降维以减少计算量，升维以保证符合计算条件。

其中identity shortcuts对bottleneck尤为重要，若shortcut中若引入projection则时间复杂度、计算量、模型尺寸、参数量都会翻倍，影响bottleneck的效能。
![experiment-6](/img/in-post/image-psab.png)
通过在ImageNet上的实验我们可以得知，在single-model下Restnet比VGG网络轻量化、准确率高，Resnet-152单模型性能超过了过去所有多模型集成的结果，到了3.57%的top-5错误率。
![experiment-7](/img/in-post/image-qdal.png)
随后作者在CIFAR-10数据集上做了实验，这里的残差网络使用GAP层替代了全连接层，如图所示共有1+2n + 2n + 2n + 1共6n+2个带权重的层

从图中我们也能知道普通网络随着层数的加深，优化问题开始出现，错误率也上升,110残差网络的效果好，但一味加深网络层数并不能提高性能，如右图所示。
![experiment-8](/img/in-post/image-yywt.png)
残差网络比普通网络输出小

BN层输出的std相当于输出信号的大小

由于残差连接的存在，如果当前层的学习不够好，F(x)拟合零映射，使输入和输出十分接近，激活函数也会将负值压制为0，由此残差网络输出响应小

且网络越深输出响应越小，网络输出较小可以帮助避免过拟合、提高训练稳定性、增强可解释性，并适应特定任务的需求。
![experiment-9](/img/in-post/image-sxuf.png)
超深网络Resnet-1202无优化困难问题能收敛

模型参数过多，表示空间过大，对小数据集没有必要

仅通过深而窄的网络结构起到正则化效果

不加花里胡哨的正则化技巧

以免偏离解决优化困难问题这个主线任务

## Conclusion
![conclusion](/img/in-post/image-mvop.png)
这篇论文通过提出残差学习框架，成功解决了深度网络训练中的一些关键问题，推动了深度学习在计算机视觉领域的发展，是深度学习领域的重要里程碑，影响了后续许多网络架构的设计和研究方向。

