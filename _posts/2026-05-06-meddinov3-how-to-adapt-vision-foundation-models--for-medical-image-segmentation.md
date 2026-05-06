---
layout: post
title: "MEDDINOV3: HOW TO ADAPT VISION FOUNDATION MODELS  FOR MEDICAL IMAGE SEGMENTATION?"
subtitle: ""
date: 2026-05-06
author: "Can"
header-img: "img/background/swan-1.jpg"
tags: ["Paper Reading", "Foundation-Model", "Segmentation", "arXiv", "Self-supervised Learning", "U-Net"]
---

## Introduction

- CT与MRI扫描中器官与肿瘤的**精确分割**对于诊断、治疗规划和疾病监测至关重要。但是分割标签的标注是成本巨大的，而自监督学习（SSL）使用无标签的数据进行训练，很好的缓解了标签标注的成本问题。
- 现有的分割方法多是任务特定的，在模态和机构差别的条件下**缺乏泛化能力**。
- 将具有强大泛化能力的视觉基础模型运用于医学影像领域存在**两大关键的挑战**：
        - 多数基础模型的**ViT主干网络**在医学图像分割任务上**仍逊色于专用CNN**。
        - 自然图像与医学图像间巨大的**领域差异**限制了其可迁移性。

## Motivation

**从网络规模自然图像中学习到的表征能否有效迁移到放射影像领域？**

将ViT进行重构，通过复用中间Transformer模块的patch tokens来实现多尺度特征聚合。

**这篇文章旨在通过改进架构设计和执行领域自适应自监督学习预训练，充分释放Vision Transformer在医学图像分割中的潜力。**

## Contribution

- 提出了一种针对**二维医学图像分割的高效设计方案**。两项关键改进：
        - 从中间补丁令牌进行多尺度token聚合
        - 高分辨率训练
- 基于CT-3M的**领域自适应预训练**。构建了CT-3M——一个从16个数据集中收集的大规模轴向CT切片库，并通过三阶段流程适配DINOv3：
        - 全局/局部自蒸馏（DINOv2风格）
        - 语法锚定以稳定补丁级一致性
        - 高分辨率适配
- 在四个多样化基准测试（AMOS22、BTCV、KiTS23、LiTS）取得先进性能（SOTA）。

## Method

### Framework

![framework](/img/in-post/meddinov3.png)

- **[Stage 1]** 提出了对视觉transformer的迭代细化，用于医学图像分割，实现了预训练SSL模型的集成。
- **[Stage 2]** 用DINOv3对CT图像进行域自适应预训练，这是一种最先进的SSL方法，以学习卓越的密集特征而闻名。
- **[Stage 3]** 将学到的表示迁移到各种医疗分割任务中。

### Rethinking transformers for medical image segmentation

- 分割模型中的Transformer模块对最终性能贡献甚微，且仍不及强大的CNN基线模型。
- 提出一种渐进式优化方法，将DINOv3主干网络适配为高效的分割骨干网络，如表1所示：

![refinements](/img/in-post/refinements.png)

