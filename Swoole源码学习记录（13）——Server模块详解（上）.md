
终于可以正式进入Server.c模块了……
在之前的分析中，可以看到很多相关模块的声明都已经写在了Server.h中，就是因为这些模块构成了Server的核心部分。而Server本身，则是一个最上层的对象，它包括了核心的Reactor和Factory模块，存放了消息队列的key值，控制着全部的Connection，所有PHP层面的回调函数也在这里指定；同时，Server存放了大量的属性值，这些值决定了整个Swoole的详细特征。

直接上源码。首先是swServer结构体，这个结构体定义了一个swoole_server对象所拥有的全部成员。其声明在Server.h文件的197 - 406 行。因为该结构体太长，因此我将分段贴出其成员并分析。
```c
    /**
     * tcp socket listen backlog
     */
    uint16_t backlog;
    /**
     * reactor thread/process num
     */
    uint16_t reactor_num;
    uint16_t writer_num;
    /**
     * worker process num
     */
    uint16_t worker_num;

    /**
     * The number of pipe per reactor maintenance
     */
    uint16_t reactor_pipe_num;

    uint8_t factory_mode;

    /**
     * run as a daemon process
     */
    uint8_t daemonize;

    /**
     * package dispatch mode
     */
    uint8_t dispatch_mode; //分配模式，1平均分配，2按FD取摸固定分配，3,使用抢占式队列(IPC消息队列)分配

    /**
     * 1: unix socket, 2: message queue, 3: memory channel
     */
    uint8_t ipc_mode;
```
backlog参数为socket监听时listen函数的参数，表示listen可以保持的最大连通数。reactor_num指定了reactor进程的数量，worker_num指定了worker进程的数量。reactor_pipe_num指定了每个reactor所拥有的管道数量。factory_mode指定了Swoole_server运行于哪种运行模式下（Base模式、单进程单线程、多进程、多线程）。daemonize指定是否作为守护进程运行（也就是后台运行）。dispatch_mode指定了收到消息时worker进程的分配模式，ipc_mode则指定了进程间的通信模式。

```c
    int worker_uid;
    int worker_groupid;

    /**
     * max connection num
     */
    uint32_t max_connection;

    /**
     * worker process max request
     */
    uint32_t max_request;
    /**
     * task worker process max request
     */
    uint32_t task_max_request;
    
    int timeout_sec;
    int timeout_usec;

    int sock_client_buffer_size; //client的socket缓存区设置
    int sock_server_buffer_size; //server的socket缓存区设置

    char log_file[SW_LOG_FILENAME]; //日志文件

    int signal_fd;
    int event_fd;

    int ringbuffer_size;
```
worker_uid和worker_groupid是两个无用变量，估计是历史遗留……max_connection指定允许的最大连接数，max_request指定了每个worker进程能处理的最大请求数，task_max_request指定了每个task worker进程能处理的最大任务请求。timeout_sec和timeout_usec定义了超时时间。signal_fd和event_fd同样是废弃变量。ringbuffer_size指定了RingBuffer的大小。
```c
    uint16_t reactor_round_i; //轮询调度
    uint16_t reactor_next_i; //平均算法调度
    uint16_t reactor_schedule_count;

    uint16_t worker_round_id;

    int udp_sock_buffer_size; //UDP临时包数量，超过数量未处理将会被丢弃

    /**
     * reactor ringbuffer memory pool size
     */
    size_t reactor_ringbuffer_size;

    /**
     * have udp listen socket
     */
    uint8_t have_udp_sock;

    /**
     * have tcp listen socket
     */
    uint8_t have_tcp_sock;

    /**
     * oepn cpu affinity setting
     */
    uint8_t open_cpu_affinity;
```
前三个属性用于Reactor的调度算法，而实际上reactor只使用了轮询调度，因此后两个属性都没有用到……worker_round_id用于worker轮询调度，reactor_ringbuffer_size指定了reactor中的RingBuffer内存池的大小。have_udp_sock和have_tcp_sock用于标记当前Server是否有udp和tcp的监听。open_cpu_affinity标记是否打开了cpu亲和性设置。

