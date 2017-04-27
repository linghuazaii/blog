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
单个CPU有三级缓存，访问的clock cycle递增，L1 Cache最快。x86用的CISC指令集，CISC指令集比RISC复杂，Decode的Cost要更高，所以有两个L1 Cache，即L1i（instruction）Cache，用来缓存Decode的CPU指令；L1d（Data）Cache，用来缓存内存数据。    
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/multi_processor.png" />    
&emsp;&emsp;多个CPU是这样的，图中给出了两个物理CPU，每个物理CPU有两个Core，每个Core有两个thread，两个thread共享L1i和L1d，Intel的thread实现是这样的，它们有各自独有的寄存器，但是有些寄存器还是共享的，以上就是Intel的Hyper Thread设计。当然，只是一个例子，不同的CPU可能用的不同的架构。
