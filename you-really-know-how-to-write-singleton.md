你真的知道怎么写Singleton吗？
============================

### 常见的Singleton写法
```c
/* singleton.h */
class Singleton {
public:
    static Singleton *getInstance();
private:
    Singleton() {}
    ~Singleton() {}
private:
    static Singleton *singleton;
};

/* singleton.cpp */
Singleton *Singleton::singleton = NULL;
Singleton *Singleton::getInstance() {
    if (singleton == NULL)
        singleton = new Singleton();
    return singleton;
}
```

### 大伙都知道的DCLP问题
&emsp;&emsp;以上的例子放在多线程环境下，可能会多个线程同时执行`singleton = new Singleton()`，造成那么一丢丢的内存泄漏。好的，我们加把锁：
```c
/* singleton.h */
class Singleton {
public:
    static Singleton *getInstance();
private:
    Singleton() {}
    ~Singleton() {}
private:
    static Singleton *singleton;
    static pthread_mutex_t mutex;
};

/* singleton.cpp */
Singleton *Singleton::singleton = NULL;
pthread_mutex_t Singleton::mutex = PTHREAD_MUTEX_INITIALIZER;
Singleton *Singleton::getInstance() {
    if (singleton == NULL) {
        pthread_mutex_lock(&mutex);
        singleton = new Singleton();
        pthread_mutex_unlock(&mutex);
    }
    return singleton;
}
```

&emsp;&emsp;这样同样会有一个问题，可能多个线程同时等待`mutex`，然后`singleton = new Singleton()`同样会被多次调用，造成那么一丢丢的内存泄漏。好的，我们在锁里再加一层判断：
```c
/* singleton.cpp */
Singleton *Singleton::singleton = NULL;
pthread_mutex_t Singleton::mutex = PTHREAD_MUTEX_INITIALIZER;
Singleton *Singleton::getInstance() {
    if (singleton == NULL) {
        pthread_mutex_lock(&mutex);
        if (singleton == NULL)
            singleton = new Singleton();
        pthread_mutex_unlock(&mutex);
    }
    return singleton;
}
```

&emsp;&emsp;干得不错，但是线程的多核CPU架构，多个CPU的cache line存在一个可见性问题，再加上`new`操作符可能被指令重排，所以运行在不同的CPU core上的多个线程仍然可能同时调用`singleton = new Singleton()`，造成那么一丢丢的内存泄漏。好的，我们用上memory model：
```c
/* singleton.cpp */
Singleton *Singleton::singleton = NULL;
pthread_mutex_t Singleton::mutex = PTHREAD_MUTEX_INITIALIZER;
Singleton *Singleton::getInstance() {
    Singleton *temp = singleton.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);
    if (temp == NULL) {
        pthread_mutex_lock(&mutex);
        temp = singleton.load(std::memory_order_relaxed);
        if (temp == NULL) {
            temp = new Singleton();
            std::atomic_thread_fence(std::memory_order_release);
            singleton.store(temp, std::memory_order_relaxed);
        }
        pthread_mutex_unlock(&mutex);
    }
    return temp;
}
```

&emsp;&emsp;以上便是DCLP(Double-Checked-Locking-Pattern)的最终版，你会发现写一个Singleton又是memory fence，又是atomic操作，简直费劲，下面我们来另一种写法。

### 懒人Singleton写法
```c
/* singleton.h */
class Singleton {
public:
    static Singleton *getInstance() {
        return &singleton;
    }
private:
    Singleton() {}
    ~Singleton() {}
private:
    static Singleton singleton;
};

/* singleton.cpp */
Singleton Singleton::singleton;
```

&emsp;&emsp;很优雅的规避了DCLP问题，并且多线程安全，为什么多线程安全呢，以下是cpp-reference上的一段解释:
> Non-local variables
> 
> All non-local variables with static storage duration are initialized as part of program startup, before the execution of the main function begins (unless deferred, see below). All variables with thread-local storage duration are initialized as part of thread launch, sequenced-before the execution of the thread function begins.

&emsp;&emsp;类里面的static成员是全局的，会在编译的时候存在.bss段，在程序启动也就是在进入到`_start`后`call main`之前，进行初始化。但是呢，这种写法会存在一些隐患。

### static initialization order fiasco
&emsp;&emsp;将设我有另一个同样的Singleton类`SingletonB`：
```c
class SingletonB {
private:
    SingletonB() {
        Singleton *singleton = Singleton::getInstance();
        singleton->do_whatever();
    }
}
```

&emsp;&emsp;假定`SingletonB`先于`Singleton`编译，由于`SingletonB`初始化的时候`Singleton`没有初始化，所以程序在`call main`之前就会crash。

### 怎么解决
&emsp;&emsp;解决方案很简单，将`static Singleton singleton;`由global scope转为function scope。
```c
/* singleton.h */
class Singleton {
public:
    static Singleton *getInstance() {
        static Singleton singleton;
        return &singleton;
    }
private:
    Singleton() {}
    ~Singleton() {}
};
/* 没有singleton.cpp */
```

&emsp;&emsp;这样只有在第一次调用`Singlton::getInstance()`才会初始化`singleton`。而且，该解决方案在C++11的前提下是线程安全的，以下是cpp-reference的解释：
> If multiple threads attempt to initialize the same static local variable concurrently, the initialization occurs exactly once (similar behavior can be obtained for arbitrary functions with std::call_once).
> 
> Note: usual implementations of this feature use variants of the double-checked locking pattern, which reduces runtime overhead for already-initialized local statics to a single non-atomic boolean comparison.	(since C++11)

&emsp;&emsp;[这个是我在StackOverflow上抛出的一个问题](https://stackoverflow.com/questions/44838641/what-bugs-will-my-singleton-class-cause-if-i-write-it-like-this)

### Reference
 - [Double Checked Locking Pattern](https://en.wikipedia.org/wiki/Double-checked_locking)
 - [c++ memory model](http://en.cppreference.com/w/cpp/language/memory_model)
 - [memory fence](https://en.wikipedia.org/wiki/Memory_barrier)
 - [what every programmer should know about memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
 - [c++ initialization](http://en.cppreference.com/w/cpp/language/initialization)
 - [static initialization fiasco](https://isocpp.org/wiki/faq/ctors#static-init-order)

### 小结
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
