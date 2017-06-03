---
title: OpenCV 基本使用
date: 2017-05-31 11:55:17
tags:
    - OpenCV
    - Python
---

> 此文章主要介绍 OpenCV 的基本使用。读者需具备 Python 的基础知识

<!-- more -->

## 前言
本文将以例子为导向，并分为三个部分进行叙述：代码实例、代码讲解、相关概念。希望能帮助初学者快速了解opencv的基本使用。执行前读者需安装 `numpy` 包。通过 setup或pip 安装均可

## 图片操作流程
如下代码：
```python
import cv2
img_path = "../img_before/lala.png"
# read img
img = cv2.imread(img_path, 0)
# print(type(img))
cv2.imshow("win_name", img)
k = cv2.waitKey() # 阻塞，作用为保留图片窗口
```
上述代码描述了图片载入到销毁的基本流程，函数名是自解释的。主要关注几点
1. img 是一个 `numpy.ndarray` 类型的变量。直接 print 出来可以发现是一个矩阵
2. 在计算机中，图片可以被存储为像素点，因此一个 x*y 像素大小的图片可以被存储为 x*y 大小的矩阵。在opencv中，每个像素点的表示也有多种方式，这里默认使用的是 RGB 模式，即 `blue, green, red` 三原色的组合。注意opencv表示的顺序为 BGR。后面对图像的切割等处理均是对矩阵的处理。了解这点很重要
3. `img[100, 100] = [255, 255, 255]` 设置具体位置的像素值
3. `imshow` 的第一个元素为展示图片的 window 窗口名称，窗口名称必须是唯一的

## 键盘交互与文件保存
```python
import cv2
img_path = "../img_before/lala.png"
img = cv2.imread(img_path, 0)
cv2.imshow("win_name", img)
k = cv2.waitKey()
if k == 27:
    cv2.destroyAllWindows() # key esc
elif k == ord('s'):
    print("your press key 's'")
    cv2.imwrite("newImgName.jpg", img)
    cv2.destroyAllWindows
```
1. `waitKey`方法阻塞进程，等待键盘输入。接收到后返回按键的ascii
2. `imwrite`方法将img保存成文件

## ROI操作
首先要先了解两个概念
1. 通道(Channels)：前面我们说过，opencv 中通过 BGR 三原色来表示一个像素点，此时我们称一个像素点具有三个维度，而维度的个数即为通道的个数。因此默认打开的文件是 3通道的。而如果将图片转换为 gray，即灰度，此时一个像素只需要一个数值即可描述，我们称之为 1通道 或 单通道
2. ROI(*region of image*)：即图像区域。通俗地说，就是图片的一部分。获取一个 ROI 即获取图片的一部分。在计算机中表示为获取图像矩阵的一部分。具体到 python，表示为对 img 的切片操作
```python
import cv2
img_path = "../../img_before/lala.png"
img = cv2.imread(img_path, 0)
roi = img[100:200, 0:100]
cv2.imshow('img', img)
cv2.imshow('roi', roi)
cv2.waitKey(0)
```

对于`roi = img[y:y+h, x:x+w]` 需要注意的几个点
1. 坐标原点在左上角
2. y表示离坐标原点的纵向距离，h表示离y的距离。x表示横向距离
3. roi 操作本质是要理解计算机操作图片实际上操作的是什么。理解了这点其他就很好理解，比如图片合并 `img[100:200, 0:100] = img2[0:100, 0:100]` 使用img2某区域图像覆盖img某区域图像（需要保证二者大小一致）

## 画图函数
这里只介绍三种常用的，剩余的查阅 [OpenCV 2.3.2 documentation][1]
```python
import cv2
img_path = "../../img_before/lala.png"
img = cv2.imread(img_path, 0)
height, width = img.shape # 获取图片的大小
# drawing line
# 起点坐标，终点坐标（注意原点在左上角）
# 画线颜色，线条像素点
cv2.line(img,(0,0), (width-1, height-1), (255,0,0),5)
# drawing rectangle
# 矩形左上角坐标、右下角坐标
cv2.rectangle(img, (0, 0), (int(width/5), int(height/5)), (0, 0, 0), 5)
# drawing circle
# circle(img, center, radius, color, thickness, lineType)
# thickness 如果为正，表示圆圈线条往外延伸的像素大小，如果为负，表示color充满整个源
cv2.circle(img, (30,30), 20, (255, 255, 255), 10)
cv2.imshow('img' ,img)
cv2.waitKey(0)
```


## 摄像头操作
```python
import cv2

# 捕捉摄像头
cap = cv2.VideoCapture(0)

# 通过获取摄像头各帧并展示出来
while (1) :
    # take each frame, type of frame is numpy.ndarray
    _, frame = cap.read()

    cv2.imshow('frame', frame)
    k = cv2.waitKey(5)

    if k == 27 :
        break

cv2.destroyAllWindows()
```
摄像头操作是非常简单的，获取数据源(VideoCapture) -> 获取视频每一帧并显示出来(imshow) -> 不断刷新(while)

