[新手乐园]和我一起学C - Hello World
===========================

### 前言
&emsp;&emsp;如果你有过任何一门语言的编程经验，请绕行。本文适用于没有任何编程经验的Newbies，哈哈！滴滴，开车啦！

### 关于中间层
&emsp;&emsp;有个逗逼曾经说过：“计算机科学的任何复杂问题都可以通过创造一个或者多个中间层来简化！” 好有道理啊，当时当时理解的并不深。010101编码的打孔机时代，蛋疼吧，后来就有了类似于指令的东西，每个指令表示一个操作，这样编码就方便了很多，所以指令到010101之间有一个translator，这个translator就是一个中间层。后来有了个东西叫做cpu就是专门用来干这个事的，这层的translator我们就称之为ISA吧，就是Instruction Set Architecture，不同的CPU可能是不同的ISA，比如说cisc指令集和risc指令集就不一样，现在的web后端 server基本都是x86的吧，不信你`lscpu`看一哈，而x86的指令集是risc，risc指令集相比cisc简单得多，不需要一个中间的microprocessor去做translator，cisc的第一个c就是complex的简写。然后啊，intel的cpu指令集也不是我们程序猿／媛HOLD得住的哈，所以又来了一层，assembly language，就是汇编啦，汇编到cpu指令集呢，也需要一个中间层translator，例如windows平台下的MASM／Linux平台下的NASM，在这里鄙视一下垃圾windows，不好用，SHIT！然后某天，一堆高级语言诞生来，像c／c++，java之类的，她们到汇编之间呢又有一层translator，java的JVM，或者gcc/g++。像什么python啊，PHP啊，ruby啊我就不提了，我是搞c/c++的，虽然没你们赚的多，但是也无所谓啦～所以呢，论架构之类的乱七八糟的玩意儿，都是都是解耦，像什么插件化设计啊，后端业务拆分微服务化啊，反正呢，总结起来就是一句话：给我一堆Virtual Machine，我能撬起整个地球！

### x86的内存模型
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/c-hello-world/memory-model.png" />

