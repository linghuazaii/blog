怎样写一个自修复的TCP连接池
===========================

### 原料
&emsp;&emsp;epoll, threadpool, pthread

### 构图
&emsp;&emsp;首先，我们怎么定义一个socket连接？其一当然是socket file descriptor；其二就是关于socket的一些设置选项，像`TCP_NODELAY`, `TCP_CORK`, `SOCK_NONBLOCK`, `TCP_KEEPALIVE`等等；其三，socket可能由于server crash或者一段时间闲置被server close，或者其他原因导致其失效，所以需要一个选项保存socket的状态；其四，这个socket是所有线程共享的，这里的共享是指socket被某一个thread里拿出来，可能因为失效，另一个做修复的thread正在修复这个socket连接。由于在连接失效时才修复连接并更改连接是否可用的选项，而且每次从池子拿连接都得判断连接是否可用，所以需要一把读写锁，因为写得少读的多嘛；其五，设置一个额外的字段留着备用。结构如下：  
```c
typedef struct tcp_connection_t {
    int fd;
    int flags;
    bool valid;
    pthread_rwlock_t rwlock;
    void *extra;
} tcp_connection_t;
```
&emsp;&emsp;其次，一个连接池需要哪些东西呢？其一，需要一个大池子`vector<tcp_connection_t *> pool`存储所有的连接信息；其二，需要一个队列存储可用的连接`queue<tcp_connection_t *> ready_pool`；其三，这些池子是共享的，所以需要一把锁，一个条件变量去检测池子里是否有连接可用；其四，池子对外接口只需要`initPool()`, `getConnection()`, `putConnection()`；其五，需要一些额外的信息保存server信息以及池子信息。类声明如下：
```c
class CharlesTcpPool {
public:
    CharlesTcpPool(const char *ip, int port, int max_size = MAX_POOL_SIZE, int init_size = INIT_POOL_SIZE);
    ~CharlesTcpPool();
    int initPool(int flags = CHARLES_OPTION_NONE);
    tcp_connection_t * newConnection(int flags = CHARLES_OPTION_NONE);
    /* flags is for newConnection when ready pool is empty but pool is not full */
    tcp_connection_t * getConnection(int timeout /* milisecond */, int flags = CHARLES_OPTION_NONE);
    void putConnection(tcp_connection_t *connection);
    int setConfig(int sock, int flags);
public:
    void watchPool();
    void repairConnection(tcp_connection_t *connection);
private:
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int max_pool_size;
    int init_pool_size;
    vector<tcp_connection_t *> pool;
    queue<tcp_connection_t *> ready_pool;
    threadpool_t *threadpool;
    int epollfd;
    char ip[IPLEN];
    int port;
    bool running;
};
```
&emsp;&emsp;然后，连接失效了我们怎么修呢？这里就需要用到`epoll`了，所有新建立的连接需要加到`epoll`里监控，只需要监控`EPOLLIN`，这里我们只能选择LT模式，因为ET模式只通知一次，为了避免数据丢失，只能选择LT模式。只要有`EPOLLIN`事件产生，就用`ioctl(fd, FIONREAD, &length)`检查一下socket可读的buffer大小，如果为`0`则表示server关闭了连接，需要我们修复。你可能会问：那buffer里有内容我一直没读怎么办？没关系，没读是你的事，等你读完了，对于该socket会一直有`EPOLLIN`事件产生，这里检查可读buffer大小依旧为`0`，此时连接才被判定为失效。一般server挂掉之后起来得花些时间，所以每一次重建连接失败的话会有一个等待延时，而且有一个重试次数，如果耗尽的话就等待下一次`epoll_wait`返回再试。这里其实可以稍微设计一下算法的，比如说每一次重建连接失败都会略微增加延时。其实也没有太大的必要，因为有延时后就减少了很大一部分CPU消耗了，可以忽略不计。

### 实现
&emsp;初始化该初始化的东西。
```c
CharlesTcpPool::CharlesTcpPool(const char *server_ip, int server_port, int max_size, int init_size) {
    strncpy(ip, server_ip, IPLEN);
    port = server_port;
    max_pool_size = max_size;
    init_pool_size = init_size;
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&cond, NULL);
    pool.reserve(max_size);
}
```  
  
&emsp;析构的时候还是加上锁吧，不然不知道会不会出什么幺蛾子。
```c
CharlesTcpPool::~CharlesTcpPool() {
    // do clean
    pthread_mutex_lock(&mutex);
    for (int i = 0; i < pool.size(); ++i) {
        tcp_connection_t *connection = pool[i];
        pthread_rwlock_wrlock(&connection->rwlock);
        charles_epoll_ctl(epollfd, EPOLL_CTL_DEL, connection->fd, NULL);
        close(connection->fd);
        connection->valid = false;
        pthread_rwlock_unlock(&connection->rwlock);
        pthread_rwlock_destroy(&connection->rwlock);
        delete connection;
    }
    running = false;
    ready_pool = queue<tcp_connection_t *>();
    pthread_mutex_unlock(&mutex);
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
}
```
  
