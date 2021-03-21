---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---
AbstractQueuedSynchronizer是JDK中实现同步工具的一个很重要的类。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4238c1860544e17b59f4938eb89f900~tplv-k3u1fbpfcp-watermark.image)

* 上述为其继承关系

今日阅读Semaphore源码时，看到release()方法:
``` Java
// Semaphore.release(int permits)方法
public void release(int permits) {
    if (permits < 0)
    	throw new IllegalArgumentException();
    sync.releaseShared(permits);
}

// Sync.releaseShared(int arg)方法，其中Sync extends AbstractQueuedSynchronizer
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        signalNext(head);
        return true;
    }
    return false;
}

// AbstractQueuedSynchronizer.tryReleasedShared(int arg)方法在Semaphore中的实现
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
           	 throw new Error("Maximum permit count exceeded");
        // 原子性更新资源数
        if (compareAndSetState(current, next))
            return true;
    }
}

// AbstractQueuedSynchronizer.signalNext(Node head)的实现
private static void signalNext(Node h) {
    Node s;
    // h节点应该是头节点，s节点是头节点后的第一个队列元素，也就是我们应该唤醒的节点，所以有s = h.next
    if (h != null && (s = h.next) != null && s.status != 0) {
        s.getAndUnsetStatus(WAITING);
        // 实际的唤醒操作，通过唤醒节点绑定的线程实现。对应睡眠尝试获取资源而不得的线程
        LockSupport.unpark(s.waiter);
    }
}
```

在我们具体深入之前，先了解一下AbstractQueuedSynchronizer的队列结构:

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17e5267bb0af46a680edab3ef7ae78bd~tplv-k3u1fbpfcp-watermark.image)

这就是一个非常典型的队列，每次插入从队尾添加，然后维护一个队头用来控制，而队尾则是最后一个节点。

理解这个之后，就很好理解关键的acquire()方法了。
``` Java
final int acquire(Node node, int arg, boolean shared, boolean interruptible, boolean timed, long time) {
    Thread current = Thread.currentThread();
    byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
    // 一开始假设当前节点插入之后不会成为第一个节点(首节点)，所以first=false
    boolean interrupted = false, first = false;
    Node pred = null;                // predecessor of node when enqueued

    /*
     * Repeatedly:
     *  Check if node now first
     *    if so, ensure head stable, else ensure valid predecessor
     *  if node is first or not yet enqueued, try acquiring
     *  else if node not yet created, create it
     *  else if not yet enqueued, try once to enqueue
     *  else if woken from park, retry (up to postSpins times)
     *  else if WAITING status not set, set and retry
     *  else park and clear WAITING status, and check cancellation
     */

    for (;;) {
        // 因为第一轮循环node=null，所以第一轮循环不走这里
        if (!first && (pred = (node == null) ? null : node.prev) != null && !(first = (head == pred))) {
            if (pred.status < 0) {
                cleanQueue();           // predecessor cancelled
                continue;
            } else if (pred.prev == null) {
                Thread.onSpinWait();    // ensure serialization
                continue;
            }
        }
        // 每次循环会看看当前节点是否是首节点，如果是的话就会尝试申请资源
        if (first || pred == null) {
            boolean acquired;
            try {
                if (shared)
                    // 再次尝试申请资源，实际上就是比较可用资源够不够，特简单一实现
                    acquired = (tryAcquireShared(arg) >= 0);
                else
                    // 同上
                    acquired = tryAcquire(arg);
            } catch (Throwable ex) {
                cancelAcquire(node, interrupted, false);
                throw ex;
            }
            // 如果资源可以运行
            if (acquired) {
                // 且当前节点还是首节点，那直接把它踢出队列，让它跑
                if (first) {
                    node.prev = null;
                    head = node;
                    pred.next = null;
                    node.waiter = null;
                    if (shared)
                        signalNextIfShared(node);
                    if (interrupted)
                        current.interrupt();
                }
                // 此时首节点已出队且已重新进行调度，进入“可运行”状态
                // 下一个节点成为新的首节点，因为下一个节点的pred == null了
                return 1;
            }
        }
        // 如果node == null，则说明是要添加节点
        if (node == null) {                 // allocate; retry before enqueue
            // 根据类型创建节点
            if (shared)
                // 此时节点创建完成，下轮循环开始
                node = new SharedNode();
            else
                // 此时节点创建完成，下轮循环开始
                node = new ExclusiveNode();
        // 此时判断前驱节点是否为空，如果为空，说明这个节点还没和前面的节点链接起来
        // 接下来就是链接到前面的节点的过程
        } else if (pred == null) {          // try to enqueue
            node.waiter = current;
            Node t = tail;
            node.setPrevRelaxed(t);         // avoid unnecessary fence
            // 看看队列首尾是否已初始化
            if (t == null)
                tryInitializeHead();
            // 把新的节点设为尾节点，成功的话casTail(Node, Node)返回true
            else if (!casTail(t, node))
            	// 说明tail被另一个线程更新了，那就取消设置前驱节点
                node.setPrevRelaxed(null);  // back out
            // 关联前面的节点和node，就是入队过程
            else
                t.next = node;
        } else if (first && spins != 0) {
            --spins;                        // reduce unfairness on rewaits
            Thread.onSpinWait();
        // 又是一轮循环，因为新的节点status默认0，所以设置其状态为WAITING
        } else if (node.status == 0) {
            node.status = WAITING;          // enable signal and recheck
        } else {
            long nanos;
            spins = postSpins = (byte)((postSpins << 1) | 1);
            // 因为前面设置了times=false，所以这里把当前线程park起来，而就是这里，对应release()的unpark()
            if (!timed)
                // 在这里，线程因为资源不足而阻塞，等到被唤醒时会从这个位置继续运行，然后又是一轮循环，看看资源数是否够用
                LockSupport.park(this);
            else if ((nanos = time - System.nanoTime()) > 0L)
                LockSupport.parkNanos(this, nanos);
            else
                break;
            node.clearStatus();
        if ((interrupted |= Thread.interrupted()) && interruptible)
            break;
        }
    }
    return cancelAcquire(node, interrupted, interruptible);
}
```

结束！

其他同步类对于acquire()的调用大同小异，理解AQS对于理解JDK同步工具有着很大的帮助。