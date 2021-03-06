## 1. 什么是管程(Monitor)

管程和信号量是等价的，所谓等价指的是用管程能够实现信号量，也能用信号量实现管程。
但是管程更容易使用，所以Java选择了管程。

管程：指的是管理共享变量以及对共享变量的操作过程，让他们支持并发。
在Java中，就是管理类的成员变量和成员方法，让这个类是线程安全的。

## 2. MESA 模型
管程发展史上先后出现过三种不同的管程模型：
- Hasen模型
- Hoare模型
- MESA模型

并发编程两大核心：
#### 2.1 互斥
同一时刻只允许一个线程访问共享资源。
==将共享变量及其对共享变量的操作统一封装起来==。管程 X 将共享变量 queue 这个队列和相关的操作入队 enq()、出队 deq() 都封装起来了; 线程 A 和线程 B 如果想访问共享变量 queue，只能通过调用管程提供的 enq()、deq() 方法来实 现;enq()、deq() 保证互斥性，只允许一个线程进入管程。

<img width="300" alt="互斥" src="https://github.com/mantoudev/routine/blob/master/assets/concurrent/jk-concurrent-monitor-1.png"/>

#### 2.2 同步
线程之间如何通信、协作。

在管程模型里，共享变量和对共享变量的操作是被封装起来的，图中最外层的框就代表封装的意
思。框的上面只有一个入口，并且在入口旁边还有一个入口等待队列。当多个线程同时试图进入
管程内部时，只允许一个线程进入，其他线程则在入口等待队列中等待。这个过程类似就医流程
的分诊，只允许一个患者就诊，其他患者都在门口等待。

条件变量，每个条件变量都对应有一个等待队列。条件变量 A 和条件变量 B 分别都有自己的等待队列。

<img width="300" alt="同步" src="https://github.com/mantoudev/routine/blob/master/assets/concurrent/jk-concurrent-monitor-2.png"/>

假设有个线程 T1 执行出队操作，不过需要注意的是执行出队操作，有个前提条件，就是队列不 能是空的，而队列不空这个前提条件就是管程里的条件变量。 如果线程 T1 进入管程后恰好发现 队列是空的，那怎么办呢?等待啊，去哪里等呢?就去条件变量对应的等待队列里面等。此时线 程 T1 就去“队列不空”这个条件变量的等待队列中等待。这个过程类似于大夫发现你要去验个 血，于是给你开了个验血的单子，你呢就去验血的队伍里排队。线程 T1 进入条件变量的等待队 列后，是允许其他线程进入管程的。这和你去验血的时候，医生可以给其他患者诊治，道理都是 一样的。

再假设之后另外一个线程 T2 执行入队操作，入队操作执行成功之后，“队列不空”这个条件对 于线程 T1 来说已经满足了，此时线程 T2 要通知 T1，告诉它需要的条件已经满足了。当线程 T1 得到通知后，会从等待队列里面出来，但是出来之后不是马上执行，而是重新进入到入口等待队 列里面。这个过程类似你验血完，回来找大夫，需要重新分诊。

条件变量及其等待队列我们讲清楚了，下面再说说 `wait()`、`notify()`、`notifyAll()` 这三个操作。前 面提到线程 T1 发现“队列不空”这个条件不满足，需要进到对应的等待队列里等待。这个过程 就是通过调用 `wait()` 来实现的。如果我们用对象 A 代表“队列不空”这个条件，那么线程 T1 需要调用 `A.wait()`。同理当“队列不空”这个条件满足时，线程 T2 需要调用 `A.notify()` 来通知 A 等待队列中的一个线程，此时这个队列里面只有线程 T1。至于 `notifyAll()` 这个方法，它可以通 知等待队列中的所有线程。
这里我还是来一段代码再次说明一下吧。下面的代码实现的是一个阻塞队列，阻塞队列有两个操
作分别是入队和出队，这两个方法都是先获取互斥锁，类比管程模型中的入口。