&emsp;`initPool()`实现，主要是新建一堆连接，启动后台的连接监测线程，还有连接修复线程池。
```c
int CharlesTcpPool::initPool(int flags) {
    epollfd = charles_epoll_create();
    if (epollfd == -1)
        return -1;
    threadpool = threadpool_create(THREADPOOL_SIZE, THREADPOOL_QUEUE_SIZE, 0);
    if (threadpool == NULL)
        return -1;

    for (int i = 0; i < init_pool_size; ++i) {
        tcp_connection_t *connection = newConnection(flags);
        if (connection == NULL) /* fail if can't create connection */
            return -1;
        pool.push_back(connection);
        ready_pool.push(connection);
    }

    pthread_t watcher;
    pthread_attr_t watcher_attr;
    pthread_attr_init(&watcher_attr);
    pthread_attr_setdetachstate(&watcher_attr, PTHREAD_CREATE_DETACHED);
    pthread_create(&watcher, &watcher_attr, watch_pool, (void *)this);
    pthread_attr_destroy(&watcher_attr);

    return 0;
}
```
  
&emsp;新建连接实现。
```c
tcp_connection_t * CharlesTcpPool::newConnection(int flags) {
    tcp_connection_t *connection = new tcp_connection_t;
    connection->fd = charles_socket(AF_INET, SOCK_STREAM, 0);
    connection->flags = flags;
    connection->valid = true;
    connection->extra = (void *)this;
    pthread_rwlock_init(&connection->rwlock, NULL);
    struct sockaddr_in server;
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    charles_inet_aton(ip, &server.sin_addr);
    if (-1 == setConfig(connection->fd, flags)) {
        close(connection->fd);
        delete connection;
        return NULL;
    }
    if (-1 == charles_connect(connection->fd, (struct sockaddr *)&server, sizeof(server))) {
        close(connection->fd);
        delete connection;
        return NULL;
    }

    return connection;
}
```
  
&emsp;`watch_pool`实现，就是为了回调进类里去。
```c
/* call back into class */
void *watch_pool(void *arg) {
    CharlesTcpPool *pool = (CharlesTcpPool *)arg;
    pool->watchPool();
}
```
  
&emsp;`watchPool()`后台监测连接线程函数实现。这里为了安全起见还是先上个锁，因为本线程启动之前可能有其他线程已经在使用池子了。所有初始的连接都加入到`epoll`里去监听，如果收到`EPOLLIN`事件并且可读buffer大小为`0`，则先将其从`epoll`监听里拿掉，送到连接修复线程池里去修复。
```c
void CharlesTcpPool::watchPool() {
    pthread_mutex_lock(&mutex);
    for (int i = 0; i < init_pool_size; ++i) {
        struct epoll_event event;
        event.events = EPOLLIN;
        event.data.ptr = (void *)pool[i];
        charles_epoll_ctl(epollfd, EPOLL_CTL_ADD, pool[i]->fd, &event);
    }
    pthread_mutex_unlock(&mutex);
    running = true;
    struct epoll_event events[max_pool_size];
    while (running) {
        int nfds = charles_epoll_wait(epollfd, events, max_pool_size);
        if (nfds == -1)
            continue;
        tcp_connection_t *connection;
        for (int i = 0; i < nfds; ++i) {
            connection = (tcp_connection_t *)events[i].data.ptr;
            if (events[i].events & EPOLLIN) {
                if (0 == get_socket_read_buffer_length(connection->fd)) {
                    pthread_rwlock_wrlock(&connection->rwlock);
                    connection->valid = false;
                    pthread_rwlock_unlock(&connection->rwlock);
                    /* remove from epoll event */
                    charles_epoll_ctl(epollfd, EPOLL_CTL_DEL, connection->fd, NULL);
                    threadpool_add(threadpool, repair_connection, (void *)connection, 0);
                }
            }
        }
    }
}
```
  
&emsp;`repair_connection()`连接修复函数实现，就是为了回调进类里面去。
```c
/* call back into class */
void repair_connection(void *arg) {
    tcp_connection_t *connection = (tcp_connection_t *)arg;
    CharlesTcpPool *pool = (CharlesTcpPool *)connection->extra;
    pool->repairConnection(connection);
}
```
  
