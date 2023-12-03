---
title: 张量分析推导 AI 反向传播中的经典案例
date: 2023-12-03 03:40:50
tags: ai
categories: AI
mathjax: true
---

# 张量分析推导 loss 对 $\mathbf{X}$ 的导数
## 利用张量分析书写矩阵乘法

在一个标准的神经网络线性层中，输出和输入的关系如下

$$
\tag{1}
\mathbf{T} = \mathbf{X} \mathbf{W} + \mathbf{b}
$$

其中
- $\mathbf{T}$ 是一个 $b \times h$ 的矩阵，这里 $b$ 一般是 `batch_size`，$h$ 是隐藏层大小
- $\mathbf{X}$ 是一个 $b \times m$ 的矩阵 ，这里 $m$ 一般是 `emb_size * block_size`
- $\mathbf{W}$ 是一个 $m \times h$ 的矩阵
- $\mathbf{b}$ 在 `pytorch` 中一般不是一个矩阵，是一个长度为 $h$ 的向量，
通过使用了 `pytorch` 的扩展第一列扩展到所有列

我们可以使用张量分量的形式来表示 

$$
\tag{2}
\mathbf{T} = (x_{ik} w^{k}_{\cdot j} + b_j)\mathbf{g}^i\mathbf{g}^j
$$

## 用张量推导导数

在深度学习中，上述说的 $\mathbf{T}$ 其实会是 $loss$ 的一个函数，然后 $\mathbf{T}$ 又是 $\mathbf{X}$ 的函数，写出来是下述的样子

$$
loss = l(\mathbf{T}(\mathbf{X}))
$$

上述 $l$ 可以看做是一个二阶张量 $\mathbf{T}$ 的标量函数，$\mathbf{T}$ 是二阶张量 $\mathbf{X}$ 的二阶张量函数，而张量函数的导数存在以下链式求导规则

$$
\frac{\partial l}{\partial \mathbf{X}} =
\frac{\partial l}{\partial \mathbf{T}}
\vcentcolon
\frac{\partial \mathbf{T}}{\partial \mathbf{X}}
$$

将上式按照张量分量展开如下

$$
\tag{4}
\begin{aligned}
\frac{\partial l}{\partial \mathbf{X}}
&= \frac{\partial l}{\partial t_{ij}}\mathbf{g}_i\mathbf{g}_j
\vcentcolon
\frac{\partial t_{kl}}{\partial x_{mn}}\mathbf{g}^k\mathbf{g}^l\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{ij}}
\frac{\partial t_{kl}}{\partial x_{mn}}
\delta_i^{\cdot k}
\delta_j^{\cdot l}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{ij}}
\frac{\partial t_{ij}}{\partial x_{mn}}
\mathbf{g}_m\mathbf{g}_n \\
\end{aligned}
$$

于是我们得到了基于张量分量的链式法则。在公式 $(4)$ 中，其实 $\frac{\partial l}{\partial t_{ij}}$ 是一个已知字段，在上一次 `backward` 中已经计算出来了，现在我们的目标是计算 $loss$ 各相对于 $\mathbf{W}$、$\mathbf{X}$ 和 $\mathbf{b}$ 的导数，将该链式法则带入 $(2)$ 式


$$
\tag{5}
\begin{aligned}
\frac{\partial l}{\partial \mathbf{X}}
&= \frac{\partial l}{\partial t_{ij}}
\frac{\partial (x_{ik} w^{k}_{\cdot j} + b_j)}{\partial x_{mn}}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{ij}}
\delta_i^{\cdot m} \delta_k^{\cdot n}
w^k_{\cdot j}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{mj}}
w^n_{\cdot j}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{mj}}
(w_j^{\cdot n})^\top
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial \mathbf{T}} \mathbf{W}^\top 
\end{aligned}
$$


上式中 $(w_{ki})^\top$ 表示矩阵的转置。因此我们将上式改为矩阵形式就可以得到

$$
\tag{6}
\frac{\partial l}{\partial \mathbf{X}} =
\frac{\partial l}{\partial \mathbf{T}}\mathbf{W}^\top 
$$

由此，我们用很简单规整的数学推导推出了 $loss$ 相对于 $\mathbf{X}$ 的导数（或者称为梯度）

同样的思路，我们可以推导出 $\mathbf{W}$ 对应的导数


$$
\tag{7}
\begin{aligned}
\frac{\partial l}{\partial \mathbf{W}}
&= \frac{\partial l}{\partial t_{ij}}
\frac{\partial (x_{ik} w^{k}_{\cdot j} + b_j)}{\partial w_{mn}}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{ij}}
x_{ik}
\delta^{km} \delta_{j}^{\cdot n}
\mathbf{g}_m\mathbf{g}_n \\

&= \frac{\partial l}{\partial t_{in}}
x_i^{\cdot m} \mathbf{g}_m\mathbf{g}_n \\

&= (x^m_{\cdot i})^\top \frac{\partial l}{\partial t_{in}}
\mathbf{g}_m\mathbf{g}_n \\

&= \mathbf{X}^\top \frac{\partial l}{\partial \mathbf{T}}
\end{aligned}
$$

和 $\mathbf{b}$ 对应的导数

$$
\tag{8}
\begin{aligned}
\frac{\partial l}{\partial \mathbf{b}}
&= \frac{\partial l}{\partial t_{ij}}
\frac{\partial (w_{ik} x^{k}_{\cdot j} + b_j)}{\partial b_{m}}
\mathbf{g}_m \\

&= \frac{\partial l}{\partial t_{ij}}
\delta_j^{\cdot m}
\mathbf{g}_m \\

&= \frac{\partial l}{\partial t_{im}}
\mathbf{g}_m \\

&= \sum_i \frac{\partial l}{\partial t_{im}}
\mathbf{g}_m \\
\end{aligned}
$$

这里比较特殊，最终得到的结果其实是 $\frac{\partial l}{\partial \mathbf{T}}$ 按照行求和得到了一个长度为 $h$ 的向量。

将上述的公式表示为 `pytorch` 代码即为

```python
dX = dT @ W.T
dW = X.T @ dT
db = dT.sum(0)
```
