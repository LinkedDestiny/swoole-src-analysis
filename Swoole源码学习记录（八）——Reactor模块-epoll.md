Swoole源码学习记录
===================
-------------
##Swoole版本：1.7.4-stable
-------------
Reactor模块可以说是Swoole中最核心的模块之一，正是这些reactor模型为swoole提供了异步操作的基础。Swoole中根据不同的内核函数，提供了四种Reactor封装，ReactorEpoll，ReactorKqueue，ReactorPoll和ReactorSelect。同时，Swoole通过结构体swReactor封装了对于reactor的操作函数和基本属性。本章，我将分析swReactor以及四种Reactor模型中的ReactorEpoll，并回顾一下epoll的相关知识。

一．swReactor
swReactor结构体声明在swoole.h的**698 – 721**行，其声明如下：
```c
struct swReactor_s
{
    void *object;
    void *ptr; //reserve
    uint32_t event_num;
    uint32_t max_event_num;
    uint16_t id; //Reactor ID
    uint16_t flag; //flag
    char running;

    swReactor_handle handle[SW_MAX_FDTYPE];       //默认事件
    swReactor_handle write_handle[SW_MAX_FDTYPE]; //扩展事件1(一般为写事件)
    swReactor_handle error_handle[SW_MAX_FDTYPE]; //扩展事件2(一般为错误事件,如socket关闭)

    int (*add)(swReactor *, int fd, int fdtype);
    int (*set)(swReactor *, int fd, int fdtype);
    int (*del)(swReactor *, int fd);
    int (*wait)(swReactor *, struct timeval *);
    void (*free)(swReactor *);
    int (*setHandle)(swReactor *, int fdtype, swReactor_handle);

    void (*onTimeout)(swReactor *); //发生超时时
    void (*onFinish)(swReactor *);  //完成一次轮询
};
typedef struct swReactor_s swReactor;  // swoole.h  354行
typedef int (*swReactor_handle)(swReactor *reactor, swEvent*event); // swoole.h 355行
```
属性分析：object存放实际reactor模型的内存地址，reserve ， event_num存放现有的事件数目，max_event_num存放允许持有的最大事件数目，id用于存放对应reactor的id，flag为标记位，running用于标记该reactor是否正在运行，接下来三个数组用于存放需要监听的事件的响应回调函数，余下的都是对应reactor的操作函数，这些函数将在具体的Reactor模型中被实现并赋值。
对应Reactor的部分通用的操作函数声明在swoole.h文件中**827 – 873**行。首先是一个枚举类型SW_EVENTS，用于指定对应的事件id，声明如下：
```c
enum SW_EVENTS
{
SW_EVENT_DEAULT = 256,
SW_EVENT_READ = 1u << 9,
SW_EVENT_WRITE = 1u << 10,
SW_EVENT_ERROR = 1u << 11,
};
```
SW_EVENTS中指定了四种事件类型：DEFAULT代表默认事件，READ和WRITE分别代表读写事件，ERROR代表错误。
接着是四个操作事件类型的函数：
```c
// 过滤fdtype中的读、写、错误事件标记
static sw_inline int swReactor_fdtype(int fdtype)
{
return fdtype & (~SW_EVENT_READ) & (~SW_EVENT_WRITE) & (~SW_EVENT_ERROR);
}
    // 判定是否为读事件和其他swFd_type类型的监听
static sw_inline int swReactor_event_read(int fdtype)
{
return (fdtype < SW_EVENT_DEAULT) || (fdtype & SW_EVENT_READ);
}
    // 判定是否为事件监听
static sw_inline int swReactor_event_write(int fdtype)
{
return fdtype & SW_EVENT_WRITE;
}
    // 判定是否为错误事件监听
static sw_inline int swReactor_event_error(int fdtype)
{
return fdtype & SW_EVENT_ERROR;
}
```
接着是四个个通用操作函数，函数声明如下：
```c
int swReactor_auto(swReactor *reactor, int max_event);
int swReactor_receive(swReactor *reactor, swEvent *event);
int swReactor_setHandle(swReactor *, int, swReactor_handle);
swReactor_handle swReactor_getHandle(swReactor *reactor, int event_type, int fdtype);
```
这四个函数分别用于自动创建可用类型的reactor模型、从reactor接收到的swEvent中读取数据、设置Reactor的回调函数、获得Reactor的回调函数。函数的具体声明在ReactorBase.c文件中。
这里补充说明swEvent结构体，其声明在swoole.h文件的**311 – 317** 行，声明如下：
```c
typedef struct _swEvent
{
    int fd; // 描述符
    int16_t from_id; // 来自哪个reactor
    uint8_t type;  // 描述符类型
    void *object;   // 数据域
} swEvent;
```
在ReactorBase.c文件中，具体实现了Reactor的四个通用操作函数。第一个函数是swReactor_auto，其核心代码如下：
int swReactor_auto(swReactor *reactor, int max_event) // 第二个参数设置允许监听的最大事件数
```c
    int ret;
#ifdef HAVE_EPOLL
    ret = swReactorEpoll_create(reactor, max_event);
#elif defined(HAVE_KQUEUE)
    ret = swReactorKqueue_create(reactor, max_event);
#elif defined(SW_MAINREACTOR_USE_POLL)
    ret = swReactorPoll_create(reactor, max_event);
#else
    ret = swReactorSelect_create(SwooleG.main_reactor)
#endif
    return ret;
```
源码解释：根据环境编译中定义的参数决定使用哪种类型的reactor模型，主线程默认使用poll模型。 

