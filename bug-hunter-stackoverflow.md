[Bug Hunter]关于stack overflow Lesson I
=======================================

### 前言
&emsp;&emsp;无论是你还是我，或者其他所谓的资深或者专家，都或多或少一个不小心就写出了烂代码，留下了价值几十万、几百万、甚至几千万的大坑！下文是第一节课，作为新手的我和大家一起分享，一起学习。
&emsp;&emsp;或许你觉得自己对于Stack已经很了解了，但是我还是得唠叨一句。Stack是一个LIFO是众所周知的，大家不知道的可能是我们日常函数调用或者局部变量使用栈的一些细节。在这里我只是简单的提一下。

### 关于Stack
&emsp;&emsp;你可以简单的写一个C程序，程序里仅仅需要有一个函数的定义，然后在`main`里调用它即可，编译的时候加上`-g`。然后`gdb -tui <executable>`，然后`set disassembly intel`设置成intel指令格式，看个人习惯吧，大部分人都习惯intel指令格式，AT&T看着有些别扭，让人不习惯。然后`layout asm`查看汇编代码，`break main`或者`break *_start`，`ni`逐步往下走。你会发现Stack的用法就是这样的：没进入一个函数，都会把当前的ESP保存到EBP，然后ESP指向当前的函数栈，这个操作叫做ENTER。然后局部变量一个个压栈，完了ret之前调用leave，就是ENTER的逆操作，在进入`main`之前，EBP的值是0x0，就是栈底啦。这个小知识虽然与本文关系不是那么大，但是万丈高楼平地起，知道的原理越接近本质，事物对于你来说就呈现出来的越简单。就好比现在的所谓高级，资深动不动扯开源框架啊什么的，作为初级的我不太喜欢这些太浮于表面到东西，要想自己写一个好轮子，必须从底层到本质开始建楼。所以我从最底层的parallel programming开始，因为更接近事物的本质，从冯·诺伊曼模型开始，怎么由SISD到SIMD再到MIMD，怎么由UMA到NUMA，为了解决什么问题，怎么解决，这些都是最本质到问题。下面回到正题！

