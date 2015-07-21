memcached 源码阅读笔记
-------------
```
	Memcached源码分析之从SET命令开始说起 http://calixwu.com/2014/11/memcached-yuanmafenxi-from-set.html
	Memcached源码分析之线程模型 http://calixwu.com/2014/11/memcached-yuanmafenxi-xianchengmoxing.html
	
```

## Memcached源码分析之线程模型
	memcached通过epoll（使用libevent，下面具体再讲）实现异步的服务器，但仍然使用多线程，主要有两种线程，分别是“主线程”和“worker线程”，一个主线程，多个worker线程。
	主线程负责监听网络连接，并且accept连接。当监听到连接时，accept后，连接成功，把相应的client fd丢给其中一个worker线程。worker线程接收主线程丢过来的client fd，加入到自己的epoll监听队列，负责处理该连接的读写事件。
	主线程和worker线程都各自有自己的监听队列，主线程监听的仅是listen fd，而worker线程监听的则是主线程accept成功后丢过来的client fd。
``` 
	1）主线程首先为自己分配一个event_base，用于监听连接，即listen fd。

	2）主线程创建n个worker线程，同时每个worker线程也分配了独立的event_base。

	3）每个worker线程通过管道方式与其它线程（主要是主线程）进行通信，调用pipe函数，产生两个fd，一个是管道写入fd，一个是管道读取fd。worker线程把管道读取fd加到自己的event_base，监听管道读取fd的可读事件，即当主线程往某个线程的管道写入fd写数据时，触发事件。

	4）主线程监听到有一个连接到达时，accept连接，产生一个client fd，然后选择一个worker线程，把这个client fd包装成一个CQ_ITEM对象（该结构体下面再详细讲，这个对象实质是起主线程与worker线程之间通信媒介的作用，主线程把client fd丢给worker线程往往不止“client fd”这一个参数，还有别的参数，所以这个CQ_ITEM相当于一个“参数对象”，把参数都包装在里面），然后压到worker线程的CQ_ITEM队列里面去（每个worker线程有一个CQ_ITEM队列），同时主线程往选中的worker线程的管道写入fd中写入一个字符“c”（触发worker线程）。

	5）主线程往选中的worker线程的管道写入fd中写入一个字符“c”，则worker线程监听到自己的管道读取fd可读，触发事件处理，而此是的事件处理是：从自己的CQ_ITEM队列中取出CQ_ITEM对象（相当于收信，看看主线程给了自己什么东西），从4）可知，CQ_ITEM对象中包含client fd，worker线程把此client fd加入到自己的event_base，从此负责该连接的读写工作。
```

解析：
![分布式 memcached](/img/distributed-memcached.png)
	1. 主线程从 work 线程各自有自己的 Event base
	2. 主线程只负责监听新过来的连接并接受他，当有连接过来时，主线程将连接和其他信息封装成一个CQ_ITEM对象，用轮询的方式选择一个work线程，并把CQ_ITEM对象分发给work线程的CQ_ITEM队列之中。注意，并不是直接放入从线程的 event 监听队列中。
	3. 主线程再向和从线程相连的管道发一个字符的信息，将从唤醒(即使从线程没有挂起也没有关系)
	4. 注意，从线程的管道的另一端是已经加入到 event 监听事件中的，用来和主线程之间通信，从线程被该事件唤醒之后，会去 CQ_ITEM 队列中获取 CQ_ITEM 对象，将其加入到 event 监听队列之中。
	


## Memcached源码分析之内存管理
### 1 模型分析

#### 概念
'''
	a) slab、chunk
	slab是一块内存空间，默认大小为1M，而memcached会把一个slab分割成一个个chunk，比如说1M的slab分成两个0.5M的chunk，所以说slab和chunk其实都是代表实质的内存空间，chunk只是把slab分割后的更小的单元而已。slab就相当于作业本中的“页”，而chunk则是把一页画成一个个格子中的“格”。
	
	b）item
	item是我们要保存的数据，例如php代码：$memcached->set(“name”,”abc”,30);代表我们把一个key为name，value为abc的键值对保存在内存中30秒，那么上述中的”name”, “abc”, 30这些数据实质都是我们要memcached保存下来的数据， memcached会把这些数据打包成一个item，这个item其实是memcached中的一个结构体（当然结构远不止上面提到的三个字段这么简单），把打包好的item保存起来，完成工作。而item保存在哪里？其实就是上面提到的”chunk”，一个item保存在一个chunk中。
	chunk是实质的内存空间，item是要保存的东西，所以关系是：item是往chunk中塞的。
	还是拿作业本来比喻，item就是相当于我们要写的“字”，把它写到作业本某一“页（slab）”中的“格子（chunk）”里。
	
	c）slabclass
	通过上面a）b）我们知道，slab（都假设为1M）会割成一个个chunk，而item往chunk中塞。
	那么问题来了：
	我们要把这个1M的slab割成多少个chunk？就是一页纸，要画多少个格子？
	我们往chunk中塞item的时候，item总不可能会与chunk的大小完全匹配吧，chunk太小塞不下或者chunk太大浪费了怎么办？就是我们写字的时候，格子太小，字出界了，或者我们的字很小写在一个大格子里面好浪费。
	所以memcached的设计是，我们会准备“几种slab”，而不同一种的slab分割的chunk的大小不一样，也就是说根据“slab分割的chunk的大小不一样”来分成“不同的种类的slab”，而 slabclass就是“slab的种类”的意思了。
	继续拿作业本来比喻：假设我们现在有很多张A4纸，有些我们画成100个格子，有些我们画成200个格子，有些300…。我们把画了相同个格子（也相同大小）的纸钉在一起，成为一本本“作业本”，每本“作业本”的格子大小都是一样的，不同的“作业本”也代表着“画了不同的大小格子的A4纸的集合”，而这个作业本就是slabclass啦！
	所以当你要写字（item）的时候，你估一下你的字有多“大”，然后挑一本作业本（slabclass），在某一页（slab）空白的格子（chunk）上写。
	每个slabclass在memcached中都表现为一个结构体，里面会有个指针，指向它的那一堆slab。
'''
































