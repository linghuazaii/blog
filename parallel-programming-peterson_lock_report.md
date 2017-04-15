[Parallel Programming]Peterson Lock 研究报告
===========================================

#### 作者信息
&emsp;&emsp;Charles, Liu&emsp;&emsp;charlesliu.cn.bj@gmail.com&emsp;&emsp;安远国际  
&emsp;&emsp;Derek, Zhang&emsp;&emsp;anonymous@anonymous.com&emsp;&emsp;百度

### 前言
&emsp;&emsp;前段时间研究Peterson Lock的时候遇到了一些问题，就将问题抛给了前Mentor（Derek, Zhang）。经过两天的讨论，将所有的细节罗列在这份报告里面，对于所有的外部资料的引用，只会贴出链接，作者并不做任何形式的翻译或者转述，这样可以保证所有信息的准确性。

### Peterson Locking Algorithm实现
&emsp;&emsp;关于Peterson Locking Algorithm，请参考Wiki：[Peterson's algorithm](https://en.wikipedia.org/wiki/Peterson's_algorithm)
```c
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
```c
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
   
&emsp;&emsp;而Memory Fence分为三种：  
 - mfence -> Performs a serializing operation on all load-from-memory and store-to-memory instructions that were issued prior the MFENCE instruction. **This serializing operation guarantees that every load and store instruction that precedes in program order the MFENCE instruction is globally visible before any load or store instruction that follows the MFENCE instruction is globally visible.** The MFENCE instruction is ordered with respect to all load and store instructions, other MFENCE instructions, any SFENCE and LFENCE instructions, and any serializing instructions (such as the CPUID instruction).
Weakly ordered memory types can be used to achieve higher processor performance through such techniques as out-of-order issue, speculative reads, write-combining, and write-collapsing.
The degree to which a consumer of data recognizes or knows that the data is weakly ordered varies among applications and may be unknown to the producer of this data. The MFENCE instruction provides a performance-efficient way of ensuring load and store ordering between routines that produce weakly-ordered results and routines that consume that data.
It should be noted that processors are free to speculatively fetch and cache data from system memory regions that are assigned a memory-type that permits speculative reads (that is, the WB, WC, and WT memory types). The PREFETCHh instruction is considered a hint to this speculative behavior. Because this speculative fetching can occur at any time and is not tied to instruction execution, the MFENCE instruction is not ordered with respect to PREFETCHh instructions or any other speculative fetching mechanism (that is, data could be speculatively loaded into the cache just before, during, or after the execution of an MFENCE instruction).
 - lfence -> Performs a serializing operation on all load-from-memory instructions that were issued prior the LFENCE instruction. This serializing operation guarantees that every load instruction that precedes in program order the LFENCE instruction is globally visible before any load instruction that follows the LFENCE instruction is globally visible. The LFENCE instruction is ordered with respect to load instructions, other LFENCE instructions, any MFENCE instructions, and any serializing instructions (such as the CPUID instruction). It is not ordered with respect to store instructions or the SFENCE instruction.
Weakly ordered memory types can be used to achieve higher processor performance through such techniques as out-of-order issue and speculative reads. The degree to which a consumer of data recognizes or knows that the data is weakly ordered varies among applications and may be unknown to the producer of this data. The LFENCE instruction provides a performance-efficient way of insuring load ordering between routines that produce weakly-ordered results and routines that consume that data.
It should be noted that processors are free to speculatively fetch and cache data from system memory regions that are assigned a memory-type that permits speculative reads (that is, the WB, WC, and WT memory types). The PREFETCHh instruction is considered a hint to this speculative behavior. Because this speculative fetching can occur at any time and is not tied to instruction execution, the LFENCE instruction is not ordered with respect to PREFETCHh instructions or any other speculative fetching mechanism (that is, data could be speculative loaded into the cache just before, during, or after the execution of an LFENCE instruction).
 - sfence -> Performs a serializing operation on all store-to-memory instructions that were issued prior the SFENCE instruction. This serializing operation guarantees that every store instruction that precedes in program order the SFENCE instruction is globally visible before any store instruction that follows the SFENCE instruction is globally visible. The SFENCE instruction is ordered with respect store instructions, other SFENCE instructions, any MFENCE instructions, and any serializing instructions (such as the CPUID instruction). It is not ordered with respect to load instructions or the LFENCE instruction.
Weakly ordered memory types can be used to achieve higher processor performance through such techniques as out-of-order issue, write-combining, and write-collapsing. The degree to which a consumer of data recognizes or knows that the data is weakly ordered varies among applications and may be unknown to the producer of this data. The SFENCE instruction provides a performance-efficient way of insuring store ordering between routines that produce weakly-ordered results and routines that consume this data.  
  