```c
    /**
     * open tcp_defer_accept option
     */
    uint8_t tcp_defer_accept; //TCP_DEFER_ACCEPT

    /* tcp keepalive */
    uint8_t open_tcp_keepalive; //开启keepalive
    uint16_t tcp_keepidle; //如该连接在规定时间内没有任何数据往来,则进行探测
    uint16_t tcp_keepinterval; //探测时发包的时间间隔
    uint16_t tcp_keepcount; //探测尝试的次数

    /* heartbeat check time*/
    uint16_t heartbeat_idle_time; //心跳存活时间
    uint16_t heartbeat_check_interval; //心跳定时侦测时间, 必需小于heartbeat_idle_time

    /**
     * 来自客户端的心跳侦测包
     */
    char heartbeat_ping[SW_HEARTBEAT_PING_LEN];
    uint8_t heartbeat_ping_length;

    /**
     * 服务器端对心跳包的响应
     */
    char heartbeat_pong[SW_HEARTBEAT_PING_LEN];
    uint8_t heartbeat_pong_length;
```
tcp_defer_accept指定是否打开TCP_DEFER_ACCEPT选项，open_tcp_keepalive用于指定是否打开TCP_KEEPALIVE探测，接下来三个属性则用于设置对应的socket选项，heartbeat_idle_time和heartbeat_check_interval用于控制心跳检测。
这里做一些简单说明。TCP_DEFER_ACCEPT选项的意义在于，如果某个客户端发起连接后并没有发送数据，则服务器会先暂时挂起这个连接，不Accept，不建立IO通道，但是保留端口号；当客户端发送数据过来后，服务器会直接Accept这个端口号，建立IO。
tcp_keepalive是tcp内部的一个连接保持机制，而heartbeat是应用层控制的连接检测。

```c
    /* one package: eof check */
    uint8_t open_eof_check; //检测数据EOF
    uint8_t package_eof_len; //数据缓存结束符长度
    //int data_buffer_max_num;             //数据缓存最大个数(超过此数值的连接会被当作坏连接，将清除缓存&关闭连接)
    //uint8_t max_trunk_num;               //每个请求最大允许创建的trunk数
    char package_eof[SW_DATA_EOF_MAXLEN]; //数据缓存结束符

    /**
     * built-in http protocol
     */
    uint8_t open_http_protocol;
    uint32_t http_max_post_size;
    uint32_t http_max_websocket_size;

    /* one package: length check */
    uint8_t open_length_check; //开启协议长度检测

    char package_length_type; //length field type
    uint8_t package_length_size;
    uint16_t package_length_offset; //第几个字节开始表示长度
    uint16_t package_body_offset; //第几个字节开始计算长度
    uint32_t package_max_length;
```
这一组变量控制着swoole不同的分包机制。第一组为eof检测，第二组为内建的http协议，第三组为包长检测。这里都有中文注释，我就不再赘述。

```c
    /**
     * Use data key as factory->dispatch() param
     */
    uint8_t open_dispatch_key;
    uint8_t dispatch_key_size;
    uint16_t dispatch_key_offset;
    uint16_t dispatch_key_type;

    /* buffer output/input setting*/
    uint32_t buffer_output_size;
    uint32_t buffer_input_size;

#ifdef SW_USE_OPENSSL
    uint8_t open_ssl;
    char *ssl_cert_file;
    char *ssl_key_file;
#endif
```
open_dispatch_key指定是否在dispatch时给数据加上key值，后面三个属性用于控制key的相关特性。buffer_output_size和buffer_input_size指定了输入输出缓存区的大小。最后一组参数用于设置SSL特性，指定了对应的私钥和证书。

```c
    void *ptr2;

    swReactor reactor;
    swFactory factory;

    swListenList_node *listen_list;

    swReactorThread *reactor_threads;
    swWorkerThread *writer_threads;

    swWorker *workers;

    swQueue read_queue;
    swQueue write_queue;

    swConnection *connection_list; //连接列表
    int connection_list_capacity; //超过此容量，会自动扩容

    /**
     * message queue key
     */
    uint64_t message_queue_key;

    swReactor *reactor_ptr; //Main Reactor
    swFactory *factory_ptr; //Factory
```
第一个ptr2变量存放了一个zval指针，存放的是一个server在PHP中的this指针（个人理解和猜测，未验证）。接下来的几个属性我想大家应该都不陌生，在之前的分析中也多次提到了这些参数。需要说明的是，最后的两个指针是遗留变量，没有实际用处。

