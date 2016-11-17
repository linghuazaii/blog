Memory management every engineer need to know!
==============================================

## Prepare reading
relevant reading:[Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/)<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/kernelUserMemorySplit.png"></img><br>
系统启动，资源load进`physical memory`，对于32位系统来说，每个进程启动都会拥有一块4G大小的`virtual memory`。我是一个叫`Nerd`的进程，启动的时候kernel给我这个玩意儿，告诉我我能使用的`User Mode Space`只有3G，然后从`0xc0000000`到`0xffffffff`是内核地址，你别动它，你没权限。`physical memory`和`virtual memory`的关系就是：`physical memory`是以PAGE的形式Map到`virtual memory`，kernel管理着所有的`page_table`,所以你程序启动过多的时候`physical memory`会吃紧，`virtual memory`依然是4G。使用`top -p <pid>`查看进程的`resident memory`就是进程实际吃掉的`physical memory`，在检查`memory leak`的时候非常有用。(TIP:对于纯C程序来说，开启`mtrace()`,即可统计内存泄漏情况，或者自己用`__malloc_hook`之类的实现内存检测)。总的来说，`virtual memory`只是kernel开给每个进程的一张货币而已，`physical memory`分配过多必然会引发通货膨胀,然后发给我`Nerd`进程的货币就贬值了，无所谓啦，我是个`Nerd`。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/virtualMemoryInProcessSwitch.png"></img><br>
`process switch`就是通过置换`virtual memory`的形式运转的,因为启动着的进程在`physical memory`里有各自的`resident memory`，所以这个置换只是暂存了`virtual memory`的状态，kernel状态，寄存器值等等进程相关的东西。而`thread`共享`process`的`TEXT`,`DATA`,`BSS`,`HEAP`,所以`thread switch` is much cheaper than `process switch`。<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/linuxFlexibleAddressSpaceLayout.png"></img><br>
更加详细的`virtual memory`是这样的，`TEXT`段存储所有的代码，`DATA`段存储所有的已初始化`global variables`，例如`const char *msg="I am a nerd!"`，就是存储在`DATA`段, `BSS`段存储所有的未初始化`global variables`并且内存全部初始化为0，例如`const char *msg`存放在`BSS`段，但是肯定不是指向`"I am a nerd!"`，因为它指向`NULL`。`memory allocator`的实现有两种方式，其一是通过`brk()`系统调用(TIPS:`sbrk()` calls `brk()`)，其二是通过`mmap()``ANONYMOUS`。`brk()`是从`program break`开始向高地址增长(TIPS:`sbrk(0)`可以获取`program break`, 任何越过`program break`的内存读写都是`illegal access`)，例如`sbrk(4096)`就表示申请`4K`的内存，`sbrk(-4096)`就是释放`4K`内存，`program break`向低地址移动`4096`(TIPS:有些platform不支持`sbrk()`传入负值)。`mmap()`申请的内存是由高地址向低地址写的。关于`stack`，使用`ulimit -s`显示最大可用栈大小，一般是`8192`即`8K`,所以谨慎使用递归算法，很容易造成`stack overflow`。使用`stack`的一个好处就是，`stack`上频繁使用的`location`很可能会被`mirror`进CPU `L1 cache`，但是也不能完全依赖这一点，因为不是我们能控制的，over-use很容易造成`stack overflow`。(relevant reading:>[stack and cache question](http://www.gamedev.net/topic/564817-stack-and-cache-question-optimizing-sw-in-c/#entry4617168) >[c++ stack memory and cpu cache](http://stackoverflow.com/questions/23760725/c-stack-memory-and-cpu-cache))<br>
如果你是一个细心的读者，可能会问：为什么`TEXT`段的起始地址是`0x08048000`。Here is the explain:<br>
使用`cat /proc/self/maps`查看当前`terminal`的内存`map`情况。对于32位系统是这样的：<br>
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

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/mappingBinaryImage.png"></img><br>
这个图是更加详细的解释，`const char *msg="I am a nerd!"`,`msg`会存在`DATA`段，`"I am a nerd!"`存在`TEXT`段，是只读的。可执行文件大小 = `TEXT` + `DATA`！<br>
<br>

&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/linuxClassicAddressSpaceLayout.png"></img><br>
这个是经典的没有各种`offset`的`virtual memory` layout。<br>
<br>



