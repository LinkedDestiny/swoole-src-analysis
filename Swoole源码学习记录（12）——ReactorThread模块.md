####**Swoole版本：1.7.5-stable**
-------------

###**ReactorThread**
这一章将分析Swoole的ReactorThread模块。虽然叫Thread，但是实际上使用的是swFactoryProcess也就是多进程模式。但是，在ReactorThread中，所有的事件监听是在线程中运行的（Rango只是简单提到了PHP不支持多线程安全，具体原因还有待请教……），比如在UDP模式下，是针对每一个监听的host开辟一个线程运行reactor，在TCP模式下，则是开启指定的reactor_num个线程用于运行reactor。

那么OK，先上swReactorThread结构体。该结构体封装的其实是一个运行着Reactor的线程Thread的相关信息，其声明在Server.h文件的**104 – 112**行，如下：
```c
typedef struct _swReactorThread
{
    pthread_t thread_id;
    swReactor reactor;
    swUdpFd *udp_addrs;
    swMemoryPool *buffer_input;
    swArray *buffer_pipe;
    int c_udp_fd;
} swReactorThread;
```
// TODO


另一个结构体用来封装需要传递给Thread的参数，其声明在swoole.h的**575 - 579**行，如下：
```c
typedef struct _swThreadParam
{
	void *object;
	int pti;
} swThreadParam;
```
第一个void*指针指向了参数内容的地址，第二个参数标记线程的id（不是pid）

ReactorThread在Server.h中一共有……嗯……12个函数……声明在Server.h的**555 - 568**行。
这里分下类，其中3个函数用于创建、启动、释放，归为**操作类函数**，7个函数用于回调操作，归为**回调函数**，剩下2个用于发送数据，归为**发送函数**。

----
首先看3个操作类函数，这三个函数的声明如下：
```c
int swReactorThread_create(swServer *serv);
int swReactorThread_start(swServer *serv, swReactor *main_reactor_ptr);
void swReactorThread_free(swServer *serv);
```
首先是**swReactorThread_create**函数，该函数实质上并不是创建一个ReactorThread，而是初始化swServer中相关变量，并创建相应的Factory。下面上核心源码：
```c
	serv->reactor_threads = SwooleG.memory_pool->alloc(SwooleG.memory_pool, (serv->reactor_num * sizeof(swReactorThread)));
    if (serv->reactor_threads == NULL)
    {
        swError("calloc[reactor_threads] fail.alloc_size=%d", (int )(serv->reactor_num * sizeof(swReactorThread)));
        return SW_ERR;
    }

#ifdef SW_USE_RINGBUFFER
    int i;
    for (i = 0; i < serv->reactor_num; i++)
    {
        serv->reactor_threads[i].buffer_input = swRingBuffer_new(SwooleG.serv->buffer_input_size, 1);
        if (!serv->reactor_threads[i].buffer_input)
        {
            return SW_ERR;
        }
    }
#endif

    serv->connection_list = sw_shm_calloc(serv->max_connection, sizeof(swConnection));
    if (serv->connection_list == NULL)
    {
        swError("calloc[1] failed");
        return SW_ERR;
    }
```
源码解释：初始化运行reactor的线程池，如果指定使用了RingBuffer，则将reactor_threads里的输入缓存区的类型设置为RingBuffer。随后，在共享内存中初始化connectoin_list连接列表的内存空间。
```c
    //create factry object
    if (serv->factory_mode == SW_MODE_THREAD)
    {
        if (serv->writer_num < 1)
        {
            swError("Fatal Error: serv->writer_num < 1");
            return SW_ERR;
        }
        ret = swFactoryThread_create(&(serv->factory), serv->writer_num);
    }
    else if (serv->factory_mode == SW_MODE_PROCESS)
    {
        if (serv->writer_num < 1 || serv->worker_num < 1)
        {
            swError("Fatal Error: serv->writer_num < 1 or serv->worker_num < 1");
            return SW_ERR;
        }
        ret = swFactoryProcess_create(&(serv->factory), serv->writer_num, serv->worker_num);
    }
    else
    {
        ret = swFactory_create(&(serv->factory));
    }
```
源码解释：判断swServer的factory_mode。如果为**SW_MODE_THREAD**(线程模式)，则创建FactoryThread；如果为**SW_MODE_PROCESS**(进程模式)，则创建FactoryProcess；否则，为**SW_MODE_BASE**(基础模式)，创建Factory。