最后还有N个回调函数指针，我就不贴代码了，我想大家都看得懂他们是干嘛的……

然后swServer大概有……17个相关操作函数，这些函数声明在Server.h文件的435 - 449行，如下：
```c
int swServer_onFinish(swFactory *factory, swSendData *resp);
int swServer_onFinish2(swFactory *factory, swSendData *resp);

void swServer_init(swServer *serv);
void swServer_signal_init(void);
int swServer_start(swServer *serv);
int swServer_addListener(swServer *serv, int type, char *host,int port);
int swServer_create(swServer *serv);
int swServer_listen(swServer *serv, swReactor *reactor);
int swServer_free(swServer *serv);
int swServer_shutdown(swServer *serv);
int swServer_addTimer(swServer *serv, int interval);
int swServer_reload(swServer *serv);
int swServer_udp_send(swServer *serv, swSendData *resp);
int swServer_tcp_send(swServer *serv, int fd, void *data, int length);
int swServer_reactor_add(swServer *serv, int fd, int sock_type); //no use
int swServer_reactor_del(swServer *serv, int fd, int reacot_id); //no use
int swServer_get_manager_pid(swServer *serv);
```
然后我尽量按照调用的先后顺序来分析这些函数

首先是swServer_init函数，这个函数主要是设置swServer的相关属性，并通过swoole_init函数设置相关的全局变量，基本都是基础的赋值。这里需要重点分析**swServerG**结构体，该结构体声明在swoole.h的1052 - 1104行，其声明如下：
```c
typedef struct
{
    swTimer timer; // 全局定时器

    int running; // 是否正在运行
    int error; // 错误码errno
    int process_type; // 标记当前进程的类型（manager or worker）
    int signal_alarm; //for timer with message queue
    int signal_fd; // 没有找到赋值语句，应该是已经废弃
    int log_fd; // 日志文件的描述符

    uint8_t use_timerfd; // 是否使用Linux提供的timerfd功能
    uint8_t use_signalfd; // 是否使用signalfd功能
    /**
     * Timer used pipe
     */
    uint8_t use_timer_pipe; // timer是否使用管道
    uint8_t task_ipc_mode; // timer的通知模式

    /**
     *  task worker process num
     */
    uint16_t task_worker_num; // task worker进程的数量

    uint16_t cpu_num; // cpu的核数

    uint32_t pagesize; // 当前进程的分页大小
    uint32_t max_sockets; // 允许的最大socket fd

    /**
     * Unix socket default buffer size
     */
    uint32_t unixsock_buffer_size;

    swServer *serv; // 指向swServer
    swFactory *factory; // 指向swServer里的factory
    swLock lock;

    swProcessPool task_workers; // task_workers的进程池
    swProcessPool *event_workers;// 用于BASE模式

    swMemoryPool *memory_pool; // 内存池
    swReactor *main_reactor; // 全局主reactor，在主进程中只用于接收TCP连接，在worker进程中还需要额外处理管道的事件

    /**
     * for swoole_server->taskwait
     */
    swPipe *task_notify;
    swEventData *task_result;

    pthread_t heartbeat_pidt; // 心跳线程

} swServerG;
```
该结构体用于存放本地全局变量，这些变量会在任何地方被使用和访问。我在每个属性后加上了注释，标明了每个属性的功能。

接下来是swServer_create函数，该函数的实际功能其实是……根据factory_mode创建对应的Factory。该函数定义在Server.c的671 - 678行，如下：
```c
    if (serv->package_eof_len > sizeof(serv->package_eof))
    {
        serv->package_eof_len = sizeof(serv->package_eof);
    }

    //初始化日志
    if (serv->log_file[0] != 0)
    {
        swLog_init(serv->log_file);
    }

    //保存指针到全局变量中去
    //TODO 未来全部使用此方式访问swServer/swFactory对象
    SwooleG.serv = serv;
    SwooleG.factory = &serv->factory;

    //单进程单线程模式
    if (serv->factory_mode == SW_MODE_SINGLE)
    {
        return swReactorProcess_create(serv);
    }
    else
    {
        return swReactorThread_create(serv);
    }
```
源码解释：如果eof的长度超过了最大长度（8字节），则设置长度为8字节。随后初始化日志，并设置全局变量SwooleG中的serv和factory，接着根据factory_mode调用具体的函数。（swReactorThread_create参考12章）

