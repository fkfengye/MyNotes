
## 一、AQS介绍

`AbstractQueuedSynchronizer (AQS)` 是 Java 并发编程的核心组件，位于 `java.util.concurrent.locks` 包下，为各种锁和同步器（如 `ReentrantLock`、`Semaphore`、`CountDownLatch`）提供了基础框架。理解 AQS 对于掌握 Java 并发编程至关重要。

### 1、AQS的诞生背景与解决的问题

#### 1.1 传统同步机制的局限性

在AQS出现之前，Java的线程同步主要依赖`synchronized`关键字。尽管`Synchronized`简单易用，但其存在以下痛点：
- **灵活性不足**：无法实现非阻塞的尝试获取锁（`tryLock`）、超时等待（`tryLock(timeout)`）等高级功能。
- **性能瓶颈**：在高并发场景下，频繁的线程阻塞与唤醒会导致上下文切换开销增大。
- **资源管理复杂**：开发者需自行实现线程队列、状态管理等逻辑，容易出错且难以维护。

#### 1.2 AQS的诞生：解决并发编程的 “共性难题”

AQS的诞生旨在解决上述问题，其核心目标包括：

 - **简化同步器开发**：通过封装线程排队、阻塞、唤醒等底层逻辑，开发者只需关注状态管理。
 - **提升性能与可靠性**：基于CAS（Compare and Swap）和volatile的无锁化设计，减少线程阻塞，降低上下文切换开销。
 - **支持多样化的同步需求**：通过独占/共享模式的分离，适配锁、信号量、倒计时门闩等多种场景。

.
### 2、AQS 的上下游技术依赖关系
#### 2.1 上游依赖：硬件与JVM支持
AQS的实现依赖于以下底层技术：
- **CAS（Compare and Swap）**：通过`sun.misc.Unsafe`类提供的`compareAndSwapInt`等方法实现原子操作。
- **Volatile变量**：确保state的可见性，避免线程缓存导致的数据不一致。
- **内存屏障（Memory Barrier）**：通过`volatile`和`LockSupport`隐式地插入内存屏障，保证指令重排序的安全性。

#### 2.2 下游依赖：构建高级同步工具

AQS作为基础框架，被广泛用于构建以下高级并发工具：
- **锁（Lock）**：`ReentrantLock`、`ReentrantReadWriteLock`。
- **信号量（Semaphore）**：通过共享模式管理资源许可。
- **倒计时门闩（CountDownLatch）**：通过共享模式实现多线程协作。
- **CyclicBarrier**：通过共享模式实现多线程同步
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1154510d78ec4019897a66c9bb3ca4ee.png#pic_center)

.
### 3、AQS的设计理念

#### 3.1 核心思想：模板方法模式 + 队列管理

AQS的设计灵感来源于 “**模板方法**” 设计模式（Template Method Pattern），其核心思想是将通用的同步逻辑抽象为模板，将具体的状态管理交由子类实现。通过这种方式，AQS实现了“**一次编写，多次复用**”的目标，开发者只需关注业务逻辑中的状态操作（如锁的获取与释放），无需重复实现线程排队、阻塞、唤醒等底层逻辑。

**`AQS 使用了模板方法模式，自定义同步器时需要重写下面几个 AQS 提供的方法：`**

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
    ..... 省略 ......
    
    /**
    * 独占方式 - 尝试获取资源，成功则返回true，失败则返回false。
    */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
    * 独占方式 - 尝试释放资源，成功则返回true，失败则返回false。
    */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
    * 共享方式 - 尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
    */
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
    * 共享方式 - 尝试释放资源，成功则返回true，失败则返回false。
    */
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
    * 该线程是否正在独占资源。只有用到condition才需要去实现它。
    */
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
    
    ..... 省略 ......
}
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/74c344214136413d9ffe689349f6e883.png#pic_center)


.
### 3.2 核心数据结构：CLH队列与状态变量
AQS的实现依赖两个核心组件：
- **`state变量`**：一个volatile int类型的字段，用于表示同步状态。不同的同步器对state的语义定义不同。例如：
  - `ReentrantLock`中，state表示锁的重入次数。
  - `Semaphore`中，state表示剩余的许可数。
- **CLH队列**：一种虚拟的双向链表（由`Node`节点组成），用于管理等待获取资源的线程。每个`Node`节点包含线程引用、状态（`waitStatus`）、前驱和后继指针。

.
### 3.3 独占与共享模式

AQS支持两种资源共享模式：
- **独占模式（Exclusive）**：同一时刻只能有一个线程获取资源（如`ReentrantLock`）。
- **共享模式（Shared）**：允许多个线程同时获取资源（如`CountDownLatch`）。

这种设计使得AQS能够适配多种并发场景，开发者只需通过重写模板方法（如`tryAcquire`、`tryRelease`）即可实现自定义的同步逻辑。


.
.
## 二、源码分析 - 独占锁 ReentrantLock

### 1.1 以ReentrantLock为例 - 下面是demo案例

```java
package com.akulaku.gw.biz.concurrent;

import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    // 创建一个公平锁（true表示公平锁，false或不传参表示非公平锁）
    private static final ReentrantLock lock = new ReentrantLock(true);
    private static int sharedCounter = 0;

    public static void main(String[] args) {
        // 创建多个线程来演示锁的使用
        Thread[] threads = new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new Worker(), "Thread-" + (i + 1));
            threads[i].start();
        }

        // 等待所有线程完成
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println("最终计数器值: " + sharedCounter);
    }

    static class Worker implements Runnable {
        @Override
        public void run() {
            // 基本加锁方式
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " 【获取了】锁");
                sharedCounter++;
                // 模拟工作
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println(Thread.currentThread().getName() + " 开始释放锁");
                lock.unlock();
            }

            // 演示可重入性
            methodA();
        }

        private void methodA() {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " 进入 Method-A");
                methodB();
            } finally {
                lock.unlock();
            }
        }

        private void methodB() {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " 重入 Method--B");
                sharedCounter++;
            } finally {
                lock.unlock();
            }
        }
    }
}
```


为了演示，上面是采用公平锁来处理（**`输出的结果更整洁一点，便于大家理解，后面分析按照常用的非公平锁的方式来给进行源码解读`**），输出结果：

```java
Thread-1 【获取了】锁
Thread-1 开始释放锁
Thread-3 【获取了】锁
Thread-3 开始释放锁
Thread-2 【获取了】锁
Thread-2 开始释放锁
Thread-4 【获取了】锁
Thread-4 开始释放锁
Thread-5 【获取了】锁
Thread-5 开始释放锁
Thread-1 进入 Method-A
Thread-1 重入 Method--B
Thread-3 进入 Method-A
Thread-3 重入 Method--B
Thread-2 进入 Method-A
Thread-2 重入 Method--B
Thread-4 进入 Method-A
Thread-4 重入 Method--B
Thread-5 进入 Method-A
Thread-5 重入 Method--B
最终计数器值: 10
```

