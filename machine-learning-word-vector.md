[Machine Learning]Word Vector
=============================

### 回顾
&emsp;&emsp;前面说过Pearson Correlation，用来计算两个矩阵的相似度，本章Word Vector里面使用Pearson计算矩阵相似度。

### 原理
&emsp;&emsp;给定两份文档如何确定其相似度？这个问题可以延伸到：抓取的文章如果确定几篇文章是相似的，以作相关推荐？如果确定微博上的两个博主具有共同的话题，观念，爱好？或者如何确定几个博客博主是同一类型的，比如说作家，程序员？  
&emsp;&emsp;Word Vector，从给定的文档里提取一定数目的关键词，形成矩阵。提取关键词的依据当然有很多（本章的例子比较简单），例如高频词`the`, `an`, `is`等等，短语`go to`, `want to`等等，以及词性做一个筛选，最终留下的词作为关键词。  
&emsp;&emsp;然后通过上一篇的Pearson Score来计算距离，举个例子，我收集了50个博主的博客文章，然后统计了300个关键词，这样就有了50个1x300的矩阵，Word Vector总是选择距离最近的两个矩阵合为一个，知道最终只剩下一个矩阵。如图：  
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/machine-learning/word_vector.png" />  

### 例子
&emsp;&emsp;我选取了RSS FEED做例子，最终结果为相似的RSS FEED会被归到一块儿。 
[feedlist.txt](https://github.com/linghuazaii/Machine-Learning/blob/master/dig_groups/feedlist.txt)
```
http://www.cbn.com/cbnnews/world/feed/
http://feeds.reuters.com/Reuters/worldNews
http://feeds.bbci.co.uk/news/rss.xml
http://news.sky.com/sky-news/rss/home/rss.xml
http://www.cbn.com/cbnnews/us/feed/
http://feeds.reuters.com/Reuters/domesticNews
http://news.yahoo.com/rss/
.....
```
  
[generate_feed_vector.py](https://github.com/linghuazaii/Machine-Learning/blob/master/dig_groups/generate_feed_vector.py)
```python
#!/bin/env python
# -*- coding: utf-8 -*-
# This file is auto-generated.Edit it at your own peril.
import feedparser
import re, sys

reload(sys)
sys.setdefaultencoding('utf-8')

def get_words(html):
    txt = re.compile(r'<[^>]+>').sub('', html)
    words = re.compile(r'[^A-Z^a-z]+').split(txt)

    return [word.lower() for word in words if word != '']

def get_word_count(url):
    doc = feedparser.parse(url)
    #print doc
    wc = {}

    for e in doc.entries:
        if 'summary' in e:
            summary = e.summary
        else:
            summary = e.description
        words = get_words(e.title + ' ' + summary)
        for word in words:
            wc.setdefault(word, 0)
            wc[word] += 1

    return doc.feed.title, wc

def create_word_vector():
    apcount = {}
    word_counts = {}
    for feedurl in file('feedlist.txt'):
        try:
            title, wc = get_word_count(feedurl)
            word_counts[title] = wc
            for word, count in wc.items():
                apcount.setdefault(word, 0)
                if count > 1:
                    apcount[word] += 1
        except Exception as e:
            print "failed for feed %s" % feedurl
            continue
    wordlist = []
    for w, bc in apcount.items():
        frac = float(bc) / len(word_counts)
        if frac > 0.1 and frac < 0.5:
            wordlist.append(w)

    out = file('blogdata.txt', 'w+')
    out.write('Blog')
    for word in wordlist:
        out.write('\t%s ' % word)
    out.write("\n")
    for blog, wc in word_counts.items():
        out.write(blog)
        for word in wordlist:
            if word in wc:
                out.write("\t%d" % wc[word])
            else:
                out.write('\t0')
        out.write('\n')

def main():
    create_word_vector()

if __name__ == "__main__":
    main()
```
  
[clusters.py](https://github.com/linghuazaii/Machine-Learning/blob/master/dig_groups/clusters.py)
```python
#!/bin/env python
# -*- coding: utf-8 -*-
# This file is auto-generated.Edit it at your own peril.
from math import sqrt

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

class bicluster:
    def __init__(self, vec, left = None, right = None, distance = 0.0, id = None):
        self.left = left
        self.right = right
        self.vec = vec
        self.id = id
        self.distance = distance

def hcluster(rows, distance = pearson):
    distances = {}
    current_clust_id = -1
    clust = [bicluster(rows[i], id = i) for i in range(len(rows))]
    while len(clust) > 1:
        lowest_pair = (0, 1)
        closest = distance(clust[0].vec, clust[1].vec)
        for i in range(len(clust)):
            for j in range(i + 1, len(clust)):
                if (clust[i].id, clust[j].id) not in distances:
                    distances[(clust[i].id, clust[j].id)] = distance(clust[i].vec, clust[j].vec)
                d = distances[(clust[i].id, clust[j].id)]
                if d < closest:
                    closest = d
                    lowest_pair = (i, j)
        mergevec = [(clust[lowest_pair[0]].vec[i] + clust[lowest_pair[1]].vec[i]) / 2.0 for i in range(len(clust[0].vec))]
        new_cluster = bicluster(mergevec, left = clust[lowest_pair[0]], right = clust[lowest_pair[1]], distance = closest, id = current_clust_id)
        current_clust_id  -= 1
        new_clust = [clust[i] for i in range(len(clust)) if i not in (lowest_pair[0], lowest_pair[1])]
        clust = new_clust
        clust.append(new_cluster)
    #print "clust len: %s" % len(clust)
    #print "clust id: %s" % clust[0].id
    return clust[0]

def print_cluster(clust, labels = None, n = 0):
    for i in range(n):
        print ' ',
    if clust.id < 0:
        print '-'
    else:
        if labels == None:
            print clust.id
        else:
            print labels[clust.id]
    if clust.left != None:
        print_cluster(clust.left, labels = labels, n = n + 1)
    if clust.right != None:
        print_cluster(clust.right, labels = labels, n = n + 1)

def create_cluster():
    blognames, words, data = readfile('blogdata.txt')
    clust = hcluster(data)
    print_cluster(clust, labels = blognames)

def main():
    create_cluster()

if __name__ == "__main__":
    main()
```

**Result**
```
-
  -
    -
      The Daily Puppy | Pictures of Puppies
      David Kleinert Photography
    -
      NASA Image of the Day
      -
        Utah Jazz
        -
          -
            Dictionary.com Word of the Day
            -
              Resources » Surfnetkids
              Utah.gov News Provider
          -
            -
              WIRED
              -
                NOVA | PBS
                -
                  Latest News Articles from Techworld
                  -
                    BBC News - Business
                    -
                      -
                        -
                          PCWorld
                          Macworld
                        -
                          Animal of the day
                          -
                            Nature - Issue - nature.com science feeds
                            Latest Science News -- ScienceDaily
                      -
                        Technology : NPR
                        -
                          BBC News - Technology
                          NYT > Technology
            -
              UEN News
              -
                Education : NPR
                -
                  -
                    AP Top Sports News at 2:56 a.m. EDT
                    -
                      Latest News
                      NYT > Sports
                  -
                    -
                      Movies : NPR
                      -
                        Arts & Life : NPR
                        Fresh Air : NPR
                    -
                      -
                        FRONTLINE - Latest Stories
                        -
                          Utah
                          -
                            -
                              Utah - The Salt Lake Tribune
                              KSL / Utah / Local Stories
                            -
                              Yahoo News - Latest News & Headlines
                              -
                                NYT > Home Page
                                -
                                  -
                                    Reuters: U.S.
                                    -
                                      Reuters: Top News
                                      Reuters: World News
                                  -
                                    AP Top Science News at 6:06 a.m. EDT
                                    -
                                      BBC News - US & Canada
                                      CNN.com - RSS Channel - HP Hero
                      -
                        AP Top U.S. News at 3:16 a.m. EDT
                        -
                          BBC News - Home
                          News : NPR
  -
    Techlearning RSS Feed
    -
      CBNNews.com
      -
        GANNETT Syndication Service
        -
          ASCD SmartBrief
          sites.google.com/view/kamagratablette/
```
&emsp;&emsp;可以看到，最终结果还凑合，如果关键词选取的更细致的话，那么结果将会更加准确。

### Reference
 - [Programming Collective Intelligence](http://shop.oreilly.com/product/9780596529321.do)

### Note
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
