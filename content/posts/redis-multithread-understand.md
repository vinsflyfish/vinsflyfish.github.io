---
title: "Redis 6.0之后的多线程实现源码分析"
date: 2022-06-05T13:05:42+08:00
draft: false
---
本文主要是介绍redis多线程部分的理解，很久以前看过单线程版本的实现，最近看了几篇文章介绍多线程的原理。有些文章只是讲了些理由，看着有点模糊，对多线程为什么能提升效率，以及准确的执行点还是有点模糊。本文主要从源码阅读的角度，来梳理下其调用链，以及根据代码实现来分析多线程的实现原理和背后的设计。

<!--more-->

## 多线程批量处理源码分析

读请求到来，会插入到  clients_pending_read
等到下一次主循环 beforeSleep 集中处理 pending_read

> `beforeSleep()` is called every time the event loop fired, Redis served a few requests, and is returning back into the event loop.

```
main -> aeMain -> aeProcessEvents (while !stop) 
```

移除掉非核心的执行逻辑， aeProcessEvents 实现是这样的
```
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
		eventLoop->beforesleep(eventLoop);
		numevents = aeApiPoll(eventLoop, tvp);
		eventLoop->aftersleep(eventLoop);
		for (j =0;j < numevents; j++) {
				int fd = eventLoop->fired[j].fd;
				aeFileEvent *fe = &eventLoop->events[fd];
				if (fe->mask & AE_READABLE) {
					fe->rfileProc(eventLoop, fd, ....);
				}
				if (fe->mask & AE_READABLE) {
					fe->wfileProc(eventLoop, fd, ....);
				}
		}
		processTimeEvents(eventLoop);
		....
}
```
总结下这里的逻辑：本次执行的beforesleep 实质上上次全部的，读写事件产生的 pending_read/pending_write

### beforesleep 缩减版代码
```
void beforeSleep(struct aeEventLoop *eventLoop) {
	//..........省略了和梳理多线程原理相关的代码
	/* We should handle pending reads clients ASAP after event loop. */
    handleClientsWithPendingReadsUsingThreads();

	//.........................................
    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWritesUsingThreads();
}
```
beforeSleep 整个函数是在主线程的eventLoop中执行的, 设计到多线程改造的逻辑
1. 使用io thread 处理 pending read
2. 使用io thread 处理 pending write

### handleClientsWithPendingReadsUsingThreads
主要流程：
1. 从 clients_pending_read 列表中取元素，依次分给io_threads （从0开始 item_id++ % io_threads_num)
2. 设置io thread接下来的操作类型 io_threads_op = IO_THREADS_OP_READ;
3. 主线程从 io_threads_list[0] 获取任务从客户端读取数据，和io线程承担对等的读取任务
4. 等待所有的io线程读取任务完成， IOPendingCount 都为0
5. 设置io thread操作类型  io_threads_op = IO_THREADS_OP_IDLE;
6. 从clients_pending_read 遍历client，解析已经读取好的命令执行，如果有reply，执行  putClientInPendingWriteQueue 投递到 pending_write 队列等待发送给客户端

这里关注下 io_threads_op 当这个值为idle时，会投递到pending列表，不为idle时执行实际的网络io.

### handleClientsWithPendingWritesUsingThreads
执行逻辑和 read类似，这里就省略掉了


接下来主要关注 rfileProc / wfileProc 读写回调在多线程情况下的处理
### clients_pending_read
通过分析代码调用链，可以得出 pending_read的来源为如下的调用链
```
readQueryFromClient -> postponeClientRead -> listAddNodeHead(server.clients_pending_read, c);
```
#### readQueryFromClient 调用流程梳理
1. tcp accept时设定了handler acceptTcpHandler
`createSocketAcceptHandler(&server.ipfd, acceptTcpHandler);`
2. acceptCommonHandler 处理新连接时，执行 createClient
3. createClient 创建连接时，设定了读回调 readQueryFromClient
`connSetReadHandler(conn, readQueryFromClient);`

#### readQueryFromClient 实现
继续去除非主干代码，其简略版本实现为:
```
void readQueryFromClient(connection *conn) {
		// 如果开启了thread I/O 会执行到这里
		if (postponeClientRead(c)) return;
		// 后面都是原来的单线程下处理的逻辑，省略
}
```
可以看到多线程下的改变就是调用了 postponeClientRead，其作用就是,把当前需要读取数据的client对象投递到链表 clients_pending_read ，等待分发到IO线程中进行实际的数据读取，在主EventLoop中不在处理实际的client的读取。

这里是多线程设计的核心，如果读事件到来，扔到pending_read中，实现的类似delay的效果，这样可以集中几个client的读一次统一处理。

##### 再次补充下 io_threads_op 这个重要的全局变量对读写逻辑的影响
在任务分发给了不同的io thread之后，主线程会设置
`io_threads_op = IO_THREADS_OP_READ;`这里是一个全局变量，这样会执行 readQueryFromClient 后面实际数据读取，而不是再投递到pending列表中。写逻辑也类似。