.
### 1.2 ReentrantLock 类的整体结构

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    
    ..... 省略 ......
    
    private final Sync sync;
    
    /**
    * 内部类实现 - AQS具体实现
    */
    abstract static class Sync extends AbstractQueuedSynchronizer {
    
    }
    
    
    /**
    * 内部类实现 - 非公平锁
    */
    static final class NonfairSync extends Sync {
    
    }
    
    
    /**
    * 内部类实现 - 公平锁
    */
    static final class FairSync extends Sync {
    
    }
    
    
    ..... 省略 ......
}
```

.
### 1.3 构造函数 - 默认构造非公平锁

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```
.
### 1.4 lock.lock() 获取锁

```java
// 尝试获取锁，先立即尝试抢占，若失败则进入正常的锁获取流程
final void lock() {
    // 尝试以原子操作的方式将锁的状态从 0（未锁定）修改为 1（已锁定）
    if (compareAndSetState(0, 1))
        // 若修改成功，说明当前线程成功获取到锁，将当前线程设置为锁的独占持有者
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 若修改失败，说明锁已经被其他线程持有
        // 调用 acquire 方法进入正常的锁获取流程，可能会将当前线程放入等待队列
        acquire(1);
}
```

 - `compareAndSetState(int expect, int update)`：这是
   `AbstractQueuedSynchronizer` 类中的方法，使用 `CAS（Compare-And-Swap）`原子操作。主要设置的是
   `state`的值（AQS同步器里面的同步状态），从 0 设置到 1
- `setExclusiveOwnerThread(Thread thread)`：同样是 `AbstractQueuedSynchronizer` 类中的方法，用于将指定线程设置为锁的独占持有者
- `acquire(int arg)`：也是 `AbstractQueuedSynchronizer` 类中的方法，用于尝试获取锁。若获取失败，会将当前线程放入等待队列，进入阻塞状态，直到获取到锁


