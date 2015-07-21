memcached 源码阅读笔记
-------------
```
    [Memcached源码分析之从SET命令开始说起] (http://calixwu.com/2016/11/memcached-yuanmafenxi-from-set.html)
	[Memcached源码分析之线程模型] (http://calixwu.com/2014/11/memcached-yuanmafenxi-xianchengmoxing.html)

```

## Memcached源码分析之线程模型

解析：
![分布式 memcached](/img/memcached_thread_model.png)
	1. 主线程从 work 线程各自有自己的 Event base
	2. 主线程只负责监听新过来的连接并接受他，当有连接过来时，主线程将连接和其他信息封装成一个CQ_ITEM对象，用轮询的方式选择一个work线程，并把CQ_ITEM对象分发给work线程的CQ_ITEM队列之中。注意，并不是直接放入从线程的 event 监听队列中。
	3. 主线程再向和从线程相连的管道发一个字符的信息，将从唤醒(即使从线程没有挂起也没有关系)
	4. 注意，从线程的管道的另一端是已经加入到 event 监听事件中的，用来和主线程之间通信，从线程被该事件唤醒之后，会去 CQ_ITEM 队列中获取 CQ_ITEM 对象，将其加入到 event 监听队列之中。
