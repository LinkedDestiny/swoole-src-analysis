Swoole源码学习记录
===================
-------------
##Swoole版本：1.7.4-stable
-------------
本章将分析FactoryProcess.c中剩下的函数，这些函数用于操作worker、manager以及writer。这些函数提供了最核心的进程创建、管理等功能，是Swoole的master-worker结构的基石。

先从worker相关的函数开始（manager相关函数基本都涉及操作worker进程）。在FactoryProcess.c中一共声明了4个操作函数，分别是：
```c
static int swFactoryProcess_worker_loop(swFactory *factory, int worker_pti);
static int swFactoryProcess_worker_spawn(swFactory *factory, int worker_pti);
static sw_inline uint32_t swServer_worker_schedule(swServer *serv, uint32_t schedule_key);
static sw_inline int swFactoryProcess_worker_excute(swFactory *factory, swEventData *task);
```
先分析两个内联函数。swServer_worker_schedule用于根据不同的调度模式查找对应的需要执行任务的worker_id。其核心源码如下：
```c
    if (serv->dispatch_mode == SW_DISPATCH_ROUND)
    {
        target_worker_id = (serv->worker_round_id++) % serv->worker_num;
    }
    //Using the FD touch access to hash
    else if (serv->dispatch_mode == SW_DISPATCH_FDMOD)
    {
        target_worker_id = schedule_key % serv->worker_num;
    }
    //Preemptive distribution
    else
    {
        if (serv->ipc_mode == SW_IPC_MSGQUEUE)
        {
            //msgsnd参数必须>0
            //worker进程中正确的mtype应该是pti + 1
            target_worker_id = serv->worker_num;
        }
        else
        {
            int i;
            sw_atomic_t *round = &SwooleTG.worker_round_i;
            for (i = 0; i < serv->worker_num; i++)
            {
                sw_atomic_fetch_add(round, 1);
                target_worker_id = (*round) % serv->worker_num;

                if (serv->workers[target_worker_id].status == SW_WORKER_IDLE)
                {
                    break;
                }
            }
            swTrace("schedule=%d|round=%d\n", target_worker_id, *round);
        }
    }
```
源码解释：根据swServer的dispatch_mode（调度模式）决定具体的选取方法。SW_DISPATCH_ROUND为轮询模式，SW_DISPATCH_FDMOD为根据FD取模（同一个fd一定会被同一个worker进程处理），否则就是抢占模式（从第一个worker开始查找，找到第一个空闲的worker就占用该worker）。另外，在抢占模式下，如果使用了消息队列，则不需要分配worker_id,每个worker会从消息队列中获取自己需要处理的数据。

