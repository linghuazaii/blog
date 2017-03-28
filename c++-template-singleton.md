Simple C++ Template Singleton Class 
===================================

## 起源
&emsp;&emsp;原始纯C时代，要引用一个全局的变量，大家习惯的用法是用`extern`，但是当这种全局变量太多了，写起代码来就得四处去找这种全局变量在哪儿定义的，对于C来说，`struct`里的所有成员都是默认公有属性，所以没有singleton这个概念。转而到Dual C/C++时代，针对`extern`的解决方案就是使用singleton，但是每个类都实现为Singleton未免太过麻烦，而且，有时候需要用Singleton，有时候不需要用Singleton的时候，这么写局限性太大。

## Note
&emsp;&emsp;为什么是Simple，因为本例子是非线程安全的，而且真正做到singleton的线程安全还有性能问题远不止加锁那么简单的事，还得考虑到编译器优化的问题，具体细节可以参考DCLP(Double-Check Locking Pattern)[here](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf).

## 源码
[class_templates.h](https://github.com/linghuazaii/C--Templates/blob/master/classes/class_templates.h)
```c++
#ifndef _CLASS_TEMPLATES_H
#define _CLASS_TEMPLATES_H
/*
 * File: class_templates.h
 * Author: Charles, Liu.
 */

template <typename T>
class Singleton {
public:
    static T *instance();
    static void destroy();
private:
    Singleton();
    ~Singleton();
private:
    static T *_instance;
};

template <typename T>
T *Singleton<T>::_instance = NULL;

template <typename T>
T *Singleton<T>::instance() {
    if (_instance == NULL) {
        _instance = new T;
    }

    return _instance;
}

template <typename T>
void Singleton<T>::destroy() {
    if (_instance != NULL)
        delete _instance;
}

template <typename T>
Singleton<T>::Singleton() {
}

template <typename T>
Singleton<T>::~Singleton() {
}

#endif
```
[myclass.h](https://github.com/linghuazaii/C--Templates/blob/master/classes/myclass.h)
```c++
#ifndef _MY_CLASS_H
#define _MY_CLASS_H
/*
 * File: myclass.h
 * Author: Charles, Liu.
 */
#include <iostream>
using namespace std;

class MyClass {
public:
    MyClass(){}
    ~MyClass(){}
    const char *print() {
        return "calling MyClass::print";
    }
};

#endif
```
[main.cpp](https://github.com/linghuazaii/C--Templates/blob/master/classes/main.cpp)
```c++
#include <iostream>
#include "class_templates.h"
#include "myclass.h"
using namespace std;
/*
 * For TESTING, have fun!!!
 */

const char *test_info = NULL;

#define INFO(msg) do { \
        test_info = msg; \
        cout << "============== " << test_info << " ==============" << endl; \
    } while (0)
#define END() cout << "============== "<< "END" << " ==============" << endl
#define TEST2(function, a, b) do {\
        INFO("TEST "#function);\
        cout << #function << "(" << a << "," << b << ")" <<endl; \
        cout << "result: " << function(a, b) <<endl; \
        END();\
    } while (0)
#define TEST(function) do {\
        INFO("TEST "#function);\
        cout << #function << "( )" <<endl; \
        cout << "result: " << function() <<endl; \
        END();\
    } while (0)


int main(int argc, char **argv) {
    {
        MyClass *single_test = Singleton<MyClass>::instance();
        TEST(single_test->print);

        MyClass *addr1 = Singleton<MyClass>::instance();
        MyClass *addr2 = Singleton<MyClass>::instance();
        cout<<"addr1: "<<static_cast<void *>(addr1)<<endl;
        cout<<"addr2: "<<static_cast<void *>(addr2)<<endl;
    }

    return 0;
}
```

## 结语
&emsp;&emsp;Have Fun!!!!
