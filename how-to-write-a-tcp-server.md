怎么从无到有写一个好的TCP Server
================================

### 前言
&emsp;&emsp;很久以前在看Libevent的源码，然后研究怎么写一个完整的高性能的TCP Server，之后由于兴之所至研究Collective Intelligence然后搁置了很长一段时间，这两天就写了一个简易的，单机QPS绝对不会低，感兴趣希望帮忙测一下。

### TCP参数以及EPOLLET
&emsp;&emsp;为什么用Nonblocking和EPOLLET？原因之一是减少`epoll_wait`调用次数，这样也就减少了系统调用的次数，这样socket上的事件只会通知一次。对于listen socket，需要一直`accept`直到`errno`变为`EAGAIN`；对于连接socket，也需要一直`read`直到`errno`变为`EAGAIN`，这样就避免了数据的丢失。还有一点就是，可以多个thread同时`epoll_wait`同一个`epoll fd`，由于是ET模式，所有事件只会通知一次，所以不会引起spurious wakeup，这样可以利用multi-core来提升整体性能，本例中并没有实现这一点。  
&emsp;&emsp;通过启用`TCP_NODELAY`来禁用Nagel算法，TCP默认是启用了Nagel算法的，对于需要快速响应的Server来说，禁用Nagel总是好的。还有一点就是TCP默认有一个ACK捎带的模式，即ACK并不是立即返回给另一方，可能会随着数据的返回而捎带过去，这样和Nagel一起使用会造成更长的延时，所以禁用Nagel就好。启用`TCP_DEFER_ACCEPT`，启用此选项的考虑点非常简单，当连接建立的时候，我们并不以为连接已经建立，而是当收到数据的时候才认为连接已经建立，这样对于不发数据但是建立了连接的client，不会加入到epoll监听，节省系统资源。同时，调用`accept`后，新加入监听的连接也会立刻产生EPOLLIN事件，因为buffer里已经有数据可读。

### 关于协议设计
&emsp;&emsp;起初我是打算将buffer这一层单独抽离出来做一层，基于此的考虑是TCP只是提供一个连续不断的流，而且TCP是全双工的。无论是业务还是Write都不可以打断我的Read。这样上层只需要做一个thread不断去buffer这一层切割完整的packet，做业务处理和Write就行，这样设计比较自然，但是越想越复杂就换了比较通用的`length|data`的设计。`length|data`的设计需要client遵守这个约定，要么将`length|data`封装成结构体，一次性Write，要么将使用`TCP_CORK`一次发送整个物理包，要么使用scatter/gather IO，即`readv/writev`。

### 线程池设计
&emsp;&emsp;如果将Server的行为拆分开来，大致可以分为5个：新建连接 => 读 => 请求处理 => 写，其间可以穿插错误处理。为了不让某个行为阻断另一个行为，举个例子，如果我建立了大小为8的线程池同时处理这五种行为，我请求处理的可能很慢，这样所有任务都会Hang死在请求处理这，所以为了避免处理速度不一致的情况将其拆分成五个线程池，这样也可以根据各个行为的耗时情况调整线程池的大小。

### 代码片段
五个线程池
```c
read_threadpool = threadpool_create(8, MAX_EVENTS, 0);
write_threadpool = threadpool_create(8, MAX_EVENTS, 0);
listener_threadpool = threadpool_create(1, MAX_EVENTS, 0);
error_threadpool = threadpool_create(2, MAX_EVENTS, 0);
worker_threadpool = threadpool_create(8, MAX_EVENTS * 2, 0);
```
  
对socket的简单封装，listen socket设置成NONBLOCK符合ET的设计。
```c
int ss_socket() {
    int sock = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    if (sock == -1)
        abort("create socket error");

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = INADDR_ANY;

    set_reuseaddr(sock);
    set_tcp_defer_accept(sock);

    int ret = bind(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
    if (ret == -1)
        abort("bind socket error");

    ret = listen(sock, BACKLOG);
    if (ret == -1)
        abort("listen socket error");

    return sock;
}
```
  
