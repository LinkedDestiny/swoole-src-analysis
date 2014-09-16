Swoole源码学习记录
===================
-------------
##Swoole版本：1.7.4-stable
-------------
Swoole中为了更好的进行内存管理，减少频繁分配释放内存空间造成的损耗和内存碎片，Rango设计并实现了三种不同功能的MemoryPool：FixedPool，RingBuffer和MemoryGlobal。
Rango声明了一个swMemoryPool结构体来表示一个内存池，该结构体在swoole.h头文件中 501-507行声明，结构如下：

    typedef struct _swMemoryPool
    {
        void *object;
        void* (*alloc)(struct _swMemoryPool *pool, uint32_t size);
        void (*free)(struct _swMemoryPool *pool, void *ptr);
        void (*destroy)(struct _swMemoryPool *pool);
    } swMemoryPool;

（PS：虽然原来也知道结构体中可以通过存放函数指针模拟一个类，但是现在才学到可以通过传入一个指针参数来模拟this指针，使结构体更像一个类）
这里看到，首先是一个void*型指针，用于标记内存池的首地址。另外三个函数大家都不会陌生，分别用于从内存池中拿到一块内存、释放一块内存、销毁整个内存池。

基类介绍完了，下面来看具体的子类实现。
首先是FixedPool。FixedPool是随机分配内存池(random alloc/free)，将一整块内存空间切分成**等大小**的一个个小块，每次分配其中的一个小块作为要使用的内存，这些小块以**链表**的形式存储。
    FixedPool的全部定义声明均在src/memory/FixedPool.c文件内。Rango声明了两个结构体swFixedPool_slice和swFixedPool来实现管理内存池。
    swFixedPool_slice为内存池中每个小块的结构声明，其定义如下：

    typedef struct _swFixedPool_slice
    {
        uint8_t lock;
        struct _swFixedPool_slice *next;
        struct _swFixedPool_slice *pre;
        char data[0];
    
    } swFixedPool_slice;