这里需要提前介绍一个枚举类型swFd_type。枚举类型swFd_type指定了描述符fd的一些特殊类型，这些类型主要用于reactor直接辨识某个fd类型的回调函数（同类型的fd共用一个回调函数）。该枚举类型声明在swoole.h文件中的**165 – 179** 行，如下：
```c
enum swFd_type
{
    SW_FD_TCP             = 0, //tcp socket
    SW_FD_LISTEN          = 1, //server socket
    SW_FD_CLOSE           = 2, //socket closed
    SW_FD_ERROR           = 3, //socket error
    SW_FD_UDP             = 4, //udp socket
    SW_FD_PIPE            = 5, //pipe
    SW_FD_WRITE           = 7, //fd can write
    SW_FD_TIMER           = 8, //timer fd
    SW_FD_AIO             = 9, //linux native aio
    SW_FD_SEND_TO_CLIENT  = 10, //sendto client
    SW_FD_SIGNAL          = 11, //signalfd
    SW_FD_DNS_RESOLVER    = 12, //dns resolver
};
```
swReactor_getHandle函数和swReactor_setHandle函数分别用于获取和设置相应的回调函数。swReactor_getHandle函数的核心代码如下：
```c
if (event_type == SW_EVENT_WRITE)
    {
        //默认可写回调函数SW_FD_WRITE
        return (reactor->write_handle[fdtype] != NULL) ? reactor->write_handle[fdtype] : reactor->handle[SW_FD_WRITE];
    }
    if (event_type == SW_EVENT_ERROR)
    {
        //默认关闭回调函数SW_FD_CLOSE
        return (reactor->error_handle[fdtype] != NULL) ? reactor->error_handle[fdtype] : reactor->handle[SW_FD_CLOSE];
    }
    return reactor->handle[fdtype];
```
源码解释：首先判定事件类型是否为写事件，如果是，判定参数fdtype指定的回调是否存在，如果不存在，默认返回SW_FD_WRITE回调，否则返回fdtype对应的回调；然后判定事件类型是否为异常事件，如果是，判定参数fdtype指定的回调是否存在，如果不存在，默认返回SW_FD_CLOSE回调，否则返回fdtype对应的回调。最后，如果事件类型为其他类型，则直接返回fdtype对应的回调。
    swReactor_setHandle函数的核心代码如下：
```c
    int fdtype = swReactor_fdtype(_fdtype);
    if (fdtype >= SW_MAX_FDTYPE)
    {
        swWarn("fdtype > SW_MAX_FDTYPE[%d]", SW_MAX_FDTYPE);
        return SW_ERR;
    }
    else
    {
        if (swReactor_event_read(_fdtype))
        {
            reactor->handle[fdtype] = handle;
        }
        else if (swReactor_event_write(_fdtype))
        {
            reactor->write_handle[fdtype] = handle;
        }
        else if (swReactor_event_error(_fdtype))
        {
            reactor->error_handle[fdtype] = handle;
        }
        else
        {
            swWarn("unknow fdtype");
            return SW_ERR;
        }
    }
```
源码解释：调用swReactor_fdtype函数去掉_fdtype参数中的SW_EVENTS类型变量，获取原始的swFd_type类型变量fdtype。如果fdtype超过了swoole规定的范围，则返回SW_ERR；否则，使用swReactor_event_*系列函数判定_fdtype的实际类型，根据不同的类型将回调函数存入reactor中对应的回调函数数组中。
swReactor_receive函数只是简单调用swRead方法从event的fd中读取了数据，不再赘述。

