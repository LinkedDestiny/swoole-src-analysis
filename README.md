Swoole源码学习记录
===================

###作者
Lancelot（李丹阳） from **LinkedDestiny**（**牵机工作室**）

###描述
这些记录是我在学习分析swoole源码时写下的，由于个人能力所限，其中的分析难免有不全或错误的地方。如果你发现了这些错误和不全，请联系我并可以做出改正。

###邮箱
simonarthur2012@gmail.com

###目录
1. [ShareMemory和MemoryPool](01.ShareMemory和MemoryPool.md)
2. [三种MemoryPool(上)](02.三种MemoryPool(上).md)
3. [三种MemoryPool(下)](03.三种MemoryPool(下).md)
4. [锁和信号(上)](04.锁和信号(上).md)
5. [锁和信号(下)](05.锁和信号(下).md)
6. [Pipe管道](06.Pipe管道.md)
7. [MsgQueue](07.MsgQueue.md)
8. [Reactor模块-epoll](08.Reactor模块-epoll.md)
9. [Factory模块(上)](09.Factory模块(上).md)
10. [Factory模块(下)](10.Factory模块(下).md)
11. [Worker,Connection](11.Worker,Connection.md)
12. [ReactorThread模块](12.ReactorThread模块.md)
13. [Server模块详解(上)](13.Server模块详解(上).md)
14. [Server模块详解(下)](14.Server模块详解(下).md)

###更新计划
 - 第十五章 Timer模块分析
 - 第十六章 http_server模块分析
 - 第十七章 swoole.c详解（上）
 - 第十八章 swoole.c详解（下）
 - 第十九章 swoole_process详解
 - 第二十章 Table.c模块分析
 - 第二十一章 总结与回顾

章节内容和章节数可能会根据实际情况调整和变更。

###说明
写下这些记录，最初的目的是希望自己能深入的理解Swoole的底层机制，从源头掌握Swoole，同时也希望自己能从学会Swoole中那些精妙的设计。另一方面，我也希望我的记录能帮助到那些同样希望理解Swoole核心的人，也能让那些怀疑、抨击Swoole的人看到Swoole强大的核心。
