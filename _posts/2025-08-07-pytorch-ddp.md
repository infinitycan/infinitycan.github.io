---
layout: post
title: "PyTorch DDP详解"
subtitle: "分布式并行 DistributedDataParallel"
date: 2025-08-07
author: "Can"
header-img: "img/post-sample-image.jpg"
tags: ["PyTorch", "DDP"]
---

## DDP框架图

![DDP Framework](/img/in-post/image-yhlu.png)


## 命令
* 在分布式训练中，使用 torch.distributed.launch 启动脚本时，它会自动为每个进程传递 local_rank 参数，因此不需要显式设置 local_rank，每个进程会分配一个唯一的 local_rank 值（从 0 到 nproc_per_node - 1）,实际运行如下所示。

```python
python -m torch.distributed.launch --nproc_per_node=4 --nnodes=1 --node_rank=0 --master_addr="localhost" --master_port=12355 ddp_train.py

python ddp.py --local_rank=0
python ddp.py --local_rank=1
python ddp.py --local_rank=2
python ddp.py --local_rank=3
```
* --nproc_per_node=4 ：每个节点（机器）上启动4个进程​​（通常对应4个 GPU），如果机器有4个GPU，则每个GPU分配一个进程，每个进程会独立加载模型和数据，通过DDP同步梯度。
* --nnodes=1 ：​​总共使用 1 个节点（单机训练）​​。如果是多机训练（如 2 台机器），则设为 --nnodes=2，并配合 --master_addr 和 --master_port 指定主节点地址。
* --node_rank=0​：​当前节点的编号为 0​​（单机时固定为 0）。主节点设为 0，其他节点依次为 1, 2, ...。
* --master_addr="localhost" ：主节点的 IP 地址，也可以设置为127.0.0.1。
* --master_port=12355​：主节点监听的端口号​​（用于进程间通信）。需确保端口未被占用（可随意选择，如 12355、29500）。

## 代码示例

```python
import torch
import torch.nn as nn
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.distributed import init_process_group, destroy_process_group

def main():
    # 1. 初始化进程组
    dist.init_process_group(
        backend="nccl",  # 使用 NVIDIA NCCL 后端（GPU 必须）
        init_method="env://",  # 通过环境变量自动获取 master_addr 和 master_port
    )
    
    # 2. 获取当前进程信息 rank 和 local_rank 是两个独立的变量。rank 是整个分布式训练的唯一ID，而 local_rank 则是针对单台机器的ID。两者相互补充，共同确保了DDP训练的正确运行。
    rank = dist.get_rank()  # 全局进程编号（Global Rank）（0 ~ nproc_per_node-1）全局rank主要用于跨机器的进程间通信。
    local_rank = int(os.environ["LOCAL_RANK"])  # 本地进程编号（Local Rank） 当前机器上所有进程的唯一标识符。

    # 3. 绑定 GPU
    torch.cuda.set_device(local_rank)
    
    # 4. 创建模型并包装为 DDP
    model = MyModel().cuda()
    model = DDP(model, device_ids=[local_rank])

    # 5. 数据加载（需使用 DistributedSampler）
    dataset = MyDataset()
    sampler = torch.utils.data.distributed.DistributedSampler(dataset)
    dataloader = torch.utils.data.DataLoader(dataset, batch_size=64, sampler=sampler)

    # 6. 训练逻辑
    for epoch in range(10):
        sampler.set_epoch(epoch)  # 确保每个 epoch 数据不同
        for batch in dataloader:
            ...

if __name__ == "__main__":
    main()
    # 7. 销毁进程组
    destroy_process_group()

```
## 常见问题

1. 为什么用 nccl 后端？​
    * NCCL（NVIDIA Collective Communications Library）是 NVIDIA 提供的用于 GPU 之间通信的库，支持 GPU 之间的高效数据传输。
    * 它是 DDP 训练中推荐的后端，因为它在 GPU 上运行速度快，支持多 GPU 之间的并行通信, 比 gloo 更快。
    * 如果不使用 NCCL 后端，DDP 训练会退化为单 GPU 训练，效率会大大降低。
2. ​​多机训练如何修改？​
    * 多机训练需要修改的地方主要有以下几点：
        * --nnodes=2 ：设置节点数量为 2 台机器。
        * --node_rank=0 和 --node_rank=1 ：分别为 2 台机器设置不同的节点编号。
        * --master_addr="主节点IP" 和 --master_port=12355 ：指定主节点的 IP 地址和端口号。
3. LOCAL_RANK是什么？​
    * 由torch.distributed.launch自动注入的环境变量，表示当前进程的 GPU 编号（0~3）。
4. ​​数据如何分配？​
    * 必须用DistributedSampler，它会自动切分数据，确保不同进程处理不同部分。