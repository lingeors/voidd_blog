---
title: OpenCV 安装
date: 2017-05-31 10:57:40
tags:
    - OpenCV-Install
    - Python
---

> OpenCV安装的坑真的很多，它不仅可能出现在安装初期，还可能出现在你用opencv开发一段时间之后，前者还好，因为总归有可能解决（windows下持疑），如果遇到后者，那就可能真是gg了。本文介绍了整个安装过程以及各个步骤需要注意的问题，包括windows，Ubuntu系统以及 OpenCV2、3两个版本

<!-- more -->
最近 嵌入式课程设计 的其中一个功能模块是人脸识别，因为时间关系，决定采用 OpenCV，一天速成后又踩了几天的坑，发现网上能找到的关于 opencv-python 的文档较少，因为版本不同也存在一些坑，因此后续希望能通过几篇文章将这几天遇到的、了解到的相关信息以及相关python实现记录下来（c++实现文档还是挺多的）。

## OpenCV 简介
维基百科介绍
> OpenCV的全称是Open Source Computer Vision Library，是一个跨平台的计算机视觉库。OpenCV可用于开发实时的图像处理、计算机视觉以及模式识别程序。

上面的解释应该是很清晰的，下面就简单说几点
1. OpenCV本质上就是一个 C++库，通过它我们可以通过非常简单的代码实现诸如 人脸识别、物体识别、图像分区等复杂的功能
2. OpenCV提供了对多种语言的支持，如 Java、Python、Matlab 等，但因为它是用 C++ 写的，所以C++相关文档最齐全，使用c++编程相对来说也更简单
3. 自3.x版本后，部分功能因为还未稳定被[分离][2]出来，如果需要用到，则需要自行编译安装，初学者很可能不了解这点，导致后期遇到很多不必要的麻烦

## 安装环境
我是在两台电脑上使用opencv，环境分别为
1. windows：win10，64位，Python3.4
2. linux: ubuntu16, 64为，Python3.5, Python2.7

## OpenCV 安装
先说我遇到的几个坑，希望可能大家可以避开，避免浪费时间
1. 关于版本选择：我一开始在win10下使用，选择了 3.x 版本，但几乎所有安装教程都没有告诉读者3.x将一些功能独立出来了（如后面会说的分类器训练），因此后面还需要重新编译安装就会很坑（并且我是使用whl安装的，gg）。
2. 关于Python版本的。我在win10下是Python3.4，但OpenCV 3.x好像对py3.x支持不是很好，可能因为这个问题导致常规方法安装失败，于是我使用了 whl 文件安装，这样又导致了后面编译安装 contrilb 失败（应该是）。因此如果是要长期使用 OpenCV 的话，可以考虑使用 低版本py或者直接使用liunx，可以省不少麻烦

windows上安装：
1. 因为我们后面使用Python开发，因此首先要安装Python。如上所述选择Python版本，如果是使用 `python2.x`，则安装步骤很简单，到官网下载 exe 文件，双击安装（假设安装目录为 `d:\opencv`）。安装完毕，将 `d:\opencv\build\python\2.7\$version\cv2.pyd` 复制到 `~/path-to-python/Lib/site-packages` 即可。测试 `import cv2` 导入成功即为安装成功
2. 如果使用 `python3.x` 版本。我按照上述过程是安装失败的，错误信息如下
    ```python
        >>> import cv2
        Traceback (most recent call last):
          File "<stdin>", line 1, in <module>
        ImportError: DLL load failed: 找不到指定的模块。
    ```
    stackoverflow 上说要安装一个 vc++ 库 `https://www.microsoft.com/en-us/download/details.aspx?id=48145 ` 我下载后，该库安装失败，提示已经安装过。此时意识到可能是python版本问题，因此采用了 whl 文件的安装方式，加上 OpenCV3.x 也算是坑了自己一把。
3. whl安装
    - 下载相关文件 `http://www.lfd.uci.edu/~gohlke/pythonlibs/#opencv`，如 `opencv_python-3.1.0-cp34-cp34m-win_amd64.whl` 置于任意目录下，执行 `pip install opencv_python-3.1.0-cp34-cp34m-win_amd64.whl` 再次测试 `import cv2` DONE
    - 整个过程是相当简单的，主要问题在于 whl 文件的选择上。第一步可能是要注意 32 位和 64 位（注意是python的位数，64位电脑也可能装的是32位的py）
    - 选择正确位数的文件后，仍然可能有问题，错误信息如下：
        ```bash
        opencv_python-3.2.0-cp35-cp35m-win_amd64.whl is not a supported wheel on this platform 
        ```
        具体原因不清楚，没深入研究，主要是要搞清楚文件名各个部分的具体含义 (如果暂时不想理，就多试几遍把 :-)
    - 此步骤参考自 [Install OpenCV 3 with Python 3 on Windows][5]
4. OpenCV 3.1.0 + opencv_contrib编译
     - 前面说过，opencv3.x 将部分不稳定功能独立出来，作为 opencv_contrib 独立存在，具体见[这里][1]和[这里][2]，因此当我们需要某些功能时，就需要再安装 [opencv_contrib][2]
     - 因为是编译安装，为了避免以后出现不必要的麻烦，建议一开始就通过这种方式
     - 具体安装步骤见此处 [OpenCV 3.1.0 + opencv_contrib编译（Windows）][3] 安装的时候 opencv 和 contrib 的版本一定要一致，否则是肯定失败的。
5. 在windows折腾了很久，特别是如果你没有安装 `visual studio` 的话，那将会非常痛苦。因此还是建议在 `linux` 下安装使用 `opencv`。具体安装步骤见这里 [ubuntu16.04下安装opencv2.4.13][4]，相对windows，非常简单。我的安装环境是 `ubuntu 16.xx 64bit`。为了方便，我直接安装的是 2.x 的最新版本。

## 测试
执行如下脚本
```python
import cv2
img = cv2.imread("test.jgp")
cv2.imshow("win_name", img)
cv2.waitKey(0) 
```
运行后应该可以得到图像
![](/images/OpenCV-Install/opencv-test.png)

## 最后
首先，在windows上安装opencv是很麻烦的，又如果是 3.x 版本，因为 contrib 的存在，必须编译安装，这是 windows 天生就不擅长的事情。因此如果是在 windows 下，有如下建议
1. 如果只是玩一玩，建议直接使用 whl 文件安装，最简单
2. 如果会涉及 contrib 相关内容，可以安装 2.x 版本
3. 如果以后会经常用到，直接用 linux 吧

[1]: https://stackoverflow.com/questions/33620527/opencv3-and-python-2-7-on-virtual-enviorement-attributeerror-module-object/39846029
[2]: https://github.com/opencv/opencv_contrib
[3]: http://blog.csdn.net/linshuhe1/article/details/51221015
[4]: http://blog.csdn.net/duwangthefirst/article/details/52288821
[5]: https://www.solarianprogrammer.com/2016/09/17/install-opencv-3-with-python-3-on-windows/
