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