.
### 1.5 AQS 锁获取流程 acquire(1)

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 中断当前线程
        selfInterrupt();
}
```

`acquire` 方法首先尝试直接获取锁，若失败则将当前线程加入等待队列，在队列中持续尝试获取锁，过程中忽略中断。若线程在等待时被中断过，方法结束前会重新设置中断标志位
- `tryAcquire(arg)`：调用 `tryAcquire` 方法尝试获取锁，该方法由子类实现。若返回 true 表示获取成功，直接跳过后续逻辑；若返回 false 则继续执行。
- `addWaiter(Node.EXCLUSIVE)`：若 `tryAcquire` 失败，调用 `addWaiter` 方法为当前线程创建一个独占模式（`Node.EXCLUSIVE` - 排他）的节点，并将其加入到等待队列中，返回该节点。
- `acquireQueued(..., arg)`：将新创建的节点传入 `acquireQueued` 方法，该方法会让线程在队列中等待获取锁。线程可能会多次阻塞和唤醒，不断尝试获取锁。若在等待过程中线程被中断，`acquireQueued` 会返回 true，否则返回 false。
- `selfInterrupt()`：若 `acquireQueued` 返回 true，表示线程在等待过程中被中断过，由于 `acquire` 方法会忽略中断，所以这里调用 `selfInterrupt` 方法重新设置当前线程的中断标志位。内部调用的 `Thread.interrupt()` 方法的主要作用是中断线程，根据线程的不同状态和阻塞情况，会有不同的处理方式。它只是给线程发送中断信号，不会强制终止线程，线程需要自行处理中断信号。

#### 1.5.1 tryAcquire(1) 尝试获取锁 - AQS 模版方法

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

AQS 里面 tryAcquire 只是一个模版方法，不提供具体实现，由子类来进行实现

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

最终会调用到 `ReentrantLock.Sync#nonfairTryAcquire` 该方法用于实现非公平锁的 `tryAcquire`逻辑。非公平锁意味着线程在尝试获取锁时，不会考虑等待队列中是否有其他线程在等待，而是直接尝试获取锁

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    
    // 用于获取锁的当前状态。0 表示锁未被持有，大于 0 表示锁已被持有且重入次数为该值
    int c = getState();
    
    // 若 c 为 0，说明锁当前未被任何线程持有
    if (c == 0) {
        // 尝试以原子操作的方式将锁的状态从 0（未锁定）修改为 1（已锁定）
        if (compareAndSetState(0, acquires)) {
            // 若修改成功，说明当前线程成功获取到锁，将当前线程设置为锁的独占持有者
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    
    // 若 c 不为 0，且当前线程就是锁的独占持有者，说明是重入操作
    else if (current == getExclusiveOwnerThread()) {
        // 计算重入后的锁状态
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
            
        // 更新锁的状态为新的重入次数，因为自己本身就是锁持有者，设置state就无需做cas操作了
        setState(nextc);
        return true;
    }
    return false;
}
```


#### 1.5.2 addWaiter(Node.EXCLUSIVE) - 加入队列尾部

`addWaiter` 方法尝试以快速方式将新节点加入队列，若失败则使用完整的入队逻辑。这种设计是为了提高性能，因为大多数情况下快速入队操作就能成功，避免了每次都执行完整的入队逻辑

```java
private Node addWaiter(Node mode) {
    // 创建一个新的 Node 节点，将当前线程和传入的模式（此处用的独占模式Node.EXCLUSIVE）作为参数传入构造函数
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 若尾节点不为 null，说明队列已经初始化，将新节点的 prev 指针指向尾节点
    if (pred != null) {
        node.prev = pred;
        // 以原子操作的方式尝试将队列的尾节点更新为新节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 完整入队操作
    enq(node);
    return node;
}
```

`enq(node)` 完整入队操作：若快速入队尝试失败（尾节点为 null 或者 `compareAndSetTail` 操作失败），调用 `enq` 方法进行完整的入队操作，确保新节点被正确加入队列，最后返回新节点

```java
private Node enq(final Node node) {
    // 使用无限循环确保节点最终能成功插入队列
    for (;;) {
        Node t = tail;
        // 若尾节点为 null，说明队列还未初始化
        if (t == null) { // Must initialize
            // 使用 CAS（Compare-And-Swap）操作尝试将一个新的空节点设置为头节点
            if (compareAndSetHead(new Node()))
                // 若 CAS 操作成功，将尾节点也设置为头节点，完成队列的初始化
                tail = head;
            
        // 队列已初始化，插入节点 - 逻辑跟 addWaiter 里面快速插入的逻辑一样了       
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;    // 注意只有此处才能退出循环
            }
        }
    }
}
```
`enq` 方法通过无限循环和 `CAS` 操作，确保节点能安全地插入到等待队列中。若队列未初始化，会先进行初始化；若队列已初始化，会将节点插入到队列尾部。该方法利用 `CAS` 操作保证了多线程环境下的线程安全

**注意：此处结束循环的地方，只有return处，那么意味着当队列还未初始化时，先会经历初始化操作，此时的节点为 new Node()；接着会将入参传过来的独占锁node节点入队到队列尾部，直到插入成功就会退出自旋，返回最新的尾部节点**
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f730f13ef014471ea99c376d7d4e9b8f.png#pic_center)

#### 1.5.3 acquireQueued(tail, 1)
`acquireQueued` 方法会让已在等待队列中的线程不断尝试获取锁，若前驱节点是头节点则尝试获取，获取成功则更新队列头节点；若获取失败，根据情况阻塞线程。在整个过程中，线程不会响应中断，若线程被中断，会记录中断状态并在最后返回。若最终获取锁失败，会取消当前的获取操作。

```java
final boolean acquireQueued(final Node node, int arg) {
    // failed 变量用于标记获取锁是否失败，初始化为 true，表示默认获取失败
    boolean failed = true;
    try {
        // interrupted 变量用于标记线程在等待过程中是否被中断，初始化为 false
        boolean interrupted = false;
        
        // 无限循环尝试获取锁
        for (;;) {
            // 获取当前节点的上一个节点
            final Node p = node.predecessor();
            /*
             若上一节点是头节点，说明当前线程是队列中第一个等待的线程，
             此时调用 tryAcquire(arg) 方法尝试获取锁。tryAcquire 方法由子类实现，
             用于尝试以独占模式获取锁
             */
            if (p == head && tryAcquire(arg)) {
                // 若获取锁成功，将当前节点设置为头节点，意味着该线程已获取到锁，从等待队列中移除
                setHead(node);
                // 将原头节点的 next 指针置为 null，帮助垃圾回收器回收原头节点
                p.next = null; // help GC
                // 标记获取锁成功
                failed = false;
                return interrupted;
            }
            
            // 满足阻塞条件情况下，阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 获取锁失败才会执行 - 对应的是上面的代码出现异常时的场景
        if (failed)
            // 取消当前线程的获取操作，将该节点从同步队列移除
            cancelAcquire(node);
    }
}
```
- `shouldParkAfterFailedAcquire(p, node)`：判断当前线程在获取锁失败后是否应该阻塞。该方法会检查前驱节点的状态，若需要等待信号则返回 true
- `parkAndCheckInterrupt()`：若应该阻塞，调用此方法阻塞当前线程，直到被唤醒。线程被唤醒后，检查是否被中断，若被中断则返回 true
- `interrupted = true`：若线程在阻塞过程中被中断，将 `interrupted` 标记为 true。能走到这一步说明`parkAndCheckInterrupt()` 里面的 `Thread.interrupted()` 返回的线程状态确实为 true 被中断的状态
- `cancelAcquire(node)`：该方法放在 `finally` 处，只有当方法出现异常、或者 return 的时候才会走。由于此处加了 failed 判断条件，而正常走 return 处的逻辑时 `failed = false；`所以该方法只有当出现异常的时候才会走了。该方法的用途是当获取锁失败，调用 `cancelAcquire(node)` 方法取消当前线程的获取操作，将节点从等待队列中移除

##### （a）shouldParkAfterFailedAcquire(p, node)

`shouldParkAfterFailedAcquire` 方法根据前驱节点的等待状态，决定当前线程在获取锁失败后是否应该阻塞。它会处理前驱节点状态为 `SIGNAL`、`CANCELLED` 以及其他状态的情况，确保线程能正确地等待或重新尝试获取锁。返回 false，表示当前线程不应该阻塞，需要重新尝试获取锁，返回true，表示要阻塞

```java
// Node 的等待状态
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;


/**
* node：为封装当前操作线程且成功添加到队列尾部的节点
* pred：node的上一个节点
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱节点 pred 的等待状态 waitStatus，该状态有多种取值，如 SIGNAL、CANCELLED 等
    int ws = pred.waitStatus;
    
    // 若前驱节点的状态为 SIGNAL，意味着前驱节点已经表明在释放锁时会通知后继节点
    if (ws == Node.SIGNAL)
        return true;
        
    // 若 ws > 0，说明前驱节点的状态为 CANCELLED（值为 1），即前驱节点因超时或中断已被取消
    if (ws > 0) {
        /*
         * 通过 do-while 循环跳过所有状态为 CANCELLED 的前驱节点，
         * 直到找到一个状态不为 CANCELLED 的节点 
         */   
        do {
            /*
             此处相当于下面两步 
                 pred = pred.prev;    pred 指向上上节点    
                 node.prev = pred;    node的上一个节点指向
             */
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        
        // 将找到的这个非CANCELLED节点的 next 指针指向当前节点 node
        pred.next = node;
    } else {
        /*
         * 1.若前驱节点状态既不是 SIGNAL 也不是 CANCELLED，那么其状态大概率是初始化0 或 PROPAGATE
         * 2.调用 compareAndSetWaitStatus 方法，尝试以原子操作将前驱节点的状态更新为 SIGNAL，
         *   表明当前节点需要前驱节点在释放锁时发出信号
         * 3.此时当前线程不应该立即阻塞，调用者需要重新尝试获取锁，确认无法获取后再考虑阻塞，
         *   所以返回 false
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

**流程图：**
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6fc091ad0da9410f9f84c14368853b94.png#pic_center)


##### （b）parkAndCheckInterrupt() - 阻塞当前线程

`parkAndCheckInterrupt` 方法的核心逻辑是先让当前线程进入阻塞状态，等待被唤醒或中断，然后检查线程是否在阻塞期间被中断过，同时清除中断状态，最后返回检查结果。这个方法在 `AbstractQueuedSynchronizer` 类的锁获取逻辑里经常被使用，用于处理线程的阻塞和中断情况

```java
private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程
    LockSupport.park(this);
    // 直到线程被中断、或是唤醒之后，继续执行下面的获取线程是否中断
    return Thread.interrupted();
}
```

- 调用 `LockSupport.park(this)` 方法使当前线程进入阻塞状态。`LockSupport` 是 Java 并发包中的一个工具类，park 方法可让线程暂停执行，进入等待状态，直到满足以下任一条件才会恢复执行：
  - 其他线程调用 `LockSupport.unpark` 方法，将当前线程作为参数传入。
  - 其他线程中断当前线程。
  - 该方法毫无理由地返回。
- 调用 `Thread.interrupted()` 方法检查当前线程是否被中断过，并且会清除线程的中断状态。若线程在阻塞期间被中断，该方法返回 true；若未被中断，返回 false

**说明：当线程被唤醒之后，需要执行 `Thread.interrupted()` 来返回线程的中断状态，这是为什么呢？
这个和线程的中断协作机制有关系，线程被唤醒之后，并不确定是被中断唤醒，还是被 `LockSupport.unpark()` 唤醒，因此需要通过线程的中断状态来判断**

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    // 阻塞线程
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```
- `setBlocker(t, blocker);`：调用 `setBlocker` 方法，将 `blocker` 对象记录到当前线程中，用于后续监控和诊断。
- `UNSAFE.park(false, 0L);`：调用 `Unsafe` 类的 `park` 方法阻塞当前线程。false 表示使用相对时间，0L 表示无限期阻塞，直到满足唤醒条件。
- `setBlocker(t, null);`：线程被唤醒后，将当前线程的 `blocker` 对象置为 null，清除记录。


##### （c）cancelAcquire(Node node)
`cancelAcquire` 方法的主要目的是将一个正在尝试获取锁的节点从同步队列中移除，并处理好相关节点的状态和链接关系，确保队列的正确性和信号的正常传递

```java
// Node 的等待状态
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;

private void cancelAcquire(Node node) {
    // 空节点检查
    if (node == null)
        return;

    // 清除线程引用，表示该节点不再关联任何线程，帮助垃圾回收
    node.thread = null;

    // 从当前节点的前驱节点开始向前遍历，跳过所有状态为 CANCELLED（waitStatus > 0）的节点，
    // 直到找到一个状态不为 CANCELLED 的节点，将其作为当前节点的新前驱节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // 记录新前驱节点的后继节点，后续的 CAS 操作会基于这个节点进行
    // 如果操作失败，说明有其他线程同时进行了取消或信号操作，此时无需再进行额外操作
    Node predNext = pred.next;

    // 将当前节点的 waitStatus 设为 CANCELLED，表示该节点已取消获取锁的操作
    // 由于这一步不需要考虑并发问题，所以直接赋值，无需使用 CAS 操作。
    node.waitStatus = Node.CANCELLED;

    /**
     * 处理当前节点为尾节点的情况
     * 如果当前节点是队列的尾节点，使用 CAS 操作尝试将队列的尾节点更新为其新前驱节点。
     * 若更新成功，再使用 CAS 操作将新前驱节点的后继节点置为 null，从而将当前节点从队列中移除
     */
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        /**
         * 若当前节点不是尾节点，分两种情况处理：
         * 1.需要传递信号：若新前驱节点不是头节点，且其状态为 SIGNAL 或者可以将其状态更新为 SIGNAL，
         *   并且新前驱节点关联着线程，就尝试将新前驱节点的后继节点更新为当前节点的后继节点，以保证信
         *   号能正确传递。
         * 2.不需要传递信号：若不满足上述条件，调用 unparkSuccessor 方法唤醒当前节点的后继节点，
         *   让其继续尝试获取锁。
         */
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        // 最后，将当前节点的 next 指针指向自身，帮助垃圾回收器回收该节点。
        node.next = node; // help GC
    }
}
```



### 1.6 lock.unlock() 锁释放

该方法可用于实现 Lock 接口的 unlock 方法

```java
public void unlock() {
    sync.release(1);
}
```

#### 1.6.1 release(1) - 释放锁

release 方法的主要逻辑是尝试释放锁，若释放成功，会检查等待队列的头节点，若有必要则唤醒头节点的后继节点对应的线程。最后返回锁是否成功释放的结果。该方法为实现 Lock 接口的 unlock 方法提供了基础。

```java
// Node 的等待状态
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;


public final boolean release(int arg) {
    // 尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒等待队列中的后继线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

- `if (tryRelease(arg))`：调用 `tryRelease` 方法尝试释放同步状态。`tryRelease` 是一个受保护的抽象方法，由子类实现。若释放成功，方法返回 true，表示释放成功；若 `tryRelease` 方法返回 false，表示释放失败，返回 false
- `if (h != null && h.waitStatus != 0)`：若头节点不为空且其等待状态不为 0（等待状态不为 0 表示头节点对应的线程可能需要被唤醒），则调用 `unparkSuccessor` 方法唤醒头节点的后继节点对应的线程。
- `unparkSuccessor(h)`：调用 `unparkSuccessor` 方法唤醒头节点的后继节点对应的线程。

#### 1.6.2 tryRelease(1) 尝试释放锁

`tryRelease` 方法的主要功能是尝试释放指定次数的锁。它会先检查当前线程是否为锁的持有者，若不是则抛出异常；接着计算释放后的锁重入次数，若重入次数为 0 则标记锁已完全释放，并清除锁的持有者信息；最后更新锁的状态并返回锁是否完全释放的结果。整个锁完全有 state 锁计数器来控制

```java
protected final boolean tryRelease(int releases) {
    // 计算释放指定次数后的锁重入次数。getState() 方法返回当前锁的重入次数
    int c = getState() - releases;
    
    // 检查当前线程是否为锁的持有者
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    
    // 标记锁是否已完全释放，初始为 false
    boolean free = false;
    
    // 若释放后锁的重入次数为 0，说明锁已完全释放
    if (c == 0) {
        free = true;
        // 将持有锁的线程置为 null，表示没有线程持有该锁
        setExclusiveOwnerThread(null);
    }
    
    // 更新锁的重入次数
    setState(c);
    return free;
}
```

有两个地方要注意一下：
- 为什么要做线程检查 `Thread.currentThread() != getExclusiveOwnerThread()`，如果不做会有什么影响：是否其他线程可以在加锁前随意执行 `tryRelease` 方法，将锁的计数器 state 扣减完释放掉
- `tryRelease`释放锁并不是在公平锁、非公平锁子类去分别实现，而是直接父类 `Sync` 中实现。而在加锁的时候，公平锁、非公平锁是单独在子类实现

#### 1.6.3 unparkSuccessor(h) - 唤醒头部开始的节点

`unparkSuccessor` 方法的核心逻辑是先尝试清除目标节点的等待状态（也就是设置为0 - 节点初始化的状态），然后查找其实际的非取消后继节点，最后唤醒该后继节点对应的线程。该方法在同步器释放资源时，用于唤醒等待队列中的下一个线程

```java
// Node 的等待状态
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;


/**
 * 唤醒指定节点的后继节点对应的线程（如果存在）
 * @param node 需要查找后继节点并唤醒其线程的目标节点
 */
private void unparkSuccessor(Node node) {
    // 获取当前节点的等待状态
    int ws = node.waitStatus;
    if (ws < 0)
        // 若等待状态为负数，尝试通过 CAS 操作将其置为 0
        compareAndSetWaitStatus(node, ws, 0);

    // 获取当前节点的直接后继节点
    Node s = node.next;
    // 若直接后继节点为空或已取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        
        // 从队列尾部开始向前遍历
        for (Node t = tail; t != null && t != node; t = t.prev)
            // 找到的是离 node 最近的非取消节点
            if (t.waitStatus <= 0)
                s = t;
    }
    
    // 若找到了合适的后继节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
```


**为什么找离 node 最近的非取消节点，从 tail 往前遍历，而不是直接从node节点往后开始找？**
个人理解哈，从尾部往前遍历可以更准确地找到应该被唤醒的线程。由于队列操作的并发特性，从后往前遍历能避免遗漏那些已经入队但 next 指针还未正确设置的节点。这样能保证唤醒操作的正确性

.
**为什么在 AQS 的 release 方法里，unparkSuccessor(h) 被调用时传入的是队列头节点 head。那头节点 head 的状态被设为 0 后，后续为何还可能执行 unparkSuccessor(h)？**

```java
// Node 的等待状态
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        // 这里条件是，head 结点不能是初始化的 0 状态
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// 此处唤醒时，始终传入的是 head 节点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    // ... 其他逻辑
}
```

**后续仍可能执行 unparkSuccessor(h) 的原因：**

**a）新节点入队并改变头节点状态**

此处我们虽然是说的排它锁，但是为了整体的完整性和连贯性，这里将共享锁释放的逻辑也拿过来

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);
            }
            // 此处会设置头结点的状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head)
            break;
    }
}
```


**b）头节点被替换**

当某个线程成功获取锁后，会更新头节点。原头节点出队，新的节点成为头节点，新头节点的状态可能不为 0，从而满足 release 方法中调用 `unparkSuccessor` 的条件

```java
final boolean acquireQueued(final Node node, int arg) {
    // ... 其他逻辑
    final Node p = node.predecessor();
    if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
    }
    // ... 其他逻辑
}