事件循环，这里踩过这么一个坑，`epoll_data`是一个union的结构，我还以为是一个结构体，然后`void *ptr`是一个存private data的指针，在这里堵了好久。我们不要使用`epoll_data.fd`，反而使用`epoll_data.ptr`，这样更灵活，而且能确保每一个连接共享一个我自定义的`ep_data_t`的结构。但是你会发现代码里并没有加锁，其实考虑的初衷是这样的：对于单一连接，正常的通信是一问一答的模式，如果偏离这种模式的话则需要对`ep_data_t`里的buffer加锁。
```c
void event_loop() {
    int listener = ss_socket();
    int epfd = ss_epoll_create();
    log("epollfd: %d\tlistenfd: %d", epfd, listener);
    ep_data_t *listener_data = (ep_data_t *)ss_malloc(sizeof(ep_data_t));
    listener_data->epfd = epfd;
    listener_data->eventfd = listener;
    struct epoll_event ev_listener;
    ev_listener.data.ptr = listener_data;
    ev_listener.events = EPOLLIN | EPOLLET;
    ss_epoll_ctl(epfd, EPOLL_CTL_ADD, listener, &ev_listener);
    epoll_ctl(epfd, EPOLL_CTL_ADD, listener, &ev_listener);
    struct epoll_event events[MAX_EVENTS];
    for (;;) {
        int nfds = ss_epoll_wait(epfd, events, MAX_EVENTS);
        for (int i = 0; i < nfds; ++i) {
            log("current eventfd: %d", ((ep_data_t *)events[i].data.ptr)->eventfd);
            if (((ep_data_t *)events[i].data.ptr)->eventfd == listener) {
                log("listenfd event.");
                threadpool_add(listener_threadpool, do_accept, events[i].data.ptr, 0);
            } else {
                if (events[i].events & EPOLLIN) {
                    log("read event.");
                    threadpool_add(read_threadpool, do_read, events[i].data.ptr, 0);
                }
                if (events[i].events & EPOLLOUT) {
                    log("write event.");
                    threadpool_add(write_threadpool, do_write, events[i].data.ptr, 0);
                }
                if (events[i].events & EPOLLERR | events[i].events & EPOLLHUP) {
                    log("close event.");
                    threadpool_add(error_threadpool, do_close, events[i].data.ptr, 0);
                }
            }
        }
    }
}
```
  
新建连接处理，每次有连接建立就循环`accpet`直到所有连接处理完，符合ET设计。这里事件并不设置EPOLLOUT，还有就是packet的设计，由于TCP只提供连续的流，所以我们收到的packet可能只是一个chunck。再就是这里的`EPOLLRDHUP`，很便于ET模式处理关闭的socket，`read`反0和这个只能二选一。
```c
void do_accept(void *arg) {
    ep_data_t *data = (ep_data_t *)arg;
    struct sockaddr_in client;
    socklen_t addrlen = sizeof(client);
    while (true) {
        int conn = accept4(data->eventfd, (struct sockaddr *)&client, &addrlen, SOCK_NONBLOCK);
        if (conn == -1) {
            if (errno == EAGAIN)
                break;
            abort("accpet4 error");
        }
        char ip[IPLEN];
        inet_ntop(AF_INET, &client.sin_addr, ip, IPLEN);
        log("new connection from %s, fd: %d", ip, conn);

        struct epoll_event event;
        //event.events = EPOLLIN | EPOLLOUT | EPOLLRDHUP; //shouldn't include EPOLLOUT '
        event.events = EPOLLIN | EPOLLET;
        ep_data_t *ep_data = (ep_data_t *)ss_malloc(sizeof(ep_data_t));
        ep_data->epfd = data->epfd;
        ep_data->eventfd = conn;
        ep_data->packet_state = PACKET_START;
        ep_data->read_callback = handle_request; /* callback for request */
        ep_data->write_callback = handle_response; /* callback for response */
        event.data.ptr = ep_data;
        ss_epoll_ctl(data->epfd, EPOLL_CTL_ADD, conn, &event);
        log("add event for fd: %d", conn);
    }
}
```
  
Read处理，一直读直到`EAGAIN`，每读到一个完整的packet就送去request线程池，避免其阻塞Read。没读完就标记为CHUNCK，等待下一次读取。当read返回0的时候说明对方已经关闭了连接。
```c
void do_read(void *arg) {
    log("enter do_read");
    ep_data_t *data = (ep_data_t *)arg;
    while (true) {
        if (data->packet_state == PACKET_START) {
            uint32_t length;
            int ret = read(data->eventfd, &length, sizeof(length));
            if (ret == -1) {
                if (errno == EAGAIN) {
                    break; // no more data to read
                }
                err("read error for %d", data->eventfd);
                return;
            }
            log("packet length: %d", length);
            if (length == 0) {/* socket need to be closed */
                do_close(data);
                break;
            }
            data->ep_read_buffer.length = length;
            data->ep_read_buffer.buffer = (char *)ss_malloc(length);
            data->ep_read_buffer.count = 0;
        }
        int count = read(data->eventfd, data->ep_read_buffer.buffer + data->ep_read_buffer.count
, data->ep_read_buffer.length - data->ep_read_buffer.count);
        if (count == 0) {
            do_close(data); /* socket need to be closed */
            break;
        }
        if (count == -1) {
            if (errno == EAGAIN) {
                break; /* no more data to read */
            }
            err("read error for %d", data->eventfd);
            return;
        }
        data->ep_read_buffer.count += count;
        if (data->ep_read_buffer.count < data->ep_read_buffer.length) {
            data->packet_state = PACKET_CHUNCK;
        } else {
            data->packet_state = PACKET_START; /* reset to PACKET_START to recv new packet */
            //data->read_callback(data); /* should not block read thread */
            request_t *req = (request_t *)ss_malloc(sizeof(request_t));
            req->ep_data = data;
            req->length = data->ep_read_buffer.length;
            req->buffer = (char *)ss_malloc(data->ep_read_buffer.length);
            memcpy(req->buffer, data->ep_read_buffer.buffer, req->length);
            threadpool_add(worker_threadpool, data->read_callback, (void *)req, 0);
        }
    }
}
```
  