创建完后，就需要启动了。**swReactorThread_start**函数倒是真的用于启动ReactorThread了……核心源码如下：
```c
	if (serv->have_udp_sock == 1)
    {
        if (swUDPThread_start(serv) < 0)
        {
            swError("udp thread start failed.");
            return SW_ERR;
        }
    }

    //listen TCP
    if (serv->have_tcp_sock == 1)
    {
        //listen server socket
        ret = swServer_listen(serv, main_reactor_ptr);
        if (ret < 0)
        {
            return SW_ERR;
        }
        //create reactor thread
        for (i = 0; i < serv->reactor_num; i++)
        {
            thread = &(serv->reactor_threads[i]);
            param = SwooleG.memory_pool->alloc(SwooleG.memory_pool, sizeof(swThreadParam));
            if (param == NULL)
            {
                swError("malloc failed");
                return SW_ERR;
            }

            param->object = serv;
            param->pti = i;

            if (pthread_create(&pidt, NULL, (void * (*)(void *)) swReactorThread_loop_tcp, (void *) param) < 0)
            {
                swError("pthread_create[tcp_reactor] failed. Error: %s[%d]", strerror(errno), errno);
            }
            thread->thread_id = pidt;
        }
    }

    //timer
    if (SwooleG.timer.fd > 0)
    {
        main_reactor_ptr->add(main_reactor_ptr, SwooleG.timer.fd, SW_FD_TIMER);
    }
```
源码解释：如果swServer需要监听UDP，则调用**swUDPThread_start**函数启动UDP监听线程；如果swServer需要监听TCP，首先调用**swServer_listen**函数在main_reactor中注册accept监听，然后创建**reactor_num**个reactor运行线程用于监听TCP连接的其他事件（读、写）。最后，如果使用了Timer，则将Timer的fd监听加入到main_reactor中。

**swReactorThread_free**函数用于释放全部的正在运行的线程以及相关资源，核心源码如下：
```c
	if (serv->have_tcp_sock == 1)
    {
        //create reactor thread
        for (i = 0; i < serv->reactor_num; i++)
        {
            thread = &(serv->reactor_threads[i]);
            if (pthread_join(thread->thread_id, NULL))
            {
                swWarn("pthread_join() failed. Error: %s[%d]", strerror(errno), errno);
            }

            for (j = 0; j < serv->worker_num; j++)
            {
                swWorker *worker = swServer_get_worker(serv, i);
                swBuffer *buffer = *(swBuffer **) swArray_fetch(thread->buffer_pipe, worker->pipe_master);
                swBuffer_free(buffer);
            }
            swArray_free(thread->buffer_pipe);

#ifdef SW_USE_RINGBUFFER
            thread->buffer_input->destroy(thread->buffer_input);
#endif
        }
    }

    if (serv->have_udp_sock == 1)
    {
        swListenList_node *listen_host;
        LL_FOREACH(serv->listen_list, listen_host)
        {
            shutdown(listen_host->sock, SHUT_RDWR);
            if (listen_host->type == SW_SOCK_UDP || listen_host->type == SW_SOCK_UDP6 || listen_host->type == SW_SOCK_UNIX_DGRAM)
            {
                if (pthread_join(listen_host->thread_id, NULL))
                {
                    swWarn("pthread_join() failed. Error: %s[%d]", strerror(errno), errno);
                }
            }
        }
    }
```
源码解释：如果使用了TCP，则遍历全部的reactor_thread，并调用pthread_join函数结束线程，并释放线程中用于管道通信的缓存区。如果使用了RingBuffer，还需要释放buffer_input输入缓存。如果使用了UDP，则首先遍历监听列表，使用shutdown终止连接，然后调用pthread_join函数结束线程。

----
接着先看发送函数。一共两个发送函数，一个用于发送数据到client客户端或者输出buffer，一个用于发送数据到worker进程。两个函数的声明如下：
```c
int swReactorThread_send(swSendData *_send);
int swReactorThread_send2worker(void *data, int len, uint16_t target_worker_id);
```

**swReactorThread_send**函数用于发送数据到客户端，也就是通过swConnection发送数据，其核心源码如下：
```c
	volatile swBuffer_trunk *trunk;
    swConnection *conn = swServer_connection_get(serv, fd);

    if (conn == NULL || conn->active == 0)
    {
        swWarn("Connection[fd=%d] is not exists.", fd);
        return SW_ERR;
    }

#if SW_REACTOR_SCHEDULE == 2
    reactor_id = fd % serv->reactor_num;
#else
    reactor_id = conn->from_id;
#endif

    swTraceLog(SW_TRACE_EVENT, "send-data. fd=%d|reactor_id=%d", fd, reactor_id);
    swReactor *reactor = &(serv->reactor_threads[reactor_id].reactor);
```
