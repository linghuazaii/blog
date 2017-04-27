[Parallel Programming]深入PThread (Lesson I)
============================================

### 前言
&emsp;&emsp;因为最近一直在研究Parallel Programming方面的东西，Lock的实现，一些Design Patterns以及atomic，fence，lock-free之类的东西。我学东西都是比较分散的，不是一个东西一本书或者一份资料啃到底。今天刚好到PThreads，将一些东西记录如下。

### Hyper Thread
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/UMA.png" />  
&emsp;&emsp;在你的印象中，内存是不是这样的？古老的UMA（Uniform Memory Access），多个CPU共享一整个RAM。
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/NUMA.png" />  
&emsp;&emsp;现在的Linux Server，基本都是这样的，是不是感觉世界瞬间不一样了？即所说的NUMA（Non-Uniform Memory Access）。  
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/cpu_cache.png" />  
&emsp;&emsp;单个CPU是这样的，内存访问也并不是你印象中的直接由RAM拿到，而是由RAM=>CPU Cache=>CPU。在公司的Server上`lscpu`结果如下：  
```
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    2
Core(s) per socket:    2
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 63
Model name:            Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz
Stepping:              2
CPU MHz:               2400.058
BogoMIPS:              4800.11
Hypervisor vendor:     Xen
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              30720K
NUMA node0 CPU(s):     0-3
```

