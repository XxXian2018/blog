# Interview Three：AQS

AbstractQueuedSynchronizer，为ReentrantLock、Semaphore、CountDownLatch等同步工具提供基础实现，是`CLH队列`的一种变体实现

```java
// CLH自旋锁
public class CLH {

    private final AtomicReference<Node> tail;

    private final ThreadLocal<Node> current;

    private final ThreadLocal<Node> pred;

    public CLH() {
        tail = new AtomicReference(new Node());
        current = ThreadLocal.withIntial(new Node());
        pred = new ThreadLocal<>();
    }

    private final CLH ins = new CLH();

    static class Node {
        volatile boolean locked = false;
    }

    static class TestTask implements Runnable {
        @Override
        public void run() throws InterruptedException {
            try {
                ins.lock();
            } catch (InterruptedException ex) {
                e.printStackTrace();
            } finally {
                ins.unlock();
            }
        }
    }

    public void lock() throws InterruptedException {
        Node c = current.get();
        c.locked = true;

        Node prev = tail.getAndSet(c);
        pred.set(prev);

        // 自旋判断前一个节点的状态
        while (prev.locked) {
            Thread.sleep(100);
        }
    }

    public void unlock() {
        Node c = current.get();
        c.locked = false;
        current.set(pred.get());
    }

    public static void main(String[] args) {
        for (int i = 0; i < 2; i++) {
            new Thread(new TestTask()).start();
        }
    }
}
```

相比CLH自旋锁busy-waiting，AQS为`sleep-waiting`，所以AQS的队列节点由前驱节点唤醒，而不像自旋锁队列的后驱节点自旋判断

抽象出**同步器的state（volatile）**，根据定义进入CLH队列的规则：

1. **CAS + volatile**方式，管理等待线程的入队出队

    ```java
    // AbstractQueuedSynchronizor.class
    public final boolean acquire(int arg) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // CAS设置新的队尾
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 上述设置队尾失败，进入enq()中自旋设置
        enq(node);
        return node;
    }
    ```

2. 每个节点入队后对会**尝试修改前一个节点的状态**，在ownner线程执行完毕后，该状态作为是否唤醒后驱节点的依据

    ```java
    // AbstractQueuedSynchronizor.class
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            // h.waitStatus在
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    ```

3. 在可以竞争到资源时（当前仅当线程节点处于head之后，且尝试获取资源成功）挂起

4. 竞争到资源的节点，在执行完毕后，会将自身节点的状态修改为0，并唤醒后续的节点

5. 线程被唤醒后，意味着队头线程已经执行完毕，将队头线程出队

> 类比：synchronized的waiting queue，collection list和entry list对应除了队头和第二个节点的CLH队列部分，on deck对应第二个节点

难点：

- 获取资源和释放资源的并发安全性，由`AQS#acquiredQueued()#tryAcquired()`方法和`AQS#release()#tryRelease()`对state的CAS操作保障

    > 前驱线程节点一定被设置成SIGNAL后，后驱线程才会被挂起，否则会出现无法唤醒的情况

    <!-- 当前驱线程不唤醒后驱线程，则肯定资源已被释放，后续线程可以获得资源不被阻塞 -->
    
    后驱线程阻塞，则队列必定有值，前驱线程在完成任务后根据SIGNAL/PROPAGATE标识，若有则唤醒后驱线程

- 唤醒的方式：独占模式**只在释放资源时唤醒**，传播模式在**获取资源和释放资源**时都会唤醒后续线程

    传播模式原理：每个线程被唤醒后，都会**根据当前的资源数是否足够**，来决定是否唤醒其后一个节点的线程

## **ReentrantLock**

独占锁，使用AQS`独占模式`实现，state表示**标识位和计数器**：
- = 0：可用
- \>= 1：有线程独占，大于1时表示线程有重入锁范围的逻辑

与synchronized没有本质区别，在有竞争时线程一样会**阻塞**，且一样**支持可重入**，额外支持**公平锁模式**和**获取尝试**

- 公平锁：在竞争资源时，发现资源可持有后不会立即独占资源，而是判断等待队列是否有其他等待的线程，若有则进入队列中排队，有效防止`线程饥饿`问题

- 可重入锁：如果发现同步器**此时的独占线程为当前线程**，则直接将资源数加1，并重新进入代码块

> 若没有可重入特性，当前线程在获得锁资源后，尝试再次获取同个锁资源时，会形成自己锁自己的现象

- 非公平锁：与synchronized一致，当有新来线程竞争资源时，会与队头第二个节点线程进行争夺，如果失败则进入队列尾，成功则意味着第二个节点需要继续排队，造成线程饥饿问题

- 性能（与synchronized）：与synchronized相比有小幅度提升，ReentrantLock在竞争失败后不会立即挂起，而是会转而加入CLH队列，并`在前驱节点为队头的情况`下，再次尝试执行`两次CAS竞争`（进入队列后先执行一次CAS，设置前驱节点状态为-1后再竞争一次），这种做法在竞争不太激烈的情况下会有更高的性能表现

