[Machine Learning]Euclidean Distance
====================================

### Euclidean Distance
&emsp;&emsp;Euclidean Distance(欧几里得几何学距离)，定义如下：  
&emsp;&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/euclidean_distance.png" />

### Distance & Similarity
&emsp;&emsp;Euclidean Distance 可以被用来计算用户之间的相关性，相距越近的用户相关性越高。举个简单的例子：我看过“楚门的世界”，给了9分，也看过“肖生克的救赎”，给了9.8；你也看过这两部电影，并且给“楚门的世界”打了6分，给“肖生克的救赎”打了9分。那么我们之间的距离可以算出来，`distance = sqrt((9 - 6)^2 + (9.8 - 9)^2) = 3.104835`。为了体现距离越近，相似度越高，通过函数`f(x) = 1 / (1 + x)`转化一下：  
&emsp;&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/function.png" />  
&emsp;&emsp;由图可知，距离为0时，相似度为1，随着距离越大，相似度递减。  

### 一个例子
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
&emsp;&emsp;以上是用户看过的电影，并打上的分值。
```json
{
    "Claudia Puig": {
        "Gene Seymour": 0.2025519956555797,
        "Jack Matthews": 0.17411239489546831,
        "Lisa Rose": 0.252650308587072,
        "Michael Phillips": 0.17491720310721423,
        "Mick LaSalle": 0.21239994067041815,
        "Toby": 0.14923419446018063
    },
    "Gene Seymour": {
        "Claudia Puig": 0.2025519956555797,
        "Jack Matthews": 0.38742588672279304,
        "Lisa Rose": 0.29429805508554946,
        "Michael Phillips": 0.1896812679802183,
        "Mick LaSalle": 0.27792629762666365,
        "Toby": 0.15776505912784203
    },
    "Jack Matthews": {
        "Claudia Puig": 0.17411239489546831,
        "Gene Seymour": 0.38742588672279304,
        "Lisa Rose": 0.2187841884486319,
        "Michael Phillips": 0.19636040545626823,
        "Mick LaSalle": 0.23800671553691075,
        "Toby": 0.1652960191502465
    },
    "Lisa Rose": {
        "Claudia Puig": 0.252650308587072,
        "Gene Seymour": 0.29429805508554946,
        "Jack Matthews": 0.2187841884486319,
        "Michael Phillips": 0.1975496259559987,
        "Mick LaSalle": 0.4142135623730951,
        "Toby": 0.15954492995986427
    },
    "Michael Phillips": {
        "Claudia Puig": 0.17491720310721423,
        "Gene Seymour": 0.1896812679802183,
        "Jack Matthews": 0.19636040545626823,
        "Lisa Rose": 0.1975496259559987,
        "Mick LaSalle": 0.23582845781094,
        "Toby": 0.16462407202206505
    },
    "Mick LaSalle": {
        "Claudia Puig": 0.21239994067041815,
        "Gene Seymour": 0.27792629762666365,
        "Jack Matthews": 0.23800671553691075,
        "Lisa Rose": 0.4142135623730951,
        "Michael Phillips": 0.23582845781094,
        "Toby": 0.16879264089884097
    },
    "Toby": {
        "Claudia Puig": 0.14923419446018063,
        "Gene Seymour": 0.15776505912784203,
        "Jack Matthews": 0.1652960191502465,
        "Lisa Rose": 0.15954492995986427,
        "Michael Phillips": 0.16462407202206505,
        "Mick LaSalle": 0.16879264089884097
    }
}
```  
&emsp;&emsp;以上是算出的用户相似度。  
&emsp;&emsp;[源码](https://github.com/linghuazaii/Machine-Learning/tree/master/recommendation)

### 小结
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