&emsp;`repairConnection()`实现，先备份下失效连接的`fd`，因为后续可能需要将其再次加入到`epoll`监听，避免将其意外关闭导致永久丢失该连接的监听。如果成功建立连接则将连接状态置为有效并关闭备份连接，完了无论是否成功建立连接，都需要将该连接重新加入到`epoll`监听等待下一次`epoll_wait`返回。
```c
void CharlesTcpPool::repairConnection(tcp_connection_t *connection) {
    /* I must backup this fd, can't close it, it must remains a valid file descriptor */
    int backup = connection->fd;
    int count = 0;
    for (; count < RETRY_COUNT; ++count) {
        connection->fd = charles_socket(AF_INET, SOCK_STREAM, 0);
        struct sockaddr_in server;
        server.sin_family = AF_INET;
        server.sin_port = htons(port);
        charles_inet_aton(ip, &server.sin_addr);
        if (-1 == setConfig(connection->fd, connection->flags)) {
            close(connection->fd);
            usleep(RETRY_PERIOD * 1000);
            continue;
        }
        if (-1 == charles_connect(connection->fd, (struct sockaddr *)&server, sizeof(server))) {
            close(connection->fd);
            usleep(RETRY_PERIOD * 1000);
            continue;
        }
        /* get a good connection */
        pthread_rwlock_wrlock(&connection->rwlock);
        connection->valid = true;
        close(backup); /* close backup if we new connection successfully */
        pthread_rwlock_unlock(&connection->rwlock);
        break;
    }
    /* add to epoll event even if this connection is not repaired, wait for next time repair */
    struct epoll_event event;
    event.events = EPOLLIN;
    if (count == RETRY_COUNT) { /* failed, add original fd to epoll events */
        connection->fd = backup;
    }
    event.data.ptr = connection;
    charles_epoll_ctl(epollfd, EPOLL_CTL_ADD, connection->fd, &event);
}
```
  
&emsp;逻辑最复杂的`getConnection()`实现，先记录一个`start_time`，如果可用的池子空了并且连接池已满，那么就等待连接可用直到`timeout`。否则，如果可用池子空了但是连接池未满，则新建一个连接返回并将其加入到连接池，但是并不加入到可用连接池，因为用完后会还给可用连接池。如果可用连接池不为空，则循环最多`max_pool_size`次数从可用连接池里拿连接直到`timeout`或者所有连接都不可用。这里为什么用`max_pool_size`而不是`read_pool.size()`，因为不仅`read_pool.size()`是动态可变的而且新建的连接可能和失效的连接可能是穿插的，有点绕，好好想一下。`max_pool_size`确保了所有连接都过了一遍，即使有的连接被过了多次也无所谓，因为连接池不会非常巨大，所以暂不考虑算法层面上的优化。
```c
tcp_connection_t *CharlesTcpPool::getConnection(int timeout /* milisecond */, int flags) {
    struct timespec ts;
    ts.tv_sec = timeout / 1000;
    ts.tv_nsec = (timeout % 1000) * 1000 * 1000;
    struct timespec start_time, end_time;
    clock_gettime(CLOCK_MONOTONIC_COARSE, &start_time);
    pthread_mutex_lock(&mutex);
    while (ready_pool.empty() && pool.size() == max_pool_size) {
        int ret = pthread_cond_timedwait(&cond, &mutex, &ts);
        if (ret == ETIMEDOUT) {
            pthread_mutex_unlock(&mutex);
            return NULL;
        }
    }
    if (ready_pool.empty()) {
        tcp_connection_t *connection = newConnection(flags);
        if (connection == NULL) {
            pthread_mutex_unlock(&mutex);
            return NULL;
        }
        pool.push_back(connection);
        pthread_mutex_unlock(&mutex);
        return connection;
    } else {
        for (int count = 0; count < max_pool_size; ++count) {
            tcp_connection_t *connection = ready_pool.front();
            ready_pool.pop();
            int valid;
            pthread_rwlock_rdlock(&connection->rwlock);
            valid = connection->valid;
            pthread_rwlock_unlock(&connection->rwlock);
            if (valid == false) {
                ready_pool.push(connection);
                clock_gettime(CLOCK_MONOTONIC_COARSE, &end_time);
                int period = (end_time.tv_sec * 1000 + end_time.tv_nsec / 1000000) - (start_time.tv_sec * 1000 + start_time.tv_nsec / 1000000);
                if (period >= timeout) {
                    pthread_mutex_unlock(&mutex);
                    return NULL;
                }
            } else {
                pthread_mutex_unlock(&mutex);
                return connection;
            }
        }
        /* at last, all connection is failed */
        pthread_mutex_unlock(&mutex);
        return NULL;
    }
}                
```
  
&emsp;`putConnection()`实现，还给池子，broadcast给所有线程，池子里有可用连接啦。
```c
void CharlesTcpPool::putConnection(tcp_connection_t *connection) {
    pthread_mutex_lock(&mutex);
    ready_pool.push(connection);
    pthread_cond_broadcast(&cond);
    pthread_mutex_unlock(&mutex);
}
```
  
&emsp;&emsp;还有一些其他函数即细节没有一一罗列，所有代码请见[Charles TCP Pool](https://github.com/linghuazaii/Charles-TcpPool)  

### 小结
&emsp;&emsp;**GOOD LUCK, HAVE FUN!**
