---
layout: post
title: "Linguistics aware Masked Image Modeling for Self supervised Scene Text Recognition"
subtitle: "Linguistics-aware Masked Image Modeling (LMIM) Model"
date: 2025-12-03
author: "Can"
header-img: "img/background/dog.jpg"
tags: ["Scene Text Recognition", "Self-supervised Learning", "Masked Image Modeling", "CVPR"]
---

这篇文章发表在**CVPR 2025**上。

## Introduction

- 场景文本识别（STR）专注于从图像中检测出的文本区域中提取文字内容，而场景文本图像通常同时包含视觉和文本信息，其中视觉元素包含结构与外观特征，而语言模式则涵盖语境与语义成分。
- 在视觉质量下降的场景中，语言模式成为理解信息的关键补充，这凸显了将视觉与语言特征相结合对于实现鲁棒场景文本识别（STR）的必要性。
- STR任务中包含序列对比学习和掩码图像建模两种主流方法，其中序列对比学习强调局部特征的对齐，而掩码图像建模（MIM）倾向于利用局部结构来重建视觉模式，从而导致其**语言知识有限**。

## Motivation

![motivation](/img/paper-reading/12-03/motivation.png)

Figure1(a)的这些方法通常需要**大量标注数据集**，这既**成本高昂**，又**因专业语言标注要求而往往难以实现**。尽管合成数据[28, 33]部分缓解了这一问题，但在**合成数据与真实数据上训练的模型之间仍存在性能差距**[5, 34]。为弥合这一差距，利用大规模未标注的真实世界数据具有重要的现实意义。

Figure1(b)指明现有的方法在**处理全局级别信息存在很大的局限性**。序列到序列对比学习与字符到字符蒸馏仅专注于在片段或字符层面对齐局部特征，未能理解文本的整体语言结构，而MIM方法将视觉模式重建置于连贯文本生成之上，主要利用局部视觉特征，而忽视了同一单词内字符的全局上下文。

在自监督学习的场景文本识别中更好地**整合全局语言信息**是非常有必要的。

这篇文章提出了**Linguistics-aware Masked Image Modeling (LMIM)**模型

- **Guidance** **View** **Generation**：生成一个**视觉样式不同但文本内容相同的辅助图像**，用于提取语言知识。
- **Linguistics Alignment**：强制模型从辅助图像中提取**与视觉无关的纯粹文本特征**，作为语言指导信号。
- **Linguistics-guided Reconstruction**：利用这些纯粹的**语言特征来指导解码器**预测被遮盖区域的像素，实现语言感知的重建。

## Methodology

![framework](/img/paper-reading/12-03/framework.png)

### Linguistics-aware Masked Image Modeling

#### Guidance View Generation

在自监督设置下，需要使用数据增强手段以增加额外的信息，通过生成保留完整语言信息的引导视图，解决视觉数据中语言线索整合的难题。传统方法可能因增强过度而导致语义错位，我们的方法利用同一文本内容的多样化视觉表征，在无需显式序列对比学习的前提下确保了语言连贯性。

#### Linguistics Alignment

为解耦视觉无关特征，利用具有不同视觉表征的输入来学习语言一致性。

将[cls]token设计为能有效封装语言信息的载体，使用对齐损失进行约束：

<img src="/img/paper-reading/12-03/distill.png" alt="distill" style="zoom:67%;" >

这种一致性确保语言信息在不同分支间得到统一整合，从而迫使模型在学习视觉结构的同时，也必须掌握全局语言信息。

#### Linguistics-guided Reconstruction

语言学指导分支处理未掩码图像，确保语言信息的完整性。此分支得到的特征表示为<img src="/img/paper-reading/12-03/feature.png" alt="feature" style="zoom: 50%;" />，而掩码重建分支在插入掩码标记后输出的特征序列记为 F。

为将Fˆ中的语言信息整合至F，我们在解码器内设计了自注意力与交叉注意力架构：

<img src="/img/paper-reading/12-03/attention.png" alt="attention" style="zoom:50%;" />

将公式(4)的输出结果与实际掩码mask做MSE损失的约束：
<img src="/img/paper-reading/12-03/recon_loss.png" alt="recon_loss" style="zoom:50%;" />

总体的损失记为：

<img src="/img/paper-reading/12-03/total_loss.png" alt="total_loss" style="zoom:50%;" />

## Experiments

### Datasets

- **[Pre-training Data]** Union14M-U, Unlabeled Chinese Text Image 11M (UCTI-11M)
- **[Fine-tuning Data]** Annotated real data (ARD), Union14M-L, Chinese benchmark (i.e., Scene, Web, Document, and Handwriting)
- **[Benchmarks]** IIIT5K-Words (IIIT5K) , ICDAR2013 (IC13) , Street View Text (SVT), ICDAR2015 (IC15) , SVT Perspective (SVTP) ,CUTE80 (CUTE), Union14M, Chinese benchmark

### Metrics

- **[English text]** WAICS (Word Accuracy Ignoring Case and Symbols)
- **[Chinese text]** sentence-level accuracy
- **[subset level]** Average accuracy (Avg)
- **[instances level]** weighted average (W-Avg)

### Implementation Details

- **[Pre-training Phase]** 使用预训练的具有12层transformer block的ViT-S作为图像编码器，Decoder包含自注意力和交叉注意力两层，输入图像统一被resize到32 x 128，文本图像的掩盖比例设置为80%，重建目标使用维度为384的MAERec 编码器所提取的特征。
- **[Fine-tuning Phase]** 本文提出的文本识别模型包含一个ViT编码器和一个Transformer Decoder（共6层），最大序列长度为25（英文）和40（中文）。

### Results

![image-20251203182418739](/img/paper-reading/12-03/result-1.png)

![image-20251203182450236](/img/paper-reading/12-03/result-2.png)

![image-20251203182509907](/img/paper-reading/12-03/result-3.png)

### Ablation Study

<img src="/img/paper-reading/12-03/ablation-1.png" alt="ablation-1" style="zoom:50%;" />

<img src="/img/paper-reading/12-03/ablation-2.png" alt="ablation-2" style="zoom:50%;" />

### Visualization

<img src="/img/paper-reading/12-03/visualization.png" alt="visualization" style="zoom:67%;" />

## Conclusion

本文提出语言学感知掩码图像建模（Linguistics-aware Masked Image Modeling，简称LMIM），这是一种专为场景文本识别设计的简洁高效自监督学习框架。该方法采用双分支结构，将语言线索融入视觉建模过程，并强调二者对场景文本识别同等重要。其特别设计的语言对齐模块能够利用视觉形态各异的图像，解耦出与视觉无关的语言特征。与仅关注视觉结构的方法不同，LMIM促使模型利用全局上下文信息进行重建。未来工作是探索更适合字符密度的掩码策略，以进一步优化模型性能，有效处理多样化的场景文本识别任务。
