[Machine Learning]Pearson Correlation
=====================================

### Covariance
&emsp;&emsp;Covariance(协方差)在probability and statistics里是为了衡量两个随机变量的联合可变性，公式如下：
&emsp;&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/covariance.png" />  
&emsp;&emsp;E(Expectation)，`E(X) = Sum(X1 -> Xn) / n`，当两个随机变量X,Y，X较大的时候，Y也较大，那么`cov(X, Y)`是一个正值，表示X和Y表现的行为是相似的；反之，X较大的时候，Y较小，X较小的时候，Y较大，那么他们的行为是相反的；如果X，Y的表现没什么关系，那么`cov(X, Y) ~ 0`。但是，协方差的大小并不能解释出更多的意义。

### Standard Deviation
&emsp;&emsp;Standard Deviation(标准差)，标准差体现一个随机变量的分布情况，如果标准差趋向于0，则该随机变量的分布比较集中，否则，则较为分散。公式如下：  
&emsp;&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/standard_deviation.png" />  

### Pearson Correlation
&emsp;&emsp;Pearson Correlation(皮尔森相关)，皮尔森相关从两个离散的随机变量里近似出一条线，这条线让所有的变量值离这条线都比较近，得到的值是从`-1 < score < 1`，`-1`表示我讨厌你，`0`表你在我这没有存在感，`1`表示我对你有兴趣。公式如下：
&emsp;&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/pearson_correlation.png" />

### Euclidean Distance的不足之处
&emsp;&emsp;如果我们看过的电影列表很相似，但是如果我们的个人观点都比较尖锐，那么我们给的评分可能差距很大，这样会导致Eucliean Distance的值偏大，但是事实上我们感兴趣的电影是差不多的，我们的相似性也是很大的。Pearson Correlation就可以比较适用于这种情况。

### 例子
```json
{
    "Claudia Puig": {
        "Just My Luck": 3.0,
        "Snakes on a Plane": 3.5,
        "Superman Returns": 4.0,
        "The Night Listener": 4.5,
        "You, Me and Dupree": 2.5
    },
    "Gene Seymour": {
        "Just My Luck": 1.5,
        "Lady in the Water": 3.0,
        "Snakes on a Plane": 3.5,
        "Superman Returns": 5.0,
        "The Night Listener": 3.0,
        "You, Me and Dupree": 3.5
    },
    "Jack Matthews": {
        "Lady in the Water": 3.0,
        "Snakes on a Plane": 4.0,
        "Superman Returns": 5.0,
        "The Night Listener": 3.0,
        "You, Me and Dupree": 3.5
    },
    "Lisa Rose": {
        "Just My Luck": 3.0,
        "Lady in the Water": 2.5,
        "Snakes on a Plane": 3.5,
        "Superman Returns": 3.5,
        "The Night Listener": 3.0,
        "You, Me and Dupree": 2.5
    },
    "Michael Phillips": {
        "Lady in the Water": 2.5,
        "Snakes on a Plane": 3.0,
        "Superman Returns": 3.5,
        "The Night Listener": 4.0
    },
    "Mick LaSalle": {
        "Just My Luck": 2.0,
        "Lady in the Water": 3.0,
        "Snakes on a Plane": 4.0,
        "Superman Returns": 3.0,
        "The Night Listener": 3.0,
        "You, Me and Dupree": 2.0
    },
    "Toby": {
        "Snakes on a Plane": 4.5,
        "Superman Returns": 4.0,
        "You, Me and Dupree": 1.0
    }
}
```  
&emsp;&emsp;测试数据
```json
{
    "Claudia Puig": {
        "Gene Seymour": 0.26747360685805954,
        "Jack Matthews": 0.2569179867629838,
        "Lisa Rose": 0.7203602702251998,
        "Michael Phillips": 0.5412073589719606,
        "Mick LaSalle": 0.2318104592769826,
        "Toby": 0.2651650429449559
    },
    "Gene Seymour": {
        "Claudia Puig": 0.26747360685805954,
        "Jack Matthews": 0.9804777648345675,
        "Lisa Rose": 0.396059017190669,
        "Michael Phillips": 0.5251050315105037,
        "Mick LaSalle": 0.4117647058823524,
        "Toby": 0.643567982536105
    },
    "Jack Matthews": {
        "Claudia Puig": 0.2569179867629838,
        "Gene Seymour": 0.9804777648345675,
        "Lisa Rose": 0.4195906791483465,
        "Michael Phillips": 0.18171094607790858,
        "Mick LaSalle": 0.5484028176193337,
        "Toby": 0.826839467166564
    },
    "Lisa Rose": {
        "Claudia Puig": 0.7203602702251998,
        "Gene Seymour": 0.396059017190669,
        "Jack Matthews": 0.4195906791483465,
        "Michael Phillips": 0.5303300858899116,
        "Mick LaSalle": 0.594088525786004,
        "Toby": 0.8622074921564968
    },
    "Michael Phillips": {
        "Claudia Puig": 0.5412073589719606,
        "Gene Seymour": 0.5251050315105037,
        "Jack Matthews": 0.18171094607790858,
        "Lisa Rose": 0.5303300858899116,
        "Mick LaSalle": 0.7351470441147054,
        "Toby": 0.33995005182504257
    },
    "Mick LaSalle": {
        "Claudia Puig": 0.2318104592769826,
        "Gene Seymour": 0.4117647058823524,
        "Jack Matthews": 0.5484028176193337,
        "Lisa Rose": 0.594088525786004,
        "Michael Phillips": 0.7351470441147054,
        "Toby": 0.7223722252956285
    },
    "Toby": {
        "Claudia Puig": 0.2651650429449559,
        "Gene Seymour": 0.643567982536105,
        "Jack Matthews": 0.826839467166564,
        "Lisa Rose": 0.8622074921564968,
        "Michael Phillips": 0.33995005182504257,
        "Mick LaSalle": 0.7223722252956285
    }
}
```
&emsp;&emsp;Pearson得分  
&emsp;&emsp;[测试源码](https://github.com/linghuazaii/Machine-Learning/blob/master/recommendation/pearson_correlation.py)

### Reference
 - [Covariance](https://en.wikipedia.org/wiki/Covariance)
 - [Standard deviation](https://en.wikipedia.org/wiki/Standard_deviation)
 - [Pearson correlation coefficient](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient)

### Note
&emsp;&emsp;大学浪费了不少时光啊，暂时不更了，补一补`probability and statistics`。  
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
