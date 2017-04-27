[Parallel Programming]深入PThread (Lesson I)
============================================

### 前言
&emsp;&emsp;因为最近一直在研究Parallel Programming方面的东西，Lock的实现，一些Design Patterns以及atomic，fence，lock-free之类的东西。我学东西都是比较分散的，不是一个东西一本书或者一份资料啃到底。今天刚好到PThreads，将一些东西记录如下。

### Hyper Thread
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/UMA.png" />  
&emsp;&emsp;在你的印象中，内存是不是这样的？古老的UMA（Uniform Memory Access），多个CPU共享一整个RAM。
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/NUMA.png" />  
&emsp;&emsp;现在的Linux Server，基本都是这样的，是不是感觉世界瞬间不一样了？即所说的NUMA（Non-Uniform Memory Access）。  
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/cpu_cache.png" />  
&emsp;&emsp;单个CPU是这样的，内存访问也并不是你印象中的直接由RAM拿到，而是由RAM=>CPU Cache=>CPU。在公司的Server上`lscpu`结果如下：  
```
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    2
Core(s) per socket:    2
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 63
Model name:            Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz
Stepping:              2
CPU MHz:               2400.058
BogoMIPS:              4800.11
Hypervisor vendor:     Xen
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              30720K
NUMA node0 CPU(s):     0-3
```
单个CPU有三级缓存，访问的clock cycle递增，L1 Cache最快。x86用的CISC指令集，CISC指令集比RISC复杂，Decode的Cost要更高，所以有两个L1 Cache，即L1i（instruction）Cache，用来缓存Decode的CPU指令；L1d（Data）Cache，用来缓存内存数据。    
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/pthreads/multi_processor.png" />    
&emsp;&emsp;多个CPU是这样的，图中给出了两个物理CPU，每个物理CPU有两个Core，每个Core有两个thread，两个thread共享L1i和L1d，Intel的thread实现是这样的，它们有各自独有的寄存器，但是有些寄存器还是共享的，以上就是Intel的Hyper Thread设计。当然，只是一个例子，不同的CPU可能用的不同的架构。你可能会问，这些和Pthread有关吗？关系不大，但是了解了这些便于你更好的理解Memory Model和内存可见性方面的问题。

