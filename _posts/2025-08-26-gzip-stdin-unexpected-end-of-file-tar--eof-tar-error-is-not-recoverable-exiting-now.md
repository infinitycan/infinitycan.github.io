---
layout: post
title: "gzip: stdin: unexpected end of file tar: 归档文件中异常的 EOF tar: Error is not recoverable: exiting now"
subtitle: ""
date: 2025-08-26
author: "Can"
header-img: "img/sea.jpg"
tags: ["DevOps", "Linux"]
---

## 问题描述
[VideoPipe](https://opendatalab.com/OpenDataLab/VideoPipe)（城市管道系统中的异常检测数据集，使用分卷压缩存储），该数据集共205GB，由于文件容量过大，使用分卷压缩存储。在Ubuntu 22.04下使用以下命令报错：
```bash
tar -zxvf tarck1_raw_video.tar.gz00
```
尝试使用以下命令均失败
```bash
tar -xvzf tarck1_raw_video.tar.gz00 --wildcards -- *.tar.gz*
tar -zxvf tarck1_raw_video.tar.gz*
```

## 解决方法
使用以下命令成功解压分卷
```bash
cat track1_raw_video.tar.gz* | tar -zxvf -
```
此脚本通过cat命令将所有的分卷压缩包先合并为一个压缩包，利用管道符｜进行输入输出的传递，将得到的总体压缩包使用tar命令进行整体解压，得到解压后的最终结果。