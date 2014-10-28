# Swoole源码学习记录

---
**swoole版本：1.7.6-stable**

上一章已经分析了如何启动swServer的相关函数。本章将继续分析swServer的相关函数，

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








分析三个结构体swPackage，swPackage_task和swPackage_response。由于第一个结构体swPackage已经被swEventData代替，因此不贴出源码。剩下两个结构体声明在Server.h文件的420 - 430行，声明如下：
```c
typedef struct
{
    int length;
    char tmpfile[sizeof(SW_TASK_TMP_FILE)];
} swPackage_task;

typedef struct
{
    int length;
    int worker_id;
} swPackage_response;
```
其中，swPackage_task用于封装内容较大的task包（超过8K），tmpfile指向一个由mkstemp函数（如果开启了HAVE_MKOSTEMP选项，则为mkostemp函数）创造的临时文件，所有的数据会被暂时存放在这个文件里。

swPackage_response结构体用于Factory回应Reactor，主要作用是通知Reactor响应数据的实际长度以及是由哪个worker处理的，然后会根据是否为Big Response决定是否从worker的共享内存中读取数据。（参考**ReactorThread**的**swReactorThread_onPipeReceive**函数以及**FactoryProcess**的**swFactoryProcess_finish**函数）

