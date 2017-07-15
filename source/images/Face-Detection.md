---
title: Face-Detection
date: 2017-06-06 23:32:32
tags:
    - opencv
    - face-detection
---
> 此文章主要介绍使用 opencv 实现人脸识别的过程

## 概述
使用opencv进行人脸识别操作非常简单，这里我们使用“特征脸算法”。
简单来说，特征脸指用于人脸识别问题的一组特征向量，这些特征向量是从高维度矢量空间的人脸图像的协方差矩阵计算而来。
操作步骤大概如下：
- 获取训练集
- 对数据集进行训练获得相关特征脸信息并保存到xml文件中
- 对要进行检测识别的人脸同样应用特征脸算法，将提取的信息与从训练集得到的数据进行比对，在误差阀值内则为同一张人脸

## 获取训练集
训练集为一些目标人脸的集合，该集合元素要求图片的大小一致。我们通过一个小脚本来实现，如下：
```python
# 检测人脸并返回人脸位置信息
def faceDetection(fonttalfaceFilePath, grayImg):
    face_cascade = cv2.CascadeClassifier(fonttalfaceFilePath)
    faces = face_cascade.detectMultiScale(grayImg,
                                          1.2,
                                          4,
                                          cv2.CASCADE_SCALE_IMAGE,
                                          (100, 100),
                                          (400,400)
                                          )
    return faces
```
这里使用了python自带的人脸数据，先通过 `cv2.CascadeClassifier` 载入xml文件，该xml文件位于 `path/to/opencv/sources/data/haarcascades`，再通过返回的人脸识别器对要识别的图像进行识别，最后返回结果信息。
`detectMultiScale` 各参数如下：grayImg-待识别图像的灰度表示，


[1]: http://img.ivsky.com/img/tupian/pre/201612/05/boshisheng_biye-002.jpg