swFactoryProcess_worker_excute函数用于处理一个worker接收到的swEventData* task，根据task的type不同执行不同的逻辑。核心代码如下：
```c
    swServer *serv = factory->ptr;
    swString *package = NULL;

    factory->last_from_id = task->info.from_id;
    //worker busy
    serv->workers[SwooleWG.id].status = SW_WORKER_BUSY;

    switch(task->info.type)
    {
    //no buffer
    case SW_EVENT_TCP:
    case SW_EVENT_UDP:
    case SW_EVENT_UNIX_DGRAM:

    //ringbuffer shm package
    case SW_EVENT_PACKAGE:
        onTask:
        factory->onTask(factory, task);

        if (!SwooleWG.run_always)
        {
            //only onTask increase the count
            worker_task_num --;
        }

        if (task->info.type == SW_EVENT_PACKAGE_END)
        {
            package->length = 0;
        }
        break;

    //package trunk
    case SW_EVENT_PACKAGE_START:
    case SW_EVENT_PACKAGE_END:
        //input buffer
        package = SwooleWG.buffer_input[task->info.from_id];
        //merge data to package buffer
        memcpy(package->str + package->length, task->data, task->info.len);
        package->length += task->info.len;
        //printf("package[%d]. from_id=%d|data_len=%d|total_length=%d\n", task->info.type, task->info.from_id, task->info.len, package->length);
        //package end
        if (task->info.type == SW_EVENT_PACKAGE_END)
        {
            goto onTask;
        }
        break;

    case SW_EVENT_CLOSE:
        serv->onClose(serv, task->info.fd, task->info.from_id);
        break;
    case SW_EVENT_CONNECT:
        serv->onConnect(serv, task->info.fd, task->info.from_id);
        break;
    case SW_EVENT_FINISH:
        serv->onFinish(serv, task);
        break;
    default:
        swWarn("[Worker] error event[type=%d]", (int)task->info.type);
        break;
    }
    //worker idle
    serv->workers[SwooleWG.id].status = SW_WORKER_IDLE;
```
源码解释：设置factory的最近一次接收的reactor的id为task的from_id。设置worker的状态值为BUSY状态。根据task的type类型决定具体的调用类型。这里的调用逻辑稍微复杂点：
1.  SW_EVENT_TCP、SW_EVENT_UDP、SW_EVENT_UNIX_DGRAM这三种类型没有缓存区，因此直接调用onTask投递任务，如果没有设置run_always（一直运行），则将剩余可处理的worker_task_num减一；
2.  SW_EVENT_PACKAGE是RingBuffer共享内存池的包，因为该package是完整的不需要拼接，因此同样直接onTask发送；
3.  SW_EVENT_PACKAGE_START 获取reactor对应的缓存，将task的data放入缓存区中；
4.  SW_EVENT_PACKAGE_END 所有数据发送完毕，跳到onTask，通过onTask投递任务，并将缓存区置空。（这里存在一个疑问，缓存区中的数据并没有通过onTask投递，也没有发现将缓存区数据存入task的行为，待解决）
5.  SW_EVENT_CLOSE、SW_EVENT_CONNECT、SW_EVENT_FINISH分别调用对用的PHP回调函数。
执行完成后，设置worker状态为IDLE。

接下来是swFactoryProcess_worker_spawn，该函数的功能是重启一个worker进程，函数核心源码如下：
```c
    pid = fork();
    if (pid < 0)
    {
        swWarn("Fork Worker failed. Error: %s [%d]", strerror(errno), errno);
        return SW_ERR;
    }
    //worker child processor
    else if (pid == 0)
    {
        ret = swFactoryProcess_worker_loop(factory, worker_pti);
        exit(ret);
    }
    //parent,add to writer
    else
    {
        return pid;
    }
```
源码解释：调用fork创建worker进程，在进程内调用swFactoryProcess_worker_loop进入worker的工作循环，循环结束后推出进程。在父进程中返回worker进程的pid。