二．ReactorEpoll
首先回顾一下epoll的相关知识（在群里很多用PHP做开发的小伙伴似乎根本不了解什么是epoll什么是异步I/O……）epoll是Linux内核提供的一个多路复用I/O模型，它提供和poll函数一样的功能：监控多个文件描述符是否处于I/O就绪状态（可读、可写）。这就是异步最核心的表现：程序不是主动等待一个描述符可以操作，而是当描述符可操作时由系统提醒程序可以操作了，程序在被提醒前可以去做其他的事情（这里的程序、描述符、系统可以更换为其他东西）
Linux提供了三个主要的系统调用：**epoll_create**，**epoll_ctl**，**epoll_wait**。
epoll_create用于创建一个epoll实例并返回这个实例的文件描述符。epoll_ctl用于将一个需要监控的文件描述符在epoll中注册对应的监听事件，该函数也可用于更改一个已注册描述符的监听事件。epoll_wait函数用于等待监听的描述符的I/O事件，如果所有描述符都没有就绪，该函数会阻塞直到有至少一个描述符进入就绪状态。（该段描述翻译自 Linux 命令：man epoll ）
    上周参与腾讯面试时就被问到了这样的问题：请说明一下epoll函数的水平触发（Level-triggered）和边缘触发（edge-triggered）两种模式的区别。结果我逗比的没答出来……在此重新复习一下这个知识……水平触发和边缘触发是epoll的两种模式，它们的区别在于：水平触发模式下，当一个fd就绪之后，如果没有对该fd进行操作，则系统会继续发出就绪通知直到该fd被操作；边缘触发模式下，当一个fd就绪后，系统仅会发出一次就绪通知。
（相关链接：http://baike.baidu.com/view/1385104.htm http://yaocoder.blog.51cto.com/2668309/888374 ）

Swoole中，所有swReactorEpoll的相关定义都在ReactorEpoll.c中实现。首先说明两个epoll相关的宏：EPOLLRDHUP和EPOLLONESHOT。
EPOLLRDHUP代表的意义是对端断开连接，这个宏是用于弥补epoll在处理对端断开连接时可能会出现的一处Bug。
EPOLLONESHOT用于标记epoll对于每个socket仅监听一次事件，如果需要再次监听这个socket，需要再次将该socket加入epoll的监听队列中。

