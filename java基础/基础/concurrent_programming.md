# Java并发编程探索

### Java并发分类
Java并发分为进程内的并发与进程之间的并发

1、进程内的并发

Java语言原生提供两种同步机制，这种同步机制核心思想是线程内存共享和原子操作

- 加在对象头上的对象监视器synchronized

- 基于共享变量的Lock

>前者是语言级别的，使用synchronized的时候对于的Object对象内存中的头会被设定为加锁状态，其他线程进入该同步代码块则必须等待；后者是代码级别的，使用共享变量的原子操作实现锁的性质。
实际上理论认为在竞争不是很激烈的情况下对象监视器的性能会优于Lock，在竞争非常激烈的情况下则相反。这里要说一下Object的wait()方法，object.wait()方法一般和synchronized(object)一起出现，注意这里的object必须是同一个对象，wait的意思是：放弃当前的锁，让出cpu，进入等待状态
---

2、进程之间的并发

进程之间的并发就需要进程锁来实现多个进程之间的共享问题。核心思想：

- 使用一个可以被共享的节点来作为锁的容器（共享一般通过网络方式）
- 使用这些容器的原子操作保证锁只能同一时间被一个任务持有

>单机情况下一个简单的进程锁实现可以是一个共享文件。简单的说，在我进入临界区操作之前先判断一下文件系统某个目录下是否存在一个.lock文件，如果文件存在则说明已经有其他进程进入了临界区，我需要等待，如果不存在我就创建一个.lock文件然后进入操作；相比于线程级别的锁，这种方式简单有效（低效）的处理多个进程之间的同步问题；
多机情况下简单有效的方式是通过进程外的共享内存（redis）来实现分布式锁
---

### 排队与锁的基本概率

#### 1、背景

计算机系统的发展经历单道批处理系统、多道批处理系统、分时系统。

>多道批处理系统和单道批处理系统的差别在于系统同一时间运行多道程序，利用错＂程序执行的方式充分利用CPU时间，分时系统的原理其实跟多道批处理类似，也是错开任务执行来提供CPU的利用率,只是多道批处理没有CPU分时管理的概念，它使用多个任务组成一个后备队列的方式来管理CPU。
分时系统关注的是CPU时间的管理，系统不考虑业务的情况，只关注CPU时间，不管业务办没办完，时间到了就需要停下手头的工作。
这种方式的好处是CPU时间管理相对简单，不需要考虑任务本身特性，利用率稳定（未必高），多个任务之间相对并行，坏处也很明显,时间片到了就需要进行上下文的切换
---

目前的很多系统都算分时系统，分时系统下

- 任务是可以并行执行的；
- 任务是可能被粗暴的打断的，这种打断会发生在任何时候，是不可预知的，比如分配的时间片结束了，再比如收到一个外部的中断信号等；

#### 2、排队和锁

>看到上面的内容，我们大致明白了多任务系统（多进程／多线程）系统的一些特征：任务在遇到共享资源操作的时候就需要排队，很多时候我们称之为同步，一个简单的理解：同步（动词）就是将这些异步的（形容词）线程／进程排个队。
这种避免资源竞争造成的保护机制我们统称为锁，实际上我觉得锁不是任何一种东西，而是一种避免发生混乱的机制，只要这个机制能保证多任务操作共享数据不会造成不一致我们都能称这种机制为锁；我们说锁的本质是使用等待排队的方式实现异步任务之间的同步以达到保护临界区安全的机制
---

锁的实现：

本质上锁的实现都需要依赖系统的原子操作和不可中断性（比如关中断，指令不是原子的，但可以通过关闭中断来实现一系列指令全部被执行），而实现的方式（调度机制）基本上可以分为两种：非阻塞锁／阻塞锁；

1、非阻塞锁：自旋（spin）

>自旋这个高大上的词曾经误导了我好久，其实这个词很好理解，自旋就是循环等待；当前任务循环等待条件的达成（条件不满足会一直循环占用CPU资源，条件满足则获得锁进入临界区）；
从操作系统层面上讲，自旋解决内核态竞争问题，内核任务对资源进行操作，对于单核CPU来说可以通过关中断的方式来解决，任务在执行的时候不会被中断，保证一系列操作的原子性（全做）；可世事无常，CPU到了SMP（对称多处理器）时代，原来的伪并行变成了真并行，这时情况就变得复杂多了，一个２核的CPU实际上能真并行的处理两个任务，如何保证运行在不同CPU上的任务在共享访问资源的时候不会出错呢？即使关闭了两个核的中断，也没啥鸟用（依然是两个任务并行在执行）
---