然后就是重要的swFactoryProcess_worker_loop函数，该函数定义了一个worker进程的主要工作循环，由于该函数较长，所以仍然是分段分析。下面上源码：
```c
    swServer *serv = factory->ptr;
    struct
    {
        long pti;
        swEventData req;
    } rdata;
    int n;
    int pipe_rd = serv->workers[worker_pti].pipe_worker;
```
源码分析：获取swServer对象，构建rdata结构体（该结构体的成员和swDispatchData结构体完全一样）用于存放对应的worker_id和需要处理的数据。同时获取对应worker的读管道。
```c
#ifdef HAVE_CPU_AFFINITY
    if (serv->open_cpu_affinity == 1)
    {
        cpu_set_t cpu_set;
        CPU_ZERO(&cpu_set);
        CPU_SET(worker_pti % SW_CPU_NUM, &cpu_set);
        if (0 != sched_setaffinity(getpid(), sizeof(cpu_set), &cpu_set))
        {
            swWarn("pthread_setaffinity_np set failed");
        }
    }
#endif
```
源码解释：如果定义了HAVE_CPU_AFFINITY宏且swServer中指定打开了cpu affinity setting（CPU亲和性设置，英文解释为always run this process on processor one，大概翻译一下就是每个进程只能在某个指定的处理器上运行（针对多核CPU）），则先通过CPU_SET将worker_id对应的CPU加入到CPU集合中，然后通过sched_setaffinity函数指定当前worker进程运行在这个CPU上。（这里就能理解为何Swoole要求将worker_num设置为CPU的核的数目）
```c
    //signal init
    swWorker_signal_init();

    //worker_id
    SwooleWG.id = worker_pti;
#ifndef SW_USE_RINGBUFFER
    int i;
    //for open_check_eof and  open_check_length
    if (serv->open_eof_check || serv->open_length_check || serv->open_http_protocol)
    {
        SwooleWG.buffer_input = sw_malloc(sizeof(swString*) * serv->reactor_num);
        if (SwooleWG.buffer_input == NULL)
        {
            swError("malloc for SwooleWG.buffer_input failed.");
            return SW_ERR;
        }
        for (i = 0; i < serv->reactor_num; i++)
        {
            SwooleWG.buffer_input[i] = swString_new(serv->buffer_input_size);
            if (SwooleWG.buffer_input[i] == NULL)
            {
                swError("buffer_input init failed.");
                return SW_ERR;
            }
        }
    }
#endif
```
源码解释：调用swWorker_signal_init指定对应信号的处理函数，并设置SwooleWG全局变量中的id为当前worker_id。接下来这段就是处理Swoole提供的自动分包功能，如果使用RingBuffer，并且设置了eof检测、长度检测、http协议中的任意一种 ，则都需要设置worker的缓存区。首先给缓存区buffer_input分配空间，开辟一个长度为reactor_num的、存放swString指针的数组，然后遍历数组，为数组中的每一位创建一个swString对象，swString的长度为指定的缓存区长度（buffer_input_size，对应config中的package_max_length）。
```c
    if (serv->ipc_mode == SW_IPC_MSGQUEUE)
    {
        //抢占式,使用相同的队列type
        if (serv->dispatch_mode == SW_DISPATCH_QUEUE)
        {
            //这里必须加1
            rdata.pti = serv->worker_num + 1;
        }
        else
        {
            //必须加1
            rdata.pti = worker_pti + 1;
        }
    }
```
源码解释：如果使用了消息队列，在抢占模式下，使用相同的队列type，其他模式下，指定rdata的进程id为worker_id + 1（我没理解为何要必须加1，后面发现了会过来补充）。
```c
    else
    {
        SwooleG.main_reactor = sw_malloc(sizeof(swReactor));
        if (SwooleG.main_reactor == NULL)
        {
            swError("[Worker] malloc for reactor failed.");
            return SW_ERR;
        }
        if (swReactor_auto(SwooleG.main_reactor, SW_REACTOR_MAXEVENTS) < 0)
        {
            swError("[Worker] create worker_reactor failed.");
            return SW_ERR;
        }
        swSetNonBlock(pipe_rd);
        SwooleG.main_reactor->ptr = serv;
        SwooleG.main_reactor->add(SwooleG.main_reactor, pipe_rd, SW_FD_PIPE);
        SwooleG.main_reactor->setHandle(SwooleG.main_reactor, SW_FD_PIPE, swFactoryProcess_worker_onPipeReceive);

#ifdef HAVE_SIGNALFD
        if (SwooleG.use_signalfd)
        {
            swSignalfd_setup(SwooleG.main_reactor);
        }
#endif
    }
```
源码解释：如果没有使用消息队列，初始化并调用swReactor_auto函数创建主要的reactor，设置读管道pipe_rd为非阻塞模式，并将该管道加入reactor中监听，并设置回调函数为swFactoryProcess_worker_onPipeReceive（从名字就能看出这是干嘛的……）。最后一段，如果设置了HAVE_SIGNALFD宏并且开启了use_signalfd，则调用swSignalfd_setup函数设置main_reactor。（对于这段的分析将在对Swoole的signal.c文件分析时给出）
```c
    if (serv->max_request < 1)
    {
        SwooleWG.run_always = 1;
    }
    else
    {
        worker_task_num = serv->max_request;
        worker_task_num += swRandom(worker_pti);
    }
```
源码解释：如果没有设置max_request（每个worker允许处理的最大请求数）,则设置run_always为true，否则，设置worker_task_num数量为max_request，然后加上一个随机数（个位数的随机数 * worker_id）
```c
    //worker start
    swServer_worker_onStart(serv);

    if (serv->ipc_mode == SW_IPC_MSGQUEUE)
    {
        while (SwooleG.running > 0)
        {
            n = serv->read_queue.out(&serv->read_queue, (swQueue_data *) &rdata, sizeof(rdata.req));
            if (n < 0)
            {
                if (errno == EINTR)
                {
                    if (SwooleG.signal_alarm)
                    {
                        swTimer_select(&SwooleG.timer);
                    }
                }
                else
                {
                    swWarn("[Worker%ld] read_queue->out() failed. Error: %s [%d]", rdata.pti, strerror(errno), errno);
                }
                continue;
            }
            swFactoryProcess_worker_excute(factory, &rdata.req);
        }
    }
    else
    {
        struct timeval timeo;
        timeo.tv_sec = SW_REACTOR_TIMEO_SEC;
        timeo.tv_usec = SW_REACTOR_TIMEO_USEC;
        SwooleG.main_reactor->wait(SwooleG.main_reactor, &timeo);
    }

    //worker shutdown
    swServer_worker_onStop(serv);
```
源码解释：首先调用swServer_worker_onStart回调（这里对应onWorkerStart回调，通知PHP程序）。如果是消息队列模式，则进入循环，每次从消息队列中取出一条消息，调用swFactoryProcess_worker_excute处理该消息；如果不是消息队列模式，则调用main_reactor的wait方法进入事件监听（详情请看第八章Reactor模块）。循环结束后，调用swServer_worker_onStop回调通知PHP程序。