&emsp;&emsp;参考Intel x86-64的Memory Ordering的设计，其实lfence和sfence在保证Memory Ordering这点上是没有意义的，因为Intel x86-64本来就是strict memory order，但是内存可见性这个点上，lfence和sfence任然有其价值，由于Peterson Lock是为了避免store&load的指令重排，所以我们使用mfence，
即Full Memory Fence。加上MFENCE之后的代码如下：
```c
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
    lock.flag[id] = true; // 子句A
    lock.victim = id; // 子句B，子句A和B的顺序很重要，如果调换的话会出现问题
    asm volatile ("mfence" : : : "memory"); // MFENCE加在store和load之间
    while (lock.flag[1 - id] && lock.victim == id);
}

void peterson_unlock(peterson_lock_t &lock, int id) {
    lock.flag[id] = false;
    lock.victim = id;
}

#endif
```
&emsp;&emsp;首先看一下MFENCE添加的位置是在store和load之间，其次思考这么一个问题，如果调换子句A和子句B的位置会出现什么情况呢？
```c
peterson_lock_0:                       peterson_lock_1:
-------------------------------------------------------
lock.victim = 0;
                                      lock.victim = 1;
                                      lock.flag[1] = true;
                                      asm volatile ("mfence" : : : "memory");
                                      while (lock.flag[0] && lock.victim == 1);
                                      // lock.flag[0] is false so
                                      // the process enters critical
                                      // section
lock.flag[0] = true;
asm volatile ("mfence" : : : "memory");
while (lock.flag[1] && lock.victim == 0);
// lock.victim is 1 so
// the process enters critical
// section
```
&emsp;&emsp;Thread0和Thread1会同时进入Critical Section，再思考如下的情况，
```c
Thread0:                                Thread1:
-------------------------------------------------------
lock.flag[0] = true;
lock.victim = 0;						
                                        lock.flag[1] = true;
                                        lock.victim = 1;
                                        mfence;
                                        如果这个时候Thread0的lock.victim = 0对于Thread1可见，
                                        那么会进入Critical Section
mfence;
如果这个时候Thread1的lock.victim = 1对于
Thread0可见，那么会进入Critical Section
```
&emsp;&emsp;以上情况并没有发生，为什么呢？请看mfence那段加粗的文字，这正是mfence的另一个作用，内存可见性。

### 关于volatile和GCC优化
&emsp;&emsp;如果去掉上面的两个volatile，不加优化，运行正常，`void peterson_lock(peterson_lock_t &lock, int id)`的汇编结果如下:
```asm
0000000000400638 <_Z13peterson_lockR15peterson_lock_ti>:
  400638:   55                      push   rbp
  400639:   48 89 e5                mov    rbp,rsp
  40063c:   48 89 7d f8             mov    QWORD PTR [rbp-0x8],rdi
  400640:   89 75 f4                mov    DWORD PTR [rbp-0xc],esi
  400643:   48 8b 55 f8             mov    rdx,QWORD PTR [rbp-0x8]
  400647:   8b 45 f4                mov    eax,DWORD PTR [rbp-0xc]
  40064a:   48 98                   cdqe
  40064c:   c6 04 02 01             mov    BYTE PTR [rdx+rax*1],0x1
  400650:   48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
  400654:   8b 55 f4                mov    edx,DWORD PTR [rbp-0xc]
  400657:   89 50 04                mov    DWORD PTR [rax+0x4],edx
  40065a:   0f ae f0                mfence
  40065d:   90                      nop
  40065e:   b8 01 00 00 00          mov    eax,0x1
  400663:   2b 45 f4                sub    eax,DWORD PTR [rbp-0xc]
  400666:   48 8b 55 f8             mov    rdx,QWORD PTR [rbp-0x8]
  40066a:   48 98                   cdqe
  40066c:   0f b6 04 02             movzx  eax,BYTE PTR [rdx+rax*1]
  400670:   84 c0                   test   al,al
  400672:   74 0c                   je     400680 <_Z13peterson_lockR15peterson_lock_ti+0x48>
  400674:   48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
  400678:   8b 40 04                mov    eax,DWORD PTR [rax+0x4]
  40067b:   3b 45 f4                cmp    eax,DWORD PTR [rbp-0xc]
  40067e:   74 de                   je     40065e <_Z13peterson_lockR15peterson_lock_ti+0x26>
  400680:   5d                      pop    rbp
  400681:   c3                      ret
```




