private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

下载下来浏览器打开自己操作一下：[ReentrantLock多线程获取锁流程详解.html
](https://github.com/fkfengye/MyNotes/blob/main/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/file/ReentrantLock%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%8E%B7%E5%8F%96%E9%94%81%E6%B5%81%E7%A8%8B%E8%AF%A6%E8%A7%A3.html)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b73d99bb434c4e468ec777f2596ed259.png#pic_center)



### 2、共享锁

`ReentrantLock` 主要用于实现独占锁，而 Java 标准库中的另外一个类 `ReentrantReadWriteLock` 可用于实现共享锁和独占锁的功能。其中读锁是共享锁，允许多个线程同时读取共享资源；写锁是独占锁，同一时间只允许一个线程进行写操作


#### 2.1 以ReentrantReadWriteLock 实现共享读锁的示例代码

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class SharedLockDemo {
    // 初始化 ReentrantReadWriteLock 实例
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    // 获取读锁
    private final ReentrantReadWriteLock.ReadLock readLock = rwLock.readLock();
    // 模拟共享资源
    private int sharedResource = 0;

    /**
     * 读取共享资源的方法
     * @return 共享资源的值
     */
    public int readSharedResource() {
        // 获取读锁
        readLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 获取读锁，正在读取共享资源");
            // 模拟读取操作耗时
            Thread.sleep(100);
            return sharedResource;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return -1;
        } finally {
            // 释放读锁
            readLock.unlock();
            System.out.println(Thread.currentThread().getName() + " 释放读锁");
        }
    }

    public static void main(String[] args) {
        SharedLockDemo demo = new SharedLockDemo();

        // 创建多个读线程
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                int value = demo.readSharedResource();
                System.out.println(Thread.currentThread().getName() + " 读取到的共享资源值为: " + value);
            }, "读线程-" + i).start();
        }
    }
}
```

输出结果：

```java
读线程-0 获取读锁，正在读取共享资源
读线程-1 获取读锁，正在读取共享资源
读线程-2 获取读锁，正在读取共享资源
读线程-3 获取读锁，正在读取共享资源
读线程-4 获取读锁，正在读取共享资源
读线程-0 释放读锁
读线程-1 释放读锁
读线程-1 读取到的共享资源值为: 0
读线程-2 释放读锁
读线程-2 读取到的共享资源值为: 0
读线程-0 读取到的共享资源值为: 0
读线程-4 释放读锁
读线程-4 读取到的共享资源值为: 0
读线程-3 释放读锁
读线程-3 读取到的共享资源值为: 0
```

.
#### 2.2 构造函数 - 默认构造非公平锁

```java
public ReentrantReadWriteLock() {
    this(false);
}

