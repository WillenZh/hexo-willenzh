title: 含L1正则项的最优化问题求解：Shooting Algorithm
tags:
  - 最优化
  - Math
categories:
  - 机器学习
date: 2016-09-09 22:23:00

---

## 前言
Shooting算法是W.J.Fu在1998年提出的，原始论文为《Penalized Regressions: The Bridge Versus the Lasso》，是一种针对包含L1正则项的最优化问题（如LASSO）的快速求解方法，据称效果与流行的LARS（Least Angle Regression）不相上下（或者更好）。本文主要参考《A tutorial on the LASSO and the ”shooting algorithm”》，介绍Shooting算法，并由此推广到Rotated Sparse Regression的问题求解上。
<!--more-->
## Shooting for LASSO：
### 最简单的单变量LASSO

$$\min_{\beta}h(\beta) = \frac{1}{2}||\bf{y}-\bf{x}\beta||^2_2+\lambda|\beta|, where\ \lambda>0$$

$x$和$y$均是$n\times 1$的向量，$\beta$是一个参数变量。目标函数由于L1项的存在，变得非平滑。
在此可以用一个变量$t$来替代：
$$\min_{\beta}\overline{h}(\beta) = \frac{1}{2}||\bf{y}-\bf{x}\beta||^2_2+\lambda t, where\ \lambda>0，t+\beta\geq0，t-\beta\geq0$$

可以证明，若$\beta^\*_1$是$h(\beta)$的解，$(\beta^\*,t^\*)$是$\overline{h}(\beta)$的解，则$\beta^\*_1=\beta^\*$。（具体证明参见《A tutorial on the LASSO and the ”shooting algorithm”》pp.7）

这个约束条件下的最优化求解，可以套用拉格朗日乘子：

$$L(\beta,t,\lambda_1,\lambda_2)=\frac{1}{2}||\bf{y}-\bf{x}\beta||^2_2+\lambda t -\lambda_1(t-\beta)-\lambda_2(t+\beta)$$

可以利用KKT条件求出解析解。
### 多变量LASSO
多变量LASSO的形式为：
$$\min_{\beta}h(\beta) = \frac{1}{2}||\bf{y}-\bf{x\beta}||^2_2+\lambda||\bf{\beta}||_1, where\ \lambda>0$$
其中，$X = [x_1,x_2,...,x_p]$，$\beta= [ \beta_1, \beta_2,..., \beta_p]^T$， {%math%}X(i) = [x_1,...,x_{i-1},x_{i+1},...,x_p]{%endmath%}， {%math%}\beta^{(-i)} =[ \beta_1, . . . ,  \beta_{i-1},\beta_{i+1}, . . . ,\beta_p]^T{%endmath%}

