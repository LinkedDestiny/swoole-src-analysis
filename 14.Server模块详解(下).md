# Swoole源码学习记录

---
**swoole版本：1.7.6-stable**

上一章已经分析了如何启动swServer的相关函数。本章将继续分析swServer的相关函数，
##【Table of Contents】
- [1.swServer函数分析](#1swserver%E5%87%BD%E6%95%B0%E5%88%86%E6%9E%90)
    - [swServer_addListener](#swserver_addlistener)
    - [swServer_listen](#swserver_listen)
    - [swServer_addTimer](#swserver_addtimer)
    - [swServer_tcp_send](#swserver_tcp_send)
    - [swServer_reload](#swserver_reload)
- [2.Server相关结构体分析](#2server%E7%9B%B8%E5%85%B3%E7%BB%93%E6%9E%84%E4%BD%93%E5%88%86%E6%9E%90)
    - [swPackage](#swsackage)
    - [swPackage_task](#swsackage_task)
    - [swPackage_response](#swsackage_response)

---
###**1.swServer函数分析**
####**swServer_addListener**
该函数用于在swServer中添加一个需要监听的host及port。函数原型如下：
```c
// Server.h 438h
int swServer_addListener(swServer *serv, int type, char *host,int port);
```
| 参数    | 说明   | 
| --------  | ------  |
| swServer *serv|swServer对象|
| int type |创建的socket类型，见枚举**swSocket_type**|
| char* host |监听地址|
| int port |监听端口|
函数核心源码：
```c
    // Server.c 900~943h
    swListenList_node *listen_host = SwooleG.memory_pool->alloc(SwooleG.memory_pool, sizeof(swListenList_node));

    listen_host->type = type;
    listen_host->port = port;
    listen_host->sock = 0;
    listen_host->ssl = 0;

    bzero(listen_host->host, SW_HOST_MAXSIZE);
    strncpy(listen_host->host, host, SW_HOST_MAXSIZE);
    LL_APPEND(serv->listen_list, listen_host);

    //UDP需要提前创建好
    if (type == SW_SOCK_UDP || type == SW_SOCK_UDP6 || type == SW_SOCK_UNIX_DGRAM)
    {
        int sock = swSocket_listen(type, listen_host->host, port, serv->backlog);
        if (sock < 0)
        {
            return SW_ERR;
        }
        //设置UDP缓存区尺寸，高并发UDP服务器必须设置
        int bufsize = serv->udp_sock_buffer_size;
        setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
        setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));

        listen_host->sock = sock;
        serv->have_udp_sock = 1;
    }
    else
    {
        if (type & SW_SOCK_SSL)
        {
            type = type & (~SW_SOCK_SSL);
            listen_host->type = type;
            listen_host->ssl = 1;
        }
        if (type != SW_SOCK_UNIX_STREAM && port <= 0)
        {
            swError("listen port must greater than 0.");
            return SW_ERR;
        }
        serv->have_tcp_sock = 1;
    }
    return SW_OK;
```
源码解释：  
在**SwooleG**的共享内存池中创建一个**swListenList_node**，并设置type等相关参数，并将该node添加进**swServer**的**listen_list**。随后，判定type类型。如果是UDP类型的socket，需要直接调用swSocket_listen进行监听，并根据**swServer**的**udp_sock_buffer_size**设置socket的缓存区大小；如果是TCP类型的socket，只针对两种特别的type类型做判定（SSL类型要设置ssl开关，非Unix Sock类型要保证监听端口大于0）。

####**swServer_listen**
该函数用于开始监听swServer中全部的TCP类型的socket。函数原型如下：
```c
// Server.h 440h
int swServer_listen(swServer *serv, swReactor *reactor);
```
| 参数    | 说明   | 
| --------  | ------  |
| swServer *serv|swServer对象|
| swReactor *reactor |Reactor对象，监听实际的Listen事件|
函数核心源码：
```c
    // Server.c 949-998h
    LL_FOREACH(serv->listen_list, listen_host)
    {
        //UDP
        if (listen_host->type == SW_SOCK_UDP || listen_host->type == SW_SOCK_UDP6 || listen_host->type == SW_SOCK_UNIX_DGRAM)
        {
            continue;
        }

        //TCP
        sock = swSocket_listen(listen_host->type, listen_host->host, listen_host->port, serv->backlog);
        if (sock < 0)
        {
            LL_DELETE(serv->listen_list, listen_host);
            return SW_ERR;
        }

        if (reactor!=NULL)
        {
            reactor->add(reactor, sock, SW_FD_LISTEN);
        }

#ifdef TCP_DEFER_ACCEPT
        int sockopt;
        if (serv->tcp_defer_accept > 0)
        {
            sockopt = serv->tcp_defer_accept;
            setsockopt(sock, IPPROTO_TCP, TCP_DEFER_ACCEPT, &sockopt, sizeof(sockopt));
        }
#endif
        listen_host->sock = sock;
        //将server socket也放置到connection_list中
        serv->connection_list[sock].fd = sock;
        serv->connection_list[sock].addr.sin_port = listen_host->port;
        //save listen_host object
        serv->connection_list[sock].object = listen_host;
    }
    //将最后一个fd作为minfd和maxfd
    if (sock >= 0)
    {
        swServer_set_minfd(serv, sock);
        swServer_set_maxfd(serv, sock);
    }
```
源码解释：  
遍历**listen_list**列表，对所有TCP类型的socket，调用**swSocket_listen**函数启动监听，并将socket添加到reactor中。如果设置了**TCP_DEFER_ACCEPT**属性，则设置相应的socket option。最后，将监听的socket加入swServer的**connection_list**,并设置swServer的minfd和maxfd为最后一个待监听的server socket。

####**swServer_addTimer**
该函数用于在swServer中添加一个定时器。函数原型如下：
```c
// Server.h 443h
int swServer_addTimer(swServer *serv, int interval);
```
| 参数    | 说明   | 
| --------  | ------  |
| swServer *serv|swServer对象|
| int interval |Timer的时间间隔|
函数核心源码：
```c
    // Server.c 263-293h
    if (serv->onTimer == NULL)
    {
        swWarn("onTimer is null. Can not use timer.");
        return SW_ERR;
    }

    //timer no init
    if (SwooleG.timer.fd == 0)
    {
        if (swTimer_create(&SwooleG.timer, interval, SwooleG.use_timer_pipe) < 0)
        {
            return SW_ERR;
        }

        if (swIsMaster())
        {
            serv->connection_list[SW_SERVER_TIMER_FD_INDEX].fd = SwooleG.timer.fd;
        }

        if (SwooleG.use_timer_pipe)
        {
            SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_TIMER, swTimer_event_handler);
            SwooleG.main_reactor->add(SwooleG.main_reactor, SwooleG.timer.fd, SW_FD_TIMER);
        }

        SwooleG.timer.onTimer = swServer_onTimer;
    }
    return swTimer_add(&SwooleG.timer, interval);
```
源码解释：
首先判定是否有onTimer回调，如果没有则返回一个Error。随后，如果没有初始化timer计时器，则调用**swTimer_create**函数创建计时器。如果当前进程是master进程，将timer的fd添加到**connection_list**中。如果timer指定使用了pipe管道，则将timer的fd添加到**SwooleG**的**main_reactor**中。最后，调用**swTimer_add**添加timer。

####**swServer_tcp_send**
该函数用于发送TCP数据。函数原型如下：
```c
// Server.h 443h
int swServer_tcp_send(swServer *serv, int fd, void *data, int length);
```
| 参数    | 说明   | 
| --------  | ------  |
| swServer *serv|swServer对象|
| int fd |发送的socket描述符|
| void *data |需要发送的数据|
| int length |数据长度|
函数核心源码：
```c
    // Server.c 788-858h
    swSendData _send;
    swFactory *factory = &(serv->factory);

#ifndef SW_WORKER_SEND_CHUNK
    /**
     * More than the output buffer
     */
    if (length >= serv->buffer_output_size)
    {
        swWarn("More than the output buffer size[%d], please use the sendfile.", serv->buffer_output_size);
        return SW_ERR;
    }
    else
    {
        _send.info.fd = fd;
        _send.info.type = SW_EVENT_TCP;
        _send.data = data;

        if (length >= SW_BUFFER_SIZE)
        {
            _send.length = length;
        }
        else
        {
            _send.info.len = length;
            _send.length = 0;
        }
        return factory->finish(factory, &_send);
    }
#else
    char buffer[SW_BUFFER_SIZE];
    int trunk_num = (length / SW_BUFFER_SIZE) + 1;
    int send_n = 0, i, ret;

    swConnection *conn = swServer_connection_get(serv, fd);
    if (conn == NULL || conn->active == 0)
    {
        swWarn("Connection[%d] has been closed.", fd);
        return SW_ERR;
    }

    for (i = 0; i < trunk_num; i++)
    {
        //last chunk
        if (i == (trunk_num - 1))
        {
            send_n = length % SW_BUFFER_SIZE;
            if (send_n == 0)
                break;
        }
        else
        {
            send_n = SW_BUFFER_SIZE;
        }
        memcpy(buffer, data + SW_BUFFER_SIZE * i, send_n);
        _send.info.len = send_n;
        ret = factory->finish(factory, &_send);

#ifdef SW_WORKER_SENDTO_YIELD
        if ((i % SW_WORKER_SENDTO_YIELD) == (SW_WORKER_SENDTO_YIELD - 1))
        {
            swYield();
        }
#endif
    }
    return ret;
#endif
```
源码解释：
如果没有定义**SW_WORKER_SEND_CHUNK**宏，执行如下操作：
如果数据长度大于swServer的输出缓存大小，则报错。否则，设置swSendData的相关属性，调用swServer中的factory的finish函数将数据发出。
如果定义了**SW_WORKER_SEND_CHUNK**宏，执行如下操作：
首先根据数据长度length计算出需要多少个trunk。随后，获取fd对应的**swConnecton**，如果找不到connection或者connection已经关闭，则报错。随后，将数据划分为一个个trunk的长度，放进swSendData中后调用swServer中的factory的finish函数将数据发出。


####**swServer_reload**
该函数用于通知Manager进程重启全部的Worker进程。函数原型如下：
```c
// Server.h 443h
int swServer_reload(swServer *serv);
```
| 参数    | 说明   | 
| --------  | ------  |
| swServer *serv|swServer对象|
函数核心源码：
```c
    // Server.c 1009-1017h
    int manager_pid = swServer_get_manager_pid(serv);
    if (manager_pid > 0)
    {
        return kill(manager_pid, SIGUSR1);
    }
    return SW_ERR;
```
源码解释：
通过kill函数向Manager进程发送SIGUSR1信号，该信号的行为是杀死并重启全部的Worker。

剩下的swServer的操作函数较为简单，在此不再贴出源码进行分析。

###**2.Server相关结构体分析**
下面分析一些声明在Server.h中的结构体。

####**swPackage**
该结构体已被swEventData替代。

####**swPackage_task**
声明：
```c
// Server.h 420-424h
typedef struct
{
    int length;
    char tmpfile[sizeof(SW_TASK_TMP_FILE)];
} swPackage_task;
```
| 成员    | 说明   | 
| --------  | ------  |
| int length|数据长度|
| char tmpfile[sizeof(SW_TASK_TMP_FILE)]|临时文件的文件名|
说明：
**swPackage_task**用于封装内容较大的task包（超过8K），tmpfile指向一个由**mkstemp**函数（如果开启了**HAVE_MKOSTEMP**选项，则为**mkostemp**函数）创造的临时文件，所有的数据会被暂时存放在这个文件里。

####**swPackage_response**
声明：
```c
// Server.h 426-430h
typedef struct
{
    int length;
    int worker_id;
} swPackage_response;
```
| 成员    | 说明   | 
| --------  | ------  |
| int length|数据长度|
| int worker_id|用于接收该应答的Worker ID|
说明：
swPackage_response结构体用于Factory回应Reactor，主要作用是通知Reactor响应数据的实际长度以及是由哪个worker处理的，然后会根据是否为Big Response决定是否从worker的共享内存中读取数据。（参考**ReactorThread**的**swReactorThread_onPipeReceive**函数以及**FactoryProcess**的**swFactoryProcess_finish**函数）