/**
 * 默认构造非公平锁
 * 同时构造读写锁
 */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

- `sync` 是 `ReentrantReadWriteLock` 的内部同步器：继承自 `AbstractQueuedSynchronizer` 根据 `fair` 参数决定创建 `FairSync` 或 `NonfairSync` 实例
- **公平锁 vs 非公平锁**：
    - **公平锁（FairSync）**：按照线程请求锁的顺序分配锁，保证了锁的获取顺序，但可能会降低吞吐量
    - **非公平锁（NonfairSync）**：允许线程插队，如果锁恰好可用，线程可以直接获取而不需要排队，这样可以提高吞吐量，但可能导致某些线程长时间等待
    - 如果你关心线程饥饿问题，应该使用公平锁；如果你更关心系统的整体吞吐量，可以使用非公平锁
- `readerLock` 和 `writerLock` 分别是读锁和写锁的实例，两者都与当前的 `ReentrantReadWriteLock` 实例关联


.
#### 2.3 readLock.lock() 获取锁

```java
public void lock() {
    sync.acquireShared(1);
}
```

- 这是读锁的加锁方法，用于获取共享模式的锁权限。允许多个线程同时持有读锁（只要没有写锁被占用）
- `acquireShared(1)`调用AQS框架的共享锁获取方法，参数1表示获取1个共享单元


.
#### 2.4 AQS 共享锁获取流程 acquireShared(1)

`acquireShared` 方法的主要作用是以共享模式获取锁，在此过程中会忽略线程中断。共享模式意味着多个线程可以同时持有锁，适用于读操作等场景

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

这段代码体现了模板方法模式，`acquireShared`是final方法，定义了算法骨架，而`tryAcquireShared`是抽象方法，需要子类根据具体同步器语义来实现。这种设计使得AQS可以支持多种同步语义，如独占模式（`ReentrantLock`）和共享模式（读写锁的读锁、`Semaphore`等）
- 首先调用`tryAcquireShared(arg)`尝试以共享模式获取锁，该方法由子类实现具体的锁获取逻辑
    - 若返回值`< 0`：表示获取失败，则调用`doAcquireShared(arg)`将线程加入同步队列并阻塞
    - 返回值 `> 0`：成功获取锁且后续线程可能也能获取
    - 返回值 `= 0`：成功获取锁但后续线程无法获取

.
##### 2.4.1 tryAcquireShared(1) - 尝试获取共享锁