设$A^{(-i)}$表示除掉第i个元素后的$A$，问题可以转化为求解如下目标函数，并采用逐个更新的方式进行参数估计（选取一个待更新变量，固定其他变量）：
$$\min_{\beta_i} h'(\beta_i) = \frac{1}{2}||\bf{y_i}-\bf{x_i}\beta_i||^2_2 +\lambda |\beta_i| + \lambda ||\bf{\beta^{(-i)}}||_1$$
其中$\bf{y_i}=\bf{y}-X^{(-i)}\beta^{(-i)}$。从上式的形式可以看出，其实是将待更新的变量提了出来。
## Shooting for RSR（Rotated Sparse Regression)
在此举一个例子来说明如何将Shooting算法应用到其他包含L1约束条件的最优化问题求解中。该例子是上篇[博文](http://jayveehe.github.io/2016/09/06/Notes-of-High-Dimensional-LBP/)中提到的Rotated Sparse Regression的矩阵学习问题。
### 原始目标函数：

$$\min_{B,R}\ ||\bf{\matrix{R^T}\matrix{Y}}-\matrix{B^T}\matrix{X}||^2_2+\lambda||\matrix{B}||_1,\quad s.t.\ R^TR=I$$

在计算映射矩阵$\matrix{B}$时，其各列是相互独立的，可以并行计算。将$\matrix{B^T}$的第$i$行提出（也即$\matrix{B}$的第$i$列），下面用$b_i$表示$\matrix{B^T}$的第$i$行。令$\widetilde{Y}=\matrix{R^TY}$，则用$y_i$表示$\widetilde{Y}$的第$i$行，$N$为样本数量``n_Sample``。因此目标函数变为类似LASSO的形式：  

$$\min_{b_i}\ \frac{1}{N}||\bf{y_i}-\matrix{b_iX}||^2_2+\lambda||b_i||_1, where\   \lambda>0$$

采用坐标下降法进行参数估计，需要对{%math%}b_i{%endmath%}的每一维坐标进行估计。在对$b\_{i,j}$进行估计时，可以采用“固定其他变量，仅估计$b\_{i,j}$”的方式进行迭代估计。
此时求解函数为：

$$\min\_{b\_{i,j}}\ \frac{1}{N}||\widetilde{\bf{y\_i}}-\matrix{b\_{i,j}X\_j}||^2_2+\lambda||b_i^{(-j)}||\_1+\lambda |b\_{i,j}| , where\   \lambda>0$$

其中$\widetilde{\bf{y\_i}}=y\_i-b\_i^{(-j)}X^{(-j)}$
### RSR目标函数的Shooting求解
类似单变量的LASSO问题，则可以将L1项用t替代，等价的目标函数为：
$$\min\_{b\_{i,j}}\ \frac{1}{N}||\widetilde{\bf{y\_i}}-\matrix{b\_{i,j}X\_j}||^2\_2+\lambda||b\_i^{(-j)}||\_1+\lambda t , where\   \lambda>0, t+b\_{i,j}\geq0, t-b\_{i,j}\geq0$$
构建拉格朗日乘子：
$$L(b\_{i,j},t,\lambda_1,\lambda_2)=\frac{1}{N}||\widetilde{\bf{y\_i}}-\matrix{b\_{i,j}X\_j}||^2_2+\lambda||b\_i^{(-j)}||\_1+\lambda t -\lambda\_1(t-b\_{i,j})-\lambda\_2(t+b\_{i,j})$$

结合KKT条件并求偏导：  

1. $$\frac{\partial{L}}{\partial{b\_{i,j}}}=\frac{2}{N}(\matrix{b\_{i,j}X\_j}-\bf{\widetilde{y\_i}})X\_j^T+\lambda\_1-\lambda\_2=0 \Rightarrow b\_{i,j}=\frac{\frac{2}{N}\widetilde{y\_i}X\_j^T-(\lambda\_1-\lambda\_2)}{\frac{2}{N}X\_jX\_j^T}$$
2. $$\frac{\partial{L}}{\partial{t}}=\lambda-\lambda\_1-\lambda\_2=0 \Rightarrow \lambda=\lambda\_1+\lambda\_2$$

#### 分情况得出解析解：  

1. 若$\frac{2}{N}\widetilde{y\_i}X\_j^T-\lambda>0$，在满足假设与KKT的条件下，$\lambda\_2=0$，此时:$$b\_{i,j}=\frac{\frac{2}{N}\widetilde{y\_i}X\_j^T-\lambda}{\frac{2}{N}X\_jX\_j^T}$$
2. 若$\frac{2}{N}\widetilde{y\_i}X\_j^T+\lambda<0$，在满足假设与KKT的条件下，$\lambda\_1=0$，此时:$$b\_{i,j}=\frac{\frac{2}{N}\widetilde{y\_i}X\_j^T+\lambda}{\frac{2}{N}X\_jX\_j^T}$$
3. 其他情况下，由KKT条件可以推出:$$b\_{i,j}=0$$

至此即可逐步完成$b\_i$的估计。

## Tricks！
在实际操作中，我们需要注意减少重复计算。分析$b\_{i,j}$的表达式可知，若采用的是坐标下降法（CD），则在每一个针对坐标的iteration中（变化的量是j，遍历计算所有的坐标j），$\widetilde{\bf{y\_i}}=y\_i-b\_i^{(-j)}X^{(-j)}$都需要重新计算（即让目标函数朝梯度方向“走”一步，每更新一个j就应该重新计算一次$\widetilde{\bf{y\_i}}$，这代表了$b\_i$得到了更新，进入了一个新状态）该计算至关重要，可以让算法更快收敛。
可以发现，在计算$b\_i^{(-j)}X^{(-j)}$时，变化的量只是下标为j-1的部分，其他部分与上一个iteration中的值一样。因此我们可以保存上一个iteration中的$b\_i^{(-j)}X^{(-j)}$值，设为$\vec{bi\_X}$，假设在本次迭代中，更新前的$b_{i,j}$为{%math%}old\_b_{i,j}{%endmath%}，更新后的为{%math%}new\_b_{i,j}{%endmath%}。则本次iteration后待更新的$b\_i^{(-j)}X^{(-j)}$公式为：
{%math%}
$b_i^{(-j)}X^{(-j)}=\vec{bi_X}-old\_b_{i,j}X_j+new\_b_{i,j}X_j=\vec{bi_X}+(new\_b_{i,j}-old\_b_{i,j})X_j$
{%endmath%}

同理，该trick可以推广到其他具体问题中，***此举可省去99%的计算量***