> synchronized最糟的情况莫过于锁最终膨胀为重量级锁，**整个开销包括了锁的升级过程**

## **Semaphore**

信号量，使用AQS`共享模式`实现，state表示**许可证**：

- = 0：不可用，许可证不足
- \>= 1：可用，最多还能有

**入口**：

- Semaphore(int count)：传入许可证的个数，后续许可证的获取和返还操作下，个数在区间\[0, count\]

- acquire(int)：`剩余值 = 当前许可证量 - 需要获取的值`，如果剩余值小于0，则进入同步阻塞队列，否则通过CAS设置当前信号量为剩余值，同时返回剩余值

- release(int)：**返还**当前信号量计数器传入个数个许可证，公平模式下如果没有出现与后驱线程竞争成功的线程，才会唤醒后续线程进行**传播**

**应用：**

1. 限速，控制应用速率

2. 反向使用也可达到阻塞效果，效果类似于闭锁的第二个情况

## **CountDownLatch**

闭锁，使用AQS`共享模式`实现，state表示**闭合开关（倒数状态）**：

- = 0：打开闭合开关，此时在队列中的所有线程都将被唤醒
- \>= 0：关闭闭合开关，此时获取该闭锁资源的线程都将进入到CLH队列中

**入口**

- CountDownLatch(int count)：传入闭合开关的大小

- await()：`sync.acquireSharedInterrupted(1)`，如果闭合开关处于开启状态，则直接进入阻塞状态，否则直接执行

- countDown()：`sync.releaseShared(1)`，每调用一次闭合开关的数量就减1，当减至0时闭合开关**关闭**

**应用**：

1. 所有参与者一同执行任务

    执行countDown()的作为观察者线程（此时构造函数通常传入1），参与者到达闭锁后执行await()方法阻塞等待，观察者可以在恰当的时间点打开闭锁

2. 某个执行任务需要等待所有参与者

- 执行await()的作为任务执行线程（此时构造函数通常传入参与者个数），参与者到达闭锁后执行countDown()方法并继续执行其任务，直到闭锁开关关闭，任务执行线程才开始执行任务

## **CyclicBarrier**

栅栏，使用内置lock（Reentrant）的trip（lock的condition）实现线程间的阻塞等待，通过`AQS.state`和`CyclicBarrier.count`两个值共同维护：

- state：表示栅栏内置的显式锁占用情况

    - = 0：当前锁未被独占，线程可以获取该锁后

    - \>= 0：当前锁已被独占

- CyclicBarrier.count：表示当前已参与的个数，配合parties（代表参与的总个数）使用

    - 0 < count <= parties：栅栏的参与个数未到达，所有线程都会进入lock的等待队列中等待

    - count == 0：最后一个参与者到达，执行栅栏回调（若有）后，唤醒当前等待队列中的所有线程，并重置当前迭代

**入口**：

- CyclicBarrier(int parties, Runnable barrierAction)：设置参与者个数和栅栏即将打开前的执行回调

- **await()**：如果栅栏的count未到达0，则调用`trip.await()`进入等待队列等待唤醒，否则执行回调，并调用`trip.signAll()`唤醒所有线程

> condition.await()对应obj.wait()，condition.signal()/signalAll()对应obj.notify()/notifyAll()，线程被唤醒后会加入到同步队列（AQS为WaitNode，Synchronized为waiting queue）中，等待调用notify()的线程执行完毕，并与其他线程竞争（排队）

**应用：** 无需第三方介入的多线程同时进行任务工具，每个参与者不止有阻塞等待逻辑，而且都会参与到栅栏的迭代逻辑中

# **交替执行**

```java
public class A1B2 {
    
    private Semaphore[] semaphores;

    private volatile int pos;

    private int raceCount;

    private ThreadLocal<Integer> sumCount = ThreadLocal.withInitial(() -> 0);

    public A1B2(char[] output) throws InterruptedException {
        this.semaphores = new Semaphore[output.length];
        for (int i = 0; i < output.length; i++) {
            semaphores[i] = new Semaphore(1);
            semaphores[i].acquire();
        }
    }

    public void run(int count, char output) throws InterruptedException {
        int sum = sumCount.get();
        while (sum < 2) {
            // 停顿
            semaphores[count].acquire();

            if (count == pos) {
                pos = (count + 1) % semaphores.length;
                System.out.println(output + ", 竞争次数：" + raceCount);
                raceCount = 0;
                sumCount.set(sum++);
            } else {
                raceCount++;
            }

            semaphores[pos].release();
        }
    }

    public void start() {
        semaphores[0].release();
    }

    public static void main(String[] args) throws InterruptedException {
        char[] output = {'A', 'B', 'C', 'D'};
        A1B2 a1B2 = new A1B2(output);
        for (int i = 0; i < output.length; i++) {
            int j = i;
            new Thread(() -> {
                try {
                    a1B2.run(j, output[j]);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        a1B2.start();
    }
}
```

# 参考
- [AQS基础——多图详解CLH锁的原理与实现](https://zhuanlan.zhihu.com/p/197840259)