该方法是`ReentrantReadWriteLock`内部同步器`Sync`类中的`tryAcquireShared`方法实现，这是读锁获取的核心逻辑，负责以共享模式尝试获取锁权限。
这段代码巧妙地利用了AQS的状态变量设计，通过高低16位分离读写锁计数，实现了高效的读写锁分离控制，同时通过各种优化（如`firstReader`缓存、`ThreadLocal`计数）减少了多线程竞争开销

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 步骤1：检查写锁状态
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
        return -1; // 有其他线程持有写锁，获取失败
    
    // 步骤2：尝试获取读锁
    int r = sharedCount(c);
    
    // 步骤3：读锁获取条件
    if (!readerShouldBlock() && r < MAX_COUNT && compareAndSetState(c, c + SHARED_UNIT)) {
        // 步骤4：处理读锁重入计数
        
        // 首次获取：记录第一个获取读锁的线程，避免ThreadLocal查找开销
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
            
        // 重入计数跟踪：对第一个线程的重入直接计数，减少ThreadLocal访问    
        } else if (firstReader == current) {  
            firstReaderHoldCount++;
        
        // 普通线程计数    
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                // 第一次做初始化操作
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                // 更新计数器
                readHolds.set(rh);
            // 重入次数 +1    
            rh.count++;
        }
        
        // 获取成功
        return 1;
    }
    
    // 步骤5：完整重试逻辑
    return fullTryAcquireShared(current);
}
```


- **步骤1**：检查写锁状态
    - `exclusiveCount(c)`：通过`c & EXCLUSIVE_MASK`计算写锁计数（低16位）
    - 若写锁已被其他线程持有，直接返回-1（AQS规定负数表示获取失败）
    - 允许写锁降级：持有写锁的线程可同时获取读锁
- **步骤2**：尝试获取读锁 `sharedCount(c)`：等价于 `c >>> 16`（高16位为读锁计数）
- **步骤3**：读锁获取条件
    - `readerShouldBlock()`：
        - 公平锁实现：`hasQueuedPredecessors()`（判断同步队列是否有前驱节点）
        - 非公平锁实现：`apparentlyFirstQueuedIsExclusive()`（判断队列头是否为写锁节点）
    - `MAX_COUNT`：读锁最大计数（65535），防止高16位溢出
    - `SHARED_UNIT`：读锁计数单位（1 << 16 = 65536），每次加1实际是状态值+65536
    - CAS操作：原子更新状态变量，保证多线程并发安全
- **步骤4**：普通线程计数部分
    - 通过`ThreadLocal<HoldCounter>`跟踪每个线程的读锁持有次数
    - `cachedHoldCounter`缓存上次访问的线程计数器，减少`ThreadLocal`查找
- **步骤5**：失败处理逻辑。当快速路径（CAS）失败时，调用`fullTryAcquireShared(current)`进入完整重试逻辑，处理
    - 读锁重入（当前线程已持有读锁）
    - CAS竞争失败后的重试
    - 读锁计数溢出检查
    - 队列阻塞策略的完整实现

###### （a）锁的几种同步状态

```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

// 提取读锁计数（高16位）
static int sharedCount(int c) { return c >>> SHARED_SHIFT; }
// 提取写锁计数（低16位）
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

- `SHARED_SHIFT = 16`
    - 作用：定义读锁（共享锁）在`state`状态变量中的位移量。
    - 设计逻辑：`ReentrantReadWriteLock`使用一个32位的int类型变量`state`同时记录读锁和写锁的重入次数：
        - 高16位：存储读锁（共享锁）的重入次数
        - 低16位：存储写锁（独占锁）的重入次数
    - 示例：通过`state >>> SHARED_SHIFT`（无符号右移16位）可提取高16位的读锁计数。
- `SHARED_UNIT = (1 << SHARED_SHIFT)`
    - 计算值：1 << 16 = 65536
    - 作用：读锁的计数单位。每次获取读锁时，`state`需要增加`SHARED_UNIT`（即高16位加1）。
    - 示例：获取一次读锁时，执行`state += SHARED_UNIT`，相当于高16位的读锁计数加1。
- `MAX_COUNT = (1 << SHARED_SHIFT) - 1`
    - 计算值：`2^16 - 1 = 65535`
    - 作用：读写锁的最大可重入次数限制（读锁和写锁各自最多65535次重入）。
    - 保护逻辑：若重入次数超过此值（如`sharedCount(c) >= MAX_COUNT`），会抛出Error("Maximum lock count exceeded")。
- `EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1`
    - 计算值：同`MAX_COUNT（65535）`
    - 作用：写锁的掩码。通过`state & EXCLUSIVE_MASK`可提取低16位的写锁计数。
    - 示例：`exclusiveCount(c) = c & EXCLUSIVE_MASK`，直接获取低16位的写锁重入次数。



###### （b）readerShouldBlock()

判断同步队列首个等待节点是否为独占模式。该方法返回 true 时，表示同步队列中第一个有效等待节点是独占模式。这一判断主要用于：
- 公平锁策略：在尝试获取锁时，若队列中已有等待的独占节点，当前线程需让步，避免“插队”
- 同步器状态校验：辅助 `tryAcquire`、`release` 等方法决策是否允许当前线程获取/释放锁

```java
final boolean apparentlyFirstQueuedIsExclusive() {
    // 分别用于存储同步队列的头节点（head）和头节点的下一个节点
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

**return 判断逻辑：**

- `(h = head) != null`：头节点 head 存在（队列非空）
- `(s = h.next) != null`：头节点的下一个节点 s 存在（队列中至少有一个等待节点）
- `!s.isShared()`：节点 s 不是共享模式（即独占模式，如 `ReentrantLock` 的写锁）
- `s.thread != null`：节点 s 关联的线程未被取消（`thread` 不为空表示有效等待）



###### （c）fullTryAcquireShared - 完整的共享锁获取逻辑

`fullTryAcquireShared` 是读锁获取的“完整路径”实现，通过循环重试、状态校验和原子操作，处理了写锁降级、公平策略、重入计数等复杂场景，是`ReentrantReadWriteLock`支持“**多读单写**”高并发特性的核心逻辑之一。该方法是 `Sync` 内部类的成员，属于 AQS（`AbstractQueuedSynchronizer`）共享模式的扩展实现，主要用于：
- 处理读锁的重入逻辑（支持写锁持有者降级为读锁）
- 校验读锁是否需要阻塞（基于公平/非公平策略）
- 确保读锁计数不超过最大值（`MAX_COUNT`）

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) { // 无限循环：CAS操作的重试机制
        int c = getState(); // 获取当前锁状态（高16位读锁计数，低16位写锁计数）

        // 步骤1. 检查是否存在写锁（独占锁）
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1; // 写锁被其他线程持有，获取读锁失败
            // 否则：当前线程持有写锁（允许降级为读锁，不阻塞）
        }

        // 步骤2. 检查读锁是否需要阻塞（公平策略决定）
        else if (readerShouldBlock()) {
            if (firstReader == current) {
                // 当前线程是第一个读锁持有者（允许重入）
            } else {
                // 非第一个读锁持有者：检查当前线程的读锁持有计数
                if (rh == null) rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current)) {
                    rh = readHolds.get(); // 从ThreadLocal获取当前线程的HoldCounter
                    if (rh.count == 0) readHolds.remove(); // 无持有计数则清理
                }
                if (rh.count == 0) return -1; // 无重入资格，获取失败
            }
        }

        // 步骤3. 检查读锁计数是否超过最大值（65535）
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");

        // 步骤4. 尝试CAS更新读锁计数（+SHARED_UNIT=65536）
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;     // 第一个读锁持有者
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;     // 第一个读锁持有者重入
            } else {
                // 非第一个读锁持有者：更新ThreadLocal中的HoldCounter
                if (rh == null) rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // 缓存HoldCounter，优化释放时的性能
            }
            return 1; // 获取成功
        }
    }
}
```

