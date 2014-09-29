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
其中thread_id为ReactorThread的id，udp_addrs和c_udp_fd专门用于处理udp请求，buffer_input为RingBuffer，用于开启了RingBuffer选项的处理，buffer_pipe用于存放来自管道的数据


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
源码解释：如果不是直传，则需要现将数据放入缓存。首先创建connection的out_buffer输出缓存，如果发送数据长度为0，则指定缓存的trunk类型为**SW_TRUNK_CLOSE**（关闭连接），如果发送数据的类型为sendfile，则调用**swConnection_sendfile**函数，否则调用**swBuffer_append**函数将发送数据加入缓存中。最后，在reactor中设置fd为可写状态。

swReactorThread_send2worker函数在此不再贴源码分析。基本思路就是消息队列模式就扔队列不是消息队列模式就扔缓存或者直接扔管道……应该都看得懂了。

这里直接上数量最多的回调函数的分析。这几个回调式用于处理接收数据的，从名字上大家基本能看出，Swoole提供的一些特性比如包长检测、eof检测还有UDP的报文接收都是通过这些不同的回调来实现的。

首先来看**swReactorThread_onReceive_no_buffer**函数，这个是最基本的接收函数，没有缓存，没有检测，收到多少数据就发给worker多少数据。下面上核心源码：
```c
#ifdef SW_USE_EPOLLET
    n = swRead(event->fd, task.data.data, SW_BUFFER_SIZE);
#else
    //非ET模式会持续通知
    n = swConnection_recv(conn, task.data.data, SW_BUFFER_SIZE, 0);
#endif

    if (n < 0)
    {
        switch (swConnection_error(errno))
        {
        case SW_ERROR:
            swWarn("recv from connection[fd=%d] failed. Error: %s[%d]", event->fd, strerror(errno), errno);
            return SW_OK;
        case SW_CLOSE:
            goto close_fd;
        default:
            return SW_OK;
        }
    }
    //需要检测errno来区分是EAGAIN还是ECONNRESET
    else if (n == 0)
    {
        close_fd:
        swTrace("Close Event.FD=%d|From=%d|errno=%d", event->fd, event->from_id, errno);
        swServer_connection_close(serv, event->fd, 1);
        /**
         * skip EPOLLERR
         */
        event->fd = 0;
        return SW_OK;
    }
```
源码解释：如果使用epoll的ET模式，则调用**swRead**函数直接从fd中读取数据；否则，在非ET模式下，调用**swConnection_recv**函数接收数据。如果接收数据失败，则根据errno执行对应的操作，如果接收数据为0，需要关闭连接，调用**swServer_connection_close**函数关闭fd。
```c
        conn->last_time =  SwooleGS->now;

        //heartbeat ping package
        if (serv->heartbeat_ping_length == n)
        {
            if (serv->heartbeat_pong_length > 0)
            {
                send(event->fd, serv->heartbeat_pong, serv->heartbeat_pong_length, 0);
            }
            return SW_OK;
        }

        task.data.info.fd = event->fd;
        task.data.info.from_id = event->from_id;
        task.data.info.len = n;

#ifdef SW_USE_RINGBUFFER

        uint16_t target_worker_id = swServer_worker_schedule(serv, conn->fd);
        swPackage package;

        package.length = task.data.info.len;
        package.data = swReactorThread_alloc(&serv->reactor_threads[SwooleTG.id], package.length);
        task.data.info.type = SW_EVENT_PACKAGE;

        memcpy(package.data, task.data.data, task.data.info.len);
        task.data.info.len = sizeof(package);
        task.target_worker_id = target_worker_id;
        memcpy(task.data.data, &package, sizeof(package));

#else
        task.data.info.type = SW_EVENT_TCP;
        task.target_worker_id = -1;
#endif

        //dispatch to worker process
        ret = factory->dispatch(factory, &task);

#ifdef SW_USE_EPOLLET
        //缓存区还有数据没读完，继续读，EPOLL的ET模式
        if (sw_errno == EAGAIN)
        {
            swWarn("sw_errno == EAGAIN");
            ret = swReactorThread_onReceive_no_buffer(reactor, event);
        }
#endif
```
源码解释：首先更新最近收包的时间。随后，检测是否是心跳包，如果接收长度等于心跳包的长度并且指定了发送心跳回应，则发送心跳包并返回。如果不是心跳包，则设置接收数据的fd、reactor_id以及长度。如果指定使用了RingBuffer，则需要将数据封装到swPackage中然后放进ReactorThread的input_buffer中。随后调用factory的**dispatch**方法将数据投递到对应的worker中。最后，如果是LT模式，并且缓存区的数据还没读完，则继续调用**swReactorThread_onReceive_no_buffer**函数读取数据。