>一种思路是大内核锁（BLK），因为之前内核设计并没有考虑多核的情况，让内核支持　SMP需要很多代码，所以理论上BLK是一种过渡机制，BLK的核心思想是当系统进入内核态前就把整个内核锁定，换句话说，内核任务只能运行在一个核上，同一时间只能运行一个内核任务；
使用一个大粒度的锁简单粗暴的锁定内核，放弃内核的并行执行是一个临时解决方案，因为内核的大部分代码其实都还是可以并行执行的，后续的版本迭代逐步的甄别那些可并行的执行代码，选用更小粒度的锁～其中spin_lock是一个选择；
自旋锁定CPU核在内存中的共享数据，在内核中被用来防止多核处理器并发进入临界区；用户态的自旋锁存在不需要上下文转＂的好处，坏处也很明显：占着茅坑不拉屎；循环等待运行空指令是需要消耗CPU时间的；所以自旋锁一般被用来等待一个很小的时间；这个时间多少好呢？一般意义上讲：这个自旋周期选择为上下文切换时间；
---

对于Java而言，用户层面包含两部分：虚拟机JVM实现(C++)的自旋锁和用户自己实现的自旋锁（Java代码）

用户层面自己实现的自旋锁
```
public class SpinLock {

　private AtomicReference<Thread> sign = new AtomicReference<>();

  public void lock(){
    Thread current = Thread.currentThread();
    while(!sign.compareAndSet(null, current)){
    }
  }

  public void unlock (){
    Thread current = Thread.currentThread();
    sign.compareAndSet(current, null);
  }
}
```

2、阻塞锁：列队等待

>上面的自旋锁采用死皮赖脸一直循环等待的方式保护临界区，对于系统层面任务（进程／线程）状态依然是runnable状态；而这里的阻塞锁则有别于这种方式，
操作系统层面的阻塞锁一般有两种：信号量（semaphore）／互斥体（mutex）；
信号量可以理解为一个信号灯（红绿灯），当条件不满足的时候任务（进程／线程）进入等待区域（考虑一下过马路的场景）；信号量本质是一个计数器＋等待列队，他有两种操作，UP／DOWN，我们认为信号量计数器０时资源不可用，任务进入等待列队，信号量＞０时，任务进入临界区，计数器－１；而mutex则关注在互斥问题，可以理解为一个二值信号量；
---

JVM层面，因为多数JVM实现都会将Java线程映射到系统线程，所以自然可以使用系统提供的锁了，上面说过＂对象监视器＂使用mutex来实现；这对于多数开发人员其实也只是一个知识点而已；实际高性能开发者JVM层面的东西跟大多数人都没有太大都关系；

锁的特性：

- 锁的公平性：公平锁与非公平锁
- 锁的可重入性：不可重入锁／可重入锁
- 锁的级别（对象）：无锁，偏向锁，轻量级锁和重量级锁
- 锁的的世界观：乐观锁／悲观锁

### volatile对指令重排的影响
volatile是轻量级的synchronized，但是volatile不会引起线程的上下文切换和调度。

1、volatile的硬件实现原理
- 为了提高处理速度，避免内存IO速度的木桶短板，现代处理器不直接和内存进行通信，而是将内存中的数据读取到CPU的内部高速缓存中（L1，L2，L3等），这里普及一下高速缓存的概念（cache），高速缓存一般集成在CPU内部，保存着CPU刚用过或循环使用的一部分数据，是内存数据的部分拷贝，计算机内部的数据通信为：CPU <--> 寄存器 <--> 高速缓存 <--> 内存
- 对volatile变量的写操作，会在正常汇编指令前加一个lock前缀的指令。lock前缀在多核处理器中会发生以下两件事情：1）lock前缀指令将引起当前处理器缓存的数据写回到系统内存。对于内存中可以缓存并已经缓存的数据，系统不会在总线上声言LOCK#信号而是锁定这块内存区域的缓存并写回到内存中（锁缓存）；如果内存中的数据没有被缓存，那么将在总线上声言LOCK#信号，锁住总线，并将数据写回到内存中。2）处理器的缓存写回到内存中将会导致其他处理器的缓存无效。 多核CPU之间的缓存一致性协议（锁缓存的保证）：每个处理器通过嗅探总线上有其他处理器写内存地址，如果处理器发现正好是自己缓存行所对应的内存地址，就会将当前处理器的缓存行设置成无效状态，当处理器再次访问这个数据时，就会重新将内存中的数据加载到处理器缓存中。当然如果是锁总线，那么就不需要缓存一致性协议来保障，因为锁总线将会独占所有共享内存。
- CPU缓存一致性（MESI 协议及 RFO 请求）