- **步骤1**：写锁降级支持
    - 若当前线程已持有写锁（`exclusiveCount(c) != 0`且`getExclusiveOwnerThread() == current`），允许其获取读锁（写锁→读锁的降级），避免死锁
- **步骤2**：公平策略校验，`readerShouldBlock()` 由具体同步器（`FairSync`/`NonfairSync`）实现：
    - 公平锁：检查同步队列中是否有前驱节点（`hasQueuedPredecessors()`），避免“插队”
    - 非公平锁：直接返回false（允许竞争）
- **步骤4**：读锁重入计数
    - `firstReader`：记录第一个获取读锁的线程（优化高频单线程读场景）
    - `readHolds`：`ThreadLocal<HoldCounter>`，存储其他线程的读锁重入次数（避免多线程竞争）
    - `cachedHoldCounter`：缓存最近使用的`HoldCounter`，减少`ThreadLocal.get()`的开销

.
##### 2.4.2 doAcquireShared - 共享锁加入队列等等

该方法是`AQS`的私有方法，属于共享模式获取锁的底层实现，与独占模式的 `acquireQueued` 对应。其核心职责是：当线程无法直接获取共享锁时，将线程加入同步队列等待，并在条件满足时唤醒。
该方法与独占模式的 `acquireQueued` 的主要区别在于：共享模式在获取成功后可能唤醒多个后续节点（通过`setHeadAndPropagate`），而独占模式仅唤醒一个节点（通过`unparkSuccessor`）

```java
private void doAcquireShared(int arg) {
    // 步骤1. 创建共享模式等待节点并加入队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        
        // 步骤2. 无限循环尝试获取锁
        for (;;) {
            final Node p = node.predecessor(); // 获取当前节点的前驱节点
            // 步骤3. 前驱是头节点（当前节点是队列中的第一个有效节点）
            if (p == head) {
                int r = tryAcquireShared(arg); // 尝试获取共享锁（由子类实现）
                if (r >= 0) { // 获取成功（r >=0 表示剩余许可足够）
                    // 步骤4. 更新头节点并传播唤醒
                    setHeadAndPropagate(node, r);
                    p.next = null; // 断开前驱节点的next引用（帮助GC）
                    if (interrupted) // 处理中断标记
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 步骤5. 获取失败时，检查是否需要阻塞当前线程
            if (shouldParkAfterFailedAcquire(p, node) && // 判断是否需要阻塞
                parkAndCheckInterrupt()) // 阻塞并检查中断
                interrupted = true;
        }
    } finally {
        if (failed) // 6. 异常或失败时取消当前节点的获取
            cancelAcquire(node);
    }
}
```

该方法就不详细说了，对比 **acquireQueued** 方法的源码解析来看


.

#### 2.5 readLock.unlock() 释放读锁

```java
public void unlock() {
    sync.releaseShared(1);
}
```

`releaseShared` 该方法是AQS（抽象队列同步器）中共享锁释放的核心入口方法，用于释放共享模式下的锁资源，并唤醒后续等待共享锁的线程

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

该方法体现了AQS对共享锁的**释放-传播**机制：通过模板方法`tryReleaseShared`将具体释放逻辑委托给子类，同时通过`doReleaseShared`统一处理队列唤醒，保证了共享锁场景下多线程唤醒的公平性和效率
- `tryReleaseShared(arg)` 尝试释放共享锁
    - 这是一个由子类（如`ReentrantReadWriteLock`的读锁）实现的模板方法，负责实际释放共享锁的状态（例如减少读锁的重入计数）。若返回true，表示共享锁已成功释放，需要进一步唤醒后续等待线程
- `doReleaseShared()` 唤醒后续共享节点
    - 若`tryReleaseShared`返回true，则调用此方法。其核心作用是：通过CAS操作设置头节点状态（`Node.SIGNAL`），并唤醒头节点的后继节点（`Node.next`），确保共享锁的释放能够传播给多个后续等待共享锁的线程（与独占锁仅唤醒一个线程不同）
- 返回值含义
    - 返回true：共享锁释放成功且已触发唤醒逻辑；
    - 返回false：共享锁未成功释放（`tryReleaseShared`返回false，可能因状态未满足释放条件）


##### 2.5.1 tryReleaseShared(arg) 尝试释放共享锁

该方法是 ReentrantReadWriteLock 中读锁（共享锁）释放的核心实现，通过维护线程本地的重入计数和原子更新同步状态（state），实现了读锁的可重入性和共享释放逻辑，是ReentrantReadWriteLock读写锁协作的核心机制之一

```java
/**
 * 尝试释放共享读锁（重入计数递减）。
 * 此方法由AQS的releaseShared调用，用于更新线程本地的读锁重入计数，并原子性更新同步状态。
 * @param unused 未使用的参数（共享锁释放通常不需要具体参数）
 * @return 若所有读锁和写锁均已释放（state=0），返回true，触发后续唤醒等待线程
 */
protected final boolean tryReleaseShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();

    // 情况1：当前线程是第一个获取读锁的线程（优化常见场景）
    if (firstReader == current) {
        // 断言：第一个读线程的重入计数应大于0
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1) {
            // 若重入计数为1（最后一次释放），清空第一个读线程记录
            firstReader = null;
        } else {
            // 否则递减重入计数
            firstReaderHoldCount--;
        }
        
    // 情况2：当前线程不是第一个读线程，通过缓存或ThreadLocal获取重入计数器    
    } else {
        HoldCounter rh = cachedHoldCounter;
        // 缓存不匹配（缓存的线程ID与当前线程不一致）
        if (rh == null || rh.tid != getThreadId(current)) {
            // 从ThreadLocal中获取当前线程的HoldCounter（存储重入计数）
            rh = readHolds.get();
        }
        // 获取当前线程的重入计数
        int count = rh.count;
        if (count <= 1) {
            // 若重入计数≤1（最后一次释放），从ThreadLocal中移除，避免内存泄漏
            readHolds.remove();
            // 防御性检查：若计数≤0，说明未匹配的unlock操作（异常）
            if (count <= 0) {
                throw unmatchedUnlockException();
            }
        }
        // 递减当前线程的重入计数
        --rh.count;
    }

    // 循环CAS更新AQS的同步状态（state）
    for (;;) {
        // 获取当前同步状态（高16位为读锁计数，低16位为写锁计数）
        int c = getState();
        // 计算释放后的状态（读锁计数减1单位，即SHARED_UNIT=1<<16）
        int nextc = c - SHARED_UNIT;
        // 原子更新状态（CAS保证线程安全）
        if (compareAndSetState(c, nextc)) {
            // 仅当所有锁（读+写）均释放时（nextc=0），返回true以触发后续唤醒
            return nextc == 0;
        }
    }
}
```

