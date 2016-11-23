Sysem Monitor For Linux
=======================
```
buddyinfo
cgroups
cmdline
consoles
cpuinfo
crypto
devices
diskstats
dma
execdomains
filesystems
interrupts
iomem
ioports
kallsyms
kcore
keys
key-users
kmsg
kpagecount
kpageflags
latency_stats
loadavg
locks
mdstat
meminfo
misc
modules
mounts
mtrr
net
pagetypeinfo
partitions
sched_debug
schedstat
self
slabinfo
softirqs
stat
swaps
sysrq-trigger
timer_list
timer_stats
uptime
version
vmallocinfo
vmstat
zoneinfo
```


 - `/proc/loadavg`: `0.24 0.33 0.46 2/847 25053`，前三列表示最后1分钟，5分钟，15分钟`CPU`和`IO`的利用率，第四列表示当前运行的`进程数/总进程数`，第五列表示最后一个用过的`进程ID`。

 - `/proc/buddyinfo`:
```
Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32  43388  23561  21073   4093    683    154      7      2      1      0      0
Node 0, zone   Normal 235635 106198  20708     50      0      0      0      0      0      1      0
```
`Node 0`表示只有一个`NUMA Node`，`zone DMA`有`16MB`内存，从低地址开始，被一些`legacy devices`使用；`zone DMA32`存在于64位机器，表示低地址开始`4GB`的内存；`zone Normal`在64位机器上表示从`4GB`开始的内存。其余几列的固定大小的内存块数目(relevant reading: [buddy algorithm](https://www.cs.fsu.edu/~engelen/courses/COP402003/p827.pdf))，分别为`free`状态的`4KB 8KB 16KB 32KB 64KB 128KB 256KB 512KB 1MB 2MB 4MB`内存块数目，Eg. `DMA Zone`有`32KB + 2 * 64KB + 128KB + 256KB + 1MB + 2MB + 3 * 4MB = 15MB544KB`处在`free`状态的内存块，这个数据可以用来查内存碎片情况。
 - `/proc/cgroups`
```
subsys_name    hierarchy       num_cgroups     enabled
cpuset         0               1               1
cpu            0               1               1
cpuacct        0               1               1
memory         0               1               1
devices        0               1               1
freezer        0               1               1
net_cls        0               1               1
blkio          0               1               1
perf_event     0               1               1
hugetlb        0               1               1
```
四列分别表示`controller name`，`hierarchy`全0表示所有`controller`挂载在`cgroups v2 single unified hierarchy`，`num_cgroups`表示有多少个`control groups`挂载在这个`controller`上，`enabled`表示`controller`状态。(relevant reading: [cgroups](http://man7.org/linux/man-pages/man7/cgroups.7.html))

 - `/proc/cmdline`:
```
root=LABEL=/ console=ttyS0 LANG=en_US.UTF-8 KEYTABLE=us
```
表示`kernel`启动的时候传给`kernel`的参数。

 - `/proc/consoles`:
```
ttyS0                -W- (EC p a)    4:64
```
所连接的`consoles`，`ttyS0`表示`device name`，`W`表示可写，`EC p a`分别表示`Enabled``Preferred console``used for printk buffer``safe to use when cpu is offline`，`4:64`表示`major number:minor number`。(relevant reading: [/proc/consoles](https://www.kernel.org/doc/Documentation/filesystems/proc.txt))

 - `/proc/cpuinfo`:
```
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 63
model name      : Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz
stepping        : 2
microcode       : 0x25
cpu MHz         : 2400.058
cache size      : 30720 KB
physical id     : 0
siblings        : 4
core id         : 0
cpu cores       : 2
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 13
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm xsaveopt fsgsbase bmi1 avx2 smep bmi2 erms invpcid
bogomips        : 4800.11
clflush size    : 64
cache_alignment : 64
address sizes   : 46 bits physical, 48 bits virtual
power management:
```
`cpu family`、`model`、`stepping`表示`cpu`架构类型区分，`microcode`记录更新的版本号或者类似的信息，`cache size`表示`cpu L2 cache`的大小，这里为`30M`，`physical id``processor``cpu cores``siblings``core id`表示只有一块物理`cpu`，但是有四个`processor`，可以同时运行四个`hyperthreads`在`0-2 core`上，`flags`表示该`cpu`的特性，`bogomips`表示每秒百万次级别衡量`cpu`什么也不做的情况，`cache_alignment`应该表示是`64bit`对齐的，即`8bytes`，具体没Google到，`address sizes`表示地址总线数，`46 bits physical`表示最大支持内存为`2 ^ 46bytes = 65536GB`，`48 bits virtual`表示`virtual memory`寻址总线数，最大支持的`virtual memory`内存大小为`2 ^ 48bytes = 262144GB`。

 - `/proc/crypto`:
```
name         : ecb(arc4)
driver       : ecb(arc4-generic)
module       : ecb
priority     : 0
refcnt       : 1
selftest     : passed
type         : blkcipher
blocksize    : 1
min keysize  : 1
max keysize  : 256
ivsize       : 0
geniv        : <default>
```
表示`kernel`支持的加密算法。

 - `/proc/devices`:
```
Character devices:
  1 mem
  4 /dev/vc/0
  4 tty
  4 ttyS
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  7 vcs
 10 misc
 13 input
108 ppp
128 ptm
136 pts
202 cpu/msr
203 cpu/cpuid
253 hidraw
254 bsg

Block devices:
259 blkext
  9 md
202 xvd
253 device-mapper
254 mdp
```
表示系统挂载的设备，以`Character device`和`Block device`区分，区别见[/proc/devices](https://www.centos.org/docs/5/html/5.1/Deployment_Guide/s2-proc-devices.html)

 - `/proc/diskstats`:
```
202 0 xvda 14461686 42539668 1278044167 78747120 281646345 132327060 17709904384 2309947396 0 124008400 2388329900
202 1 xvda1 14461153 42539028 1278039312 78746860 281646343 132327052 17709904304 2309947396 0 124008176 2388340472
```
表示`block device`的`I/O`状态，各列分别表示`major number`,`minor number`,`device name`,`read成功次数`,`read merge次数`,`sectors read次数`,`reading总耗时(ms)`,`write成功次数`,`write merge次数`,`sector write次数`,`write总耗时(ms)`,`当前正在处理的I/O个数`,`I/O总耗时(ms)`,`I/O加权总耗时(ms)`。