2、volatile的特性

- 可见性：对一个volatile变量的读，总是能看到其他线程对这个变量最新的修改。
- 原子性：volatile变量的单个读/写操作是原子性的且具有可见性，复合操作（依赖当前值的读写复合操作等，比如i++；以及该变量包含在具有其他变量的不变式中）不具有原子性。

3、volatile的内存语义
>java虚拟机内存模型中存在主内存和工作内存之分，实际上并不存在，只是抽象出来的用来表述问题的概念。
主内存中存储的是各个线程共享的数据，每一个线程都有一份与其他线程隔离独立的数据空间叫做工作内存，工作内存一部分存储的是线程私有数据，一部分是主内存中共享数据的拷贝。
---
- 对于普通共享变量：线程持有主内存中共享变量的数据拷贝，当发生读操作时，线程首先在自己的拷贝中查找，如果没有则从主内存中拷贝；发生写操作时，将会修改线程拷贝的数据，而不是主内存中的共享数据，所以无法保证共享变量的可见性。
- volatile的写内存语义：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。和锁synchronized的释放内存语义一致
- volatile的读内存语义：当读一个volatile变量时，JMM会把该线程对应的本地内存设置为无效。线程接下来将从主内存中读取共享变量。和锁synchronized的获取内存语义一致

4、指令重排序与内存屏障

1、指令重排序

指令重排是指JVM在编译Java代码的时候，或者CPU在执行JVM字节码的时候，对现有的指令顺序进行重新排序

```
boolean contextReady = false;

// 线程A中执行
context = loadContext();
contextReady = true;

// 线程B中执行
while( ! contextReady ){ 
   sleep(200);
}
doAfterContextReady (context);

```

对于这段代码，看似没问题，但是，如果线程A执行的代码发生了指令重排，初始化和contextReady的赋值交换了顺序
```
boolean contextReady = false;

// 在线程A中执行:
contextReady = true;
context = loadContext();
 
// 在线程B中执行:
while( ! contextReady ){ 
   sleep(200);
}
doAfterContextReady (context);
```
可以看出指令重排序后会导致结果错误

2、内存屏障

内存屏障共分为四种类型

- LoadLoad屏障：Load1; LoadLoad; Load2。Load1 和 Load2 代表两条读取指令。在Load2要读取的数据被访问前，保证Load1要读取的数据被读取完毕
- StoreStore屏障：Store1; StoreStore; Store2。Store1 和 Store2代表两条写入指令。在Store2写入执行前，保证Store1的写入操作对其它处理器可见
- LoadStore屏障：Load1; LoadStore; Store2。在Store2被写入前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：Store1; StoreLoad; Load2。在Load2读取操作执行前，保证Store1的写入对所有处理器可见。StoreLoad屏障的开销是四种屏障中最大的。

在一个变量被volatile修饰后，JVM会为我们做两件事：

- 在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障。
- 在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障。

从而保障程序的正确执行

volatile关键字只能保证线程读到最新的，但是不能控制线程写的瞬间那个值还是最新的，由此可见volatile适合多线程读的场景，并不适合写的场景

>如果线程1读取了共享的counter变量value 0到他的CPU cache中，增加其到1但还没来得及写回到主存中，线程2也可以从主存中读取同一个counter变量（此时这个变量还是0），线程2也可以令这个counter从0变成1，也还没来得及写回主存。
线程1和线程2现在出现了不同步的现象，counter的真实值应该是2，尽管每个线程都会直接把他们的counter值写回到主存中，但是这个counter的值依然是错误的
---