### pthread_create
&emsp;&emsp;我们调用`pthread_create(&tid, NULL, start_routine, args)`会经历哪些步骤呢？将attribute置为NULL是为了简化问题，下文我们会提到的。
```c
int __pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
                      void *(*start_routine) (void *), void *arg)
{
  const struct pthread_attr *iattr = (struct pthread_attr *) attr;
  struct pthread_attr default_attr;
  if (iattr == NULL)
    {
      lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
      default_attr = __default_pthread_attr;
      lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
      iattr = &default_attr;
    }

  struct pthread *pd = NULL;
  int err = ALLOCATE_STACK (iattr, &pd);
  pd->start_routine = start_routine;
  pd->arg = arg;
  atomic_increment (&__nptl_nthreads);
  bool thread_ran = false;
  retval = create_thread (pd, iattr, true, STACK_VARIABLES_ARGS,
}
```
&emsp;&emsp;函数精简后大致是这样的，但是我没有找到`__default_pthread_attr`设置`cpuset`的地方，`pthread_create`会接着调用`create_thread`。
```c
static int create_thread (struct pthread *pd, const struct pthread_attr *attr,
               bool stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
{
  /* We rely heavily on various flags the CLONE function understands:
     CLONE_VM, CLONE_FS, CLONE_FILES
        These flags select semantics with shared address space and
        file descriptors according to what POSIX requires.
     CLONE_SIGHAND, CLONE_THREAD
        This flag selects the POSIX signal semantics and various
        other kinds of sharing (itimers, POSIX timers, etc.).
     CLONE_SETTLS
        The sixth parameter to CLONE determines the TLS area for the
        new thread.
     CLONE_PARENT_SETTID
        The kernels writes the thread ID of the newly created thread
        into the location pointed to by the fifth parameters to CLONE.
        Note that it would be semantically equivalent to use
        CLONE_CHILD_SETTID but it is be more expensive in the kernel.
     CLONE_CHILD_CLEARTID
        The kernels clears the thread ID of a thread that has called
        sys_exit() in the location pointed to by the seventh parameter
        to CLONE.
     The termination signal is chosen to be zero which means no signal
     is sent.  */
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
                           | CLONE_SIGHAND | CLONE_THREAD
                           | CLONE_SETTLS | CLONE_PARENT_SETTID
                           | CLONE_CHILD_CLEARTID
                           | 0);
  ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
                                    clone_flags, pd, &pd->tid, tp, &pd->tid);
  *thread_ran = true;
  if (attr != NULL)
    {
      INTERNAL_SYSCALL_DECL (err);
      if (attr->cpuset != NULL)
        {
          res = INTERNAL_SYSCALL (sched_setaffinity, err, 3, pd->tid,
                                  attr->cpusetsize, attr->cpuset);
        }
      if ((attr->flags & ATTR_FLAG_NOTINHERITSCHED) != 0)
        {
          res = INTERNAL_SYSCALL (sched_setscheduler, err, 3, pd->tid,
                                  pd->schedpolicy, &pd->schedparam);
        }
    }
  return 0;
}
```
&emsp;&emsp;这里需要注意的是上面的一堆`flags`和它们的说明，可以发现，对于所有\*nix系统来说，线程就是轻量级的进程，线程无非就是共享了除栈外几乎所有进程所有的资源，所以线程的切换消耗无非就是保存寄存器的值和线程自己的栈数据。下次如果有人问你线程和进程的区别，一定要回答的深入一点，这样就可以吊打面试官啦！
```c
#define ARCH_FORK() \
  INLINE_SYSCALL (clone2, 6,                              \
          CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD,        \
          NULL, 0, NULL, &THREAD_SELF->tid, NULL)
#define ARCH_CLONE __clone2
```
```asm
/* int  __clone2(int (*fn) (void *arg), void *child_stack_base,         */
/*               size_t child_stack_size, int flags, void *arg,         */
/*               pid_t *parent_tid, void *tls, pid_t *child_tid)        */

#define CHILD   p8
#define PARENT  p9

ENTRY(__clone2)
        .prologue
        alloc r2=ar.pfs,8,1,6,0
        cmp.eq p6,p0=0,in0
        cmp.eq p7,p0=0,in1
        mov r8=EINVAL
        mov out0=in3            /* Flags are first syscall argument.    */
        mov out1=in1            /* Stack address.                       */
(p6)    br.cond.spnt.many __syscall_error       /* no NULL function pointers */
(p7)    br.cond.spnt.many __syscall_error       /* no NULL stack pointers */
        ;;
        mov out2=in2            /* Stack size.                          */
        mov out3=in5            /* Parent TID Pointer                   */
        mov out4=in7            /* Child TID Pointer                    */
        mov out5=in6            /* TLS pointer                          */
        /*
         * clone2() is special: the child cannot execute br.ret right
         * after the system call returns, because it starts out
         * executing on an empty stack.  Because of this, we can't use
         * the new (lightweight) syscall convention here.  Instead, we
         * just fall back on always using "break".
         *
         * Furthermore, since the child starts with an empty stack, we
         * need to avoid unwinding past invalid memory.  To that end,
         * we'll pretend now that __clone2() is the end of the
         * call-chain.  This is wrong for the parent, but only until
         * it returns from clone2() but it's better than the
         * alternative.
         */
        mov r15=SYS_ify (clone2)
        .save rp, r0
        break __BREAK_SYSCALL
        .body
        cmp.eq p6,p0=-1,r10
        cmp.eq CHILD,PARENT=0,r8 /* Are we the child?   */
(p6)    br.cond.spnt.many __syscall_error
        ;;
(CHILD) mov loc0=gp
(PARENT) ret
        ;;
        tbit.nz p6,p0=in3,8     /* CLONE_VM */
(p6)    br.cond.dptk 1f
        ;;
        mov r15=SYS_ify (getpid)
(p7)    break __BREAK_SYSCALL
        ;;
        add r9=PID,r13
        add r10=TID,r13
        ;;
        st4 [r9]=r8
        st4 [r10]=r8
        ;;
1:      ld8 out1=[in0],8        /* Retrieve code pointer.       */
        mov out0=in4            /* Pass proper argument to fn */
        ;;
        ld8 gp=[in0]            /* Load function gp.            */
        mov b6=out1
        br.call.dptk.many rp=b6 /* Call fn(arg) in the child    */
        ;;
        mov out0=r8             /* Argument to _exit            */
        mov gp=loc0
        .globl HIDDEN_JUMPTARGET(_exit)
        br.call.dpnt.many rp=HIDDEN_JUMPTARGET(_exit)
                                /* call _exit with result from fn.      */
        ret                     /* Not reached.         */
PSEUDO_END(__clone2)
```
&emsp;&emsp;虽然我看不懂C里面的asm但是它调用了Linux的`clone2`，但是我没找到`clone2`的源码，反正肯定会调用`sys_clone`。`sys_clone`会创建一个新的进程，与`fork`不同的是，`sys_clone`容许新的进程共享父进程的内存，各种不同的段以及其他的东西，得到的pid就是新建的线程的thread id。继续回到之前，线程创建之后会根据`thread_attr`接着调用`sched_setaffinity`。`sched_setaffinity`的作用就是将CPU Core和Thread绑定，让thread只在指定的CPU上执行，这样做可以提升Performance。上文提到过，不同的CPU Core是不共享L1i和L1d的，这样线程去另一个CPU Core执行的话，会导致Cache Invalidation，这样就会导致内存读取，还是一个很大的损耗的，所以对于CPU Bound的程序来说，设置CPU affinity可以提升Performance。接着会调用`sched_setscheduler`，设置线程调度的优先级和调度策略。

### 结语
&emsp;&emsp;本课内容看似简短，其实扩展开来都是很复杂很深的课题。扩展的活儿就交给你自己啦，任何问题都可以撩博主，与博主交流！**Good Luck!Have Fun!**
