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