### 例子源码
```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
	int value = 5;
	char buffer_one[8], buffer_two[8];

	strcpy(buffer_one, "one"); /* Put "one" into buffer_one. */
	strcpy(buffer_two, "two"); /* Put "two" into buffer_two. */

	printf("[BEFORE] buffer_two is at %p and contains \'%s\'\n", buffer_two, buffer_two);
	printf("[BEFORE] buffer_one is at %p and contains \'%s\'\n", buffer_one, buffer_one);
	printf("[BEFORE] value is at %p and is %d (0x%08x)\n", &value, value, value);

	printf("\n[STRCPY] copying %d bytes into buffer_two\n\n", strlen(argv[1]));
	strcpy(buffer_two, argv[1]); /* Copy first argument into buffer_two. */

	printf("[AFTER] buffer_two is at %p and contains \'%s\'\n", buffer_two, buffer_two);
	printf("[AFTER] buffer_one is at %p and contains \'%s\'\n", buffer_one, buffer_one);
	printf("[AFTER] value is at %p and is %d (0x%08x)\n", &value, value, value);

	if (value == 5)
        printf("execute as expected!\n");
    else if (value == 1801675112) 
        printf("hacked, now we can do a little bad things at this branch!!!\n");
    else
        printf("execute unexpected, fucked up!\n");

    return 0;
}
```
&emsp;&emsp;如果你够仔细到话，你可能会发现这里用到是`strcpy`而不是`strncpy`，甚至还有这么一句`strcpy(buffer_two, argv[1])`，这样就可以简单的构造一个栈溢出，大型的项目里，这种Bug会藏的很深，不然就不会有那么多0day漏洞了。编译一下，`gcc example.c -o example`。  
&emsp;&emsp;运行正常情况`./example 123456`，结果如下：  
> [BEFORE] buffer_two is at 0x7fff9fa850a0 and contains 'two'  
> [BEFORE] buffer_one is at 0x7fff9fa850b0 and contains 'one'  
> [BEFORE] value is at 0x7fff9fa850bc and is 5 (0x00000005)  
>  
> [STRCPY] copying 6 bytes into buffer_two  
>  
> [AFTER] buffer_two is at 0x7fff9fa850a0 and contains '123456'  
> [AFTER] buffer_one is at 0x7fff9fa850b0 and contains 'one'  
> [AFTER] value is at 0x7fff9fa850bc and is 5 (0x00000005)  
> execute as expected!   
&emsp;&emsp;关于为什么`buffer_one`和`buffer_tow`的地址间隔是16，而不是8，我没整明白，如果你知道的话，请不吝赐教。从目前得到的信息来看，`buffer_one`比`buffer_one`高出16个字节，而`value`的地址比`buffer_one`高出12个字节而不是4个，这一点我也没整明白，也请不吝赐教。那么我猜想，如果我输入的参数为16个字节的话，按照`strcpy`的规则来讲，`buffer_one`会被覆盖为0x0，再次运行`./example 1234567890123456`，结果如下：  
> [BEFORE] buffer_two is at 0x7ffd538796d0 and contains 'two'  
> [BEFORE] buffer_one is at 0x7ffd538796e0 and contains 'one'  
> [BEFORE] value is at 0x7ffd538796ec and is 5 (0x00000005)  
>  
> [STRCPY] copying 16 bytes into buffer_two
>  
> [AFTER] buffer_two is at 0x7ffd538796d0 and contains '1234567890123456'  
> [AFTER] buffer_one is at 0x7ffd538796e0 and contains ''  
> [AFTER] value is at 0x7ffd538796ec and is 5 (0x00000005)  
> execute as expected!  
&emsp;&emsp;和预想的一样，那么我们就可以控制`buffer_one`的内容了，再次运行`./example 1234567890123456fuckedup`，结果如下：  
> [BEFORE] buffer_two is at 0x7ffc299904f0 and contains 'two'  
> [BEFORE] buffer_one is at 0x7ffc29990500 and contains 'one'  
> [BEFORE] value is at 0x7ffc2999050c and is 5 (0x00000005)  
>   
> [STRCPY] copying 24 bytes into buffer_two  
>  
> [AFTER] buffer_two is at 0x7ffc299904f0 and contains '1234567890123456fuckedup'  
> [AFTER] buffer_one is at 0x7ffc29990500 and contains 'fuckedup'  
> [AFTER] value is at 0x7ffc2999050c and is 5 (0x00000005)  
> execute as expected!  
&emsp;&emsp;再次如愿以偿，`buffer_one`的内容变成了`fuckedup`，进而我们可以控制`value`的值，让程序换个分支运行，再次运行`./example '1234567890123456fucked up!!!1'`，结果如下：  
> [BEFORE] buffer_two is at 0x7fff596de720 and contains 'two'  
> [BEFORE] buffer_one is at 0x7fff596de730 and contains 'one'  
> [BEFORE] value is at 0x7fff596de73c and is 5 (0x00000005)  
>  
> [STRCPY] copying 29 bytes into buffer_two  
>  
> [AFTER] buffer_two is at 0x7fff596de720 and contains '1234567890123456fucked up!!!1'  
> [AFTER] buffer_one is at 0x7fff596de730 and contains 'fucked up!!!1'  
> [AFTER] value is at 0x7fff596de73c and is 49 (0x00000031)  
> execute unexpected, fucked up!  
&emsp;&emsp;请打开ASCII码表查看1的ASCII码值，十进制和十六进制。是不是一切尽在掌握呢，当然，这个权限还是太小，随着以后的分享，我们一步步深入了解。  

### 留给读者的问题
&emsp;&emsp;还有一个分支是我留给广大的读者朋友的，哈哈，构造怎样的输入才能执行第二个分支呢？请将你的答案在此项目上open一个issue，注明[Bug Hunter Lesson I] + 你的答案，第一位给出正确答案的读者，博主将发一个10块钱的微信红包作为奖励^_^！博主很穷，挣的非常少，还请谅解TAT

### 结语
&emsp;&emsp;没点关注的点一波关注，谢谢！Have Fun！See You！
