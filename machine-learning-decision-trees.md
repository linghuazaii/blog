[Machine Learning]Decision Trees
================================

### Decision Trees
&emsp;&emsp;决策树(Decision Trees)是用来做预测的比较常用的一个工具，设想一下，你收集了大量的商品浏览购买的信息，包括用户信息，用户浏览过的商品信息，商品价格，商品分类，商品标签，用户跳转过来的连接等等一系列毫不相关的数据，我们称之为特征。你有100亿条这种数据，然后你就可以根据某个特征将数据分为两个或者多个集合，然后再取特征再继续细分，直到无法再分为止，这样就形成了一棵决策树。怎么选取特征来做到最优的集合分割呢？

### Gini Impurity
&emsp;&emsp;基尼不纯度(Gini Impurity)是衡量标准之一。基尼不纯度衡量的是集合元素的聚类情况，当所有元素聚在一个分类的时候，基尼不纯度为0，值越大，表明分类越差。即，我们选取不同的特征进行集合分类，所得到的基尼不纯度越低，分类越准确。基尼不纯度计算方式如下：  
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/gini_impurity.png" />  

### Entropy
&emsp;&emsp;Entropy(熵)，entropy在统计学中用来衡量事件的不确定性。当熵值为0的时候表明事件是可以直接预测的，准确度为100%。同样，我们选取不同的特征来进行分类，所得到的熵值也是不同的，熵值越低，表明分类越准确。熵的计算公式如下：  
&emsp;&emsp;&emsp;<img src = "https://github.com/linghuazaii/blog/blob/master/image/machine-learning/entropy.png" />  

### 例子
```python
my_data=[
['slashdot','USA','yes',18,'None'],
['google','France','yes',23,'Premium'],
['digg','USA','yes',24,'Basic'],
['kiwitobes','France','yes',23,'Basic'],
['google','UK','no',21,'Premium'],
['(direct)','New Zealand','no',12,'None'],
['(direct)','UK','no',21,'Basic'],
['google','USA','no',24,'Premium'],
['slashdot','France','yes',19,'None'],
['digg','USA','no',18,'None'],
['google','UK','no',18,'None'],
['kiwitobes','UK','no',19,'None'],
['digg','New Zealand','yes',12,'Basic'],
['slashdot','UK','no',21,'None'],
['google','UK','yes',18,'Basic'],
['kiwitobes','France','yes',19,'Basic']
]
```
&emsp;&emsp;以上是测试数据，根据其建立的决策树如下：  
&emsp;&emsp;&emsp;<img src = "https://github.com/linghuazaii/Machine-Learning/blob/master/decision_trees/treeview.jpg" />  
&emsp;&emsp;如果预测`['(direct)', 'USA', 'yes', 5]`的结果，流程如下：  
&emsp;&emsp;&emsp;<img src = "https://github.com/linghuazaii/Machine-Learning/blob/master/decision_trees/treeview_predict.jpg" />  

&emsp;&emsp;这样细分的决策树会对数据有很大的偏向性，这时需要根据特征分类值来merge树叶，以1.0为例，修剪后的结果如下：  
&emsp;&emsp;&emsp;<img src = 'https://github.com/linghuazaii/Machine-Learning/blob/master/decision_trees/pruned_tree_1.0.jpg' />  
  
&emsp;&emsp;[代码 treepredict.py](https://github.com/linghuazaii/Machine-Learning/blob/master/decision_trees/treepredict.py)

### 后话
&emsp;&emsp;以决策树的方式做预测，容易理解也很好实现，当特征越多，数据越多的时候，预测才会越准确。如果不理解，可以想想分类后熵值或者基尼不纯度的变化，肯定是越来越小的。就好比那句话，“菩提本无树，明镜亦非台。本来无一物，何处惹尘埃。”一切皆有因果，我们虽然抓不到那条线，但是我们有数据呀，有历史呀，我想这就是统计学的真谛吧。

### Reference
 - [Decision tree learning](https://en.wikipedia.org/wiki/Decision_tree_learning)
 - [Programming Collective Intelligence](http://shop.oreilly.com/product/9780596529321.do)

### 小结
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
