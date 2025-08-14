---
layout: post
title: "PyTorch大数据集加载优化思路"
subtitle: ""
date: 2025-08-14
author: "Can"
header-img: "img/poland.jpg"
tags: ["PyTorch"]
---

## 现有问题
使用他人项目中MS-COCO_2014数据集的加载思路，加载公开的多标签排水管缺陷数据集Sewer-ML。由于COCO数据集本身数据量就不大，且在_load_dataset这个操作之前，我们就已经提前对coco的标签数据进行了处理，处理后的json文件仅剩下（图片相对路径、对应的label、所有类别），故使用常规的for循环，在for循环中添加路径替换和one-hot-encoding是不会消耗太多时间的。
```python
//MS-COCO_2014数据集加载
class MultiLabelClassification(BaseImageDataset):
    def __init__(self, root='', verbose=True, **kwargs):
        super(MultiLabelClassification, self).__init__()
        self.dataset_dir = root
        self.train_file = os.path.join(self.dataset_dir, 'cvt_train.json')
        self.test_file = os.path.join(self.dataset_dir, 'cvt_val.json')
        # self.train_file = os.path.join(self.dataset_dir, 'train_anno_2014.json')
        # self.test_file = os.path.join(self.dataset_dir, 'val_anno_2014.json')

        self._check_before_run()

        train, class2idx, classnames = self._load_dataset(self.dataset_dir, self.train_file, shuffle=True)
        test, _, _ = self._load_dataset(self.dataset_dir, self.test_file, shuffle=False)
        self.train = train
        self.test = test
        self.class2idx = class2idx
        if verbose:
            print("=> Multi-Label Dataset loaded")
            self.print_dataset_statistics(train, test)
        self.classnames = classnames

    def _check_before_run(self):
        """Check if all files are available before going deeper"""
        if not os.path.exists(self.dataset_dir):
            raise RuntimeError("'{}' is not available".format(self.dataset_dir))
        if not os.path.exists(self.train_file):
            raise RuntimeError("'{}' is not available".format(self.train_file))
        if not os.path.exists(self.test_file):
            raise RuntimeError("'{}' is not available".format(self.test_file))

    def _load_dataset(self, data_dir, annot_path, shuffle=True):
        out_data = []
        with open(annot_path) as f:
            annotation = json.load(f)
            classes = sorted(annotation['classes'])
            class_to_idx = {classes[i]: i for i in range(len(classes))}
            images_info = annotation['images']
            img_wo_objects = 0
            for img_info in images_info:
                labels_idx = list()
                rel_image_path, img_labels = img_info
                full_image_path = os.path.join(data_dir, rel_image_path)
                labels_idx = [class_to_idx[lbl] for lbl in img_labels if lbl in class_to_idx]
                labels_idx = list(set(labels_idx))
                # transform to one-hot
                onehot = np.zeros(len(classes), dtype=int)
                onehot[labels_idx] = 1
                assert full_image_path
                if not labels_idx:
                    img_wo_objects += 1
                out_data.append((full_image_path, onehot))
        if img_wo_objects:
            print(f'WARNING: there are {img_wo_objects} images without labels and will be treated as negatives')
        if shuffle:
            random.shuffle(out_data)
        return out_data, class_to_idx, classes
```
但将代码修改迁移到Sewer-ML数据集上后就出现了问题，由于Sewer-ML的训练注释文件（SewerML_Train.csv）和验证注释文件（SewerML_Valid.csv）总共有110w条记录，且在csv文件中已经对下水道缺陷的17个类进行了标注（可以认为已经完成了one-hot-encoding），使用逐记录loc的方式进行加载的数据特别慢（大概22个小时）。
```python
import os
import numpy as np
import random
import pandas as pd
import dask.dataframe as dd
from tqdm import tqdm
from utils.iotools import mkdir_if_missing
from datasets.bases import BaseImageDataset

DefectLabels = ["RB", "OB", "PF", "DE", "FS", "IS", "RO", "IN", "AF", "BE", "FO", "GR", "PH", "PB", "OS", "OP",
                "OK", "VA", "ND"]

class SewerMLClassification(BaseImageDataset):
    def __init__(self, root='', verbose=True, **kwargs):
        print("Loading SewerML dataset...")
        super(SewerMLClassification, self).__init__()
        self.dataset_dir = root
        # self.train_file = os.path.join(self.dataset_dir,'train_10w.csv')
        # self.test_file = os.path.join(self.dataset_dir,'val_1w.csv')
        self.train_file = os.path.join(self.dataset_dir,'SewerML_Train.csv')
        self.test_file = os.path.join(self.dataset_dir,'SewerML_Valid.csv')
        # 确保只使用17个缺陷类
        self.LabelNames = DefectLabels.copy()
        self.LabelNames.remove("VA")
        self.LabelNames.remove("ND")

        self._check_before_run()
        print("Loading SewerML labels=>")
        train, class2idx, classnames = self._load_dataset(self.dataset_dir, self.train_file, split='Train',shuffle=True)
        test, _, _ = self._load_dataset(self.dataset_dir, self.test_file, split='Valid',shuffle=False)
        self.train = train
        self.test = test
        self.class2idx = class2idx
        if verbose:
            print("=> Multi-Label Dataset loaded")
            self.print_dataset_statistics(train, test)
        self.classnames = classnames

    def _check_before_run(self):
        """Check if all files are available before going deeper"""
        if not os.path.exists(self.dataset_dir):
            raise RuntimeError("'{}' is not available".format(self.dataset_dir))
        if not os.path.exists(self.train_file):
            raise RuntimeError("'{}' is not available".format(self.train_file))
        if not os.path.exists(self.test_file):
            raise RuntimeError("'{}' is not available".format(self.test_file))

     def _load_dataset(self, data_dir, annot_path, split, shuffle=True):
         out_data = []
         # 加载注释文件，获取指定列
         annotation = pd.read_csv(annot_path, sep=',', encoding='utf-8', usecols=DefectLabels + ["Filename", "Defect"])
         self.imgPaths = annotation['Filename'].values
         classes = self.LabelNames
         class_to_idx = {classes[i]: i for i in range(len(classes))}
    
         # tqdm 添加进度条，设置总长度为文件路径的数量
         for img_path in tqdm(self.imgPaths, desc="Loading dataset", total=len(self.imgPaths)):
             full_image_path = os.path.join(data_dir, split, img_path)
             # 获取缺陷标签
             labels = annotation.loc[annotation['Filename'] == img_path, classes].values.flatten()
             # 添加到输出数据列表
             out_data.append((full_image_path, labels))
    
         if shuffle:
             random.shuffle(out_data)
    
         return out_data, class_to_idx, classes
```

