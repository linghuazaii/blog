[Parallel Programming]深入PThread (Lesson II)
=============================================

### 回顾
&emsp;&emsp;上一节课我们讲了`pthread_create()`，这节课讲`mutex`和`condition variable`，东西有点多，我还以为我讲过`mutex`了，我有讲过吗？

### 前言
&emsp;&emsp;`Pthreads`所有的东西的实现都是基于`Futex`，但是不打算讲`Futex`，我没看，感兴趣可以去这挖一挖。  
&emsp;&emsp;[Futex Wiki](https://en.wikipedia.org/wiki/Futex)

### pthread_mutex_lock & pthread_mutex_unlock
&emsp;&emsp;其实Linux内核包括`Futex`，所有锁的实现都是spinlock，即test&set。关于test&set可以参考[Wiki](https://en.wikipedia.org/wiki/Test-and-set),
