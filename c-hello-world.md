[新手乐园]和我一起学C - Hello World
===========================

### 前言
&emsp;&emsp;如果你有过任何一门语言的编程经验，请绕行。本文适用于没有任何编程经验的Newbies，哈哈！滴滴，开车啦！

### 关于中间层
&emsp;&emsp;有个逗逼曾经说过：“计算机科学的任何复杂问题都可以通过创造一个或者多个中间层来简化！” 好有道理啊，当时当时理解的并不深。010101编码的打孔机时代，蛋疼吧，后来就有了类似于指令的东西，每个指令表示一个操作，这样编码就方便了很多，所以指令到010101之间有一个translator，这个translator就是一个中间层。后来有了个东西叫做cpu就是专门用来干这个事的，这层的translator我们就称之为ISA吧，就是Instruction Set Architecture，不同的CPU可能是不同的ISA，比如说cisc指令集和risc指令集就不一样，现在的web后端 server基本都是x86的吧，不信你`lscpu`看一哈，而x86的指令集是risc，risc指令集相比cisc简单得多，不需要一个中间的microprocessor去做translator，cisc的第一个c就是complex的简写。然后啊，intel的cpu指令集也不是我们程序猿／媛HOLD得住的哈，所以又来了一层，assembly language，就是汇编啦，汇编到cpu指令集呢，也需要一个中间层translator，例如windows平台下的MASM／Linux平台下的NASM，在这里鄙视一下垃圾windows，不好用，SHIT！然后某天，一堆高级语言诞生来，像c／c++，java之类的，她们到汇编之间呢又有一层translator，java的JVM，或者gcc/g++。像什么python啊，PHP啊，ruby啊我就不提了，我是搞c/c++的，虽然没你们赚的多，但是也无所谓啦～所以呢，论架构之类的乱七八糟的玩意儿，都是都是解耦，像什么插件化设计啊，后端业务拆分微服务化啊，反正呢，总结起来就是一句话：给我一堆Virtual Machine，我能撬起整个地球！

### x86的内存模型
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/memory-model.png" />
<br>
&emsp;&emsp;我来一个个解释哈，后面我会直接上一个x86-64汇编版本的Hello World。现在的内存模型，早已不是远古时期DOS时代的真实地址模型了，virtual memory，也就是每个程序先给你一个4G的Sandbox玩，寄存器里存的各个段的地址都是偏移地址，真实地址CPU会帮我们算出来。而且呢，程序的起始偏移地址并不是0x00000000,而是0x400000，这段内存里存的是标准库的一些信息，我们不用理她。程序呢，是一段一段组成的，data段啊，text段啊，bss段啊，各种SS，DS，CS，ES，FS，GS寄存器就是存一个段起始值，然后SP，DI，SI啊可以存一个偏移，如图中DS:SI就指向了一个具体的地址。现在我只需要向你强调一件事：地址这个东西非常非常重要，它是程序跑起来的核心！

### 那些逗逼寄存器们
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/Register386.png" />
<br>
&emsp;&emsp;eax，ebx，ecx，edx这几个是通用的，32位，去掉e，例如ax就是16位，ah或者al就是高八位和低八位，esi，edi，esp，ebp没八位那一说，用法看CPU喜好。CS即Code Segment，DS即Data Segment，SS即Stack Segment，ES，FS，GS是扩展用的，因为可能不止一个段，而且一条指令可能需要多个数据段的数据，所以这几个是扩展用的。EFLAGS这是是存CPU状态的，像什么溢出啊之类的。最经典的EIP/RIP，这个大妹子可厉害了，存的是下一条指令的地址。先恭喜一下360安全团队夺取本届Pwn2Own世界冠军。如果你能够找到一个程序的漏洞，并且这个漏洞可以用来获得EIP/RIP控制权，那么你就可以执行任意代码，随意更改代码逻辑，如果你发现知名厂商有这种漏洞的话，那么恭喜你，0day漏洞，拿去卖吧，买房不是梦～寄存器就是这样的，我知道你还没听懂，没关系，下面我们来干活，抄起键盘就是干～

