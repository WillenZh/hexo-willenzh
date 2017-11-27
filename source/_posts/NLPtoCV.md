title: NLP选手的CV初探：图像LBP特征建模
tags:
  - CV
categories:
  - 机器学习
date: 2016-08-28 18:44:00
---

#### 前言：
最近因为实习的关系，半只脚走进了CV的门里，剩下半只还在自己折腾。作为在研究生阶段搞了快2年NLP的渣渣，机器学习的一些基本理论还是懂一些的，所以大部分还是从ML通用的方面开始入手。下面总结一下最近接触和学习到的东西。
<!--more-->

---

## 涉及的领域
CV（Computer Vision）总体上说，和NLP一样都是一个很大很大的坑：研究点很多，涉及的应用领域也很广。单就本次接触到的人脸领域而言，就包括（且不限于）：人脸检测（从图像中找出人脸部分）、人脸对齐（定位人脸上的关键点）、人脸匹配（匹配出同一个人在不同光照、表情、背景等条件下的照片）、人脸属性标注（年龄、性别、种族等特征信息的识别）等等。

具体而言，我做的是人脸属性标注这块。这是一个很典型的分类、回归问题，只需要一些图像方面的知识，加上基本的统计学习基础即可胜任。

### 人脸属性标注
大众可能相对熟悉的人脸属性标注应用，说起来应该可以算是前段时间火起来的[HowOld.net](http://www.how-old.net)以及微软小冰提供的合影测年龄、颜值打分等应用。此外，与一些商用的视频监控设备配合，可以在商场、路口等人流量大的地方及相关的场景下做人群信息的统计分析。
仅以属性标注这一应用来说，背后的算法不复杂，主要涉及两点：人脸特征的提取与标注集的准备。
和一般的ML套路一样，接下来的内容大部分会集中在特征的提取之上。


## 图像处理中的特征（Feature）
常用的图像特征有SIFT、Haar、HoG、Gabor、LBP等，网上有很多[介绍的文章](http://blog.csdn.net/zouxy09/article/details/7929348)，在此不做赘述。由于我主要参考的是MSRA在2013年发表的论文《[Blessing of Dimensionality: High-dimensional Feature and Its Efficient Compression for Face Verification](http://www.cv-foundation.org/openaccess/content_cvpr_2013/html/Chen_Blessing_of_Dimensionality_2013_CVPR_paper.html)》，所以下面稍微介绍一下LBP特征。

### 原始的LBP（Local Binary Patterns）特征
对于任意一副灰度图，可以用一个二维矩阵$M$表示，其中$M_{i,j}$为坐标(i,j)上的像素灰度值。则坐标图中(i,j)处的LBP值可以按照如下的方法计算：
1. 首先选定一个LBP算子窗口大小，通常情况下选取3*3的窗口，并以目标点(i,j)为窗口中心。
2. 以目标点(i,j)的灰度值$M_{i,j}$作为参考值，分别与(i,j)周边的8个灰度值进行比较，大于参考值为1，否则为0.
3. 如此可以得到一个8 Bit的序列，进行进制转换后可得到0~255之间的一个数字。

![](http://img.my.csdn.net/uploads/201208/31/1346398527_7290.jpg)

对于某一幅图片，常用的LBP建模表示方法是：
1. 首先将整幅图划分为k个小区域（cell）。
2. 对于每个cell中的一个像素，计算其LBP值。
3. 计算每个cell的直方图，即每个数字（假定是十进制数LBP值）出现的频率；然后对该直方图进行归一化处理。如此，每个cell可以得到一个256维的特征向量。
4. 最后将得到的每个cell的统计直方图进行连接成为一个特征向量，也就是整幅图的LBP纹理特征向量（k*256维）；

### 旋转不变性（Rotation Invariant）LBP
易知，原始的LBP特征具有灰度不变性，即一张图篇的总体灰度不会影响到局部的LBP特征；但是却不具有旋转不变性，即图片如果发生了一定的旋转，则某一个像素的LBP会由于二进制序列的顺序变化而产生一定的差异。
针对这一个问题，旋转不变LBP被提出。通过对二进制序列进行循环移位得到一组序列值，并以这组序列值的最小值作为这一组序列的代表。一个简单的例子参见下图：
![](http://img.my.csdn.net/uploads/201208/31/1346398586_1563.jpg)
如此一来，得到的LBP特征就不会因为图片的旋转而有所变化。

### Uniform Pattern LBP
上述提到的LBP算法，会用统计直方图进行图像建模。在实际情况中，直方图的结果往往是稀疏的，这会造成很大的浪费并引起维度灾难，Uniform Pattern LBP就是为解决这一个问题而提出的：
>Ojala提出了采用一种“等价模式”（Uniform Pattern）来对LBP算子的模式种类进行降维。Ojala等认为，在实际图像中，绝大多数LBP模式最多只包含两次从1到0或从0到1的跳变。因此，Ojala将“等价模式”定义为：当某个LBP所对应的循环二进制数从0到1或从1到0最多有两次跳变时，该LBP所对应的二进制就称为一个等价模式类。如00000000（0次跳变），00000111（只含一次从0到1的跳变），10001111（先由1跳到0，再由0跳到1，共两次跳变）都是等价模式类。除等价模式类以外的模式都归为另一类，称为混合模式类，例如10010111（共四次跳变）

Uniform Pattern LBP总共只有58个值，相比于原始的256个值，减少了很多。

---

下一篇博客我们讲讲High-dimensional LBP的相关内容。


## 参考文献

1. 《目标检测的图像特征提取之（二）LBP特征》 http://blog.csdn.net/zouxy09/article/details/7929531
2. 《Blessing of Dimensionality: High-dimensional Feature and Its Efficient Compression for Face Verification》  http://www.cv-foundation.org/openaccess/content_cvpr_2013/html/Chen_Blessing_of_Dimensionality_2013_CVPR_paper.html