接着是swServer_start函数，该函数用于启动一个swServer。其定义在Server.c文件的456 - 620行，因为函数较长，再此做分段分析。

```c
    swFactory *factory = &serv->factory;
    int ret;

    ret = swServer_start_check(serv);
    if (ret < 0)
    {
        return SW_ERR;
    }

#if SW_WORKER_IPC_MODE == 2
    serv->ipc_mode = SW_IPC_MSGQUEUE;
#endif

    if (serv->message_queue_key == 0)
    {
        char path_buf[128];
        char *path_ptr = getcwd(path_buf, 128);
        serv->message_queue_key = ftok(path_ptr, 1) + getpid();
    }

    if (serv->ipc_mode == SW_IPC_MSGQUEUE)
    {
        SwooleG.use_timerfd = 0;
        SwooleG.use_signalfd = 0;
        SwooleG.use_timer_pipe = 0;
    }

#ifdef SW_USE_OPENSSL
    if (serv->open_ssl)
    {
        if (swSSL_init(serv->ssl_cert_file, serv->ssl_key_file) < 0)
        {
            return SW_ERR;
        }
    }
#endif
```
源码解释：首先通过**swServer_start_check**函数检测相关属性是否设置正确。随后设置ipc_mode，如果消息队列key值为0，则生成一个key。如果使用了消息队列，则关闭timerfd，signalfd和timer管道。如果开启了SSL选项，则调用**swSSL_init**初始化SSL设置。

```c
    //run as daemon
    if (serv->daemonize > 0)
    {
        /**
         * redirect STDOUT to log file
         */
        if (SwooleG.log_fd > STDOUT_FILENO)
        {
            if (dup2(SwooleG.log_fd, STDOUT_FILENO) < 0)
            {
                swWarn("dup2() failed. Error: %s[%d]", strerror(errno), errno);
            }
        }
        /**
         * redirect STDOUT_FILENO/STDERR_FILENO to /dev/null
         */
        else
        {
            int null_fd = open("/dev/null", O_WRONLY);
            if (null_fd > 0)
            {
                if (dup2(null_fd, STDOUT_FILENO) < 0)
                {
                    swWarn("dup2(STDOUT_FILENO) failed. Error: %s[%d]", strerror(errno), errno);
                }
                if (dup2(null_fd, STDERR_FILENO) < 0)
                {
                    swWarn("dup2(STDERR_FILENO) failed. Error: %s[%d]", strerror(errno), errno);
                }
            }
            else
            {
                swWarn("open(/dev/null) failed. Error: %s[%d]", strerror(errno), errno);
            }
        }

        if (daemon(0, 1) < 0)
        {
            return SW_ERR;
        }
    }
```
源码解释：如果设置为守护进程，如果设置了日志文件，则将输出定向到日志文件中，否则将输入定向到/dev/null中（抹去输出）

```c
    //master pid
    SwooleGS->master_pid = getpid();
    SwooleGS->start = 1;
    SwooleGS->now = SwooleStats->start_time = time(NULL);

    serv->reactor_pipe_num = serv->worker_num / serv->reactor_num;

    //设置factory回调函数
    serv->factory.ptr = serv;
    serv->factory.onTask = serv->onReceive;

    if (serv->have_udp_sock == 1 && serv->factory_mode != SW_MODE_PROCESS)
    {
        serv->factory.onFinish = swServer_onFinish2;
    }
    else
    {
        serv->factory.onFinish = swServer_onFinish;
    }

    serv->workers = SwooleG.memory_pool->alloc(SwooleG.memory_pool, serv->worker_num * sizeof(swWorker));
    if (serv->workers == NULL)
    {
        swWarn("[Master] malloc[object->workers] failed");
        return SW_ERR;
    }

    /*
     * For swoole_server->taskwait, create notify pipe and result shared memory.
     */
    if (SwooleG.task_worker_num > 0 && serv->worker_num > 0)
    {
        int i;
        SwooleG.task_result = sw_shm_calloc(serv->worker_num, sizeof(swEventData));
        SwooleG.task_notify = sw_calloc(serv->worker_num, sizeof(swPipe));
        for (i = 0; i < serv->worker_num; i++)
        {
            if (swPipeNotify_auto(&SwooleG.task_notify[i], 1, 0))
            {
                return SW_ERR;
            }
        }
    }
```
源码解释：设置master进程id为当前进程id，设置start标记并储存启动时间。接下来设置reactor的管道数量，设置factory的onTask回调。如果有UDP监听并且没有使用多进程模式，则设置factory的onFinish回调为swServer_onFinish2回调（当前的swoole模式下基本没用）；否则，使用swServer_onFinish回调。接着，在全局内存池中分配worker结构体数组所需要的内存。最后，如果task_worker_num大于0，则在全局变量中为task_result分配共享内存，并创建task_notify管道数组。

