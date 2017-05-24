[Parallel Programming]深入PThread (Lesson II)
=============================================

### 回顾
&emsp;&emsp;上一节课我们讲了`pthread_create()`，这节课讲`mutex`和`condition variable`，东西有点多，我还以为我讲过`mutex`了，我有讲过吗？

### 前言
&emsp;&emsp;`Pthreads`所有的东西的实现都是基于`Futex`，但是不打算讲`Futex`，我没看，感兴趣可以去这挖一挖。  
&emsp;&emsp;[Futex Wiki](https://en.wikipedia.org/wiki/Futex)

### pthread_mutex_lock & pthread_mutex_unlock
&emsp;&emsp;其实Linux内核包括`Futex`，所有锁的实现都是spinlock，即test&set。关于test&set可以参考[Wiki](https://en.wikipedia.org/wiki/Test-and-set)。锁的本质其实就是原子操作，而原子操作需要hardware级别的支持，好在`xchg`系列交换值的指令是存在的，而且是原子性的，上层的调用类似`atomic_compare_and_exchange_val_acq`到汇编这一层就是这些原子指令。关于CAS（compare-and-swap）锁的例子，我自己写了一个，[看这里](https://github.com/linghuazaii/parallel_programming/tree/master/test_and_set_lock)。那么我们`pthread_mutex_lock`和`pthread_mutex_unlock`我们就讲完啦。我们不讲Futex，所以也不讲`pthread_mutex_t`之类的结构，只谈本质的话，锁就是这么简单。

### pthread_cond_wait
&emsp;&emsp;同样也不讲`pthread_cond_t`之类的结构，理由同上，但是还是有些细节要讲一讲。
```c
__pthread_cond_wait (pthread_cond_t *cond, pthread_mutex_t *mutex)
{
    err = __pthread_mutex_unlock_usercnt (mutex, 0);
    cond->__data.__nwaiters += 1 << COND_NWAITERS_SHIFT;
    do {
        lll_futex_wait (&cond->__data.__futex, futex_val, pshared);
    } while (val == seq || cond->__data.__woken_seq == val);
    ++cond->__data.__woken_seq;
    return __pthread_mutex_cond_lock (mutex);
}
```
&emsp;&emsp;代码简化完后大致干了这几件事，先解锁`mutex`，不然发送signal的线程没法获得这把锁，然后调用`lll_futex_wait`一直等待自己被唤醒，完了将`mutex`锁住。有人或许要问：为什么`condition variable`需要配一把锁呢？简而言之，设计使然，而且这个设计也是一石二鸟啊。首先等待的`condition`肯定是共享的资源，而且条件变量自身也是共享的资源，一把锁解决了两个共享资源的问题，棒不棒？！如果没有锁会怎么样？看看常见的条件变量用法：
```c
Thread 1                                      Thread2
-----------------------------                 -------------------------------------
//pthread_mutex_lock(&mutex);                 //pthread_mutex_lock(&mutex);
while(wait condition become true)             change conditon to true;
    pthread_cond_wait(&cond, &mutex);         pthread_cond_signal(&cond);
do things;                                    //pthread_mutex_unlock(&mutex);
//pthread_mutex_unlock(&mutex);
```
&emsp;&emsp;我们把锁注掉看会发生什么，假设`Thread1`运行在`CPU0`，`Thread2`运行在`CPU1`，首先考虑一下指令重排的情况：如果`condition = true`和`pthread_cond_signal`重排了，信号发了，在`pthread_cond_wait`之前，那么这个信号丢失了，`Thread1`永远Block在`pthread_cond_wait`。再考虑指令不重排的情况：由于是不同的CPU，所以彼此的cacheline并不一定可见，所以`thread2`运行`condition = true`然后发送`pthread_cond_signal`，对于`thread1`，`condition = true`暂时不可见，就会进入`while`，信号丢失，`pthread_cond_wait`永远Block。所以呢，加这把锁就很好的防止了信号丢失的问题。聪明的你可能会继续问：那为什么要`while`循环呢？这是为了保证系统的健壮性，对于内核来说，并不是什么都是确定的，可能由于某种意外情况，就好比意外怀孕一样，这个信号在条件不满足的时候也发出去了呢？这种Wakeup并不违反标准，所以Pthreads这么设计也是为了迎合标准，感兴趣可以Google下**Spurious Wake Up**。

### pthread_cond_signal & pthread_cond_broadcast
&emsp;&emsp;线程唤醒的原则遵循scheduler的标准，即wakeup的时候，谁的priority大，就唤醒谁，像`SCHED_OTHER`这种默认的`min/max priority`都为`0`的时间片抢占的情况来说，first-in-first-out，很公平。如果你要设计`priority`的话，类似于所有的实时程序设计，使用`SCHED_RR`或者`SCHED_FIFO`。  
&emsp;&emsp;`pthread_cond_signal`简化一下就一行代码`lll_futex_wake (&cond->__data.__futex, 1, pshared)`；`pthread_cond_broadcast`简化一下也是一行代码`lll_futex_wake (&cond->__data.__futex, INT_MAX, pshared)`。简而言之，`pthread_cond_signal`只唤醒一个线程，根据scheduler策略来，`pthread_cond_broadcast`唤醒所有。

### 小结
&emsp;&emsp;本课内容好像并不多也，**Good Luck! Have Fun!**