volatile想象不到的陷阱：不需并发控制的普通共享变量也会因为处于同一缓存行而不能同时被访问

### Java concurrent原理

CAS原子操作
>对于内存中的某一个值V，提供一个旧值A和一个新值B。如果提供的旧值V和A相等就把B写入V。这个过程是原子性的。
CAS执行结果要么成功要么失败，对于失败的情形下一班采用不断重试或者放弃。
---

ABA问题
>如果另一个线程修改V值假设原来是A，先修改成B，再修改回成A。
当前线程的CAS操作无法分辨当前V值是否发生过变化
---

ABA解决办法：添加版本号

ABA问题的一个实例
>小明在提款机，提取了50元，因为提款机问题，
有两个线程，同时把余额从100变为50
线程1（提款机）：获取当前值100，期望更新为50，
线程2（提款机）：获取当前值100，期望更新为50，
线程1成功执行，线程2某种原因block了，这时，某人给小明汇款50
线程3（默认）：获取当前值50，期望更新为100，
这时候线程3成功执行，余额变为100，
线程2从Block中恢复，获取到的也是100，compare之后，继续更新余额为50！！！
此时可以看到，实际余额应该为100（100-50+50），但是实际上变为了50（100-50+50-50）
这就是ABA问题带来的成功提交
---

Java concurrent原理

concurrent有三个包，包含了大概五部分内容

- 原子Value（atomic）
- LOCK 框架（locks）：Lock，Condition，ReadWriteLock等
- 线程池部分：Executor，ExecutorService，Executors工具类
- 并发集合部分：ConcurrentMap，BlockingQueue等
- 同步辅助类：CountDownLatch，CyclicBarrier，Semaphore等

线程池、并发集合、同步辅助类是建立在LOCK框架和atomic的基础之上的，我们有了锁和原子基本类型数据操作之后就能演化出很多复杂的东西

#### 1、原子类
看一个典型的AtomicInteger，一个提供原子操作的Integer类，通过线程安全的方式操作加减

AtomicInteger实现乐观锁

```
public final int incrementAndGet() {
        // 自旋，循环不断尝试compareAndSet操作（乐观锁），典型的乐观锁思路
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
}
```

