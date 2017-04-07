[Bug Hunter]浅说shellcode
========================================

### 回顾
&emsp;&emsp;上一篇我们进一步了解了stackoverflow Bug的利用点，就是通过复写函数返回地址的方式来控制程序的逻辑，本篇对shellcode进行一个简单的介绍。虽然本篇暂时和上连篇没有联系，但是我会在Lesson IV里将shellcode和stackoverflow的Bug结合起来。

### 关于shellcode
&emsp;&emsp;何为shellcode呢，shellcode就是一段可执行的二进制代码，可执行程序的代码段是只读的，即`.text`段是只读的，那么在我们无法更改程序`.text`段的情况下怎么执行我们自己的代码呢，那就是我们前两篇一直提到的Stack，但是本篇例子放shellcode的地方并不在Stack上，而是在`.data`段，因为`const char code[]`是一个全局已经初始化的变量，但是无关本篇的结论，你愿意的话可以把它放到`main`里去，并不影响结果。

### shellcode测试模板
```c++
/*shellcodetest.c*/ 
const char code[] = "bytecode will go here!";
int main(int argc, char **argv)
{
	int (*func)();
	func = (int (*)()) code;
	(int)(*func)();
}
```
&emsp;&emsp;这段代码比较简单，稍微解释一下吧，函数名在汇编这一层也就是一个label，一个具体地址的替代而已，程序的执行就是顺着地址走的，地址里存的就是具体的cpu指令，而我们的shellcode就是cpu指令，下面我会show给你看的。所以这个例子就是讲code地址转换为函数指针，然后执行函数，就是执行我们的shellcode。原理清楚了，我们试着跑一下这个测试模板，可想而知，`code`的数据并不是cpu指令，所以会引发segment fault。你可以编译运行一下以上的代码段，结果如下：  
> Segmentation fault  

### 测试一个可用的shellcode
```asm
; exit.asm
[SECTION .text]
global _start
_start:
    xor eax, eax
    mov al, 1
    mov ebx, 10
    int 0x80
```
&emsp;&emsp;这段汇编的作用就是调用了系统调用`sys_exit(10)`，这里有x86的系统调用列表，[Linux System Call Table](https://www.informatik.htw-dresden.de/~beck/ASM/syscall_list.html)。先生成shellcode，`nasm -f elf exit.asm`，然后`ld exit.o -o exiter -melf_i386`，然后`objdump -d exiter >log`，这里是log文件的内容：  
> exiter:     file format elf32-i386  
> Disassembly of section .text:  
> 08048060 <_start>:  
> 8048060:       31 c0                   xor    %eax,%eax  
> 8048062:       b0 01                   mov    $0x1,%al  
> 8048064:       31 db                   xor    %ebx,%ebx  
> 8048066:       cd 80                   int    $0x80  

&emsp;&emsp;那么`\x31\xc0\xb0\x01\x31\xdb\xcd\x80`，就是我们要找的cpu指令，即所谓的shellcode，我们可以直接用工具截取`objdump`内容的shellcode，[odfhex](https://github.com/linghuazaii/hacking-stuff/tree/master/odfhex)。代码也比较简单，当然作者并不是博主。`./odfhex log`，得到结果如下：  
> Odfhex - object dump shellcode extractor - by steve hanna - v.01  
> Trying to extract the hex of log which is 279 bytes long  
> "\x31\xc0\xb0\x01\xbb\x0a\x00\x00\x00\xcd\x80";  
> 11 bytes extracted.  

&emsp;&emsp;用以上shellcode替换`code[]`的内容，编译运行，并没有segment fault，程序正常退出，在shell里`echo $?`获取程序执行的返回码，为10，证明我们的shellcode成功执行了。

### 一个简单的hello hacking world的例子
```
;hello.asm
[SECTION .text]
global _start
_start:
    jmp short ender

    starter:
    xor eax, eax    ;clean up the registers
    xor ebx, ebx
    xor edx, edx
    xor ecx, ecx

    mov al, 4       ;syscall write
    mov bl, 1       ;stdout is 1
    pop ecx         ;get the address of the string from the stack
    mov dl, 20       ;length of the string
    int 0x80

    xor eax, eax
    mov al, 1       ;exit the shellcode
    xor ebx,ebx
    int 0x80

    ender:
    call starter	;put the address of the string on the stack
    db 'hello hacking world',0x0a
```
&emsp;&emsp;简单说下这段汇编代码，程序由`_start`开始执行，定义局部变量，存的是字符串`hello hacking world\n`，`0x0a`就是`\n`，调用`call`会直接将下一条指令压栈，所以跳转到`starter`，清空寄存器，`xor`可以用来快速将寄存器清零，然后调用`sys_write`，然后调用`sys_exit()`。编译`hello.asm`，`nasm -f elf hello.asm`，然后`ld hello.o -o hello -melf_i386`，然后`objdump -d hello >log`，然后`./odfhex log`，结果如下：  
> Odfhex - object dump shellcode extractor - by steve hanna - v.01  
> Trying to extract the hex of log which is 1265 bytes long  
> "\xeb\x19\x31\xc0\x31\xdb\x31\xd2\x31\xc9\xb0\x04\xb3\x01\x59\xb2\x14"\  
> "\xcd\x80\x31\xc0\xb0\x01\x31\xdb\xcd\x80\xe8\xe2\xff\xff\xff\x68\x65"\  
> "\x6c\x6c\x6f\x20\x68\x61\x63\x6b\x69\x6e\x67\x20\x77\x6f\x72\x6c\x64"\  
> "\x0a";  
> 52 bytes extracted.  

&emsp;&emsp;将得到的shellcode写到`code[]`数组里面去，编译运行，结果如下：  
> hello hacking world  

### 展望
&emsp;&emsp;以上例子仅仅是为了说明shellcode的基本原理，让我们来展望一下，如果执行我们的shellcode的是一个root user，那么说明什么呢？首先我们可以轻松构造一个shellcode拿到对方机器的shell，然后可以干什么呢？可以干任何事，对方的机器对我们来说就是一个肉鸡啦。下一期我会将shellcode和stackoverflow结合起来，因为shellcode本来自身就是一个非常深的话题，我们这里浅尝辄止，感兴趣可以关注这里的资源[The Shellcoder's Handbook](https://www.amazon.com/Shellcoders-Handbook-Discovering-Exploiting-Security/dp/047008023X)。

### 结语
&emsp;&emsp;没点关注的点一波关注，谢谢！Have Fun！See You！
