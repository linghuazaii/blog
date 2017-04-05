[Bug Hunter]关于stack overflow Lesson II
=======================================

### 回顾
&emsp;&emsp;上一篇我们已经简单了解了stackoverflow的简单原理，以及怎么利用stackoverflow去控制程序的逻辑，本节Lesson更加深入的探讨一下stackoverflow的进一步利用点。这一个系列是根据博主本人的学习进度写成，所以速度可能偏慢，因为我也有活要干，同时我也是个新手。


### 例子一
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int check_authentication(char *password) {
	int auth_flag = 0;
	char password_buffer[16];

	strcpy(password_buffer, password);
	
	if(strcmp(password_buffer, "brillig") == 0)
		auth_flag = 1;
	if(strcmp(password_buffer, "outgrabe") == 0)
		auth_flag = 1;

	return auth_flag;
}

int main(int argc, char *argv[]) {
	if(argc < 2) {
		printf("Usage: %s <password>\n", argv[0]);
		exit(0);
	}
	if(check_authentication(argv[1])) {
		printf("\n-=-=-=-=-=-=-=-=-=-=-=-=-=-\n");
		printf("      Access Granted.\n");
		printf("-=-=-=-=-=-=-=-=-=-=-=-=-=-\n");
	} else {
		printf("\nAccess Denied.\n");
   }
}
```
	
&emsp;&emsp;本例是一个简单的认证判断，如果输入参数是`brillig`或者`outgrabe`，那么输出`Access Granted.`。显然这里也有一个stackoverflow的bug，在`strcpy`函数这里。在Stack上，`auth_flag`先入栈，`password_buffer`后入栈，所以`auth_flag`的地址比`password_buffer`高，这样就可以覆盖`auth_flag`的值。`gcc example.c -o example -g`，生成可执行程序。然后`gdb -tui example`获取一些必要的信息如下。  


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
