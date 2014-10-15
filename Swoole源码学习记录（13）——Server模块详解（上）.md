
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