## 解决思路
* **减少逐行操作**：每次在循环中用 annotation.loc 按 Filename 查找行会导致大量的重复操作，极大地拖慢速度。可以提前将 annotation 数据转换为一个字典，基于 Filename 进行查找，显著提升效率。
* **使用Dask**：Dask 是一个并行计算库，特别适用于处理大型数据集。可以使用 Dask 来并行加载和处理数据，充分利用多核处理器的性能。
* **向量化处理**：使用 pandas 的向量化操作（如 apply 或 map）来处理数据，避免显式循环。

## 代码实现
```python
def _load_dataset(self, data_dir, annot_path, split, shuffle=True):
        out_data = []
        # 使用 Dask 读取 CSV 文件
        annotation = dd.read_csv(annot_path, sep=',', usecols=DefectLabels + ["Filename", "Defect"])
        annotation = annotation.compute()  # 转换为 Pandas DataFrame
        self.imgPaths = annotation['Filename'].values
        classes = self.LabelNames
        class_to_idx = {classes[i]: i for i in range(len(classes))}
    
        # 将 DataFrame 转换为字典 {Filename: [labels]}
        annotation_dict = annotation.set_index('Filename')[classes].to_dict(orient='index')

        # tqdm 添加进度条
        for img_path in tqdm(self.imgPaths, desc="Loading dataset", total=len(self.imgPaths)):
            full_image_path = os.path.join(data_dir, split, img_path)
            labels = np.array(list(annotation_dict.get(img_path, {}).values()))
            out_data.append((full_image_path, labels))
    
        if shuffle:
            random.shuffle(out_data)
    
        return out_data, class_to_idx, classes
```