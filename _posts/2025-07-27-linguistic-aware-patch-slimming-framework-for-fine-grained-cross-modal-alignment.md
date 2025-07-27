---
layout: post
title: "Linguistic Aware Patch Slimming Framework for Fine grained Cross Modal Alignment"
subtitle: "跨模态对齐语言感知型补丁精简框架"
date: 2025-07-27
author: "Can"
header-img: "img/post-bg-2015.jpg"
tags: ["人工智能", "多模态", "多标签分类"]
---
这篇工作被<strong>CVPR 2024</strong>接收。

<a href="https://openaccess.thecvf.com/content/CVPR2024/papers/Fu_Linguistic-Aware_Patch_Slimming_Framework_for_Fine-grained_Cross-Modal_Alignment_CVPR_2024_paper.pdf" target="_blank">文章链接</a>

<a href="https://github.com/CrossmodalGroup/LAPS" target="_blank">代码链接</a>

<h1>背景</h1>
现有的跨模态对齐方法可以分为以下两类
<ul>
    <li>粗粒度对齐：分别编码整张图像和整段文字到一个统一的嵌入空间，然后直接计算全局的嵌入相似度</li>
    <li>细粒度对齐：对视觉和文本特征进行局部交互，然后聚合局部特征得到累计的相似度</li>
</ul>
<img src="/img/in-post/image-lXYW.png">
先前的细粒度方法主要采用基于检测器（detector）的方法，这种方式依赖于检测器，并且存在额外的运算开销。最近又出现了基于Transformer的方法，将图像分为不重叠的patch，然后执行patch-word对齐，然而仅采用这种原始的patch-word对齐会存在<b>冗余块（patch redundancy）</b>和<b>块意义不明确（patch ambiguity）</b>的问题。
<h1>动机</h1>
通过语言模态的监督消除冗余的块，对显著块的语义和结构信息进行校正，将平均语义表达转化为特定图像的最优语义表达。

<h1>方法</h1>
<h2>Framework</h2>
<img src="/img/in-post/image-fqFs.png" >
<h2>Token Feature Extraction</h2>
使用纯Transformer架构对特征进行提取。<br>
对图像使用ViT提取，并使用[CLS] token，得到维度为(N+1)xD的特征矩阵，N为图像中的块数量。<br>
对文本使用BERT提取，得到维度MxD的特征矩阵，M为文本中的词数量。
<h2>Language-Context Patch Selection</h2>
在得到了视觉和文本特征之后，就要选择显著图像块。
<h3>Significance Score Estimation</h3>
将这个过程看作识别任务，对每个patch预测一个显著性分数，并根据显著性分数来选择patch，首先对第i个块计算显著性分数：
<img src="/img/in-post/image-bAal.png" style="width: 70%;">
然而仅依赖显著性分数预测显著patch而不是用文本进行监督的话，不足以进行跨模态的对齐，因此还需要对视觉patch和文本word之间计算注意力分数来引入语言上下文。
为了计算视觉和文本之间的注意力分数，引入两种不同的注意力计算方式：
<ul>
    <li>文本和视觉之间的交叉注意力（Cross-Attention）作为文本相对注意力分数，反应视觉块是否与文本语义相关。</li>
    <li>视觉块的自注意力作为图像显著性注意力，区分如前景和背景特征。</li>
    <img src="/img/in-post/image-ocSv.png" style="width: 70%;">
</ul>
在得到上面三种注意力分数后，使用β作为权重对其进行聚合，得到最终的注意力分数。
<img src="/img/in-post/image-ucKK.png" style="width: 70%;">
<h3>Differentiable Decision Matrix</h3>
对块进行选择时，会对N个显著性分数转换为二值决策矩阵{0,1}N代表是否选择这个块。简单的采样方法如基于显著性分数选择top-K个块的方法是不可微的，导致了端到端优化不可行，因此利用Gumbel-Softmax技术平滑这一过程。
Gumbel-Softmax矩阵定义为：
<img src="/img/in-post/image-YAxz.png" style="width: 70%;">
M为NxL维矩阵，L是类别数量，Gi=-log(-log(Ui))为Gumbel分布为采样过程添加噪声，Ui为均匀分布，τ控制M的平滑程度。推理时使用arg-max对M进行采样，得到决策矩阵D,训练时用M作为近似离散决策进行反向传播：
<img src="/img/in-post/image-BAaC.png" style="width: 70%;">
<h2>Semantic-Spatial Patch Calibration</h2>
在选择了显著视觉块后，就需要增强显著块的语义表达能力。

使用聚合网络学习多个聚合权重，然后聚合Ns个显著块生成Nc个包含信息的块：
<img src="/img/in-post/image-RwNB.png" style="width: 70%;">
W为权重矩阵，由MLP和softmax使用显著块学习而来：W=softmax(MLP(Vs))。

此外冗余块中可能还包含对跨模态对齐十分重要的视觉信息，因此，将冗余块融合成一个块保留：
<img src="/img/in-post/image-dRCS.png" style="width: 70%;">
最后得到过滤校正的块集合：
<img src="/img/in-post/image-NmnT.png" style="width: 30%;">
<h2>Sparse Patch-Word Alignment</h2>
使用先前得到的块和原始文本执行细粒度的跨模态对齐。首先计算逐词相似度，生成NcxM的块-词相似度矩阵：
<img src="/img/in-post/image-NCnl.png" style="width: 30%;">
其中A中元素代表第i个视觉块和第j个词之间的相似度。

接下来，使用最大对应交互来聚合对齐：首先为每个块（或词）挑选最对齐的词（或块）。

然后对这些分数计算平均表示图像和文本间总体对齐分数：
<img src="/img/in-post/image-JUHg.png" style="width: 70%;">
使用双向三元组硬负样本挖掘策略作为损失
<img src="/img/in-post/image-QpxX.png" style="width: 70%;">
其中α为边缘参数，[x]+=max(x,0),(I,T)为正图像-文本对。

另外，对选择的块用一个预定义的参数ρ进行约束以稳定训练，并采用MSE损失监督这个过程，总的损失表示为：
<img src="/img/in-post/image-tZkt.png" style="width: 70%;">

<h1>实验</h1>
<h2>Dataset</h2>
Flickr30K、MS-COCO
<h2>Evaluation Metric</h2>
recall R@K、rSum
<h2>Results</h2>
<img src="/img/in-post/image-fwMu.png">
<img src="/img/in-post/image-ldxO.png">
<img src="/img/in-post/image-DtFw.png">
<h2>Ablation Study</h2>
<img src="/img/in-post/image-IywG.png">
<img src="/img/in-post/image-Xonk.png">
<h2>Visualization</h2>
<img src="/img/in-post/image-eQsi.png">
<img src="/img/in-post/image-iXNn.png">
<h1>总结</h1>
本文提出了一种新颖的跨模态对齐语言感知型补丁精简框架(LAPS)，这是首个在纯基于Transformer的架构上显式解决补丁冗余与模糊性问题、专注于补丁-单词对齐的研究。LAPS通过语言监督识别关键视觉补丁，进而修正语义与结构信息以构建更精准一致的对齐关系。在多种基准测试和视觉编码器上的大量实验验证了本框架的优越性。