```
public final boolean compareAndSet(int expect, int update) {
　　　　 // unsafe.compareAndSwapInt 方法是一个native方法，使用CPU原子指令CAS实现
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

atomicXXX使用volatile保持线程之间对值的可见性，以及单值原子操作。但是注意单值原子性不代表CPU之间的同步操作，CPU1对值的修改和CPU2对值的修改还是可以同时的。

![cup_cache](image/cup_cache.png)

由此可知上面的自增操作是不具备原子性的，它包括读取变量的原始值，进行加1操作，写入内存，这里的volatile只能保证值+1操作这一个指令是原子的，三个子操作可能依然会分开执行，多线程对volatile变量操作依然会出现问题
所以需要基于CAS（乐观锁机制）实现三个操作的原子性

jdk1.8 incrementAndGet的实现改成用unsafe来实现了
```
public final int incrementAndGet() {
    // 使用CPU原子指令CAS实现
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

看一个AtomicInteger的实例，摘自IT宅
```
import java.io.File;
import java.io.FileFilter;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicTest {

    private static long randomTime() {
        return (long) (Math.random() * 1000);
    }

    public static void main(String[] args) {

        // 阻塞队列，能容纳100个文件
        final BlockingQueue<File> queue = new LinkedBlockingQueue<>(100);
        // 线程池
        final ExecutorService exec = Executors.newFixedThreadPool(5);

        final File root = new File("D:\\ISO");
        // 完成标志
        final File exitFile = new File("");
        // 原子整型，读个数
        // AtomicInteger可以在并发情况下达到原子化更新，避免使用了synchronized，而且性能非常高。
        final AtomicInteger rc = new AtomicInteger();
        // 原子整型，写个数
        final AtomicInteger wc = new AtomicInteger();

        // 读线程
        Runnable read = new Runnable() {

            public void run() {
                scanFile(root);
                scanFile(exitFile);
            }

            void scanFile(File file) {
                if (file.isDirectory()) {
                    File[] files = file.listFiles(new FileFilter() {
                        public boolean accept(File pathname) {
                            return pathname.isDirectory() || pathname.getPath().endsWith(".iso");
                        }
                    });
                    assert files != null;
                    for (File one : files)
                        scanFile(one);
                } else {
                    try {
                        // 原子整型的incrementAndGet方法，以原子方式将当前值加 1，返回更新的值
                        int index = rc.incrementAndGet();
                        System.out.println("Read0: " + index + " " + file.getPath());
                        // 添加到阻塞队列中
                        queue.put(file);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        };
        // submit方法提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。
        exec.submit(read);

        // 四个写线程
        for (int index = 0; index < 4; index++) {
            // write thread
            final int num = index;
            Runnable write = new Runnable() {
                String threadName = "Write" + num;

                public void run() {
                    while (true) {
                        try {
                            Thread.sleep(randomTime());
                            // 原子整型的incrementAndGet方法，以原子方式将当前值加 1，返回更新的值
                            int index = wc.incrementAndGet();
                            // 获取并移除此队列的头部，在元素变得可用之前一直等待（如果有必要）。
                            File file = queue.take();
                            // 队列已经无对象
                            if (file == exitFile) {
                                // 再次添加"标志"，以让其他线程正常退出
                                queue.put(exitFile);
                                break;
                            }
                            System.out.println(threadName + ": " + index + " " + file.getPath());
                        } catch (InterruptedException ignored) {
                        }
                    }
                }

            };
            exec.submit(write);
        }
        exec.shutdown();
    }
}
```

#### 2、LOCK

1、一些比较

前面我们说过synchronized的线程释放锁的情况有两种:

- 代码块或者同步方法执行完毕
- 代码块或者同步方法出现异常有jvm自动释放锁

>从上面的synchronized释放锁可以看出，只有synchronized代码块执行完毕或者异常才会释放，如果代码块中的程序因为IO原因阻塞了，那么线程将永远不会释放锁，但是此时另外的线程还要执行其他的程序，极大的影响了程序的执行效率，
现在我们需要一种机制能够让线程不会一直无限的等待下去，能够响应中断，这个通过lock就可以办到。另外如果有一个程序，包含多个读线程和一个写线程，我们可以知道synchronized只能一个一个线程的执行，但是我们需要多个读线程同时进行读，那么使用synchronized肯定是不行的，但是我们使用lock同样可以办到
 ---
 
LockSupport与ReentrantLock的区别
 
>LockSupport是基于unsafe类的操作，但是LockSupport是阻塞的，它不会发生Thread.suspend 和 Thread.resume所可能引发的死锁问题。
JDK1.8后，ReentrantLock及ReentrantReadWriteLock是基于AQS实现的，AQS内部也使用了unsafe类进行操作，而AQS是非阻塞机制。
---

LockSupport.park()和unpark()，与object.wait()和notify()的区别

>主要的区别应该说是它们面向的对象不同。阻塞和唤醒是对于线程来说的，LockSupport的park/unpark更符合这个语义，以线程作为方法的参数， 语义更清晰，使用起来也更方便。
而wait/notify的实现使得阻塞/唤醒对线程本身来说是被动的，要准确的控制哪个线程、什么时候阻塞/唤醒很困难， 要不随机唤醒一个线程（notify）要不唤醒所有的（notifyAll）
---

2、Lock框架

>Lock框架的核心是Lock和Condition两个接口
---

锁的实现

>Lock的机制依然是依赖volatile和CAS乐观锁机制, ReentrantLock的实现为例，ReentrantLock内部使用同步器Sync来实现，可以看看lock方法最终会调用Sync的lock
---

1、非公平实现NonfairSync
```
//这个值在Sync的父类中,通过偏移得到,需要注意的是这里的volatile
private volatile int state; 

final void lock() {
    if (compareAndSetState(0, 1))//CAS操作,公平的方式没有这个操作,直接进入acquire
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

2、公平实现FairSync
```
final void lock() {
    acquire(1);
}
```

>acquire之前非公平的实现会尝试一次compareAndSetState（CAS），如果此时正好有释放的锁则不从队列头取线程而是直接获得锁（独占setExclusiveOwnerThread），不在进入队列了（所以不公平的嘛）
---

获取锁

无法acquire到授权，就必须等待, 通过一个双向链表来存储阻塞的线程
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

调度

>ReentrantLock的Sync类继承AbstractQueuedSynchronizer，通过名字（队列同步器）你大致能知道他是通过对线程进行列队的方式进行同步和调度的。
Sync的上面两个实现NonfairSync和FairSync，分别实现非公平锁和公平所机制机制。
---

重入锁实现

```
// 非公平锁的获取方法
final boolean nonfairTryAcquire(int acquires) {
    // 首先获取当前线程
    final Thread current = Thread.currentThread();
    // 获取锁的状态
    int c = getState();
    // 如果为0则继续通过原子操作设置state，如果成功则设置获取锁的线程为当前线程并返回成功
    if (c == 0) {
	if (compareAndSetState(0, acquires)) {
	    setExclusiveOwnerThread(current);
	    return true;
	}
    }
    // 否则锁已经被某个线程获取到，判断是否为当前线程
    else if (current == getExclusiveOwnerThread()) {
	// 如果是当前线程则将state+1，可以看出锁的可重入性就体现在这里
	int nextc = c + acquires;
	if (nextc < 0) // overflow
	    throw new Error("Maximum lock count exceeded");
	// 设置状态，并返回成功
	setState(nextc);
	return true;
    }
    // 否则该锁已经被其他线程占用，因此后面需要考虑阻塞该线程。
    return false;
}
```
```
// 公平锁的获取
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 否则锁已经被某个线程获取到，判断是否为当前线程
    else if (current == getExclusiveOwnerThread()) {
    // 如果是当前线程则将state+1，可以看出锁的可重入性就体现在这里
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

#### 3、线程池

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 根据ctl的值, 获取线程池中的有效线程数 workerCount, 如果 workerCount小于核心线程数 corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 调用addWorker()方法, 将核心线程数corePoolSize设置为线程池中线程数的上限值, 将此次提交的任务command作为参数传递进去, 
        // 然后再次获取线程池中的有效线程数 workerCount, 如果 workerCount依然小于核心
        // 线程数 corePoolSize, 就创建并启动一个线程, 然后返回 true结束整个
        // execute()方法. 如果此时的线程池已经关闭, 或者此时再次获取到的有
        // 效线程数 workerCount已经 >= 核心线程数 corePoolSize, 就再继续执
        // 行后边的内容. 
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    
    /***** 分析1 ****/
    // 如果情况1的判断条件不满足, 则直接进入情况2. 如果情况1的判断条件满足, 
    // 但情况1中的 addWorker()方法返回 false, 也同样会进入情况2.  
    // 总之, 进入情况2时, 线程池要么已经不处于RUNNING(运行)状态, 要么仍处于RUNNING
    // (运行)状态但线程池内的有效线程数 workerCount >= 核心线程数 corePoolSize
    /***** 分析2 ****/
    // 经过上一段分析可知, 进入这个情况时, 线程池要么已经不处于RUNNING(运行)
    // 状态, 要么仍处于RUNNING(运行)状态但线程池内的有效线程数 workerCount
    // 已经 >= 核心线程数 corePoolSize
    // 如果线程池未处于RUNNING(运行)状态, 或者虽然处于RUNNING(运行)状态但线程池
    // 内的阻塞队列 workQueue已满, 则跳过此情况直接进入情况3.
    // 如果线程池处于RUNNING(运行)状态并且线程池内的阻塞队列 workQueue未满, 
    // 则将提交的任务 command 添加到阻塞队列 workQueue中.
    if (isRunning(c) && workQueue.offer(command)) {
         int recheck = ctl.get();
         // 再次判断线程池此时的运行状态. 如果发现线程池未处于 RUNNING(运行)
         // 状态, 由于先前已将任务 command加入到阻塞队列 workQueue中了, 所以需
         // 要将该任务从 workQueue中移除. 一般来说, 该移除操作都能顺利进行. 
         // 所以一旦移除成功, 就再调用 handler的 rejectedExecution()方法, 根据
         // 该 handler定义的拒绝策略, 对该任务进行处理. 当然, 默认的拒绝策略是
         // AbortPolicy, 也就是直接抛出 RejectedExecutionException 异常, 同时也
         // 结束了整个 execute()方法的执行.
         if (! isRunning(recheck) && remove(command))
            reject(command);
         // 再次计算线程池内的有效线程数 workerCount, 一旦发现该数量变为0, 
         // 就将线程池内的线程数上限值设置为最大线程数 maximumPoolSize, 然后
         // 只是创建一个线程而不去启动它, 并结束整个 execute()方法的执行.
         else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
         // 如果线程池处于 RUNNING(运行)状态并且线程池内的有效线程数大于0, 那么就直接结束该 
         // execute()方法, 被添加到阻塞队列中的该任务将会在未来的某个时刻被执行.
    }
    /********************************* 情况3 ************************************/

    /***** 分析3 ****/
    // 如果该方法能够执行到这里, 那么结合分析1和分析2可知, 线程池此时必定是
    // 下面两种情况中的一种:
    // ① 已经不处于RUNNING(运行)状态
    // ② 处于RUNNING(运行)状态, 并且线程池内的有效线程数 workerCount已经
    //   >= 核心线程数 corePoolSize, 并且线程池内的阻塞队列 workQueue已满

    // 再次执行addWorker() 方法, 将线程池内的线程数上限值设置为最大线程数 
    // maximumPoolSize, 并将提交的任务 command作为被执行的对象, 尝试创建并
    // 启动一个线程来执行该任务. 如果此时线程池的状态为如下两种中的一种, 
    // 就会触发 handler的 rejectedExecution()方法来拒绝该任务的执行:
    // ① 未处于RUNNING(运行)状态.
    // ② 处于RUNNING(运行)状态, 但线程池内的有效线程数已达到本次设定的最大
    // 线程数 (另外根据分析3可知, 此时线程池内的阻塞队列 workQueue已满).

    // 如果线程池处于 RUNNING(运行)状态, 但有效线程数还未达到本次设定的最大
    // 线程数, 那么就会尝试创建并启动一个线程来执行任务 command. 如果线程的
    // 创建和启动都很顺利, 那么就直接结束掉该 execute()方法; 如果线程的创建或
    // 启动失败, 则同样会触发 handler的 rejectedExecution()方法来拒绝该
    // 任务的执行并结束掉该 execute()方法.
    else if (!addWorker(command, false))
        reject(command);
}
```

#### 4、并发集合

- 非阻塞式安全列表 - ConcurrentLinkedDeque
- 阻塞式安全列表 - LinkedBlockingDeque
- 优先级排序阻塞式安全列表 - PriorityBlockingQueue
- 延迟元素线程安全列表 - DelayQueue
- ConcurrentHashMap

#### 5、同步辅助类

- CountDownLatch
- CyclicBarrier
- Semaphore

1、CountDownLatch

>CountDownLatch实现维护一个计数器，这个类有两个核心方法await和countDown，一般被用在线程等待Ｎ个线程的情况。
调用await方法会使当前线程阻塞等待，直到这个计数器归零才会继续被唤醒执行。
典型的用法是，一个线程Ａ调用await（很多等待方法都被定义为await，区别object.wait()～），等待其他Ｎ个线程完成操作，每完成一个操作就countDown一次，当所有任务都完成之后线程Ａ继续执行。
---
```
class Driver2 { // ...
   void main() throws InterruptedException {
     CountDownLatch doneSignal = new CountDownLatch(N);
     Executor e = ...
     for (int i = 0; i < N; ++i) // create and start threads
       e.execute(new WorkerRunnable(doneSignal, i));
     doneSignal.await();　// wait for all to finish
   }
 }

 class WorkerRunnable implements Runnable {
   private final CountDownLatch doneSignal;
   private final int i;
   WorkerRunnable(CountDownLatch doneSignal, int i) {
      this.doneSignal = doneSignal;
      this.i = i;
   }
   public void run() {
      try {
        doWork(i);
        //工作线程执行完之后countDown一次
        doneSignal.countDown();
      } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }
```

2、CyclicBarrier

>CountDownLatch是等待计数器归零，而CyclicBarrier则是等待其他线程进入等待状态。
假设你有一个线程，需要等待其他５个线程进入等待状态，然后继续执行，你就可以使用这个东西。
---

3、Semaphore

>Semaphore是信号量的意思，和操作系统书籍里的信号量语义层面是一样的。
Semaphore有两个操作UP(+1)和DOWN(-1)，或者说P操作(-1)和V(+1)操作。
---
