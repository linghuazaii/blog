[Parallel Programming]Active Object Design Pattern
===================================================

### 前言
&emsp;&emsp;试着思考这样一个问题，假设我们有一个文字处理应用，点击保存按钮的时候，不要阻塞在保存这里，而是将保存按钮灰色显示，等到保存功能完成后，自动将保存按钮恢复正常显示。在单线程版本下，必须得等保存完成，这对任何GUI来说都是致命的。或许你会说，这期起一个后台线程去做保存功能，可以是可以，但并不是一个好的设计。而Active Object Design Pattern或者说Concurrent Object Design Pattern就是用来做异步的设计，常用于GUI，Web Server的设计。本文要实现一个简易的Producer&Consumer的消息系统，当然并不是进程之间的，是进程内多线程之间的。

### Active Object Design Pattern
&emsp;&emsp;Active Object Design Pattern的核心思想就是函数调用和函数实现相分离，这样抽象出两个角色，一个是`Proxy`，另一个是`Servant`。`Proxy`负责所有Client的函数调用，`Servant`负责所有`Proxy`函数调用的具体实现。函数调用到函数实现是在不同的线程里完成，那么怎么实现一个同步呢？这里需要一个`ActiveObjectQueue`，这个队列里装的是所有从`Proxy`生成的Active Object。动态维护这个队列需要抽象出一个`Scheduler`，`Scheduler`负责将`Proxy`的Request加入到`ActiveObjectQueue`队列，并且一直循环分发这些Request到具体的`Servant`实现。在本例子中，我们将ActiveObject抽象为一个抽象类`MethodRequest`。而不同线程之间的消息传递抽象为`MessageFuture`，将所有的消息抽象为`Message`。以上文字可能并不能在你的脑海里勾画出一个完成的架构图，下面我们画一画架构图。

### 架构图
&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/active_object/active_object_design.png" />  
&emsp;&emsp;如图所示，只有Proxy对于Client是可见的，其他部分对于Client都是不可见的。可能这个图还是不能在你的脑海里勾勒出这个设计的所有细节，确实，我在学这个设计模式的时候，也懵逼了好久好久，知道我读到《[The Art Of MultiProcessor Programming](http://coolfire.insomnia247.nl/c&mt/Herlihy,%20Shavit%20-%20The%20art%20of%20multiprocessor%20programming.pdf)》里面的Concurrent Object的时候我才豁然开朗，实现了本文的例子。下面我们各个击破，将每个部分分解出来。

### Proxy
&emsp;&emsp;只有`Proxy`对于Client是可见的，以下是`Proxy`的实现。
```cpp
#ifndef _PROXY_H
#define _PROXY_H
/*
 * File: proxy.h
 * Author: Charles, Liu.
 * Mailto: charlesliu.cn.bj@gmail.com
 */
#include "servant.h"
#include "method_request.h"
#include "message.h"
#include "message_future.h"
#include "scheduler.h"
#include "producer.h"
#include "consumer.h"

class Proxy {
public:
    enum { MAX_SIZE = 1000 };
    Proxy(size_t size = MAX_SIZE) {
        scheduler_ = new Scheduler(size);
        servant_ = new Servant(size);
        scheduler_->run();
    }
    ~Proxy() {
        delete scheduler_;
        delete servant_;
    }
    void produce(Message &msg) {
        MethodRequest *method_request = new Producer(servant_, msg);
        scheduler_->enqueue(method_request);
    }
    MessageFuture * consume() {
        MessageFuture *result = new MessageFuture;
        MethodRequest *method_request = new Consumer(servant_, result);
        scheduler_->enqueue(method_request);
        return result;
    }
private:
    Servant *servant_;
    Scheduler *scheduler_;
};


#endif
```
&emsp;&emsp;本例实现的是一个Producer&Consumer消息系统，所以Client只需要调用`produce()`和`consumer()`，所以Proxy只有`produce()`和`consume()`两个方法。`Proxy`有一个`Servant`实例，正如上文所说，`Proxy`负责函数调用，`Servant`负责函数实现。`Proxy`还有一个`Scheduler`实例，`Scheduler`负责将所有的Active Object加到`ActiveObjectQueue`队列。`produce()`调用生成一个叫`Producer`的Active Object，`consume()`生成一个`Consumer`的Active Object,并且返回一个`MessageFuture`实例，这个`MessageFuture`会在`Servant`调用其`consume()`方法之后填充`Message`。

### Message
&emsp;&emsp;本例中的`Message`就是`std::string`的一个type definition。
```cpp
#ifndef _MESSAGE_H
#define _MESSAGE_H
/*
 * File: message.h
 * Description: type definition for Message
 * Autor: Charles, Liu.
 * Mailto: charlesliu.cn.bj@gmail.com
 */
#include <string>

typedef std::string Message;

#endif
```

### MessageFuture
&emsp;&emsp;`MessageFuture`包含`consume()`填充的`Message`，并且包含一个`hasMessage()`方法用来判断`Message`是否填充，这样Consumer可以通过这个标志来轮询所有的`MessageFuture`，这样可以避免消息的丢失。
```cpp
#ifndef _MESSAGE_FUTURE_H
#define _MESSAGE_FUTURE_H
/*
 * File: message_future.h
 * Description: type definition for class MessageFuture
 * Autor: Charles, Liu.
 * Mailto: charlesliu.cn.bj@gmail.com
 */
#include "message.h"

class MessageFuture {
public:
    MessageFuture() : has_message(false) {}
    ~MessageFuture() {}
    bool hasMessage() {
        return has_message;
    }
    Message getMessage() {
        return msg_;
    }
    void setMessage(Message &msg) {
        msg_ = msg;
        has_message = true;
    }
private:
    Message msg_;
    bool has_message;
};

#endif
```




