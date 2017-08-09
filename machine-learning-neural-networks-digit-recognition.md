Neural Networks Backpropagation做MNIST数字识别
==============================================

### Neural Networks
&emsp;&emsp;不同种属的生物大脑所含有的神经元数量有着显著的不同，一个成年人的大脑约含有850亿~860亿神经元，其中163亿位于大脑皮质，690亿位于小脑。相比而言，[秀丽隐杆线虫线虫](Caenorhabditis elegans)仅仅只有302个神经元，而[黑腹果蝇](https://en.wikipedia.org/wiki/Drosophila_melanogaster)则有约100000神经元，并且能表现出比较复杂的行为。本例所使用的神经元数量大约只有1000个，三层神经网络，只有一层hidden layer。  
&emsp;&emsp;对于人的大脑来说，不同的部分之间具有通用性，一个有语言障碍的人，负责语言部分的大脑会适应于其他部分的行为。就好比有的聋哑人视觉会比较好，有的瞎子听觉却比常人更好，其原因就是他们的缺陷导致相应的大脑部位去适应另外的能力。  
&emsp;&emsp;神经元的行为非常复杂，我们只抽象出我们所能理解的比较简单的一部分，但是Neural Networks的对不同问题领域的适应能力和学习能力还是保留了的。一个神经元简化为树突，对应input，胞体对应`f(x)`，早期Perceptron神经元设计比较简单，`f(x) = b + W*X`，局限性比较大，只能输出`0`和`1`，不过能实现所有的门电路逻辑。后来发展为sigmoid神经元，`f(x) = 1 / (1 + e^-(W*X + b))`，输出是一个平滑的从0到1的S曲线。然后轴突对应output。神经元在输入达到一个阈值的时候才会被激发，这样通过动态调整weights和biases来控制Neural Networks的神经元行为进而学习一个问题领域。下面是一个简单的神经网络，input layer拥有784个神经元，hidden layer有15个神经元，output layer有10个神经元：  
&emsp;&emsp;<img src='http://neuralnetworksanddeeplearning.com/images/tikz12.png' />  

### Weights And Biases
&emsp;&emsp;对于上图，如果我们有M个training data，则一次正向的传导为`sigmoid(W2 * sigmoid(W1 * R^(M*784) + b1) + b2)`，最终得到`R^(M*10)`的output矩阵，然后计算Cost，然后多次迭代得到优化后的Weights和Biases。本例用上篇提到的Stochastic Gradient Descent去找优化的解。

### Backpropagation
&emsp;&emsp;下面是Backpropagation的公式，我们来一一推导：  
&emsp;&emsp;<img src='http://neuralnetworksanddeeplearning.com/images/tikz21.png' />  
&emsp;&emsp;<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/backpropagation.png' />  

### MNIST
&emsp;&emsp;[MNIST](http://yann.lecun.com/exdb/mnist/)包含手写的数字，有60000条training data，10000条test data。下面的图片提取自MNIST：  
&emsp;&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit1_5.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit2_0.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit3_4.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit4_1.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit5_9.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit6_2.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit7_1.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit8_3.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit9_1.png' />&emsp;
<img src='https://github.com/linghuazaii/digit-recognition/blob/master/digits/digit10_4.png' />&emsp;  
&emsp;提取training data  
```python
def load_train_data():
    fimg = gzip.open('train-images-idx3-ubyte.gz', 'rb')
    flabel = gzip.open('train-labels-idx1-ubyte.gz', 'rb')
    magic_img, total_img, rows, cols = struct.unpack('>IIII', fimg.read(16))
    magic_label, total_label = struct.unpack('>II', flabel.read(8))
    train_data = list()
    for i in xrange(total_img):
        img = np.reshape(np.fromstring(fimg.read(rows * cols), dtype = np.uint8), (rows * cols, 1))
        train_data.append(img)
    train_label = vectorize_result(np.fromstring(flabel.read(total_label), dtype = np.uint8))
    train = zip(train_data, train_label)

    fimg.close()
    flabel.close()

    return train
```

&emsp;提取test data
```python
def load_test_data():
    fimg = gzip.open('t10k-images-idx3-ubyte.gz', 'rb')
    flabel = gzip.open('t10k-labels-idx1-ubyte.gz', 'rb')
    magic_img, total_img, rows, cols = struct.unpack('>IIII', fimg.read(16))
    magic_label, total_label = struct.unpack('>II', flabel.read(8))
    test_data = list()
    for i in xrange(total_img):
        img = np.reshape(np.fromstring(fimg.read(rows * cols), dtype = np.uint8), (rows * cols, 1))
        test_data.append(img)
    test_label = np.fromstring(flabel.read(total_label), dtype = np.uint8)
    test = zip(test_data, test_label)

    fimg.close()
    flabel.close()

    return test
```

### Neural Networks & Backpropagation
&emsp;定义Neural Network
```python
class NeuralNetwork(object):
    def __init__(self, sizes):
        self.num_layers = len(sizes)
        self.sizes = sizes
        self.biases = [np.random.randn(y, 1) for y in sizes[1:]]
        self.weights = [np.random.randn(y, x) for x, y in zip(sizes[:-1], sizes[1:])]
```

&emsp;sigmoid函数和导数
```python
def sigmoid(z):
    for (x, y), val in np.ndenumerate(z):
        if val >= 100:
            z[x][y] = 1.
        elif val <= -100:
            z[x][y] = 0.
        else:
            z[x][y] = 1.0 / (1.0 + np.exp(-val))

    return z

def sigmoid_derivative(z):
    return sigmoid(z) * (1. - sigmoid(z))

```

&emsp;Stochastic Gradient Descent
```python
# eta is learning rate
def stochasticGradientDescent(self, training_data, epochs, mini_batch_size, eta, test_data = None):
    if test_data:
        n_test = len(test_data)
    n_train = len(training_data)
    for j in xrange(epochs):
        random.shuffle(training_data)
        mini_batches = [training_data[k: k + mini_batch_size] for k in xrange(0, n_train, mini_batch_size)]
        for mini_batch in mini_batches:
            self.updateMiniBatch(mini_batch, eta)
        if test_data:
            print 'Epoch %s: %s / %s' % (j, self.evaluate(test_data), n_test)
        else:
            print 'Epoch %s complete.' % j

def updateMiniBatch(self, mini_batch, eta):
    nabla_b = [np.zeros(b.shape) for b in self.biases]
    nabla_w = [np.zeros(w.shape) for w in self.weights]
    for x, y in mini_batch:
        delta_nabla_b, delta_nabla_w = self.backprob(x, y)
        nabla_b = [nb + dnb for nb, dnb in zip(nabla_b, delta_nabla_b)]
        nabla_w = [nw + dnw for nw, dnw in zip(nabla_w, delta_nabla_w)]
    self.weights = [w - (eta / len(mini_batch)) * nw for w, nw in zip(self.weights, nabla_w)]
    self.biases = [b - (eta / len(mini_batch)) * nb for b, nb in zip(self.biases, nabla_b)]
```

&emsp;Backpropagation
```python
def backprob(self, x, y):
    nabla_b = [np.zeros(b.shape) for b in self.biases]
    nabla_w = [np.zeros(w.shape) for w in self.weights]

    activation = x
    activations = [x]
    zs = []
    for b, w in zip(self.biases, self.weights):
        z = np.dot(w, activation) + b
        zs.append(z)
        activation = sigmoid(z)
        activations.append(activation)
    delta = self.cost_derivative(activations[-1], y) * sigmoid_derivative(zs[-1])
    nabla_b[-1] = delta
    nabla_w[-1] = np.dot(delta, activations[-2].transpose())
    for l in xrange(2, self.num_layers):
        z = zs[-l]
        sd = sigmoid_derivative(z)
        delta = np.dot(self.weights[-l + 1].transpose(), delta) * sd
        nabla_b[-l] = delta
        nabla_w[-l] = np.dot(delta, activations[-l - 1].transpose())
    return (nabla_b, nabla_w)
```
  
&emsp;&emsp;hidden layer 30个Neurons，准确率约为82%.  
&emsp;&emsp;<img src='https://github.com/linghuazaii/blog/blob/master/image/machine-learning/hidden-layer-30.png' />  
&emsp;&emsp;hidden layer 100个Neurons，准确率约为90%.  
&emsp;&emsp;完整代码见[Digit Recognition](https://github.com/linghuazaii/digit-recognition)

### Reference
 - [Andrew NG. Machine Learning](https://www.coursera.org/learn/machine-learning/home/welcome)
 - [Michael Nielsen. Neural Networks And Deep Learning](http://neuralnetworksanddeeplearning.com)


### 小结
&emsp;&emsp;**Learning is a HOBBY!**
