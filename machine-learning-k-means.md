[Machine Learning]K-Means
==========================

### K-means
&emsp;&emsp;上一篇介绍了Word Vector，其有一个很明显的缺点，就是数据量太大的时候，运算量是非常惊人的。本章介绍K-means，K-means的原理比较简单。举个例子，我们要将50个新闻RSS FEED分成K个类别的话，只需要生成5个随机的Vector，然后分别将50个RSS FEED与5个随机的Vector求Pearson距离，这一轮的最终结果会将所有RSS FEED归为5个类别。所谓means就是中值嘛，然后去5个分组的中值作为5个新的Vector，再次计算距离归类。循环以上迭代，直到得到的分组不再变化。如图：  
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/k-means.png" />

### 例子
[k_means_cluster.py](https://github.com/linghuazaii/Machine-Learning/blob/master/dig_groups/k_means_cluster.py)
```python
#!/bin/env python
# -*- coding: utf-8 -*-
# This file is auto-generated.Edit it at your own peril.
from math import sqrt
import random

def readfile(filename):
    lines = [line for line in file(filename)]
    colnames = lines[0].strip().split('\t')[1:]
    rownames = []
    data = []
    for line in lines[1:]:
        p = line.strip().split('\t')
        rownames.append(p[0])
        data.append([float(i) for i in p[1:]])

    return rownames, colnames, data

def pearson(v1, v2):
    sum1 = sum(v1)
    sum2 = sum(v2)

    sum1Sq = sum([pow(v, 2) for v in v1])
    sum2Sq = sum([pow(v, 2) for v in v2])

    pSum = sum([v1[i] * v2[i] for i in range(len(v1))])
    num = pSum - (sum1 * sum2 / len(v1))
    den = sqrt((sum1Sq - pow(sum1, 2) / len(v1)) * (sum2Sq - pow(sum2, 2) / len(v1)))
    if den == 0:
        return 0

    return 1.0 - num / den

def kcluster(rows, distance = pearson, k = 7):
    ranges = [(min([row[i] for row in rows]), max([row[i] for row in rows])) for i in range(len(rows[0]))]
    # Create k random centroids
    clusters = [[random.random() * (ranges[i][1] - ranges[i][0]) + ranges[i][0] for i in range(len(rows[0]))] for j in range(k)]

    last_matches = None
    for t in range(100):
        print 'Iteration %d' % t
        best_matches = [[] for i in range(k)]

        for j in range(len(rows)):
            row = rows[j]
            best_match = 0
            for i in range(k):
                d = distance(clusters[i], row)
                if d < distance(clusters[best_match], row):
                    best_match = i
            best_matches[best_match].append(j)

        if best_matches == last_matches:
            break
        last_matches = best_matches

        for i in range(k):
            avgs = [0.0] * len(rows[0])
            if len(best_matches[i]) > 0:
                for row_id in best_matches[i]:
                    for m in range(len(rows[row_id])):
                        avgs[m] += rows[row_id][m]
                    for j in range(len(avgs)):
                        avgs[j] /= len(best_matches[i])
                    clusters[i] = avgs

    return best_matches

def print_cluster(clusters, blognames):
    for i in range(len(clusters)):
        print "{"
        for j in clusters[i]:
            print "\t%s," % blognames[j]
        print "}"


def create_cluster():
    blognames, words, data = readfile('blogdata.txt')
    kclust = kcluster(data)
    print_cluster(kclust, blognames)


def main():
    create_cluster()

if __name__ == "__main__":
    main()
```
  
**Result**:
```
{
        sites.google.com/view/kamagratablette/,
        BBC News - Technology,
        David Kleinert Photography,
        Techlearning RSS Feed,
}
{
        NASA Image of the Day,
}
{
        ASCD SmartBrief,
        CBNNews.com,
        GANNETT Syndication Service,
}
{
        Reuters: U.S.,
        NYT > Home Page,
        News : NPR,
        Reuters: Top News,
        Reuters: World News,
}
{
        Education : NPR,
        Technology : NPR,
        WIRED,
        UEN News,
}
{
        Nature - Issue - nature.com science feeds,
        NOVA | PBS,
        Latest Science News -- ScienceDaily,
        Resources » Surfnetkids,
        Movies : NPR,
        Utah.gov News Provider,
        PCWorld,
        BBC News - Business,
        The Daily Puppy | Pictures of Puppies,
        Animal of the day,
        Latest News Articles from Techworld,
        Utah Jazz,
        Dictionary.com Word of the Day,
        Macworld,
}
{
        FRONTLINE - Latest Stories,
        Utah - The Salt Lake Tribune,
        Utah,
        Arts & Life : NPR,
        BBC News - US & Canada,
        Latest News,
        AP Top Science News at 6:06 a.m. EDT,
        BBC News - Home,
        Fresh Air : NPR,
        NYT > Sports,
        AP Top Sports News at 2:56 a.m. EDT,
        CNN.com - RSS Channel - HP Hero,
        KSL / Utah / Local Stories,
        AP Top U.S. News at 3:16 a.m. EDT,
        NYT > Technology,
        Yahoo News - Latest News & Headlines,
}
```
&emsp;&emsp;得出的结果并不太准确~

### Reference
 - [Programming Collective Intelligence](http://shop.oreilly.com/product/9780596529321.do)

### 小结
&emsp;&emsp;**GOOD LUCK，HAVE FUN!**