## 阀值操作(Thresholding)
阀值，可以理解为一个人为定义的临界点，在opencv中，阀值主要用来对特定像素点执行操作，简单而言就是当像素点超过阀值，则可能会被设置为某个值，否则可能会被设置为另一个值。
opencv提供了几个函数，这里我们只讨论 `threshold, adaptiveThreshold` 两个

**对于 threshold**
```python
cv2.threshold(src, thresh, maxval, type[, dst]) → retval, dst
```
函数原型如上，我们先解释type。type指像素点与阀值比较后的执行策略，有如下几种

突然发现不支持 LaTeX....
又不想截图，偷懒一下，诸位移步[这里][2]，不难。要关注一下几种type类型的val值表示。它们和上面的参数名很多对应。看懂了 几类 type，也就看懂了上述函数的参数

关于此函数，还需要关注一点：thresh阀值以及设置的maxval均是标量，意味着该函数只支持单通道图像，也就是说，src 必须先转换成灰度图像。
代码如下：
```python
# 这里使用了 matplotlib包，需要安装。如不安装，
# 则代码稍微修改即可
import cv2
import matplotlib.pyplot as plt

img1_path = "../../img_before/lala.png"

img = cv2.imread(img1_path)
img = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY) # 转换成灰度图像

# 测试多种阀值函数
ret,thresh1 = cv2.threshold(img, 127, 255, cv2.THRESH_BINARY)
ret,thresh2 = cv2.threshold(img, 127, 255, cv2.THRESH_BINARY_INV)
ret,thresh3 = cv2.threshold(img, 127, 255, cv2.THRESH_TRUNC)
ret,thresh4 = cv2.threshold(img, 127, 255, cv2.THRESH_TOZERO)
ret,thresh5 = cv2.threshold(img, 127, 255, cv2.THRESH_TOZERO_INV)

# 通过 plt 画图，可以将多个图像画在一个页面。工科生应该都知道吧（毕竟matlab饶过谁）
# 亦可通过imshow。
titles = ['origin', 'THRESH_BINARY', 'THRESH_BINARY_INV', 'THRESH_TRUNC', 'THRESH_TOZERO', 'THRESH_TOZERO_INV']
images = [img, thresh1, thresh2, thresh3, thresh4, thresh5]

for i in range(6):
    plt.subplot(2, 3, i+1), plt.imshow(images[i], 'gray')
    plt.title(titles[i])
    plt.xticks([]), plt.yticks([])

plt.show()
```
结果如下：
![](threshold.png)


**对于adaptiveThreshold**
`threshold` 使用固定的 阀值，而有时根据图片的不同部分使用不同的阀值效果可能更好。
`adaptiveThreshold` 可以根据图片的不同部分使用不同的阀值。
阀值选择的策略目前有两种。分别是：1.  `ADAPTIVE_THRESH_MEAN_C` 相邻区域的平均值； 2. `ADAPTIVE_THRESH_GANSSIAN_C` 相邻值高斯窗的和
函数原型如下
```python
cv2.adaptiveThreshold(src, maxValue, adaptiveMethod, thresholdType, blockSize, C[, dst]) → dst

adaptiveMethod 即为阀值选择策略
blockSize 为选取的区域大小
剩下的和 threshold 基本一致
```

代码如下：
```python
import cv2
import matplotlib.pyplot as plt

img1_path = "../../img_before/lala.png"

img = cv2.imread(img1_path, 0)

ret, th1 = cv2.threshold(img, 127, 255, cv2.THRESH_BINARY)
th2 = cv2.adaptiveThreshold(img, 255, cv2.ADAPTIVE_THRESH_MEAN_C, cv2.THRESH_BINARY, 11, 2)
th3 = cv2.adaptiveThreshold(img, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C, cv2.THRESH_BINARY, 11, 2)

titles = ['Original Image', 'Global Thresholding (v = 127)',
            'Adaptive Mean Thresholding', 'Adaptive Gaussian Thresholding']
images = [img, th1, th2, th3]

for i in range(4):
    plt.subplot(2, 3, i+1), plt.imshow(images[i], 'gray')
    plt.title(titles[i])
plt.show()
```
结果如下：
![](adaptiveThreshold.png)

## 最后
上面简述了opencv的几种简单操作并给出了相应的 python 实现，对于后面人脸识别，上述语言基础应该是差不多了

## 参考
[OpenCV: OpenCV-Python Tutorials][4]
[Image Filtering — OpenCV 2.3.2 documentation][3]

[1]: http://www.opencv.org.cn/opencvdoc/2.3.2/html/modules/core/doc/drawing_functions.html?highlight=circle#cv2.circle
[2]: http://www.opencv.org.cn/opencvdoc/2.3.2/html/modules/imgproc/doc/miscellaneous_transformations.html?highlight=threshold#cv2.threshold
[3]: http://www.opencv.org.cn/opencvdoc/2.3.2/html/modules/imgproc/doc/filtering.html?highlight=medianblur
[4]: http://docs.opencv.org/3.1.0/d6/d00/tutorial_py_root.html