ReactorEpoll首先声明了一个结构体swFd用于封装一个描述符类型，其声明如下：
```c
#pragma pack(4)
typedef struct _swFd
{
    uint32_t fd;
    uint32_t fdtype;
} swFd;
#pragma pack()
```
其中, #pragma pack(4)的含义是指定结构体内的成员变量按照4字节对齐（关于字节对齐请参考http://baike.baidu.com/view/2317161.htm 
http://www.cppblog.com/tauruser/archive/2007/02/28/19049.html  ）
该结构体存放两个变量，一个变量为文件描述符，另一个变量为描述符类型。
同样的，Swoole也封装了一个结构体swReactorEpoll用于存放epoll的描述符以及监听的事件列表。该结构体的声明如下：
```c
struct swReactorEpoll_s
{
    int epfd;
    struct epoll_event *events;
};
```
创建一个ReactorEpoll的函数声明在swoole.h文件中的**852**行，其声明如下：
```c
int swReactorPoll_create(swReactor *reactor, int max_event_num);
```
该函数的核心源码如下：
```c
    swReactorEpoll *reactor_object = sw_malloc(sizeof(swReactorEpoll));
    if (reactor_object == NULL)
    {
        swWarn("malloc[0] failed.");
        return SW_ERR;
    }
    bzero(reactor_object, sizeof(swReactorEpoll));
    reactor->object = reactor_object;
    reactor->max_event_num = max_event_num;

    reactor_object->events = sw_calloc(max_event_num, sizeof(struct epoll_event));

    if (reactor_object->events == NULL)
    {
        swWarn("malloc[1] failed.");
        return SW_ERR;
    }
    //epoll create
    reactor_object->epfd = epoll_create(512);
    if (reactor_object->epfd < 0)
    {
        swWarn("epoll_create failed. Error: %s[%d]", strerror(errno), errno);
        return SW_ERR;
    }
```
源码解释：申请一个swReactorEpoll结构体并初始化。设置reactor的object和max_event_num参数。调用epoll_create函数创建一个epoll实例，参数512指定最大的监听fd数量。
swReactorEpoll共有5个操作函数，其声明如下：
```c
static int swReactorEpoll_add(swReactor *reactor, int fd, int fdtype);
static int swReactorEpoll_set(swReactor *reactor, int fd, int fdtype);
static int swReactorEpoll_del(swReactor *reactor, int fd);
static int swReactorEpoll_wait(swReactor *reactor, struct timeval *timeo);
static void swReactorEpoll_free(swReactor *reactor);
```
这5个函数基于epoll函数家族以及close函数实现，用于对epoll实例的添加fd、设置fd监听事件、移除fd、等待fd事件以及释放epoll实例。同时Swoole还声明了一个内联函数swReactorEpoll_event_set用于将自定义的SW_EVENTS类型转变为标准的epoll事件类型（EPOLLET、EPOLLIN、EPOLLOUT、EPOLLRDHUP）。下面将一一分析这些函数。
1.  swReactorEpoll_add
核心源码：
```c
    swReactorEpoll *object = reactor->object;
    struct epoll_event e;
    swFd fd_;
    int ret;
    bzero(&e, sizeof(struct epoll_event));

    fd_.fd = fd;
    fd_.fdtype = swReactor_fdtype(fdtype);
    e.events = swReactorEpoll_event_set(fdtype);

    memcpy(&(e.data.u64), &fd_, sizeof(fd_));
    ret = epoll_ctl(object->epfd, EPOLL_CTL_ADD, fd, &e);
    if (ret < 0)
    {
        swWarn("add event failed. Error: %s[%d]", strerror(errno), errno);
        return SW_ERR;
    }
    swTraceLog(SW_TRACE_EVENT, "add event[reactor_id=%d|fd=%d]", reactor->id, fd);
    reactor->event_num++;
```
源码解释：获取reactor中的swReactorEpoll结构体，创建一个epoll_event结构体e和一个swFd结构体fd_,初始化fd_参数并将该对象存放到epoll_event的data域中的u64变量中。调用epoll_ctl添加对该fd的监听，并将reactor的event_num计数加一。
2.  swReactorEpoll_set
核心代码与swReactorEpoll_add基本一致，唯一不同在于epoll_ctl函数的第二个参数由EPOLL_CTL_ADD变成EPOLL_CTL_MOD，代表设置fd的监听事件（而不是新增）。
3.  swReactorEpoll_del
核心源码：
```c
    swReactorEpoll *object = reactor->object;
    int ret;

    if (fd <= 0)
    {
        return SW_ERR;
    }
    
    ret = epoll_ctl(object->epfd, EPOLL_CTL_DEL, fd, NULL);
    if (ret < 0)
    {
        swWarn("epoll remove fd[=%d] failed. Error: %s[%d]", fd, strerror(errno), errno);
        return SW_ERR;
    }
    //close时会自动从epoll事件中移除
    //swoole中未使用dup
    ret = close(fd);
    if (ret >= 0)
    {
        (reactor->event_num <= 0) ? reactor->event_num = 0 : reactor->event_num--;
    }
```
源码解释：获取reactor中的swReactorEpoll结构体，创建一个epoll_event结构体e，设置e的data域的fd变量为指定需要删除的fd。调用epoll_ctl并指定操作位EPOLL_CTL_DEL用监听队列中移除对应的监听，并close对应的fd。如果移除成功，更改reactor的event_num计数。
4.  swReactorEpoll_wait
由于该函数较长且比较重要，在此将分段分析该函数。
```
    swEvent ev;
    swReactorEpoll *object = reactor->object;
    swReactor_handle handle;
    int i, n, ret, usec;

    int reactor_id = reactor->id;
    int epoll_fd = object->epfd;
    int max_event_num = reactor->max_event_num;
    struct epoll_event *events = object->events;

    if (timeo == NULL)
    {
        usec = SW_MAX_UINT;
    }
    else
    {
        usec = timeo->tv_sec * 1000 + timeo->tv_usec / 1000;
    }
```
源码解释：该段源码声明了所需使用的全部临时变量。ev是相应事件数据的封装，是回调函数handle的第二个参数。object为swReactorEpoll结构体变量。n为每一次epoll_wait响应后返回的当前处于就绪状态的fd的数量，usec为epoll_wait的timeout超时时间，由swReactorEpoll_wait函数的第二个参数struct timeval *timeo指定。接下来的几个参数，reactor_id为swReactor的标记，epoll_fd为epoll实例的描述符，max_event_num为允许监听的最大事件数量，events用于存放epoll函数发现的处于就绪状态的事件。
```c
while (SwooleG.running > 0)
```
源码解释：这是一个核心循环，之所以单独提出来是因为SwooleG变量非常重要。该变量声明在Server.c文件中的**53**行，并在swoole.h的**1081**行中通过extern关键字修饰使之可以被其他关联文件访问。该结构体中主要存放了整个swoole运行中需要的一些全局变量，在这里使用running变量用于标记swoole主循环是否正在执行。
```c
    n = epoll_wait(epoll_fd, events, max_event_num, usec);
    if (n < 0)
    {
        if (swReactor_error(reactor) < 0)
        {
            swWarn("[Reactor#%d] epoll_wait failed. Error: %s[%d]", reactor_id, strerror(errno), errno);
            return SW_ERR;
        }
        else
        {
            continue;
        }
    }
    else if (n == 0)
    {
        if (reactor->onTimeout != NULL)
        {
            reactor->onTimeout(reactor);
        }
        continue;
    }
```
源码解释：调用epoll_wait函数获取已经处于就绪状态的fd的集合，该集合存放在events结构体数组中，其数目为返回值n。如果没有任何描述符处于就绪状态，该函数会阻塞直到有描述符就绪。如果在usec毫秒后仍没有描述符就绪，则返回0。
```c
        for (i = 0; i < n; i++)
        {
            ev.fd = events[i].data.u64;
            ev.from_id = reactor_id;
            ev.type = events[i].data.u64 >> 32;

            //read
            if (events[i].events & EPOLLIN)
            {
                //read
                handle = swReactor_getHandle(reactor, SW_EVENT_READ, ev.type);
                ret = handle(reactor, &ev);
                if (ret < 0)
                {
                    swWarn("[Reactor#%d] epoll [EPOLLIN] handle failed. fd=%d. Error: %s[%d]", reactor_id, ev.fd, strerror(errno), errno);
                }
            }
            //write, ev.fd == 0, connection is closed.
            if ((events[i].events & EPOLLOUT) && ev.fd > 0)
            {
                handle = swReactor_getHandle(reactor, SW_EVENT_WRITE, ev.type);
                ret = handle(reactor, &ev);
                if (ret < 0)
                {
                    swWarn("[Reactor#%d] epoll [EPOLLOUT] handle failed. fd=%d. Error: %s[%d]", reactor_id, ev.fd, strerror(errno), errno);
                }
            }
            //error
#ifndef NO_EPOLLRDHUP
            if ((events[i].events & (EPOLLRDHUP | EPOLLERR | EPOLLHUP)) && ev.fd > 0)
#else
            if ((events[i].events & (EPOLLERR | EPOLLHUP)) && ev.fd > 0)
#endif
            {
                handle = swReactor_getHandle(reactor, SW_EVENT_ERROR, ev.type);
                ret = handle(reactor, &ev);
                if (ret < 0)
                {
                    swWarn("[Reactor#%d] epoll [EPOLLERR] handle failed. fd=%d. Error: %s[%d]", reactor_id, ev.fd, strerror(errno), errno);
                }
            }
        }
```
源码解释：这是核心的事件处理逻辑了。循环遍历n个待处理事件，前三行设置swEvent的对应参数（这里需要注意，data域中的u64变量是一个uint64_t类型，该变量被写入了一个swFd结构体，低位32位存放的是uint32_t类型的fd，高位32位存放的是uint32_t类型的fdtype，该fdtype的取值为枚举类型swFd_type），然后根据events的不同类型进入不同的处理逻辑，如下：
a.  EPOLLIN 读事件，根据swEvent中的type类型（fdtype）获取对应的读操作的回调函数，通过该回调将swEvent类型发出。
b.  EPOLLOUT 写事件，根据swEvent中的type类型（fdtype）获取对应的写操作的回调函数，通过该回调将swEvent类型发出。
c.  异常事件，根据swEvent中的type类型（fdtype）获取对应的异常操作的回调函数，通过该回调将swEvent类型发出。
5.  swReactorEpoll_free
调用close函数关闭epoll实例的描述符，并释放申请的内存空间

至此，swReactorEpoll分析已经结束。下一章将分析剩下的三种类型poll，select和kqueue。
