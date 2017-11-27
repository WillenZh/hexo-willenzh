title: Notes of High-Dimensional LBP
tags:
  - CV
categories:
  - 机器学习
date: 2016-09-06 16:25:00
---
## 前言
MSRA在CVPR 2013上的一篇《[*Blessing of Dimensionality: High-dimensional Feature and Its Efficient Compression for Face Verification*]( http://www.cv-foundation.org/openaccess/content_cvpr_2013/html/Chen_Blessing_of_Dimensionality_2013_CVPR_paper.html)》，相信已经成为很多人入门人脸相关算法的必读论文之一。网上关于这篇论文的文章，大多只是简单地介绍了一下论文中所提方法的思想与流程，很多实现的细节被一笔带过（或者相关博主也仅仅是阅读了论文而没做实现）。下面我将总结一下个人在实现该论文方法的过程中所遇到的问题与相关的思考。

<!--more-->

## 一、论文主要内容
论文的题目已经很完美地概括了其核心内容：高维度特征。论文要点如下：
1. 高维特征在人脸识别（Face Recognition)的相关任务中是很有必要的。该部分介绍了如何构建高维特征，并用数据证明了特征维度的提升可以带来识别准确率的提高（83%提升至93%+）。
2. 提出了基于Rotated Sparse Regression的压缩方式，利用稀疏性大幅度压缩模型大小（200倍），并仅仅受到低于0.1%的准确度损失。
3. 用各种实验数据证明方法的优越性。

### 1.1 作者所提出的方法主要流程
![main_process](http://7qnaj2.com1.z0.glb.clouddn.com/sparse_regression.jpeg?imageView2/2/w/600)

从图中和注释可知，在**训练阶段**：该方法首先从训练集中构建高维特征$X$，然后利用无监督的降维方法PCA及有监督的降维方法（如LDA）进行特征降维，得到低维特征$Y$。接着利用原始的高维特征及降维后得到的低维特征，学习出一个映射矩阵$B$。
在**测试阶段**：用同样的方法构建高维特征，并利用映射矩阵$B$直接获取降维后的特征$Y$



下面主要讲一下高维特征的构建及**Rotated Sparse Regression**。

## 二、构建高维特征
一般地，利用LBP特征对一幅图像进行建模，需要将图像分割为若干个blocks，并求得每个block中LBP数值的归一化统计直方图，将每个block的归一化直方图连接成一个长向量，即可利用该向量表示图像的特征。
而在人脸建模领域，可以仅对人脸的关键点（人脸对齐后得到的landmarks）周边进行建模，如此可以去掉大部分冗余信息。作者提出在多尺度下对人脸关键点周边进行采样建模，以此获取高维特征。作者同时发现，选取更稠密的关键点能够获得更好的效果。
![](http://7qnaj2.com1.z0.glb.clouddn.com/hd_features_construction.jpeg?imageView2/2/w/600)
如上图所示，在进行多尺度采样时，需要设定一个固定的``block_size``和``cell_size``（而不是改变这两者的值来达到缩放效果）。从小到大的缩放尺度能够分别获取到局部细节特征和全局（相对的）轮廓特征。
易知，假定选取了``n``个关键点landmarks，使用``block_size``大小的block窗口，进行``s``次缩放，且每个点的LBP值有`k`种可能（这取决于采用的LBP种类，参见另一篇博文《[图像LBP特征建模](http://jayveehe.github.io/2016/08/28/NLPtoCV/)》），则最终每幅图片的特征维度为： 
{% math %}
$n\_Feature=n*k*s*block\_size^2$
{% endmath %}
我在实现该方法时，选取了``n=25``，``k=59``，``s=3``，``block_size=4``，最终的特征维度是70800维。

## 三、Rotated Sparse Regression
数据证明，高维特征能够有效地提升模型效果，这也是机器学习中的一个常识（当然一味地提高维度也会造成过拟合）。但是高维度往往带来高复杂度（空间复杂度与时间复杂度都会指数级增长），给计算和存储带来很大的问题，即“维度灾难”。
作者提出了一个高效的稀疏映射方法对高维特征进行压缩，在时间与空间上极度优化以后，仅仅带来了很少的准确性损失，不得不再次感慨标题中的*Blessing of Dimensionality*用得十分传神。
Rotated Sparse Regression的主要目标是求解一个稀疏的映射矩阵$B$，其目标函数为：

$$\min_{B,R}\ ||\bf{\matrix{R^T}\matrix{Y}}-\matrix{B^T}\matrix{X}||^2_2+\lambda||\matrix{B}||_1,\quad s.t.\ R^TR=I$$

其中$Y$是按列存储的目标特征矩阵（低维,``n_Target``行，``n_Sample``列），$X$是按列存储的原始特征矩阵（高维，``n_Source``行，``n_Sample``列），$B$是``n_Source``行``n_Target``列映射矩阵，$R$为``n_Target``行，``n_Target``列的旋转矩阵。
在进行参数求解时，可以采用固定$R$计算$B$和固定$B$计算$R$的两步迭代法进行求解。
### 3.1 计算$\matrix{B}$
可以发现，在计算映射矩阵$\matrix{B}$时，其各列是相互独立的，可以并行计算。将$\matrix{B^T}$的第$i$行提出（也即$\matrix{B}$的第$i$列），下面用$b_i$表示$\matrix{B^T}$的第$i$行。令$\widetilde{Y}=\matrix{R^TY}$，则用$y_i$表示$\widetilde{Y}$的第$i$行，$N$为样本数量``n_Sample``。因此目标函数变为类似LASSO的形式：  

$$\min_{b_i}\ \frac{1}{N}||\bf{y_i}-\matrix{b_iX}||^2_2+\lambda||b_i||_1, where  \lambda>0$$

若采用SGD方法进行参数估计，则需要求解目标函数的梯度：
{% math %}
$\frac{\partial{}}{\partial{b_{i,j}}}=\frac{2}{N}(\bf{b_i}X-\bf{y_i})X_j^T+\lambda sign(b_{i,j})$
{% endmath %}
其中，$b_{i,j}$是$b_i$的第$j$个坐标上的值，$\bf{X^T_j}$是$X$矩阵的第$j$行的转置。需要注意的是，1-范数在0处无导数，此处使用了符号函数进行次梯度近似。
得到梯度后，即可采用梯度下降的套路进行参数更新，其中涉及的步长设置、L1稀疏度控制等，此处不做赘述（下一篇再讲讲最优化方法）。
并行求解各$b_i$后，重新组合成矩阵$B$。
### 3.2 计算$\matrix{R}$
论文中给出了R的闭式解，设$\matrix{YX^TB}$的SVD结果是$\matrix{UDV^T}$，则：
$$R=UV^T$$
如此便完成了一次更新。

## 四、补充
1. 梯度中包含的$\frac{1}{N}$是为了求梯度均值而加，经验所致。可以用更小的学习率替代，以防止发散。
2. 由于加入了旋转矩阵$R$，可以带来一部分的旋转自由度，进而有利于增加映射矩阵$B$的稀疏性。同时作者也指出并用数据证明，旋转矩阵的加入是该方法在具备高压缩比的同时又仅有很低的准确率损失的关键所在。
3. 相较于直接使用PCA等降维方法，稀疏矩阵映射的方法能够大大减少计算量及空间占用。


---

接下来的文章会介绍一下参数估计的最优化方法及相关的Tricks。