接下来是**swReactorThread_onReceive_buffer_check_length**函数，该函数用于接收开启了包长检测的数据包。包长检测是Swoole用于支持固定包头+包体的自定义协议的特性，当然有不少小伙伴不理解怎么使用这个特性……下面上核心源码：
```c
        //new package
        if (conn->object == NULL)
        {
            do_parse_package:
            do
            {
                package_total_length = swReactorThread_get_package_length(serv, (void *)tmp_ptr, (uint32_t) tmp_n);

                //Invalid package, close connection
                if (package_total_length < 0)
                {
                    goto close_fd;
                }
                //no package_length
                else if(package_total_length == 0)
                {
                    char recv_buf_again[SW_BUFFER_SIZE];
                    memcpy(recv_buf_again, (void *) tmp_ptr, (uint32_t) tmp_n);
                    do
                    {
                        //前tmp_n个字节存放不完整包头
                        n = recv(event->fd, (void *)recv_buf_again + tmp_n, SW_BUFFER_SIZE, 0);
                        try_count ++;

                        //连续5次尝试补齐包头,认定为恶意请求
                        if (try_count > 5)
                        {
                            swWarn("No package head. Close connection.");
                            goto close_fd;
                        }
                    }
                    while(n < 0 && errno == EINTR);

                    if (n == 0)
                    {
                        goto close_fd;
                    }
                    tmp_ptr = recv_buf_again;
                    tmp_n = tmp_n + n;

                    goto do_parse_package;
                }
                //complete package
                if (package_total_length <= tmp_n)
                {
                    tmp_package.size = package_total_length;
                    tmp_package.length = package_total_length;
                    tmp_package.str = (void *) tmp_ptr;

                    //swoole_dump_bin(buffer.str, 's', buffer.length);
                    swReactorThread_send_string_buffer(swServer_get_thread(serv, SwooleTG.id), conn, &tmp_package);

                    tmp_ptr += package_total_length;
                    tmp_n -= package_total_length;
                    continue;
                }
                //wait more data
                else
                {
                    if (package_total_length >= serv->package_max_length)
                    {
                        swWarn("Package length more than the maximum size[%d], Close connection.", serv->package_max_length);
                        goto close_fd;
                    }
                    package = swString_new(package_total_length);
                    if (package == NULL)
                    {
                        return SW_ERR;
                    }
                    memcpy(package->str, (void *)tmp_ptr, (uint32_t) tmp_n);
                    package->length += tmp_n;
                    conn->object = (void *) package;
                    break;
                }
            }
            while(tmp_n > 0);
            return SW_OK;
        }
```
源码解释：如果connection连接中没有缓存数据，则判定为新的数据包，进入接收循环。在接收循环中，首先从数据缓存中尝试获取包体的长度（这个长度存在包头中），如果长度小于0，说明这个数据包不合法，丢弃并关闭连接；如果长度等于0，说明包头还没接收完整，继续接收数据，如果连续5次补全包头失败，则认定为恶意请求，关闭连接；如果长度大于0并且已经接收的数据长度超过或等于包体长度，则说明已经收到一个完整的数据包，通过**swReactorThread_send_string_buffer**函数将数据包放入缓存；如果已接收数据长度小于包体长度，则将不完整的数据包放入connection的object域，等待下一批数据。
```c
        else
        {
            package = conn->object;
            //swTraceLog(40, "wait_data, size=%d, length=%d", buffer->size, buffer->length);

            /**
             * Also on the require_n byte data is complete.
             */
            int require_n = package->size - package->length;

            /**
             * Data is not complete, continue to wait
             */
            if (require_n > n)
            {
                memcpy(package->str + package->length, recv_buf, n);
                package->length += n;
                return SW_OK;
            }
            else
            {
                memcpy(package->str + package->length, recv_buf, require_n);
                package->length += require_n;
                swReactorThread_send_string_buffer(swServer_get_thread(serv, SwooleTG.id), conn, package);
                swString_free((swString *) package);
                conn->object = NULL;

                /**
                 * Still have the data, to parse.
                 */
                if (n - require_n > 0)
                {
                    tmp_n = n - require_n;
                    tmp_ptr = recv_buf + require_n;
                    goto do_parse_package;
                }
            }
        }
```
源码解释：如果connecton的object已经存有数据，则先判断这个数据包还剩下多少个字节未接受，并用当前接收的数据补全数据包，如果数据不够，则继续等待下一批数据；如果数据够，则补全数据包并将数据包发送到缓存中，并清空connection的object域。如果在补全数据包后仍有剩余数据，则开始下一次数据包的解析。

