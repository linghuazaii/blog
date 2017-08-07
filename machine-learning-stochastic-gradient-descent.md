Stochastic Gradient Descent
===========================

### Neural Networks
&emsp;&emsp;Neural Networks是挺有意思的，如果你先前没有了解过，我给你推荐以下的资源：  
 - [Andrew NG. Machine Learning Coursera](https://www.coursera.org/learn/machine-learning/home/welcome)
 - [Ra´ul Rojas Neural Networks](https://page.mi.fu-berlin.de/rojas/neural/neuron.pdf)
 - [Neural Networks And Deep Learning](http://neuralnetworksanddeeplearning.com)  

&emsp;&emsp;如果你想对Neural Networks有一个直观的感受，可以在这里体验一下Neural Networks Classification：  
 - [A Neural Network Playground](http://playground.tensorflow.org)  

&emsp;&emsp;个人推荐先去Rojas的Neural Networks里先了解下Neural Networks的发展，在此之前你得先学习一下Andrew NG的Coursera课程的Linear Regression和Logistic Regression，Neural Networks。Andrew NG的课程有些东西的来由并未说明，这时需要你去拓展，其一是写代码直观感受，其二是Rojas的Neural Networks追本溯源，其三是Michael Nielsen的Neural Networks And Deep Learning一步步优化Neural Networks做[MNIST](http://yann.lecun.com/exdb/mnist/)识别。但是本文不牵扯Neural Networks的一些东西，只说Stochastic Gradient Descent。

### Stochastic Gradient Descent
&emsp;&emsp;就由MSE来做Cost函数，因为简单也易于理解，缺点也很明显，受异常值的影响比较大。  
&emsp;<img src='http://latex.codecogs.com/gif.latex?Cost%28w%2Cb%29%3D%5Cfrac%7B1%7D%7B2n%7D%5Csum_1%5En%28h%28x_%7Bi%7D%29-y_%7Bi%7D%29%5E2' />  
&emsp;&emsp;其中w表示weight，b表示bias。  
&emsp;<img src='http://latex.codecogs.com/gif.latex?%5Ctriangle%20C%5Capprox%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20v1%7D%5Ctriangle%20v1&plus;%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20v2%7D%5Ctriangle%20v2&plus;%5C%20...%5C%20&plus;%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20vn%7D%5Ctriangle%20vn' />  
&emsp;&emsp;其中v(1...n)表示bias和weight的向量。  
&emsp;<img src='http://latex.codecogs.com/gif.latex?%5Ctriangledown%20C%5Cequiv%28%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20v1%7D&plus;%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20v2%7D&plus;%5C%20...%5C%20&plus;%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20vn%7D%29%5ET' />  
&emsp;<img src='http://latex.codecogs.com/gif.latex?%5Ctriangle%20C%20%5Capprox%20%5Ctriangledown%20C%20%5Ctimes%20%5Ctriangle%20v' />  
&emsp;&emsp;如果`△v=-η▽C`，那么：  
&emsp;<img src='http://latex.codecogs.com/gif.latex?%5Ctriangle%20C%20%5Capprox%20-%5Ceta%5Ctimes%20%5Ctriangledown%20C%5E2' />  
&emsp;&emsp;其中η就是learning rate，由上式可知，对于MSE来说，如果η取值合理，Cost会一直下降，直到得到一个最优的解。如果η取值过大，最终不会找到最优解，反而会跨过最优解一直反弹，如果η取值太小，经过的经过的迭代次数会更多，导致计算量变大，这个算是Gradient Descent的由来。  
&emsp;<img src='http://latex.codecogs.com/gif.latex?v%5Crightarrow%20v%5E%5Cprime%20%3D%20v-%5Ceta%5Ctriangledown%20C%3Dv-%5Cfrac%7B%5Ceta%7D%7Bn%7D%5Csum_1%5En%5Cfrac%7B%5Cpartial%20C_%7BX_%7Bj%7D%7D%7D%7B%5Cpartial%20v%7D' />  
&emsp;&emsp;对于Neural Networks来说，一般数据量都是非常大的，所以每次计算所有数据的gradient是一个很大的消耗，所以就有了Stochastic Gradient Descent：  
&emsp;<img src='http://latex.codecogs.com/gif.latex?%5Cfrac%7B%5Csum_1%5Em%5Cpartial%20C_%7BX_%7Bj%7D%7D%7D%7Bm%7D%20%5Capprox%20%5Cfrac%7B%5Csum_1%5En%5Cpartial%20C_%7BX_%7Bj%7D%7D%7D%7Bn%7D%20%3D%20%5Ctriangledown%20C' />  
&emsp;&emsp;把n个数据以m大小均分，将会得到`n/m`个minimal batch，对每个batch做Gradient Descent直到处理完所有数据，每一次对所有数据的处理为一次迭代，经过多次迭代达到最优解。  

### 小结
&emsp;&emsp;Learning is fun. **GOOD LUCK, HAVE FUN!**
