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