接下来分析**swReactorThread_onReceive_buffer_check_eof**函数。这个函数用于检测用户定义的数据包的分割符，用于解决TCP长连接发送数据的粘包问题，保证onReceive回调每次拿到的是一个完整的数据包。下面是核心源码：
```c        
//读满buffer了,可能还有数据
 if ((buffer->trunk_size - trunk->length) == n)
 {
     recv_again = SW_TRUE;
 }

 trunk->length += n;
 buffer->length += n;

 //over max length, will discard
 //TODO write to tmp file.
 if (buffer->length > serv->package_max_length)
 {
     swWarn("Package is too big. package_length=%d", buffer->length);
     goto close_fd;
 }

//        printf("buffer[len=%d][n=%d]-----------------\n", trunk->length, n);
 //((char *)trunk->data)[trunk->length] = 0; //for printf
//        printf("buffer-----------------: %s|fd=%d|len=%d\n", (char *) trunk->data, event->fd, trunk->length);

 //EOF_Check
 isEOF = memcmp(trunk->store.ptr + trunk->length - serv->package_eof_len, serv->package_eof, serv->package_eof_len);
//        printf("buffer ok. EOF=%s|Len=%d|RecvEOF=%s|isEOF=%d\n", serv->package_eof, serv->package_eof_len, (char *)trunk->data + trunk->length - serv->package_eof_len, isEOF);

 //received EOF, will send package to worker
 if (isEOF == 0)
 {
     swReactorThread_send_in_buffer(swServer_get_thread(serv, SwooleTG.id), conn);
     return SW_OK;
 }
 else if (recv_again)
 {
     trunk = swBuffer_new_trunk(buffer, SW_TRUNK_DATA, buffer->trunk_size);
     if (trunk)
     {
         goto recv_data;
     }
 }
```
源码解释：这里使用了connection的in_buffer输入缓存。首先判定trunk的剩余空间，如果trunk已经满了，此时可能还有数据，则需要新开一个trunk接收数据，因此设定recv_again标签为true。随后判定已经接收的数据长度是否超过了最大包长，如果超过，则丢弃数据包（这里可以看到TODO标签，Rango将在后期把超过包长的数据写入tmp文件中）。随后判定EOF，将数据末尾的长度为eof_len的字符串和指定的eof字符串对比，如果相等，则将数据包发送到缓存区，如果不相等，而recv_again标签为true，则新开trunk用于接收数据。

这里需要说明，Rango只是简单判定了数据包末尾是否为eof，而不是在已经接收到的字符串中去匹配eof，因此并不能很准确的根据eof来拆分数据包。所以各位如果希望能准确解决粘包问题，还是使用固定包头+包体这种协议格式较好。

剩下的几个回调函数都较为简单，有兴趣的读者可以根据之前的分析自己尝试解读一下这几个函数。

至此，ReactorThread模块已全部解析完成。可以看出，ReactorThread模块主要在处理连接接收到的数据，并把这些数据投递到对应的缓存中交由Worker去处理。因此可以理出一个基本的结构：
Reactor响应fd请求->ReactorThread接收数据并放入缓存->ReactorFactory将数据从缓存取出发送给Worker->Worker处理数据。