---
layout: post
title: "Failed to initialize NVML: Driver/library version mismatch"
subtitle: "Ubuntu CUDA驱动 NVML Kernel——Module"
date: 2025-10-22
author: "Can"
header-img: "img/boats.jpg"
tags: ["CUDA", "Ubuntu"]
---

## 问题复现
今早组里师弟反馈：在服务器上使用nvidia-smi以及nvitop命令检查显卡时，出现了如下错误：
```bash
Failed to initialize NVML: Driver/library version mismatch
```
系统环境为：
- Ubuntu 24.04.3 LTS
- CUDA 13.0
- NVIDIA Driver 580.65.06

## 排查思路
1. 报错信息提示了Driver/library version mismatch，即驱动版本与库版本不匹配，我的第一想法其实就是CUDA驱动自己更新了。
2. 这里的Driver其实是在用户使用nvidia-smi命令时通过系统调用NVML库来实现的，而NVML库的版本与驱动版本不一致，导致了报错。
3. 检查CUDA内核驱动的版本，可以知道**内核中加载**的驱动版本是`580.65.06`，然而，`apt`已经升级并安装了新的驱动文件（包括 NVML 库文件），其版本为 `580.95.05`。**NVML 库版本与内核加载的旧版本驱动不匹配，是导致报错的直接原因。**
```bash
cat /proc/driver/nvidia/version
```
4. 也就是说，**Ubuntu系统通过apt命令自动升级了CUDA驱动**，导致了NVML库版本与驱动版本不一致的问题。NVML 库 (libnvidia-ml.so) 是 NVIDIA 驱动软件包的一部分。当 apt 升级驱动时，它会替换用户空间的 NVML 库文件，但**不会立刻替换内核中正在运行的旧驱动模块（Kernel Module）**。因此，用户空间的库版本（新）与内核模块版本（旧）产生了不匹配。
5. 为印证以上推断，我们检索了apt的升级记录，发现有升级CUDA驱动的记录。
```bash
tail /var/log/apt/history.log 
```
![apt-upgrade-cuda-drivers](/img/in-post/cuda-driver-auto-upgrade.jpg)

## 解决方法
1. 为了解决这个问题，我们需要确保CUDA驱动版本与NVML库版本一致。
2. 重启系统以使得系统加载新的CUDA驱动版本。
    * 系统关闭 (Shutdown)：所有正在运行的程序和内核模块都被安全地卸载。
    * 系统启动 (Boot)：操作系统内核开始启动。
    * 新模块加载：系统会检查 /lib/modules 目录下最新且可用的 NVIDIA 内核模块文件 (nvidia.ko，即 580.95.05 版本)。
    * 加载完成：新版本的驱动模块被加载到内核中运行。此时，用户空间的 NVML 库 (580.95.05) 与内核中的驱动模块 (580.95.05) 版本一致，问题解决。
```bash
sudo reboot
```
3. 系统重启后再检查CUDA版本与NVML库版本，确认一致。
```bash
nvidia-smi
```
![nvidia-smi](/img/in-post/nvidia-smi.jpg)
![nvml-version](/img/in-post/nvml-version.jpg)
4. 为了防止未来再次出现类似问题，我们可以禁用自动升级CUDA驱动。
```bash
sudo apt-mark hold nvidia-driver-580（这里的版本号需要根据实际情况替换）
```

## 总结反思
* 禁用自动升级CUDA驱动可以避免由于系统自动升级导致的NVML库版本与驱动版本不一致的问题。
* 需要自己了解一下CUDA体系结构，包括CUDA驱动、CUDA库、CUDA Runtime等组件，以及它们之间的关系。

## 参考链接
* [ubuntu20.04 nvidia-smi命令报错Failed to initialize NVML: Driver/library version mismatch解决办法--重启电脑](https://blog.csdn.net/ccsodefhy/article/details/122846921)