### clients_pending_write
```
handleClientsWithPendingReadsUsingThreads ->
putClientInPendingWriteQueue -> listAddNodeHead(server.clients_pending_write,c);
```
handleClientsWithPendingReadsUsingThreads 主线程会在读线程数据全部读取完后，在线程中执行命令，再将执行完的命令投递到 pending_write列表中

## pending_read/write 不加锁的设计
IOThreadMain  listNext 从 io_threads_list[id] 这个列表取数据，只读不删
主线程 listAddNodeTail 给 io_threads_list[id] 增加数据
主线程 listDelNode 给 io_threads_list[id] 删除数据

如果只从这个改写数据的模式，单独单写来讲，不需要加锁。从数据交互上讲，写完数据，只有消费完了才能移除掉，这里需要一种合适的通讯方式，能保证删除前，数据被确定的消费了。数据的增删都由一个线程维护，数据是安全的。主要是读取逻辑如果位于增删之间，结果是不确定性的。一旦有了同步机制，能保证顺序的执行，就可以不加锁了。按照这个思想再回到redis对 pending list的处理

```
// handleClientsWithPendingReadsUsingThreads
int handleClientsWithPendingReadsUsingThreads(void) {
	// 1. 列表增加数据添加
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }
	// 2. 列表数据添加完之后，集中设定 setIOPendingCount 
	//   这里其实相当于一个通知信号，数据准备就绪，可以消费了
    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

	//3. 等待 getIOPendingCount 信号被设定为0
    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }
	//4. 列表数据删除
    /* Run the list of clients again to process the new buffers. */
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        listDelNode(server.clients_pending_read,ln);
        c->pending_read_list_node = NULL;
    }

    return processed;
}
```
这里的IOPendingCount是一个原子操作保证线程安全 atomicSetWithSync(io_threads_pending[i], count);

补充一点，基于对原子操作的理解，上面的实现还有一个问题，就是指令重排的问题，虽然保证了 IOPendingCount的原子性，但无法保证1,2,3,4是按顺序执行的，指令重排有可能打乱这种顺序。抱着这样的想法，在列一下，改原子操作对应的内存屏障相关的代码
```
#define atomicGetWithSync(var,dstvar) do { \
    dstvar = atomic_load_explicit(&var,memory_order_seq_cst); \
} while(0)

#define atomicGetWithSync(var,dstvar) do { \
    dstvar = __atomic_load_n(&var,__ATOMIC_SEQ_CST); \
} while(0)

// 内建的同步，采用的是全内存屏障
#define atomicGetWithSync(var,dstvar) { \
    dstvar = __sync_sub_and_fetch(&var,0,__sync_synchronize); \
    ANNOTATE_HAPPENS_AFTER(&var); \
} while(0)
```
文档上也清晰的标注了，使用了线程间的同步
> atomicGetWithSync(var,value)  -- 'atomicGet' with inter-thread synchronization
> atomicSetWithSync(var,value)  -- 'atomicSet' with inter-thread synchronization

## IOThreadMain 实现补充
IOThreadMain 源码解析
```
void *IOThreadMain(void *myid) {
    /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
     * used by the thread to just manipulate a single sub-array of clients. */
    long id = (unsigned long)myid;
    char thdname[16];

	// 设置线程名称，方便调试和定位问题
    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id);
    redis_set_thread_title(thdname);
	// 设置cpu亲和度
    redisSetCpuAffinity(server.server_cpulist);
    makeThreadKillable();

    while(1) {
        /* Wait for start */
        for (int j = 0; j < 1000000; j++) {
            if (getIOPendingCount(id) != 0) break;
        }

		// 每轮检查下 IOPendingCount 如果为0，检查下锁，保证主线程可以通过锁block住io线程
        /* Give the main thread a chance to stop this thread. */
        if (getIOPendingCount(id) == 0) {
			// 创建线程前initThreadedIO中 io_threads_mutex 被lock住
			// 导致这里阻塞住，直到startThreadedIO执行unlock
            pthread_mutex_lock(&io_threads_mutex[id]); 
			pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }
		// 走到这里来 iopendingcount 不应该为0，属于服务器自己的逻辑断言
        serverAssert(getIOPendingCount(id) != 0);

        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
				// io线程当前全局的操作类型为写数据
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
				// io线程当前全局的操作类型为读数据
                readQueryFromClient(c->conn);
            } else {
				// 走到这里表示主线程没有合理的设定io线程的操作类型，但是分配了任务，io线程无法判断要做什么
                serverPanic("io_threads_op value is unknown");
            }
        }
		// 列表已经访问完了，清空
        listEmpty(io_threads_list[id]);
		// 设定标记，通知主线程任务完成
        setIOPendingCount(id, 0);
    }
}
```
## 总结
redis多线程，解决的是将一次eventloop执行的读写操作并行的放入io线程执行，实际上的命令执行还是在主线程中单线程执行。这样将多个命令的sum(IO1 + IO2 + IO3) 缩减为 max(IO1,IO2,IO3).
通过原子操作以及内存屏障控制主线程和io线程的同步实现无锁控制。锁本身弱化为，提供主线程block的机智，而不是用于数据交互或者同步。
