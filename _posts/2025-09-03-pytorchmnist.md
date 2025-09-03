---
layout: post
title: "PyTorch——MNIST数据集手写数字识别"
subtitle: ""
date: 2025-09-03
author: "Can"
header-img: "img/sea-1.jpg"
tags: ["MNIST", "PyTorch", "Python"]
---

## Introduction
### PyTorch
PyTorch是一种流行的深度学习框架，具有多个核心优势，使其在研究和工业界广受欢迎：
1. **动态计算图：**PyTorch采用动态计算图（Define-by-Run），这意味着计算图在运行时构建，允许用户在每次迭代中修改网络结构。这种灵活性使得调试和开发更加直观和方便。
2. **强大的GPU加速：**PyTorch支持GPU加速，能够高效地处理大规模的张量运算。这使得训练深度学习模型的速度显著提升，尤其是在处理复杂的神经网络时。
3. **易于使用和学习：**PyTorch的API设计简洁明了，易于上手，特别适合初学者。其与Python的紧密集成使得用户能够利用Python的丰富生态系统。
4. **丰富的社区支持：**PyTorch拥有一个活跃的社区，提供大量的教程、文档和开源项目。这使得用户能够快速找到解决方案和学习资源。
5. **与NumPy兼容：**PyTorch的张量操作与NumPy非常相似，用户可以轻松地在两者之间切换，利用NumPy的功能进行数据处理。
6. **自动微分：**PyTorch自动计算梯度，这使得训练模型时无需手动计算梯度，简化了模型训练的过程。

PyTorch因其灵活性、易用性和强大的性能，成为了深度学习领域的重要工具，适合从研究到生产的各种应用场景。

### MNIST Dataset
MNIST（Modified National Institute of Standards and Technology）数据集是一个广泛使用的手写数字识别数据集，常用于机器学习和深度学习的研究与实验。它包含了大量的手写数字图像，具体特点如下：
1. **数据规模：**MNIST数据集包含60,000个训练样本和10,000个测试样本。这些样本都是28x28像素的灰度图像，表示数字0到9。
2. **数据来源：**数据集由美国国家标准与技术研究院（NIST）提供，经过修改和标准化，以便于机器学习算法的训练和测试。
3. **应用广泛：**MNIST是机器学习领域的“Hello World”数据集，因其简单性和易用性，成为了许多算法和模型的基准测试数据集。
4. **分类任务：**任务是将每个28x28的图像分类为对应的数字（0-9），这使得MNIST成为图像分类问题的经典示例。
5. **性能评估：**研究人员和开发者通常使用MNIST来评估新算法的性能，许多现代深度学习模型在该数据集上取得了很高的准确率。

MNIST数据集因其简单性和广泛的应用，成为了机器学习和深度学习领域的重要基准数据集，适合用于初学者的学习和研究。

## Implementation
* Load Modules
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
import cv2
import numpy as np
from torchvision import datasets,transforms
from torch.utils.data import DataLoader
```
* Set Hyperparameters
```python
BATCH_SIZE=64
DEVICE = torch.device("cuda" if torch.cuda.is_available()else "cpu")
EPOCHS=20 # 训练次数
```

* Pipeline
```python
pipeline=transforms.Compose([
    transforms.ToTensor(), 
    transforms.Normalize((0.1307),(0.3081)) 
])
```

* Load Dataset
```python
train_dataset = datasets.MNIST('data', train=True, download=True, transform=pipeline)
test_dataset = datasets.MNIST('data', train=False, transform=pipeline)
train_loader=DataLoader(train_dataset,batch_size=BATCH_SIZE,shuffle=True)
test_loader=DataLoader(test_dataset,batch_size=BATCH_SIZE,shuffle=True)
```

* Build Model
```python
class Digit(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1=nn.Conv2d(1,10,kernel_size=5)
        self.conv2=nn.Conv2d(10,20,kernel_size=3)
        self.fc1=nn.Linear(20*10*10,500)
        self.fc2=nn.Linear(500,10)

    def forward(self,x):
        input_size=x.size(0)
        x=self.conv1(x)
        x=F.relu(x)
        x=F.max_pool2d(x,2,2)

        x=self.conv2(x)
        x=F.relu(x)

        x=x.view(input_size,-1)

        x=self.fc1(x)
        x=F.relu(x)
        x=self.fc2(x)

        output=F.log_softmax(x,dim=1)
        return output
```
* Optimizer
```python
model=Digit().to(DEVICE)
optimizer=optim.Adam(model.parameters(),lr=0.001)
```
* Train Function
```python
def train_model(model,device,train_loader,optimizer,epoch):
    # 模型训练
    model.train()
    for batch_index,(data,target) in enumerate(train_loader):
        # 部署到DEVICE上面
        data,target=data.to(device),target.to(device)
        # 梯度初始化为0
        optimizer.zero_grad()
        # 前向传播获取结果
        output=model(data)
        # 计算损失
        loss=F.cross_entropy(output,target) # 多分类用交叉验证
        # 反向传播
        loss.backward()
        # 参数优化
        optimizer.step()
        if batch_index%3000==0:
           print("Train Epoch :{} \t Loss :{:.6f}".format(epoch,loss.item())) 
```
* Test Function
```python
def test_model(model,device,test_loader):
    # 模型验证
    model.eval()
    # 定义正确率
    correct=0.0
    # 定义测试损失
    test_loss=0.0
    # 计算损失
    with torch.no_grad():
        for data,target in test_loader:
            # 部署到device上
            data,target=data.to(device),target.to(device)
            # 测试数据
            output=model(data)
            # 计算测试损失
            test_loss+=F.cross_entropy(output,target).item()
            # 找到概率最大的下标
            pred=output.argmax(1)
            # 累计正确的值
            correct+=pred.eq(target.view_as(pred)).sum().item()
        test_loss/=len(test_loader.dataset)
        print("Test--Average Loss:{:.4f},Accuarcy:{:.3f}\n".format(test_loss,100.0 * correct / len(test_loader.dataset)))
```
* Call Functions
```python
for epoch in range(1,EPOCHS+1):
    train_model(model,DEVICE,train_loader,optimizer,epoch)
    test_model(model,DEVICE,test_loader)
```

## Result
![result](/img/in-post/image-mdkh.png)
![result-1](/img/in-post/image-lswt.png)

## Conclusion
MNIST数据集上进行实验，不仅能够掌握深度学习的基本概念，还能为其后的研究和应用打下坚实的基础。MNIST的成功应用展示了神经网络在图像分类任务中的强大能力，也激励了更复杂的深度学习模型的发展。