1. 对于入队操作，如果队列已满，就需要等待直到队列不满，所以这里用了 `notFull.await()`;。
2. 对于出队操作，如果队列为空，就需要等待直到队列不空，所以就用了 `notEmpty.await()`;。
3. 如果入队成功，那么队列就不空了，就需要通知条件变量:队列不空notEmpty对应的等待队列。
4. 如果出队成功，那就队列就不满了，就需要通知条件变量:队列不满notFull对应的等待队列。


```
public class BlockedQueue<T> {
    final Lock lock = new ReentrantLock();
    //条件变量：队列不满
    final Condition notFull = lock.newCondition();
    //条件变量：队列不空
    final Condition notEmpty = lock.newCondition();

    /**
     * 入队
     *
     * @param x
     */
    void enq(T x) {
        lock.lock();
        while (队列已满) { //TODO:队列已满
            try {
                //等待队列不满
                notFull.await();
                //省略入队操作...
                //入队后，通知可以出队
                notEmpty.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

    /**
     * 出队
     *
     * @param x
     */
    void deq(T x) {
        lock.lock();
        while (队列已空) { //TODO:队列已空
            try {
                //等待队列不满
                notEmpty.await();
                //省略入队操作...
                //入队后，通知可以出队
                notFull.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }
}
```

`await()` 和前面我们提到的 `wait()` 语义是一样的;`signal()` 和前面我们提到的 `notify()` 语义是一样的。

### wait()
对于 MESA 管程来说，有一个编程范式，就是需要在一个 while 循
环里面调用 wait()。**这个是 MESA 管程特有的。**


```
while(条件不满足) {
wait();
}
```
Hasen 模型、Hoare 模型和 MESA 模型的一个核心区别就是当条件满足后，如何通知相关线程。管程要求同一时刻只允许一个线程执行，那当线程 T2 的操作使线程 T1 等待的条件满足 时，T1 和 T2 究竟谁可以执行呢?

1. Hasen 模型里面，要求 notify() 放在代码的最后，这样 T2 通知完 T1 后，T2 就结束了，然 后 T1 再执行，这样就能保证同一时刻只有一个线程执行。
2. Hoare 模型里面，T2 通知完 T1 后，T2 阻塞，T1 马上执行;等 T1 执行完，再唤醒 T2，也 能保证同一时刻只有一个线程执行。但是相比 Hasen 模型，T2 多了一次阻塞唤醒操作。
3. MESA 管程里面，T2 通知完 T1 后，T2 还是会接着执行，T1 并不立即执行，仅仅是从条件变 量的等待队列进到入口等待队列里面。这样做的好处是 notify() 不用放到代码的最后，T2 也 没有多余的阻塞唤醒操作。但是也有个副作用，就是当 T1 再次执行的时候，可能曾经满足的 条件，现在已经不满足了，所以需要以循环方式检验条件变量。

## 4. notify()
除非 经过深思熟虑，否则尽量使用 notifyAll()。那什么时候可以使用 notify() 呢?需要满足以下三个 条件:
1. 所有等待线程拥有相同的等待条件;
2. 所有等待线程被唤醒后，执行相同的操作;
3. 只需要唤醒一个线程。

## 5.总结
Java 参考了 MESA 模型，语言内置的管程(synchronized)对 MESA 模型进行了精简。MESA 模型中，条件变量可以有多个，Java 语言内置的管程里只有一个条件变量。具体如下图所示。

<img width="300" alt="同步" src="https://github.com/mantoudev/routine/blob/master/assets/concurrent/jk-concurrent-monitor-3.png"/>

Java 内置的管程方案(synchronized)使用简单，synchronized 关键字修饰的代码块，在编译期会自动生成相关加锁和解锁的代码，但是仅支持一个条件变量;而 Java SDK 并发包实现的管程支持多个条件变量，不过并发包里的锁，需要开发人员自己进行加锁和解锁操作。