- 选用**AMOS22**来训练和评估每个优化步骤。AMOS22是一个包含300例CT影像与60例MRI影像的腹部器官分割数据集，其中标注了15个器官。
- 遵循与Primus相似的策略，仅使用默认五**折交叉验证方案中的单折数据（80/20划分）**进行训练与评估。根据默认nn-UNet设置，我们共**训练1000个轮次**，超参数调整参考自[Primus](https://arxiv.org/pdf/2503.01835)。
- 使用DINOv3 ViT编码器**（ViT-B）**和由背对反卷积、层归一化及GELU激活函数构成的Primus解码器构建基线模型，将图像块标记上采样为全分辨率分割图。
- 基于DINOv3在**LVD-1689M**数据集上预训练的权重进行二次预训练。

#### Multi-scale token aggregation

- Primus**仅使用最后一个Transformer块作为解码器输入**，因此假设这种层级先验的缺失**阻碍了ViT学习强局部特征**。
- 提出**复用多个中间层（第2、5、8、11块）的patch tokens**，并将其拼接作为解码器输入。这一步骤增强了ViT原本较弱的空间先验。

#### Higher resolution training

- 现有视觉基础模型鲜少使用8的块尺寸进行预训练，这可能是由于计算开销增加所致。
- 提出通过将轴向切片重采样至更薄层间距来实现高分辨率分割训练。遵循DINOv3的方法，保持896×896的输入分辨率。

### Domain-adaptive pretraining on CT-3M

#### Data curation

从**16个公开数据集（BTCV, Pancreas-CT (TCIA) , CHAOS , LiTS , KiTS , WORD , AbdomenCT-1K , AMOS22 , and five CT tasks from Medical Segmentation Decathlon (Liver, Lung, Pancreas, Hepatic Vessel, Spleen, Colon) , CT-ORG , TotalSegmentator and AbdomenAtlas 3.0 ）**中聚集了总共3,868,833 个轴向切片。所有三维体积数据均重采样至平面内间距0.45毫米×0.45毫米，随后统一调整为**256×256的标准分辨率**。

#### Training Pipeline

- [Stage 1] 使用DINOv2的设定进行预训练，包含全局CLS token的自蒸馏损失和局部patch token的图像重建损失，以及正则化损失：


<img src="/img/in-post/stage-1.png" width="50%" alt="stage-1" />

- [Stage 2] 采用Gram Anchoring缓解长时间训练导致的局部图像特征的退化问题：

<img src="/img/in-post/gram.png" width="50%" alt="gram" />

<img src="/img/in-post/stage-2.png" width="50%" alt="stage-2" />

- [Stage 3] 最终阶段对预训练模型进行调整，使其能够处理更高分辨率的图像，这与我们的任务尤为相关。遵循DINOv3的方法，混合使用多种分辨率的全局裁剪与局部裁剪（例如全局裁剪512–768，局部裁剪112–336）。重要的是**保留了Gram Anchoring技术，以确保图像块相似性结构保持稳定。**此阶段持续进行1万次迭代。发现高分辨率适配显著提升了模型的密集特征质量（图3）。

<img src="/img/in-post/visualization.png" width="50%" alt="visualization" />

#### Implementation details

- 对MedDINOv3进行了总计**12万次迭代的预训练**，使用的**全局批次大小为512**。
- 模型初始化采用在**LVD-1689M数据集上预训练的DINOv3 ViT-B检查点**。
- 在**第一阶段**，使用**2e-4的学习率进行10万次迭代训练**。
- 在**第二阶段**，我们选择**第2万次迭代的EMA模型作为Gram教师模型**，并继续使用**Gram锚定**目标进行**1万次迭代的预训练**，该阶段学习率为**5e-5**。
- 在第三步中，使用第二阶段得到的模型初始化教师和学生模型，并以**2.5e-5**的学习率再进行**1万次迭代训练。**

## Experiments

### Evaluation Dataset

KiTS23, LiTS, BTCV

### Metrics

dice similarity coefficient (DSC)

normalized surface dice (NSD)

### Results

#### Comparisons with SOTA methods

![sota](/img/in-post/meddinov3-sota.png)

#### Ablation study

![ablation](/img/in-post/meddinov3-ablation.png)

![consine-sim](/img/in-post/meddinov3-consine.png)

在第二阶段采用**Gram Anchoring并未观察到明显增益**。推测这是因为**在第一阶段预训练期间，图像块标记的质量并未显著下降**。为验证这一点，在图4中对其进行了可视化。尽管如此，将模型调整至更高分辨率仍使DSC提升了0.84%，并保持了特征图的一致性。

## Conclusion

### Architectural Refinements

MedDINOv3 未采用复杂的定制网络,而是基于标准的 **Vision Transformer (ViT)** 进行了两项针对性改进:

- **多尺度 Token 聚合 (Multi-scale Token Aggregation)**:通过聚合不同尺度的特征增强空间先验 (Spatial Priors),这对精准定位人体器官至关重要。
- **高分辨率训练 (High-resolution Training)**:专门用于保留医疗影像中的复杂局部细微结构。

### Data & Pretraining

作者提出了**三阶段预训练配方**,并在自建的大规模数据集 **CT-3M** 上进行领域自适应预训练:

- **阶段 1 (DINOv2 风格自蒸馏)**:显著提升特征的可迁移性。
- **阶段 2 (Gram Anchoring)**:在当前设置下仅带来微小收益。
- **阶段 3 (高分辨率适配)**:对性能提升有实质性贡献。

### Performance

- **全能性**:在危及器官 (OAR) 分割任务中持续优于或比肩强力 CNN 和 Transformer 基线。
- **肿瘤分割**:在更具挑战性的肿瘤分割任务中达到极具竞争力的水平。
- **弥合差距**:证明基础模型 (Foundation Model) 配合适当的自适应训练,可以抹平甚至超越专门设计的 CNN 架构。

### Key Takeaway

**MedDINOv3 证明:**无需过度复杂的模型设计。通过**针对性的架构优化**结合**领域对齐的预训练**,简单的 ViT 即可成为医疗影像分割领域强大且通用的解决方案。