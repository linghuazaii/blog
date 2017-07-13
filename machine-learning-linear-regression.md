Linear Regression(线性回归)
==========================

# 前言
&emsp;&emsp;本文所有测试数据来自于[Kaggle](https://www.kaggle.com/)，图形使用[matplotlib](https://matplotlib.org)绘制，数学公式来自于[hostmath](http://www.hostmath.com/)

# Linear Regression
&emsp;&emsp;在统计学上，线性回归被用来对一个独立标量y和多个独立变量x1, x2, x3 ...之间联系建立线性模型。在推荐系统里，用来建模做预测。比如说一个卖房子的网站，将房子的信息抽象成许多特征，其中包括房子价格，如果给定一个房子需要预测价格的话， 其中价格就是y，其他特征就是x1, x2, x3 ... 先看一个简单的例子，只有一个特征。

# Simple Linear Regression
&emsp;&emsp;先看如下数据：  
&emsp;[train.csv](https://github.com/linghuazaii/Machine-Learning/blob/master/linear_regression/train.csv)  
&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/train2.png" />  
&emsp;&emsp;现在需要根据这组数据求出y和x的关系函数，先了解一下[Mean Squared Error](https://en.wikipedia.org/wiki/Mean_squared_error)

# Mean Squared Error
&emsp;&emsp;[Mean Squared Error](https://en.wikipedia.org/wiki/Mean_squared_error), 在统计学上用来衡量估算函数的质量。如上例子，可知估算函数是一个线性函数，如下:  
&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/hx.png" />  
&emsp;&emsp;MSE的计算函数如下：  
&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/mse.png" />  
&emsp;&emsp;MSE的值越小，表示该估算函数越准确，以上多出来的`1/2`是为了便于做Gradient Descent。那么怎样调整θ的值使得MSE的值最小呢？先看一个简单的例子。

# 例子
&emsp;&emsp;如下例子，我们给出`y = x`上的三个点，不同的θ值得到的MSE的值是怎样的呢？
&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/linear_example.JPG" height = '320px' width = '480px'/>  
&emsp;&emsp;MSE值如图，图中的cost即为MSE：  
&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/cost.JPG" height = '320px' width = '480px'/>  
&emsp;&emsp;这是cost和θ取值的关系图，可知θ取1的时候，MSE最小，即求`d(MSE) / d(θ) = 0`时θ的值，但是电脑只会计算，不会解方程，而且`y = kx + b`会让问题变得更复杂，这样就有了一个方法去计算参数的值: [Gradient Descent](https://en.wikipedia.org/wiki/Gradient_descent).  

# Gradient Descent
&emsp;&emsp;[Gradient Descent](https://en.wikipedia.org/wiki/Gradient_descent)是一个通过不断迭代的方式去求函数极值的算法。对于：  
&emsp;<img src = 'https://github.com/linghuazaii/blog/blob/master/image/machine-learning/gradient.png' />  
&emsp;&emsp;其中α表示learning rate，这样就可以通过不断调整θ的值使得`d(MSE) / d(θ) = 0`,最终`θ(n) = θ(n) - 0 = θ(n)`，求出了MSE的极小值，同时也得到了`θ(n)`的值。由于`θ(1...n)`是彼此独立的，所以写代码的时候注意每一次迭代，他们的值根据上一次的迭代结果计算。关于learning rate，控制着每一步gradient descent的步长，如果learning rate过大，得到的结果是这样的：  
&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/big_learning_rate.png" />  
&emsp;&emsp;迭代求得的倒数越来越大，最终得到的全是`inf`，得不到极小值。

# 本例
&emsp;&emsp;对于本例，估算函数和Gradient Descent是这样的：  
&emsp;<img src = 'https://github.com/linghuazaii/blog/blob/master/image/machine-learning/example_gradient.png' />  
&emsp;&emsp;代码如下：
```python
# cost 函数，即 MSE
def cost(data, fx, k, b):
    cost_val = Decimal(0.0)
    for point in data:
        cost_val += pow(fx(k, b, point[0]) - point[1], 2)
    cost_val /= 2 * len(data)
    return cost_val

# d(cost) / d(k)
def derivate_k(data, fx, k, b):
    d = Decimal(0.0)
    for point in data:
        d += fx(k, b, point[0]) - point[1]
    d /= len(data)
    return d

# d(cost) / d(b)
def derivate_b(data, fx, k, b):
    d = Decimal(0.0)
    for point in data:
        d += (fx(k, b, point[0]) - point[1]) * point[0]
    d /= len(data)
    return d

# loop 直到cost值的变化趋向于一个非常小的值
def calc_line_function(data, learn_rate):
    '''
        cost = 1/(2m) * E(1->m)(Y(i) - y(i))^2
        Y(i) = k + b * x(i)
        cost(k)' = d(cost)/d(k) = 1/m * E(1->m)(Y(i) - y(i))
        cost(b)' = d(cost)/d(b) = 1/m * E(1->m)((Y(i) - y(i)) * x(i))
        for gradient descent, a means learning rate.
        update k:
            k = k - a * cost(k)'
            b = b - a * cost(b)'
    '''
    k = Decimal(0.0)
    b = Decimal(0.0)
    a = Decimal(learn_rate)
    fx = lambda m, n, x: (m + n * x)
    last_cost = cost(data, fx, k, b)
    while True:
        new_k = k - a * derivate_k(data, fx, k, b)
        new_b = b - a * derivate_b(data, fx, k, b)
        k = new_k
        b = new_b
        new_cost = cost(data, fx, k, b)
        #print "k:%s  b:%s  %s" % (k, b, math.fabs(new_cost - last_cost))
        print "k:%s  b:%s  %s" % (k, b, new_cost)
        if math.fabs(new_cost - last_cost) <= 0.0001:
            return k, b
        last_cost = new_cost
```
&emsp;&emsp;所有代码见 [linear regression code](https://github.com/linghuazaii/Machine-Learning/tree/master/linear_regression)

# More
&emsp;&emsp;Gradient Descent求得的值是局部最优解，并非全局最优解。  
&emsp;&emsp;对于linear regression，`y = k*x + b`的情况，`cost(k, b)`是一个三维的图，对于更多的变量，则维度更高，无法呈现。如图，即bowl-shape：  
&emsp;<img src = 'https://github.com/linghuazaii/blog/blob/master/image/machine-learning/cost_pic01.png' />  
&emsp;&emsp;你可能会问，如果图是这样的，那么得到的只是局部最优解，但是对于linear regression，都是bowl-shape，即这个局部最优解，就是全局最优解：  
&emsp;<img src = 'https://github.com/linghuazaii/blog/blob/master/image/machine-learning/cost_pic02.png' />  
&emsp;&emsp;针对上图，如果要获得全局最优解也是有办法的，以前的博文里也有说过，即[Simulated Annealing](https://en.wikipedia.org/wiki/Simulated_annealing)

# 小结
&emsp;&emsp;有例子和代码能给人更直观的感受，只看是没多大用的。**GOOD LUCK，HAVE FUN!**
