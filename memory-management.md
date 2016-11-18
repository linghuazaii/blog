Memory management every engineer need to know!
==============================================

## Prepare reading
### relevant reading:
 - [Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/)<br>
 - [How the Kernel Manages Your Memory](http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory/)<br>
 <br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/kernelUserMemorySplit.png"></img><br><br>
系统启动时，**kernel**资源以及其他启动进程资源被load进`physical memory`。对于32位系统来说，进程启动时，它会拥有一块4G大小的`virtual memory`。我是一个叫`Nerd`的进程，启动的时候**kernel**给我这张图片，告诉我：你能使用的`User Mode Space`只有`3G`，然后从`0xc0000000`到`0xffffffff`是内核地址，你别动它，你没权限。`physical memory`和`virtual memory`的关系就是：`physical memory`是以PAGE的形式**Map**到`virtual memory`，**kernel**管理着所有的`page_table`,所以你程序启动过多的时候`physical memory`会吃紧，但是每次新启进程的`virtual memory`依然是4G。使用`top -p <pid>`查看进程的`resident memory`就是进程实际吃掉的`physical memory`，在检查`memory leak`的时候非常有用。(**TIP**:对于纯C程序来说，开启`mtrace()`,即可统计内存泄漏情况，或者自己用`__malloc_hook`之类的实现内存检测)。总的来说，`virtual memory`只是**kernel**开给每个进程的一张货币而已，`physical memory`分配过多必然会引发通货膨胀,然后发给我`Nerd`进程的货币就贬值了，无所谓啦，我是个`Nerd`。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/virtualMemoryInProcessSwitch.png"></img><br><br>
`process switch`就是通过置换`virtual memory`的形式运转的,因为启动着的进程在`physical memory`里有各自的`resident memory`，所以这个置换只是暂存了`virtual memory`的状态，**kernel**状态，寄存器值等等进程相关的东西。而`thread`共享`process`的`TEXT`段,`DATA`段,`BSS`段,`HEAP`,所以`thread switch` is much cheaper than `process switch`。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/linuxFlexibleAddressSpaceLayout.png"></img><br><br>
更加详细的`virtual memory`是这样的，`TEXT`段存储所有的代码，`DATA`段存储所有的已初始化`global variables`，例如`const char *msg="I am a nerd!"`，`msg`就是存储在`DATA`段, `BSS`段存储所有的未初始化`global variables`,并且内存全部初始化为0，例如`const char *msg`存放在`BSS`段，但是肯定不是指向`"I am a nerd!"`，因为它指向`NULL`。`memory allocator`的实现有两种方式，其一是通过`brk()`系统调用(**TIPS**:`sbrk()` calls `brk()`)，其二是通过`mmap()``ANONYMOUS`或者`/dev/zero`(`mmap()`实现的`calloc()`就是这么干的)。`brk()`是从`program break`开始向高地址增长(**TIPS**:`sbrk(0)`可以获取`program break`, 任何越过`program break`的内存读写都是`illegal access`)，例如`sbrk(4096)`就表示申请`4K`的内存，`sbrk(-4096)`就是释放`4K`内存，`program break`向低地址移动`4096`(**TIPS**:有些platform不支持`sbrk()`传入负值)。`mmap()`申请的内存是由高地址向低地址写的。关于`stack`，使用`ulimit -s`显示最大可用栈大小，一般是`8192`即`8K`,所以谨慎使用递归算法，很容易造成`stack overflow`。使用`stack`的一个好处就是，`stack`上频繁使用的`location`很可能会被`mirror`进CPU `L1 cache`，但是也不能完全依赖这一点，因为不是我们能控制的，over-use很容易造成`stack overflow`。(relevant reading:>[stack and cache question](http://www.gamedev.net/topic/564817-stack-and-cache-question-optimizing-sw-in-c/#entry4617168) >[c++ stack memory and cpu cache](http://stackoverflow.com/questions/23760725/c-stack-memory-and-cpu-cache))<br>
如果你是一个细心的读者，可能会问：为什么`TEXT`段的起始地址是`0x08048000`。__Here is the explain__:<br>
使用`cat /proc/self/maps`查看当前`terminal`的内存`map`情况。<br>
对于32位系统是这样的：<br>
```
001c0000-00317000 r-xp 00000000 08:01 245836     /lib/libc-2.12.1.so
00317000-00318000 ---p 00157000 08:01 245836     /lib/libc-2.12.1.so
00318000-0031a000 r--p 00157000 08:01 245836     /lib/libc-2.12.1.so
0031a000-0031b000 rw-p 00159000 08:01 245836     /lib/libc-2.12.1.so
0031b000-0031e000 rw-p 00000000 00:00 0 
00376000-00377000 r-xp 00000000 00:00 0          [vdso]
00852000-0086e000 r-xp 00000000 08:01 245783     /lib/ld-2.12.1.so
0086e000-0086f000 r--p 0001b000 08:01 245783     /lib/ld-2.12.1.so
0086f000-00870000 rw-p 0001c000 08:01 245783     /lib/ld-2.12.1.so
08048000-08051000 r-xp 00000000 08:01 2244617    /bin/cat
08051000-08052000 r--p 00008000 08:01 2244617    /bin/cat
08052000-08053000 rw-p 00009000 08:01 2244617    /bin/cat
09ab5000-09ad6000 rw-p 00000000 00:00 0          [heap]
b7502000-b7702000 r--p 00000000 08:01 4456455    /usr/lib/locale/locale-archive
b7702000-b7703000 rw-p 00000000 00:00 0 
b771b000-b771c000 r--p 002a1000 08:01 4456455    /usr/lib/locale/locale-archive
b771c000-b771e000 rw-p 00000000 00:00 0 
bfbd9000-bfbfa000 rw-p 00000000 00:00 0          [stack]
```
`0x08048000`之前是`library kernel maaped for syscalls`,事实上呢，你可以`map`任何你想要的东西到这块内存，`128M`哟。<br>
对于64位系统是这样的：<br>
```
00400000-0040b000 r-xp 00000000 ca:01 400116                             /bin/cat
0060a000-0060c000 rw-p 0000a000 ca:01 400116                             /bin/cat
0062c000-0064d000 rw-p 00000000 00:00 0                                  [heap]
7f38ab82e000-7f38b1d55000 r--p 00000000 ca:01 454475                     /usr/lib/locale/locale-archive
7f38b1d55000-7f38b1f0c000 r-xp 00000000 ca:01 396116                     /lib64/libc-2.17.so
7f38b1f0c000-7f38b210c000 ---p 001b7000 ca:01 396116                     /lib64/libc-2.17.so
7f38b210c000-7f38b2110000 r--p 001b7000 ca:01 396116                     /lib64/libc-2.17.so
7f38b2110000-7f38b2112000 rw-p 001bb000 ca:01 396116                     /lib64/libc-2.17.so
7f38b2112000-7f38b2117000 rw-p 00000000 00:00 0
7f38b2117000-7f38b2138000 r-xp 00000000 ca:01 396509                     /lib64/ld-2.17.so
7f38b2323000-7f38b2326000 rw-p 00000000 00:00 0
7f38b2337000-7f38b2338000 rw-p 00000000 00:00 0
7f38b2338000-7f38b2339000 r--p 00021000 ca:01 396509                     /lib64/ld-2.17.so
7f38b2339000-7f38b233a000 rw-p 00022000 ca:01 396509                     /lib64/ld-2.17.so
7f38b233a000-7f38b233b000 rw-p 00000000 00:00 0
7ffcffe94000-7ffcffeb5000 rw-p 00000000 00:00 0                          [stack]
7ffcfffa1000-7ffcfffa3000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```
`libc`和`ld`被`map`到了堆内存，`0x00400000`以下的地址可能有东西，也可能没东西，你同样可以`map`你想要的东西到这块内存。<br>
更加细心的读者可能要问：为什么要有`random brk offset`,`random stack offset`和`random mmap offset`?这是为了避免直接算得某个进程的`virtual memory`详细地址，然后可以利用这个远程`PWN`(反正我不会，大牛可以教教我^_^)。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/mappingBinaryImage.png"></img><br><br>
这个图是更加详细的解释，`const char *msg="I am a nerd!"`,`msg`会存在`DATA`段，`"I am a nerd!"`存在`TEXT`段，是只读的。可执行文件大小 = `TEXT` + `DATA`！<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/linuxClassicAddressSpaceLayout.png"></img><br><br>
这个是经典的没有各种`offset`的`virtual memory` layout。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/mm_struct.png"></img><br>
<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/memoryDescriptorAndMemoryAreas.png"></img><br><br>
我就不解释了，不会kernel，详情自己看[relevant reading: How the Kernel Manages Your Memory](#relevant-reading)

## Dinner time: Memory allocator and memory management of glibc

 - `glibc`内存管理基于`ptmalloc`,`ptmalloc`基于`dlmalloc`
 - `dlmalloc`源码：[dlmalloc](https://github.com/linghuazaii/dlmalloc)
 - `ptmalloc`源码：[ptmalloc](http://www.malloc.de/malloc/ptmalloc3-current.tar.gz)
<br><br>
```
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;
};
```
`dlmalloc`使用`malloc_chunk`结构体存储每块申请的内存的具体信息，在32位系统里`sizeof(malloc_chunk) = 16`，在64位系统里`sizeof(malloc_chunk) = 32`，**example:**<br>
```
int main(int argc, char **argv) {
    void *mem = malloc(0);
    malloc_stats();

    return 0;
}
```
我的机器是64位的，调用`malloc(0)`其实也申请了内存，这块内存空间大小就是`sizeof(malloc_chunk) = 32`，运行以上代码将显示如下结果:<br>
> Arena 0:<br>
> system bytes     =     135168<br>
> **in use bytes     =         32**<br>
> Total (incl. mmap):<br>
> system bytes     =     135168<br>
> in use bytes     =         32<br>
> max mmap regions =          0<br>
> max mmap bytes   =          0<br>  

所以，你申请的内存看上去是这样子的。<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/malloc_chunk.png"></img><br><br>

```
struct malloc_state {
  /* The maximum chunk size to be eligible for fastbin */
  INTERNAL_SIZE_T  max_fast;   /* low 2 bits used as flags */
  /* Fastbins */
  mfastbinptr      fastbins[NFASTBINS];
  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr        top;
  /* The remainder from the most recent split of a small request */
  mchunkptr        last_remainder;
  /* Normal bins packed as described above */
  mchunkptr        bins[NBINS * 2];
  /* Bitmap of bins. Trailing zero map handles cases of largest binned size */
  unsigned int     binmap[BINMAPSIZE+1];
  /* Tunable parameters */
  CHUNK_SIZE_T     trim_threshold;
  INTERNAL_SIZE_T  top_pad;
  INTERNAL_SIZE_T  mmap_threshold;
  /* Memory map support */
  int              n_mmaps;
  int              n_mmaps_max;
  int              max_n_mmaps;
  /* Cache malloc_getpagesize */
  unsigned int     pagesize;
  /* Track properties of MORECORE */
  unsigned int     morecore_properties;
  /* Statistics */
  INTERNAL_SIZE_T  mmapped_mem;
  INTERNAL_SIZE_T  sbrked_mem;
  INTERNAL_SIZE_T  max_sbrked_mem;
  INTERNAL_SIZE_T  max_mmapped_mem;
  INTERNAL_SIZE_T  max_total_mem;
};

typedef struct malloc_state *mstate;
static struct malloc_state av_;
```
这个名叫`av_`的结构体就是用来存储内存申请的具体信息的，无论是`brk`的也好，还是`mmap`的也好，都会记录在案。<br>
<br>

**关于malloc**<br>
 - 如果`malloc`申请内存大小超过`M_MMAP_THRESHOLD`即`128 * 1024`并且`free list`里没有满足需要的内存大小，`malloc`就会调用`mmap`申请内存。因为有一个`header`，所以大小为`128 * 1024 - 32`，具体信息可以`man mallopt`，我就不贴`dlmalloc`的源码了，感兴趣可以自己去翻。**example:(strace跟踪系统调用)**<br>
```
int main(int argc, char **argv) {
    void *mem = malloc(1024 * 128 - 24);
    malloc_stats();

    return 0;
}
```
编译以上代码，`strace`结果：<br>
> brk(0)                                  = 0xa2c000<br>
> brk(0xa6d000)                           = 0xa6d000<br>  

```
int main(int argc, char **argv) {
    void *mem = malloc(1024 * 128 - 24 + 1);
    malloc_stats();

    return 0;
}
```
编译以上代码，`strace`结果：<br>
> mmap(NULL, 135168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f65339a6000<br>  

为什么是24而不是`sizeof(malloc_chunk)`，我猜`glibc`修改了`dlmalloc`实现，因为我看到的`dlmalloc`源码不是这样子的。<br>
<br>