swFactoryProcess_worker_onPipeReceive函数的核心源码如下：
```c
    if (read(event->fd, &task, sizeof(task)) > 0)
    {
        /**
         * Big package
         */
        ret = swFactoryProcess_worker_excute(factory, &task);
        if (task.info.type == SW_EVENT_PACKAGE_START)
        {
            //no data
            if (ret < 0 && errno == EAGAIN)
            {
                return SW_OK;
            }
            else if (ret > 0)
            {
                goto read_from_pipe;
            }
        }
        return ret;
    }
```
源码解释：从管道中读出一个task数据，然后调用swFactoryProcess_worker_excute函数执行任务。（这里的task来源就是reactor的监听）

这里可以整理出一个完整的worker流程：swFactoryProcess_worker_spawn创建worker进程，随后通过swFactoryProcess_worker_loop进入worker的主循环；当有任务来临时，通过swServer_worker_schedule获取对应的worker，调用swFactoryProcess_worker_excute执行任务。

FactoryProcess中共声明了两个manager操作函数，如下：
```c
static int swFactoryProcess_manager_loop(swFactory *factory);
static int swFactoryProcess_manager_start(swFactory *factory);
```
swFactoryProcess_manager_start函数用于启动一个manager进程，该manager进程用于创建和管理每个worker进程。其核心源码较长，分段分析：
```c
    if (serv->ipc_mode == SW_IPC_MSGQUEUE)
    {
        //读数据队列
        if (swQueueMsg_create(&serv->read_queue, 1, serv->message_queue_key, 1) < 0)
        {
            swError("[Master] swPipeMsg_create[In] fail. Error: %s [%d]", strerror(errno), errno);
            return SW_ERR;
        }
        //为TCP创建写队列
        if (serv->have_tcp_sock == 1)
        {
            //写数据队列
            if (swQueueMsg_create(&serv->write_queue, 1, serv->message_queue_key + 1, 1) < 0)
            {
                swError("[Master] swPipeMsg_create[out] fail. Error: %s [%d]", strerror(errno), errno);
                return SW_ERR;
            }
        }
    }
```
源码解释：消息队列模式下，首先创建读数据队列，如果swServer设置了TCP sock的监听，则需创建写队列。
```c
else
    {
        object->pipes = sw_calloc(serv->worker_num, sizeof(swPipe));
        if (object->pipes == NULL)
        {
            swError("malloc[worker_pipes] failed. Error: %s [%d]", strerror(errno), errno);
            return SW_ERR;
        }
        //worker进程的pipes
        for (i = 0; i < serv->worker_num; i++)
        {
            if (swPipeUnsock_create(&object->pipes[i], 1, SOCK_DGRAM) < 0)
            {
                return SW_ERR;
            }
            serv->workers[i].pipe_master = object->pipes[i].getFd(&object->pipes[i], 1);
            serv->workers[i].pipe_worker = object->pipes[i].getFd(&object->pipes[i], 0);
        }
    }
```
源码解释：非消息队列模式下，创建通讯管道（使用unix sock管道，以此保证管道是双向可用的）
```c
    if (SwooleG.task_worker_num > 0)
    {
        key_t msgqueue_key = 0;
        if (SwooleG.task_ipc_mode > 0)
        {
            msgqueue_key =  serv->message_queue_key + 2;
        }

        if (swProcessPool_create(&SwooleG.task_workers, SwooleG.task_worker_num, serv->task_max_request, msgqueue_key) < 0)
        {
            swWarn("[Master] create task_workers failed.");
            return SW_ERR;
        }

        swWorker *worker;
        for(i = 0; i < SwooleG.task_worker_num; i++)
        {
             worker = swServer_get_worker(serv, serv->worker_num + i);
             if (swWorker_create(worker) < 0)
             {
                 return SW_ERR;
             }
        }

        //设置指针和回调函数
        SwooleG.task_workers.ptr = serv;
        SwooleG.task_workers.onTask = swTaskWorker_onTask;
        SwooleG.task_workers.onWorkerStart = swTaskWorker_onWorkerStart;
        SwooleG.task_workers.onWorkerStop = swTaskWorker_onWorkerStop;
    }
```
源码解释：如果设置了task_worker_num，则创建一个PorcessPool进程池（这里将在对Swoole的进程池、线程池做分析时给出详解）用于管理多个task进程，随后循环创建指定个数的task进程并设置swServer指针和回调函数。
```c
    pid = fork();
    switch (pid)
    {
    //创建manager进程
    case 0:
        //创建子进程
        for (i = 0; i < serv->worker_num; i++)
        {
            //close(worker_pipes[i].pipes[0]);
            reactor_pti = (i % serv->writer_num);
            serv->workers[i].reactor_id = reactor_pti;
            pid = swFactoryProcess_worker_spawn(factory, i);
            if (pid < 0)
            {
                swError("Fork worker process fail");
                return SW_ERR;
            }
            else
            {
                serv->workers[i].pid = pid;
            }
        }
        /**
         * create task worker pool
         */
        if (SwooleG.task_worker_num > 0)
        {
            swProcessPool_start(&SwooleG.task_workers);
        }
        //标识为管理进程
        SwooleG.process_type = SW_PROCESS_MANAGER;
        ret = swFactoryProcess_manager_loop(factory);
        exit(ret);
        break;
        //主进程
    default:
        SwooleGS->manager_pid = pid;
        break;
    case -1:
        swError("fork() failed.");
        return SW_ERR;
    }
```
源码解释：调用fork函数创建manager进程，在manager进程中，调用swFactoryProcess_worker_spawn创建子进程，调用swProcessPool_start启动task进程池，标记该进程为管理进程，并调用swFactoryProcess_manager_loop进入管理进程核心工作循环。

