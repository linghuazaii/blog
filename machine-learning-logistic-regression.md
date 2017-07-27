Andrew NG. Logistic Regression小结与示例
========================================

### Logistics Regression
&emsp;&emsp;在统计学上，Logistics Regression的例子有不少，虽然叫Regression，其实是Supervise Classification。举个教科书上经常举的例子，就是给定一堆肿瘤的数据，然后根据这些数据去预测未来的肿瘤是良性还是恶性。再具体一点比如说肿瘤的大小，患者年龄，肿瘤良性/恶性，给定肿瘤大小和患者年龄，如何预测肿瘤是良性还是恶性的问题。这就属于一个classification的问题。看下图：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/data.png' />  
&emsp;&emsp;数据有三个特征，`grade1`, `grade2`, 以及`positive/negative`，Logistics Regression就是用来将数据进行分类，找boundary。

### 理论
&emsp;&emsp;和Linear Regression不同的是，我们的`h(x)`变了。因为预测结果只有0和1，`h(x)`如下：  
<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/logistic_regression_cost.png' />  
&emsp;&emsp;这个是`g(x) = 1 / (1 + e^-x)`的图像：  
&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/sigmoid_function.png">  
&emsp;&emsp;Cost函数的倒数如下：  
&emsp;<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/logistic_regression_cost_gradient.png'>  
&emsp;&emsp;和Linear Regression的倒数一样，但是不可以通过`(X'X)^-1X'Y`的方式计算β数组。证明过程见：[derivative of cost function for Logistic Regression](https://math.stackexchange.com/questions/477207/derivative-of-cost-function-for-logistic-regression).  
&emsp;&emsp;由于选择不同的训练模型得到的结论还是差距很大的，而且现实生活中不像例子一样可以很直观的通过图像去感受，由于特征过多，而且选取也不同，所以就有了Regularized Logistics Regression，它的Cost函数如下：  
&emsp;<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/regular_logistic_regression_cost.png'>  
&emsp;&emsp;倒数如下：  
&emsp;<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/regular_logistic_regression_cost_gradient.png' />

### 具体的例子
&emsp;&emsp;给定一组测试数据[data.csv](https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/data.csv)，由于R画图比较方便省事，所以我们先用R看一下数据的整体分布，然后再整Python代码。如果你的R没有安装`lattice`包，可以先装一个:  
```r
install.packages('lattice')
```
&emsp;&emsp;我也不了解R，它的各种plot库非常繁杂，这个库也是我随便找的一个，然后plot数据。
```r
require('lattice')
data = read.csv('data.csv', header=TRUE)
attach(data)
xyplot(grade1 ~ grade2, data, groups = label, pch = 20)
```
&emsp;&emsp;数据分布如下图:  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/data-R.png' />  
&emsp;&emsp;可以看出可以近似通过一条直线来分割曲线，这样的话训练模型为`f(x) = a + bx1 + cx2`，其实这条线还是有一个弧度的，所以也可以去训练模型为`f(x) = a + bx1 + cx2 + dx1x2`，这样分类更加准确，但是可能引起overfit的问题，我们通过调整λ值来微调。本例两种模型都实现了。  

### 代码
load数据
```python
def load_data(data_file):
    data = np.loadtxt(open(data_file, 'rb'), delimiter = ',', skiprows = 1, usecols = (1,2,3))
    X = data[:, 0:2]
    #以下注释掉的行在做f(x) = a + bx1 + cx2 + dx1x2x训练的时候需要注掉
    #X = np.append(X, np.reshape(X[:,0] * X[:,1], (X.shape[0], 1)), axis = 1)
    Y = data[:, 2]
    Y = np.reshape(Y, (Y.shape[0], 1))
    return X, Y
```

`sigmoid`函数
```python
# 因为有一个log(0)的问题所以给0一个近似的极小值
def sigmoid(x):
    rs = 1.0 / (1.0 + np.exp(-x))
    for (i, j), value in np.ndenumerate(rs):
        if value < 1.0e-10:
            rs[i][j] = 1.0e-10
        elif value > 1.0 - 1.0e-10:
            rs[i][j] = 1.0 - 1.0e-10
    return rs
```

`cost`函数
```python
# 在做f(x) = a + bx1 + cx2 + dx1x2x训练的时候需要去掉以下的所有注释
def cost(theta, x, y, lam = 0.):
    m = x.shape[0]
    theta = np.reshape(theta, (len(theta), 1))
    #lamb = theta.copy()
    #lamb[0][0] = 0.
    J = (-1.0 / m) * (y.T.dot(np.log(sigmoid(x.dot(theta)))) + (1 - y).T.dot(np.log(1 - sigmoid(x.dot(theta)))))# + lam / (2 * m) * lamb.T.dot(lamb)
    return J[0][0]
```

`gradient`函数
```python
# 在做f(x) = a + bx1 + cx2 + dx1x2x训练的时候需要去掉以下的所有注释
def grad(theta, x, y, lam = 0.):
    m = x.shape[0]
    theta = np.reshape(theta, (len(theta), 1))
    #lamb = theta.copy()
    #lamb[0][0] = 0.
    grad = (1.0 / m) * (x.T.dot(sigmoid(x.dot(theta) - y)))# + (lam / m) * lamb
    grad = grad.flatten()
    return grad
```

计算最优解
```python
# 在做f(x) = a + bx1 + cx2 + dx1x2x训练的时候， theta = np.random.randn(4)
theta = np.random.randn(3)
X_new = np.append(np.ones((X.shape[0], 1)), X, axis = 1)
theta_final = opt.fmin_tnc(cost, theta, fprime = grad, args = (X_new, Y), approx_grad = True, epsilon = 0.001, maxfun = 10000)
```

&emsp;&emsp;`f(x) = a + bx1 + cx2)`训练的结果如下：
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/linear.png' />  
&emsp;&emsp;`f(x) = a + bx1 + cx2 + dx1x2x`，`λ = 0.`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary01.png' />  
&emsp;&emsp;`λ = 0.1`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary0_1.png' />  
&emsp;&emsp;`λ = 2.0`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary02.png' />  
&emsp;&emsp;`λ = 3.0`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary03.png' />  
&emsp;&emsp;`λ = 4.0`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary04.png' />  
&emsp;&emsp;`λ = 8.0`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary08.png' />  
&emsp;&emsp;`λ = 16.0`时训练结果如下：  
&emsp;<img src='https://github.com/linghuazaii/Machine-Learning/blob/master/logistic_regression/boundary16.png' />  
&emsp;&emsp;你可以将这几张图片下载到电脑上浏览，这样对不同的λ取值导致overfit和underfit会有一个很直观的感受。

### 小结
**GOOD LUCK, HAVE FUN!**
