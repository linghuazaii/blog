[Parallel Programming]Peterson Lock 研究报告
===========================================

#### 作者信息
&emsp;&emsp;Charles, Liu&emsp;&emsp;charlesliu.cn.bj@gmail.com&emsp;&emsp;安远国际  
&emsp;&emsp;Derek, Zhang&emsp;&emsp;anonymous@anonymous.com&emsp;&emsp;百度

### 前言
&emsp;&emsp;前段时间研究Peterson Lock的时候遇到了一些问题，就将问题抛给了前Mentor（Derek, Zhang）。经过两天的讨论，将所有的细节罗列在这份报告里面，对于所有的外部资料的引用，只会贴出链接，作者并不做任何形式的翻译或者转述，这样可以保证所有信息的准确性。

### Peterson Locking Algorithm实现
&emsp;&emsp;关于Peterson Locking Algorithm，请参考Wiki：[Peterson's algorithm](https://en.wikipedia.org/wiki/Peterson's_algorithm)
```
#ifndef _PETERSON_LOCK_H
#define _PETERSON_LOCK_H
/*
 * Description: implementing peterson's locking algorithm
 * File: peterson_lock.h
 * Author: Charles, Liu.
 * Mailto: charlesliu.cn.bj@gmail.com
 */
#include <pthread.h>

typedef struct {
    volatile bool flag[2];
    volatile int victim;
} peterson_lock_t;

void peterson_lock_init(peterson_lock_t &lock) {
    lock.flag[0] = lock.flag[1] = false;
    lock.victim = 0;
}

void peterson_lock(peterson_lock_t &lock, int id) {
    lock.flag[id] = true; 
    lock.victim = id; 
    while (lock.flag[1 - id] && lock.victim == id);
}

void peterson_unlock(peterson_lock_t &lock, int id) {
    lock.flag[id] = false;
    lock.victim = id;
}

#endif
```
&emsp;&emsp;以下是测试程序：
```
#include <stdio.h>
#include "peterson_lock.h"

peterson_lock_t lock;
int count = 0;

void *routine0(void *arg) {
    int *cnt = (int *)arg;
    for (int i = 0; i < *cnt; ++i) {
        peterson_lock(lock, 0);
        ++count;
        printf("count: %d", count);
        peterson_unlock(lock, 0);
    }

    return NULL;
}

void *routine1(void *arg) {
    int *cnt = (int *)arg;
    for (int i = 0; i < *cnt; ++i) {
        peterson_lock(lock, 1);
        ++count;
        peterson_unlock(lock, 1);
    }
}

int main(int argc, char **argv) {
    peterson_lock_init(lock);
    pthread_t thread0, thread1;
    int count0 = 10000;
    int count1 = 20000;
    pthread_create(&thread0, NULL, routine0, (void *)&count0);
    pthread_create(&thread1, NULL, routine1, (void *)&count1);

    pthread_join(thread0, NULL);
    pthread_join(thread1, NULL);

    printf("Expected: %d\n", (count0 + count1));
    printf("Reality : %d\n", count);

    return 0;
}
```
&emsp;&emsp;如果你编译运行以上的测试，会发现Peterson Lock的实现并没有达到目的，原因就是编译时期和运行时期都会有指令重排的问题。

### Memory Ordering
&emsp;&emsp;这里我不打算讲Memory Ordering的具体细节，所有信息都在下面的链接里：  
  - 这里是Memory Ordering的Wiki -> [Memory ordering](https://en.wikipedia.org/wiki/Memory_ordering)
  - 这里的一篇文章讲到了Memory Ordering和Peterson Lock的问题 -> [Who ordered memory fences on an x86?](https://bartoszmilewski.com/2008/11/05/who-ordered-memory-fences-on-an-x86/)  
  - 这里有更详细的关于Intel IA64 Memory Ordering的资料 -> [Intel® 64 Architecture Memory Ordering White Paper](http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)
  - C++ <atomic>里Memory Order相关的资料(由于atomic本身就用到了原子操作，所以我并不打算使用<atomic>) -> [std::memory_order](http://en.cppreference.com/w/cpp/atomic/memory_order)  
  
&emsp;&emsp;请你务必仔细看完以上的三份资料，不然无法继续进行接下来的内容。  
&emsp;&emsp;以上Peterson Lock实现失败的原因就显而易见了，博主的机器是Intel x86-64 smp的机器，拥有strict memory order，也就是只有在store&load的
情况下才会有指令重拍的问题，而以上Peterson Lock的实现正好就是store&load的情况，为此我们必须避免CPU的指令重排。

### 关于Memory Fence  
&emsp;&emsp;上节提到避免指令重排需要加Memory Fence，指令重排分为两种:  
 - 一种是编译时期的指令重排，可以通过这个来防止：`asm volatile ("" : : : "memory")`  
 - 一种是运行时期的cpu指令重排，同时也包含了防止编译时期的指令重排：`asm volatile ("mfence" : : : "memory")` or `asm volatile ("lfence" : : : "memory")` or `asm volatile ("sfence" : : : "memory")`  
   
&emsp;&emsp;yi





















