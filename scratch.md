Memory management every engineer need to know!
==============================================

## Prepare reading
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/memory_management/kernelUserMemorySplit.png"></img><br>
&emsp;系统启动，资源load进`physical memory`，对于32位系统来说，每个进程启动都会拥有一块4G大小的`virtual memory`。我是一个叫`Nerd`的进程，启动的时候kernel给我这个玩意儿，告诉我我能使用的`User Mode Space`只有3G，然后从`0xc0000000`到`0xffffffff`是内核地址，你别动它，你没权限。`physical memory`和`virtual memory`的关系就是：`physical memory`是以PAGE的形式Map到`virtual memory`，kernel管理着所有的`page_table`,所以你程序启动过多的时候`physical memory`会吃紧，`virtual memory`依然是4G。使用`top -p <pid>`查看进程的`resident memory`就是进程实际吃掉的`physical memory`，在检查`memory leak`的时候非常有用。(TIP:对于纯C程序来说，开启`mtrace()`,即可统计内存泄漏情况，或者自己用`__malloc_hook`之类的实现内存检测)。总的来说，`virtual memory`只是kernel开给每个进程的一张货币而已，`physical memory`分配过多必然会引发通货膨胀,然后发给我`Nerd`进程的货币就贬值了，无所谓啦，我是个Nerd。<br>
<br>

For 0x08048000 in 32x and 0x00400000 in 64x system
32x:
`cat /proc/self/maps`
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
**library kernel mapped for syscalls, albeit you can map anything you like here**

64x:
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