可以看出，每个slice为一个链表节点（双向链表），unit8_t lock用于标记该节点是否被占用（0代表空闲，1代表已占用），next和pre为该节点的后继、前驱指针，char data[0]为内存空间指针（关于0长度数组的解释，请看[http://blog.csdn.net/liuaigui/article/details/3680404](http://blog.csdn.net/liuaigui/article/details/3680404)）

swFixedPool为内存池实体，包括了内存池相关信息。定义如下：

    typedef struct _swFixedPool
    {
        void *memory;   // 内存指针，指向一片内存空间
        size_t size;    // 内存空间大小
    
        swFixedPool_slice *head;    // 链表头部节点
        swFixedPool_slice *tail;    // 链表尾部节点，两个指针用于快速访问和移动节点
    
        /**
         * total memory size，节点数目
         */
        uint32_t slice_num;
    
        /**
         * memory usage ，已经使用的节点
         */
        uint32_t slice_use;
    
        /**
         * Fixed slice size
         */
        uint32_t slice_size; 节点大小
    
        /**
         * use shared memory
         */
        uint8_t shared; 是否共享内存
    
    } swFixedPool;

（参数太多就不文字叙述，直接在变量名后以注释格式解释）
这里先看swFixedPool_new的具体定义。其声明如下：

    /**
    * FixedPool, random alloc/free fixed size memory
    * Location: swoole.h 512L
    */
    swMemoryPool * swFixedPool_new(uint32_t size, uint32_t trunk_size, uint8_t shared);
第一个参数size为每一个小块的大小，第二个参数trunk_size为小块的总数，第三个参数shared代表该内存池是否为共享内存，返回的是被创建的内存池指针。下面贴出代码：

    size_t size = slice_size * slice_num;
    size_t alloc_size = size + sizeof(swFixedPool) + sizeof(swMemoryPool);
    void *memory = (shared == 1) ? sw_shm_malloc(alloc_size) : sw_malloc(alloc_size);
    
    swFixedPool *object = memory;
    memory += sizeof(swFixedPool);
    bzero(object, sizeof(swFixedPool));
    
    object->shared = shared;
    object->slice_num = slice_num;
    object->slice_size = slice_size;
    object->size = size;
    
    swMemoryPool *pool = memory;
    memory += sizeof(swMemoryPool);
    pool->object = object;
    pool->alloc = swFixedPool_alloc;    // 指定三个操作函数
    pool->free = swFixedPool_free;
    pool->destroy = swFixedPool_destroy;
    
    object->memory = memory;
    
    swFixedPool_init(object);
    
前两行是计算出内存池实际大小：slice大小*slice数量+MemoryPool头部大小+FixedPool头部大小。接下来根据是否是共享内存来判定使用哪种分配内存的方法（其中sw_shm_malloc方法将在上一章补充）。接下来依次填充swFixedPool、swMemoryPool的相关属性，实际内存空间起始地址存放于swFixedPool的memory中。接下来调用swFixedPool_init方法初始化内存池。

FixedPool同时拥有四个操作函数分别用于初始化、分配空间、释放空间、销毁内存池。定义如下：

    static void swFixedPool_init(swFixedPool *object);
    static void* swFixedPool_alloc(swMemoryPool *pool, uint32_t size);
    static void swFixedPool_free(swMemoryPool *pool, void *ptr);
    static void swFixedPool_destroy(swMemoryPool *pool);

 首先是swFixedPool_init函数。该函数用于初始化一个内存池结构体，其核心代码如下：

    void *cur = object->memory;
    void *max = object->memory + object->size;
    do
    {
        slice = (swFixedPool_slice *) cur;
        bzero(slice, sizeof(swFixedPool_slice));

        if (object->head != NULL)
        {
            object->head->pre = slice;
            slice->next = object->head;
        }
        else
        {
            object->tail = slice;
        }

        object->head = slice;
        cur += (sizeof(swFixedPool_slice) + object->slice_size);
        slice->pre = (swFixedPool_slice *) cur;
    } while (cur < max);
源码解释：从内存空间的首部开始，每次初始化一个slice大小的空间，并插入到链表的头部。至于为什么使用链表这个结构会稍后说明。
    内存分配好了，如何使用呢？这里再看swFixedPool_alloc函数。这里要注意，因为FixedPool的内存块大小是固定的，所以函数中第二个参数size只是为了符合swMemoryPool中的声明，不具有实际作用，可以随便填一个数。其核心代码如下：
    
    slice = object->head;
    if (slice->lock == 0)
    {
        slice->lock = 1;
        /**
         * move next slice to head (idle list)
         */
        object->head = slice->next;
        slice->next->pre = NULL;

        /*
         * move this slice to tail (busy list)
         */
        object->tail->next = slice;
        slice->next = NULL;
        slice->pre = object->tail;
        object->tail = slice;

        return slice->data;
    }
    else
    {
        return NULL;
    }
源码解释：首先获取内存池链表首部的节点，并判断该节点是否被占用，如果被占用，说明内存池已满，返回null（因为所有被占用的节点都会被放到尾部）；如果未被占用，则将该节点的下一个节点移到首部，并将该节点移动到尾部，标记该节点为占用状态，返回该节点的数据域。

当一个内存块用完，需要释放内存，则调用swFixedPool_free方法。该函数第二个参数为需要释放的数据域。其核心代码如下：
    
    slice = ptr - sizeof(swFixedPool_slice);
    slice->lock = 0;

    //list head, AB
    if (slice->pre == NULL)
    {
        return;
    }
    //list tail, DE
    if (slice->next == NULL)
    {
        slice->pre->next = NULL;
    }
    //middle BCD
    else
    {
        slice->pre->next = slice->next;
        slice->next->pre = slice->pre;
    }
    slice->pre = NULL;
    slice->next = object->head;
    object->head->pre = slice;
    object->head = slice;
源码解释：首先通过移动ptr指针获得slice对象，并将占用标记lock置为0。如果该节点为头节点，则直接返回。如果不是头节点，则将该节点移动到链表头部。
swFixedPool_destroy为直接释放整片内存池，在此不再详解。
（PS：目前先放出FixedPool的源码分析，今晚再放出RingBuffer和MmemoyGlobal的分析。）