### x86-64版本Hello World
```
; This file is auto-generated.Edit it at your own peril.
section .data
msg: db "Hello World!",10
msglen: equ $-msg

section .text

global _start
_start:
    nop ; make gdb happy
    ; put your experiments between these nop
    mov eax,1
    mov edi,1
    mov esi,msg
    mov edx,msglen
    syscall
    ; put your expeirments between these nop
    nop ; make gdb happy
    
    ; exit 
    mov eax,60 ; system call 60: exit
    xor edi, edi ; set exit status to zero
    syscall ; call the operating system

section .bss
```
&emsp;&emsp;出家人不打诳语，你看你看，`.data`段，存了一个`msg`变量，以及`msglen`，就是已经初始化的全局变量，所谓的静态存储区，`.text`段存的是代码，`_start`就是程序的起始地址，你会发现c程序的symbol table里面也有个`_start`符号。<br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/instruction.png" />
<br>
&emsp;&emsp;这里我就直接上指令了，`mov eax, 1`表示我们要调用`sys_write`，`mov edi,1`表示我们要写描述符1，即`stdout`，`mov edi,msg`表示传入要写内容的地址，`mov edx,msglen`表示传入消息长度，然后`syscall`中断，让操作系统干事去。你仔细看，其实这段汇编完全遵循着intel x86的指令集表，是不是？你现在在把前面指令集啊，寄存器啊啥的联系起来想一想，程序其实就是这么回事儿。然后后端就是调用退出指令：<br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/exit-instruction.png" />
<br>
&emsp;&emsp;我没骗你吧，遵循着那张指令集表呢。如果你翻过Linux内核源码你就会发现很多`_sys`打头的函数，看看那堆乱七八糟的宏定义～
<br>
&emsp;&emsp;`nasm -f elf64 -g -F stabs test.asm -o test.o`，然后`ld test.o -o test`。让我们调试一下：`gdb -tui test`
<br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/test.asm.png" />
<br>
&emsp;&emsp;操作细节我就不写了，你自己慢慢玩嘛，看看寄存器的变化啥的。然后我们看看symbol table，`readelf -s test`<br>
```
Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000004000b0     0 SECTION LOCAL  DEFAULT    1
     2: 00000000006000d4     0 SECTION LOCAL  DEFAULT    2
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 002.asm
     6: 00000000006000d4     1 OBJECT  LOCAL  DEFAULT    2 msg
     7: 000000000000000d     0 NOTYPE  LOCAL  DEFAULT  ABS msglen
     8: 00000000004000b0     0 NOTYPE  GLOBAL DEFAULT    1 _start
     9: 00000000006000e1     0 NOTYPE  GLOBAL DEFAULT    2 __bss_start
    10: 00000000006000e1     0 NOTYPE  GLOBAL DEFAULT    2 _edata
    11: 00000000006000e8     0 NOTYPE  GLOBAL DEFAULT    2 _end
```
<br>
&emsp;&emsp;然后我们再反编译一下对比一下，`objdump -S --disassemble test` <br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/reverse-test.png" />
<br>
&emsp;&emsp;和汇编版本差别不大是不是，但是对于C来说可就大了去了～
<br>
### C语言版本Hello World  

