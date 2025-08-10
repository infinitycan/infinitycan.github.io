---
layout: post
title: "Pytorch DDP运行时报错"
subtitle: "参数没有参与loss运算 DDP同步梯度报错"
date: 2025-08-09
author: "Can"
header-img: "img/post-bg-2015.jpg"
tags: ["PyTorch", "DDP"]
---

## 问题描述
<img src="/img/in-post/image-uLqf.png" alt="检索文件系统" style="zoom:60%;" />
最近在写论文的过程中，需要做模块的消融实验，假设我的模型结构是模块A+模块B+模块C，现在我需要单独验证模块B对于模型的提升效果，我直接在模型的forward()函数中注释掉了有关于模块A和模块B的调用代码，这样在单卡训练的时候是没有问题（无报错信息），但是在DDP（DistributedDataParallel）中就会出现上图的错误。

**该错误的大致意思是：如果模型中的某些参数没有参与 forward 并传播到 loss（即计算图中没有梯度回传路径），那么 DDP 在多卡同步梯度时就会报错。**

## 排查思路
在 torch.nn.parallel.DistributedDataParallel 模式下，每次训练：
1. 每张卡上的进程都会调用模型的forward()函数。
2. 每张卡上的进程都会调用loss.backward()函数。
4. DDP 会在 backward 之后同步每个参数的梯度，但只同步参与了 loss 计算的参数。
所以如果：
* 模块 self.xxx = SomeModule(...) 定义了（即参数被注册了）
* 但在 forward 里没用它（或条件下跳过）
* 那它的参数没有被“用到”，梯度为 None
* DDP 就会 报错：这个参数没有参与 loss 的计算，不能同步！

## 解决方法
* **方式一：**设置find_unused_parameters=True **（不推荐）**,这种方法能够直接解决RuntimeError的报错问题，实际上这样会让pytorch自动查找没有使用的参数，防止爆炸，但是这个设置有性能开销，并且不是从根本上解决问题（尤其你本身知道哪些模块没用）。
```python
model = torch.nn.parallel.DistributedDataParallel(
            model, device_ids=[local_rank], broadcast_buffers=False, find_unused_parameters=False)  
```
* **方式二：**找到没有参与梯度运算的参数，将其注释或删除，或者将其设置为requires_grad=False,当你并没有使用到某个模块的时候，可以在__init__声明的时候将其注释掉，这样在DDP运算时即使没有使用到这些注释掉的模块，也不会出现RuntimeError的问题。
```python
class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(10, 10)
        # self.linear2 = nn.Linear(10, 10)
    def forward(self, x):
        x = self.linear(x)
        # x = self.linear2(x)
        return x
```
* **方式三：**增加额外的loss项，这样dual_channel_attn这个模块并没有被实际调用，但是参数 “假装” 参与了计算图，DDP不会报错，且这种方式不会影响实际的运行结果。
```
# dummy loss 防止 DDP 报错
dummy = sum(p.sum() * 0.0 for p in self.dual_channel_attn.parameters())
logits = logits + dummy  # 加入计算图中，骗 DDP 一下
```