接下来是swFactoryProcess_manager_loop函数，该函数定义了manager进程的主工作流程，其核心工作是监听worker的exit事件并创建新的worker进程。核心源码如下：
```c
    swServer *serv = factory->ptr;
    swWorker *reload_workers;

    swSignal_set(SIGTERM, swWorker_signal_handler, 1, 0);

    if (serv->onManagerStart)
    {
        serv->onManagerStart(serv);
    }

    reload_worker_num = serv->worker_num + SwooleG.task_worker_num;
    reload_workers = sw_calloc(reload_worker_num, sizeof(swWorker));
    if (reload_workers == NULL)
    {
        swError("[manager] malloc[reload_workers] failed");
        return SW_ERR;
    }

    //for reload
    swSignal_add(SIGUSR1, swManager_signal_handle);
```
设置Term终止事件的信号监听，如果设置了onManagerStart回调，调用它（onStart）。设置需要reload的worker数量为worker_num + task_worker_num.并分配reload_workers数组的空间用于临时存放worker结构体。同时设置针对reload信号的监听。这里的SIGUSR1信号为自定义信号，该信号的响应事件就是将manager_worker_reloading变量置为1，并将manager_reload_flag置0.
```c
    while (SwooleG.running > 0)
    {
        pid = wait(&worker_exit_code);

        if (pid < 0)
        {
            if (manager_worker_reloading == 0)
            {
                swTrace("[Manager] wait failed. Error: %s [%d]", strerror(errno), errno);
            }
            else if (manager_reload_flag == 0)
            {
                memcpy(reload_workers, serv->workers, sizeof(swWorker) * serv->worker_num);
                if (SwooleG.task_worker_num > 0)
                {
                    memcpy(reload_workers + serv->worker_num, SwooleG.task_workers.workers,
                            sizeof(swWorker) * SwooleG.task_worker_num);
                }
                manager_reload_flag = 1;
                goto kill_worker;
            }
        }
        if (SwooleG.running == 1)
        {
            for (i = 0; i < serv->worker_num; i++)
            {
                //compare PID
                if (pid != serv->workers[i].pid)
                {
                    continue;
                }
                else
                {
                    if (serv->onWorkerError!=NULL && WEXITSTATUS(worker_exit_code) > 0)
                    {
                        serv->onWorkerError(serv, i, pid, WEXITSTATUS(worker_exit_code));
                    }
                    pid = 0;
                    while (1)
                    {
                        new_pid = swFactoryProcess_worker_spawn(factory, i);
                        if (new_pid < 0)
                        {
                            usleep(100000);
                            continue;
                        }
                        else
                        {
                            serv->workers[i].pid = new_pid;
                            break;
                        }
                    }
                }
            }

            //task worker
            if (pid > 0)
            {
                swWorker *exit_worker = swHashMap_find_int(SwooleG.task_workers.map, pid);
                if (exit_worker != NULL)
                {
                    swProcessPool_spawn(exit_worker);
                }
            }
        }
        //reload worker
        kill_worker: if (manager_worker_reloading == 1)
        {
            //reload finish
            if (reload_worker_i >= reload_worker_num)
            {
                manager_worker_reloading = 0;
                reload_worker_i = 0;
                continue;
            }
            ret = kill(reload_workers[reload_worker_i].pid, SIGTERM);
            if (ret < 0)
            {
                swWarn("[Manager]kill() failed, pid=%d. Error: %s [%d]", reload_workers[reload_worker_i].pid, strerror(errno), errno);
                continue;
            }
            reload_worker_i++;
        }
    }
```
源码解释：进入manager的核心工作循环。调用wait等待worker_exit事件的发生。wait函数会返回exit的worker进程的pid。
如果pid小于0，说明wait函数出错，如果manager_worker_reloading标记为0，说明当前并没有调用reload函数，则报错。如果manager_worker_reloading为1，且manager_reload_flag为0，则将所有worker进程的结构体copy进reload_workers中，，设置manager_reload_flag为1（代表正在重启进程），随后goto到kill_worker标签处；

如果pid大于0且swServer正在运行，首先遍历worker数组找到pid对应的worker进程，如果设置了onWorkerError回调并且worker是异常退出的，就调用该回调。随后，循环调用swFactoryProcess_worker_spawn函数创建新的worker进程直到创建成功，设置worker pid为新的pid。如果是task进程，则先通过swHashMap_find_int函数从SwooleG全局变量的task_workers里的map中找到对应的task_worker结构体，随后调用swProcessPool_spawn函数重建该task进程。

**kill_worker**标签处，如果manager_worker_reloading为1，说明调用了reload函数，如果reload_worker_i索引大于或等于reload_worker_num,说明所有进程都重启了一遍，因此重置manager_worker_reloading标记和reload_worker_i索引，继续下一次循环；否则，调用kill函数杀死reload_worker_i索引指向的worker进程，将reload_worker_i后移一位。

这里处理了两种情况，一种是worker进程自己退出，这时仅重启退出的worker进程。一种是调用了reload函数，这时要重启全部的worker进程。

到此，FactoryProcess模块全部分析完毕。下一章将分析Swoole的Worker.c模块、Connection.c模块和ReactorProcess模块。