```
#include <stdio.h>
int main(int argc, char **argv) {
    printf("Hello World!");

    return 0;
}
```
<br>
&emsp;&emsp;C语言版本够简单把，才这几行～ 我们看看符号表，`readelf -s a.out`<br>
```
Symbol table '.symtab' contains 69 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
    32: 0000000000000000     0 SECTION LOCAL  DEFAULT   32
    33: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    34: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   21 __JCR_LIST__
    35: 0000000000400460     0 FUNC    LOCAL  DEFAULT   14 deregister_tm_clones
    36: 0000000000400490     0 FUNC    LOCAL  DEFAULT   14 register_tm_clones
    37: 00000000004004d0     0 FUNC    LOCAL  DEFAULT   14 __do_global_dtors_aux
    38: 000000000060102c     1 OBJECT  LOCAL  DEFAULT   26 completed.6337
    39: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   20 __do_global_dtors_aux_fin
    40: 00000000004004f0     0 FUNC    LOCAL  DEFAULT   14 frame_dummy
    41: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   19 __frame_dummy_init_array_
    42: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS h.c
    43: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    44: 0000000000400718     0 OBJECT  LOCAL  DEFAULT   18 __FRAME_END__
    45: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   21 __JCR_END__
    46: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
    47: 0000000000600e18     0 NOTYPE  LOCAL  DEFAULT   19 __init_array_end
    48: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   22 _DYNAMIC
    49: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   19 __init_array_start
    50: 00000000004005f0     0 NOTYPE  LOCAL  DEFAULT   17 __GNU_EH_FRAME_HDR
    51: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   24 _GLOBAL_OFFSET_TABLE_
    52: 00000000004005c0     2 FUNC    GLOBAL DEFAULT   14 __libc_csu_fini
    53: 0000000000601028     0 NOTYPE  WEAK   DEFAULT   25 data_start
    54: 000000000060102c     0 NOTYPE  GLOBAL DEFAULT   25 _edata
    55: 00000000004005c4     0 FUNC    GLOBAL DEFAULT   15 _fini
    56: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@@GLIBC_2.2.5
    57: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    58: 0000000000601028     0 NOTYPE  GLOBAL DEFAULT   25 __data_start
    59: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    60: 00000000004005d8     0 OBJECT  GLOBAL HIDDEN    16 __dso_handle
    61: 00000000004005d0     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
    62: 0000000000400550   101 FUNC    GLOBAL DEFAULT   14 __libc_csu_init
    63: 0000000000601030     0 NOTYPE  GLOBAL DEFAULT   26 _end
    64: 0000000000400430     0 FUNC    GLOBAL DEFAULT   14 _start
    65: 000000000060102c     0 NOTYPE  GLOBAL DEFAULT   26 __bss_start
    66: 000000000040051d    37 FUNC    GLOBAL DEFAULT   14 main
    67: 0000000000601030     0 OBJECT  GLOBAL HIDDEN    25 __TMC_END__
    68: 00000000004003c8     0 FUNC    GLOBAL DEFAULT   11 _init
```
<br>
&emsp;&emsp;这符号表是不是复杂多了，但是`_start`入口还是一样会有，当然也少不了`main`啦,让我们调试一下`gdb -tui a.out`<br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/main.png" />
<br>
&emsp;&emsp;从`_start`开始会调到`__libc_start_main@plt`,关于plt我这里要唠叨两句，如果你写过插件化的架构或者其他的东西肯定会用到`dlopen`，这又要扯到什么链接装载与库，什么乱七八糟的东西，我不想扯。连接器将一堆.o链接成一个可执行文件的时候就是找符号，没有的就向后找，凡是使用到的都去找，第一个找到的扔进plt表，如果一个都找不到那么就出现`Undefined Reference`错误。看上边的符号表里也有`__libc_start_main`，对吧，联系起来了吧。然后我们继续调试会发现函数调用次序是这样的`printf@plt` => `vfprintf` => `strchrnul`，反正具体什么玩意我也不是很懂，毕竟我也是个Newbie。到这里呢，如果你也是跟着我一起gdb了，你会发现，一些都是地址，地址指向代码段，数据段，然后整到寄存器里，然后送到CPU，CPU把一切都搞定了，程序就跑起来啦。
<br>
&emsp;&emsp;让我们反编译一下a.out看看是些什么鬼东西把～ `objdump -S --disassemble a.out`,汇编指令我就省略掉大部分
<br>
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/reverse-a.out.png" />
&emsp;&emsp;你发现了吗，符号表里的symbol对应着段，段对应着一个起始地址，起始地址之后是一段汇编代码，一切的一切在这里都联系起来了吧，一切的一切都是地址，给我地址我就能拿到所有能拿到的东西。

### 小结
&emsp;&emsp;小结个屁，这都晚上12:30了，看了这么久，给你们出个简单的小题目，看你们看懂了没：<br>
&emsp;&emsp;我现在有两个库a.so和b.so，两个库里面都有一个print 函数，a.so里输出A，b.so里输出B，然后我有一个主程序文件main.c，<br>
&emsp;&emsp;如果我`gcc -L. main.c -o test -la -lb`，运行程序输出啥？<br>
&emsp;&emsp;如果我`gcc -L. main.c -o test -lb -la`，运行程序输出啥？<br>
&emsp;&emsp;欢迎评论，就算你评论了，我也不会理你的～
&emsp;&emsp;**GLHF~**