请求处理，这里的处理很简单，client说什么，然后在其内容前加一个 "Hi there, you just said: "。
```c
void handle_request(void *data) {
    request_t *req = (request_t *)data;
    char *resp = (char *)ss_malloc(req->length + 1024);
    const char *msg = "Hi there, you just said: ";
    memcpy(resp, msg, strlen(msg));
    memcpy(resp + strlen(msg), req->buffer, req->length);
    response(req, resp, strlen(resp));
    ss_free(resp);
}
```
  
response函数，只是简单的做write buffer的处理，加上`EPOLLOUT`事件，告诉其可以写了，因为TCP是全双工的，所以要一直是监控可读状态。
```c
void response(request_t *req, char *resp, uint32_t length) {
    ep_data_t *data = req->ep_data;
    data->ep_write_buffer.buffer = (char *)ss_malloc(length);
    memcpy(data->ep_write_buffer.buffer, resp, length);
    data->ep_write_buffer.length = length;
    struct epoll_event event;
    event.data.fd = data->eventfd;
    event.data.ptr = data;
    event.events = EPOLLIN | EPOLLOUT | EPOLLET | EPOLLRDHUP; /* we should always allow read */
    ss_epoll_ctl(data->epfd, EPOLL_CTL_MOD, data->eventfd, &event);
    ss_free(req->buffer);
    ss_free(req);
}
```
  
Write处理，write同样要一直写到`EAGAIN`为止，没写完的的接着写，写完了重置write buffer。
```c
void do_write(void *arg) {
    ep_data_t *data = (ep_data_t *)arg;
    while (true) {
        int count = write(data->eventfd, data->ep_write_buffer.buffer + data->ep_write_buffer.co
unt, data->ep_write_buffer.length - data->ep_write_buffer.count);
        if (count == -1) {
            if (errno == EAGAIN) {
                break; /* wait for next write */
            }
            err("write error for %d", data->eventfd);
            return;
        }
        data->ep_write_buffer.count += count;
        if (data->ep_write_buffer.count < data->ep_write_buffer.length) {
            /* write not finished, wait for next write */
        } else {
            //data->write_callback(data); /* should not block write */
            threadpool_add(worker_threadpool, data->write_callback, NULL, 0);
            reset_epdata(data);
            break;
        }
    }
}
```
  
reset\_epdata函数，释放buffer的内存，并将该socket事件监听改为读。
```c
void reset_epdata(ep_data_t *data) {
    ss_free(data->ep_write_buffer.buffer);
    ss_free(data->ep_read_buffer.buffer);
    memset(&data->ep_write_buffer, 0, sizeof(data->ep_write_buffer));
    memset(&data->ep_read_buffer, 0, sizeof(data->ep_read_buffer));
    /* remove write event */
    struct epoll_event event;
    event.data.ptr = (void *)data;
    event.events = EPOLLIN | EPOLLET;
    ss_epoll_ctl(data->epfd, EPOLL_CTL_MOD, data->eventfd, &event);
}
```
  
do\_close函数，关闭连接，去掉监听。
```c
void do_close(void *arg) {
    log("do close, free data!");
    ep_data_t *data = (ep_data_t *)arg;
    if (data != NULL) {
        ss_epoll_ctl(data->epfd, EPOLL_CTL_DEL, data->eventfd, NULL);
        close(data->eventfd);
        if (data->ep_read_buffer.buffer != NULL)
            ss_free(data->ep_read_buffer.buffer);
        if (data->ep_write_buffer.buffer != NULL)
            ss_free(data->ep_write_buffer.buffer);
        ss_free(data);
    }
}
```
  
[所有代码在这个](https://github.com/linghuazaii/Simple-Server)

### 补充说明
&emsp;&emsp;我比较懒，没有将文件分开写。设计上请求处理函数和处理完后的通知函数都设计成了callback，所以如果进行一个完整封装的话，只有`handle_request`和`handle_response`函数是对上层可见的。这样基于此又可以设计更多样的上层协议，无论是做RPC也好还是做其他的也好，我也正打算用这个简易的server写一个服务发现的server，然后可以写一个简易的分布式cache，然后可以写一个简易的http server，后端的这一套就算基本齐全了吧。  

### Note
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
