Swoole源码学习记录
===================

###作者
Lancelot（李丹阳） from **LinkedDestiny**（**牵机工作室**）

###描述
这些记录是我在学习分析swoole源码时写下的，由于个人能力所限，其中的分析难免有不全或错误的地方。如果你发现了这些错误和不全，请联系我并可以做出改正。

###邮箱
simonarthur2012@gmail.com

###目录
-[ShareMemory共享内存](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94ShareMemory%E5%92%8CMemoryPool.md)
-[三种MemoryPool（上）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%BA%8C%EF%BC%89%E2%80%94%E2%80%94%E4%B8%89%E7%A7%8DMemoryPool%EF%BC%88%E4%B8%8A%EF%BC%89.md)
-[三种MemoryPool（下）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%B8%89%EF%BC%89%E2%80%94%E2%80%94%E4%B8%89%E7%A7%8DMemoryPool%EF%BC%88%E4%B8%8B%EF%BC%89.md)
-[锁和信号（上）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E5%9B%9B%EF%BC%89%E2%80%94%E2%80%94%E9%94%81%E5%92%8C%E4%BF%A1%E5%8F%B7%EF%BC%88%E4%B8%8A%EF%BC%89.md)
-[锁和信号（下）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%BA%94%EF%BC%89%E2%80%94%E2%80%94%E9%94%81%E5%92%8C%E4%BF%A1%E5%8F%B7%EF%BC%88%E4%B8%8B%EF%BC%89.md)
-[Pipe管道](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E5%85%AD%EF%BC%89%E2%80%94%E2%80%94Pipe%E7%AE%A1%E9%81%93.md)
-[MsgQueue消息队列](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%B8%83%EF%BC%89%E2%80%94%E2%80%94MsgQueue.md)
-[Reactor模块-epoll](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E5%85%AB%EF%BC%89%E2%80%94%E2%80%94Reactor%E6%A8%A1%E5%9D%97-epoll.md)
-[Factory模块（上）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E4%B9%9D%EF%BC%89%E2%80%94%E2%80%94Factory%E6%A8%A1%E5%9D%97%EF%BC%88%E4%B8%8A%EF%BC%89.md)
-[Factory模块（下）](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E5%8D%81%EF%BC%89%E2%80%94%E2%80%94Factory%E6%A8%A1%E5%9D%97%EF%BC%88%E4%B8%8B%EF%BC%89.md)
-[Worker和Connection](https://github.com/LinkedDestiny/swoole-src-analysis/blob/master/Swoole%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94Worker%2CConnection.md)
-[ReactorThread模块]()

###更新计划
 - 第十一章 Worker，Connection 
 - 第十二章 ReactorThread
 - 第十三章 Server模块详解（上）
 - 第十四章 Server模块详解（下）
 - 第十五章 （补）Core核心模块
 - 第十六章 MemoryGlobal全局内存池
 - 第十七章 swoole.c详解（上）
 - 第十八章 swoole.c详解（下）
 - 第十九章 swoole_process详解
 - 第二十章 Table.c模块
 - 第二十一章 swoole_table详解
 - 第二十二章 总结与回顾

章节内容和章节数可能会根据实际情况调整和变更。

###说明
写下这些记录，最初的目的是希望自己能深入的理解Swoole的底层机制，从源头掌握Swoole，同时也希望自己能从学会Swoole中那些精妙的设计。另一方面，我也希望我的记录能帮助到那些同样希望理解Swoole核心的人，也能让那些怀疑、抨击Swoole的人看到Swoole强大的核心。