- 线程本地计数管理：通过`firstReader`（优化单线程重复获取）和`readHolds`（`ThreadLocal`存储其他线程计数）维护各线程的读锁重入次数
- 内存泄漏预防：当线程的重入计数`≤1`时，从`readHolds`中移除计数器，避免无用对象残留
- 原子状态更新：通过循环CAS确保同步状态（`state`）的原子性修改，仅当所有锁释放时触发唤醒，保证写锁的独占性
- 锁语义保证：仅当所有读锁释放（`state == 0`）时才唤醒写线程，确保写锁的“**独占性**”——写锁需等待所有读锁释放后才能获取



##### 2.5.2 doReleaseShared() 唤醒后续共享节点

`doReleaseShared` 是实现传播唤醒机制的核心方法。`AQS`通过一个双向链表（同步队列）管理等待锁的线程节点（`Node`）。共享锁（如`ReentrantReadWriteLock`的读锁）允许多个线程同时持有锁，因此释放共享锁时需要唤醒多个后续共享节点，而非像独占锁（如`ReentrantLock`）仅唤醒一个节点。

独占锁释放（如`ReentrantLock`的`release`方法）通过`unparkSuccessor`唤醒单个后继节点，而共享锁释放（`doReleaseShared`）通过循环和`PROPAGATE`状态唤醒多个后续共享节点，确保共享模式下多线程的并发获取能力。

```java
private void doReleaseShared() {  // 共享模式下释放锁的传播唤醒方法
    /*
     * 设计目标：确保释放操作的传播性，即使存在并发的获取/释放操作。
     * 核心逻辑：循环检查头节点状态，若需唤醒则执行unparkSuccessor；若无需唤醒但状态为0，则标记为PROPAGATE以保证后续传播。
     */
     
    // 步骤1：无限循环处理，直到头节点未变化时退出
    for (;;) {
        Node h = head;  // 获取当前同步队列的头节点（虚拟头节点，不存储实际线程）

        // 步骤2：头节点存在且不是尾节点（队列中至少有一个实际等待节点）
        if (h != null && h != tail) {
            int ws = h.waitStatus;  // 获取头节点的等待状态（Node的waitStatus字段）

            // 分支1：头节点状态为SIGNAL（表示后继节点需要被唤醒）
            if (ws == Node.SIGNAL) {
                // 尝试CAS将头节点状态从SIGNAL重置为0（避免并发干扰）
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;  // CAS失败（被其他线程修改），重试循环
                unparkSuccessor(h);  // 唤醒头节点的后继节点（实际等待线程）
            }

            // 分支2：头节点状态为0（无等待唤醒需求），尝试标记为PROPAGATE
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;  // CAS失败（被其他线程修改），重试循环
        }

        // 步骤3：退出循环条件 - 头节点未被其他线程修改（h == head说明当前循环处理有效，无需继续）
        if (h == head)
            break;
    }
}
```

- **步骤1**：无限循环的作用，通过 `for (;;)` 无限循环处理头节点状态，确保以下两种情况被覆盖：
    - 并发修改头节点：当多个线程同时释放共享锁时，头节点可能被其他线程修改（如`acquireShared`成功后更新头节点），需要重试以保证一致性
    - 传播唤醒需求：即使当前头节点无需唤醒，标记为`PROPAGATE`后，后续释放操作仍可通过该状态继续传播唤醒
- **步骤2**：头节点状态的处理，头节点的 `waitStatus` 字段是核心状态标识，影响唤醒逻辑：
    - `Node.SIGNAL`（值为-1）：表示头节点的后继节点被阻塞（`park`），需要被唤醒。此时通过 `unparkSuccessor(h)` 唤醒后继线程，使其尝试获取共享锁
    - `Node.PROPAGATE`（值为-3）：标记共享锁的释放需要传播到后续节点。当多个共享锁同时释放时，该状态确保即使当前头节点无直接后继需要唤醒，后续释放操作仍会继续检查并唤醒更多节点
    - `状态为0`：头节点无等待唤醒需求，但通过 `compareAndSetWaitStatus(h, 0, Node.PROPAGATE)` 标记为`PROPAGATE`，为后续可能的共享锁释放保留传播能力
- **CAS操作的必要性**：`compareAndSetWaitStatus（CAS）`是原子操作，用于确保多线程并发修改头节点状态时的线程安全。若CAS失败（说明状态被其他线程修改），则通过 `continue` 重试循环，重新获取最新的头节点状态
- **步骤3**：退出循环的条件
    - 当 `h == head` 时退出循环，说明当前循环处理的头节点未被其他线程修改，本次释放操作的传播逻辑已完成。若头节点被修改（如其他线程唤醒了新的节点并更新了头），则继续循环处理新的头节点



.

**设计意义：**
`doReleaseShared` 的核心设计是共享锁的传播唤醒机制，通过以下两点保证高效并发：
- **避免重复唤醒**：仅当头节点状态为`SIGNAL`时唤醒后继，避免无意义的`unpark`操作
- **传播性保证**：通过`PROPAGATE`状态标记，确保多个共享锁释放时，唤醒操作能持续传播到后续所有共享节点（如读锁允许多个线程同时持有，需唤醒所有等待的读线程）


.

**下载下来自己打开浏览器操作一下：**

* 如果只有读锁的情况下的变化：[ReentrantReadWriteLock 共享读锁流程详解.html](https://github.com/fkfengye/MyNotes/blob/main/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/file/ReentrantReadWriteLock%20%E5%85%B1%E4%BA%AB%E8%AF%BB%E9%94%81%E6%B5%81%E7%A8%8B%E8%AF%A6%E8%A7%A3.html)
* 读写都有的情况下：[ReentrantReadWriteLock 读写锁流程详解.html](https://github.com/fkfengye/MyNotes/blob/main/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/file/ReentrantReadWriteLock%20%E8%AF%BB%E5%86%99%E9%94%81%E6%B5%81%E7%A8%8B%E8%AF%A6%E8%A7%A3.html)


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f8a5347416f3427797a6cd6161c70cac.png#pic_center)
