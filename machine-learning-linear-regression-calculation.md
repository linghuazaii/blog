Linear Regression矩阵计算
=========================

# 前言
&emsp;&emsp;在Coursera里，Andrew NG直接给定了Linear Regression的矩阵计算方法，但是并没有给出证明，这里做一个简单的推理。

# Linear Regression的历史
&emsp;&emsp;早在十九世纪初，Gauss就发不了一篇论文 - [method of least squares](https://en.wikipedia.org/wiki/Least_squares)，根据Gauss的公式一步步推导，就是Linear Regression，如下：  
&emsp;<img src = 'https://github.com/linghuazaii/blog/blob/master/image/machine-learning/linear-regression-form.png' />  
&emsp;&emsp;关于矩阵导数的求法，请参阅:[Matrix Differentiation](http://www.atmos.washington.edu/~dennis/MatrixCalculus.pdf)  
&emsp;&emsp;其中求出的矩阵θ就是训练的出model，是一个`p x 1`的矩阵，做预测的话 `y = Xθ`即可求得所有预测结果值。  
&emsp;&emsp;算下训练时间复杂度就是`max(O(n*p), O(p^3))`，所以当数据的特征并不多的时候，举个例子，特征为10000个的话，计算量级也就10^12,而且一般做个性化推荐也好，搜索也好，很难有这么多特征，所以直接矩阵运算比较快，也比较方便。特征非常多的时候就用上一篇[博文](https://github.com/linghuazaii/blog/wiki/Linear-Regression%28%E7%BA%BF%E6%80%A7%E5%9B%9E%E5%BD%92%29)里介绍的方法迭代就好。


# 小结
**GOOD LUCK, HAVE FUN!**