```c
    //factory start
    if (factory->start(factory) < 0)
    {
        return SW_ERR;
    }
    //Signal Init
    swServer_signal_init();

    //标识为主进程
    SwooleG.process_type = SW_PROCESS_MASTER;

    //启动心跳检测
    if (serv->heartbeat_check_interval >= 1 && serv->heartbeat_check_interval <= serv->heartbeat_idle_time)
    {
        swTrace("hb timer start, time: %d live time:%d", serv->heartbeat_check_interval, serv->heartbeat_idle_time);
        swServer_heartbeat_start(serv);
    }

    if (serv->factory_mode == SW_MODE_SINGLE)
    {
        ret = swReactorProcess_start(serv);
    }
    else
    {
        ret = swServer_start_proxy(serv);
    }

    if (ret < 0)
    {
        SwooleGS->start = 0;
    }

    //server stop
    if (serv->onShutdown != NULL)
    {
        serv->onShutdown(serv);
    }
    swServer_free(serv);
    return SW_OK;
```
源码解释：调用factory的start函数启动factory，调用**swServer_signal_init**函数初始化信号回调，并设置当前进程类型为master类型。如果设置了心跳检测，则通过**swServer_heartbeat_start**函数启动心跳检测。如果使用单进程模式，则直接调用**swReactorProcess_start**函数启动。否则，调用**swServer_start_proxy**函数启动Server。Server关闭后，调用onShutdown回调，并释放内存。

那么这里需要分析swServer_start_proxy函数。该函数定义了一个proxy模式，在单独的n个线程（进程）中接受维持TCP连接。该函数定义在Server.c文件的405 - 454行，如下：
```c
int ret;
    swReactor *main_reactor = SwooleG.memory_pool->alloc(SwooleG.memory_pool, sizeof(swReactor));

#ifdef SW_MAINREACTOR_USE_POLL
    ret = swReactorPoll_create(main_reactor, 10);
#else
    ret = swReactorSelect_create(main_reactor);
#endif

    if (ret < 0)
    {
        swWarn("Reactor create failed");
        return SW_ERR;
    }
    ret = swReactorThread_start(serv, main_reactor);
    if (ret < 0)
    {
        swWarn("ReactorThread start failed");
        return SW_ERR;
    }
    SwooleG.main_reactor = main_reactor;
    main_reactor->id = serv->reactor_num; //设为一个特别的ID
    main_reactor->ptr = serv;
    main_reactor->setHandle(main_reactor, SW_FD_LISTEN, swServer_master_onAccept);

    main_reactor->onFinish = swServer_master_onReactorFinish;
    main_reactor->onTimeout = swServer_master_onReactorTimeout;

#ifdef HAVE_SIGNALFD
    if (SwooleG.use_signalfd)
    {
        swSignalfd_setup(main_reactor);
    }
#endif
    //SW_START_SLEEP;
    if (serv->onStart != NULL)
    {
        serv->onStart(serv);
    }
    struct timeval tmo;
    tmo.tv_sec = SW_MAINREACTOR_TIMEO;
    tmo.tv_usec = 0;

    //先更新一次时间
    swServer_update_time();

    return main_reactor->wait(main_reactor, &tmo);
```
源码解释：在全局内存池中创建一个主reactor，调用swReactorThread_start函数启动这个reactor，并设置全局变量SwooleG中的main_reactor为该reactor。接着设置reactor的相关属性和回调函数，设置该reactor监听LISTEN事件并设置回调为**swServer_master_onAccept**。如果serv设置了onStart回调，调用它。随后，调用swServer_update_time函数更新当前时间，然后直接进入reactor的**wait**函数开始监听连接。

至此，一个完整的swServer启动流程的相关函数以及分析完成。下一章将分析余下的函数以及相关的回调函数。


