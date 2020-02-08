# 一. Java 中线程池是如何实现的？创建线程池的几个核心构造参数是什么？

## 1.1 实现

1. 线程数小于 coreSize，创建线程，直到 coreSize 的数量；
2. BlockingQueue 一直堆积线程；堆积到 BlockingQueue 的最大容量，此时开始开启线程，直到 maxSize；
3. 如果到了 maxSize 的线程数，BlockingQueue 依然是满的，则开始对新添加进入的任务实行拒绝（按照拒绝策略）；
4. 如果线程数量大于 coreSize，而且有的线程空闲时间超过了 keepTimeAlive，则释放该资源；

注：1, 2, 3 步骤在 ThreadPoolExecutor # execute(Runnable command) 方法中；

## 1.2 核心构造参数

- coreSize: 核心线程数量，平时维持的线程数量；
- maxSize: 最大线程数量，如果 BlockingQueue 任务队列里堆积的任务过多，超过了队列限定最大值，则线程增多，增多至 maxSize 的线程数量；
- BlockingQueue\<Runnable>: 任务队列，向其中放任务；可通过设定队列最大长度，设置最大任务数量；
- keepTimeAlive: 如果当前线程数量大于 coreSize，且线程空闲超过 keepTimeAlive 值，则释放该线程，最多释放到 coreSize 的线程数量；
- RejectPolicy: 拒绝策略；当队列已满，再向其中塞任务时的拒绝策略；
- threadFactory: 线程工厂，使用各自的方法生产线程；比如可以使用 NamedThreadFactory，可以为线程池命名；

## 1.3 拒绝策略

- **ThreadPoolExecutor.AbortPolicy**：丢弃任务并抛出 RejectedExecutionException 异常。 
- **ThreadPoolExecutor.DiscardPolicy**：也是丢弃任务，但是不抛出异常。 
- **ThreadPoolExecutor.DiscardOldestPolicy**：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）；
- **ThreadPoolExecutor.CallerRunsPolicy**：由调用线程处理该任务 ；

> 参考地址：[《线程池ThreadPoolExecutor实现原理》](https://www.jianshu.com/p/125ccf0046f3)

# 二. Volatile 和 Synchronize 的区别？

从并发的三大特性角度来看 volatile 和 synchronized：

- 原子性：volatile 不保证原子性，synchronized 用锁的方式 (lock, unlock) 保证原子性；
- 可见性：在线程 A 中修改主内存变量的值，其他线程也会立即获得该变量的新值；volatile 与 synchronized 都保证可见性；
	- volatile 依靠其特性，在线程中修改值后，会立即向主内存中进行赋值，实现其可见性；
	- synchronized 依靠 lock 的特性，用 synchronized 修饰的方法，字节码上都处于 moniterenter 与 moniterexit 之间，这部分字节码会严格按照顺序执行；
	- volatile关键字主要用于解决变量在多个线程之间的可见性，而 synchronized关键字解决的是多个线程之间访问资源的同步性。
- 有序性：保证代码的顺序执行；
	- volatile 的自身特性：禁止指令的重排列；
	- synchronized 依旧依靠 lock 的特性；

> 注：  
> 
> - synchronized 是重量级的锁。具体含义，是指 Java 线程是映射到操作系统内核的**轻量级进程**执行的；而执行一个轻量级进程，就需要从用户模态切换到系统模态。如果 synchronized 使用次数过多，就意味着模态切换次数太多，消耗内核资源过多。
> - 锁优化（自旋锁、锁释放、锁粗化、轻量级锁、偏向锁）是针对 synchronized 关键字进行优化；

# 三. 高并发场景下如何防止死锁，保证数据的一致性？

## 1. 什么是死锁

死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等的进程称为死锁进程。

## 2. 死锁产生的四个必要条件

- 互斥条件：指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放
- 请求和保持条件：指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放
- 不剥夺条件：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放
- 环路等待条件：指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

## 3. 如何处理死锁

### (1) 锁模式

1. 共享锁（S)
	- 由读操作创建的锁，防止在读取数据的过程中，其它事务对数据进行更新；其它事务可以并发读取数据。共享锁可以加在表、页、索引键或者数据行上。在SQL SERVER默认隔离级别下数据读取完毕后就会释放共享锁，但可以通过锁提示或设置更高的事务隔离级别改变共享锁的释放时间。
2. 独占锁(X)
	- 对资源独占的锁，一个进程独占地锁定了请求的数据源，那么别的进程无法在此数据源上获得任何类型的锁。独占锁一致持有到事务结束。
3. 更新锁(U)
	- 更新锁实际上并不是一种独立的锁，而是共享锁与独占锁的混合。当SQL SERVER执行数据修改操作却首先需要搜索表以找到需要修改的资源时，会获得更新锁。
	- 更新锁与共享锁兼容，但只有一个进程可以获取当前数据源上的更新锁，
	- 其它进程无法获取该资源的更新锁或独占锁，更新锁的作用就好像一个序列化阀门 (serialization gate)，将后续申请独占锁的请求压入队列中。持有更新锁的进程能够将其转换成该资源上的独占锁。更新锁不足以用于更新数据—实际的数据修改仍需要用到独占锁。对于独占锁的序列化访问可以避免转换死锁的发生，更新锁会保留到事务结束或者当它们转换成独占锁时为止。
4. 意向锁(IX,IU,IS)	
	- 意向锁并不是独立的锁定模式，而是一种指出哪些资源已经被锁定的机制。
	- 如果一个表页上存在独占锁，那么另一个进程就无法获得该表上的共享表锁，这种层次关系是用意向锁来实现的。进程要获得独占页锁、更新页锁或意向独占页锁，首先必须获得该表上的意向独占锁。同理，进程要获得共享行锁，必须首先获得该表的意向共享锁，以防止别的进程获得独占表锁。
5. 特殊锁模式 (Sch_s,Sch_m,BU)
	- SQL SERVER提供3种额外的锁模式：架构稳定锁、架构修改锁、大容量更新锁。
6. 转换锁(SIX,SIU,UIX)
	- 转换锁不会由SQL SERVER 直接请求，而是从一种模式转换到另一种模式所造成的。SQL SERVER 2008支持3种类型的转换锁：SIX、SIU、UIX.其中最常见的是SIX锁，如果事务持有一个资源上的共享锁（S），然后又需要一个IX锁，此时就会出现SIX。
7. 键范围锁
	- 键范围锁是在可序列化隔离级别中锁定一定范围内数据的锁。保证在查询数据的键范围内不允许插入数据。

### (2) 锁粒度

SQL SERVER 可以在表、页、行等级别锁定用户的数据资源即非系统资源（系统资源是用闩锁来保护的）。此外SQL SERVER 还可以锁定索引键和索引键范围。  
通过sys.dm_tran_locks视图可以查看谁被锁定了（如行，键，页）、锁的模式以及特定资源的标志符。基于sys.dm_tran_locks视图创建如下视图用于查看锁定的资源以及锁模式（通过这个视图可以查看事务锁定的表、页、行以及加在数据资源上的锁类型）。

```sql
CREATE VIEW dblocks AS
SELECT request_session_id AS spid,
DB_NAME(resource_database_id) AS dbname,
CASE WHEN resource_type='object'
THEN OBJECT_NAME(resource_associated_entity_id)
WHEN resource_associated_entity_id=0 THEN'n/a'
ELSE OBJECT_NAME (p.object_id) END AS entity_name,
index_id,
resource_type AS RESOURCE,
resource_description AS DESCRIPTION,
request_mode AS mode,
request_status AS STATUS
FROM sys.dm_tran_locks t LEFTJOIN sys.partitions p 
ON p.partition_id=t.resource_associated_entity_id
WHERE resource_database_id=DB_ID()
```

### (3) 如何跟踪死锁

通过选择sql server profiler 事件中的如下选项就可以跟踪到死锁产生的相关语句。

### (4) 死锁案例分析

在该案例中process65db88, process1d0045948为语句1的进程，process629dc8 为语句2的进程； 语句2获取了1689766页上的更新锁，在等待1686247页上的更新锁；而语句1则获取了1686247页上的更新锁在等待1689766页上的更新锁，两个语句等待的资源形成了一个环路，造成死锁。

### (5) 如何解决死锁

针对如上死锁案例，分析其对应语句执行计划如下：

通过执行计划可以看出，在查找需要更新的数据时使用的是索引扫描，比较耗费性能，这样就造成锁定资源时间过长，增加了语句并发执行时产生死锁的概率。

处理方式：

1. 在表上建立一个聚集索引。
2. 对语句更新的相关字段建立包含索引。

优化后该语句执行计划如下：

- 优化后的执行计划使用了索引查找，将大幅提升该查询语句的性能，降低了锁定资源的时间，同时也减少了锁定资源的范围，这样就降低了锁资源循环等待事件发生的概率，对于预防死锁的发生会有一定的作用。
- 死锁是无法完全避免的，但如果应用程序适当处理死锁，对涉及的任何用户及系统其余部分的影响可降至最低（适当处理是指发生错误1205时，应用程序重新提交批处理，第二次尝试大多能成功。一个进程被杀死，它的事务被取消，它的锁被释放，死锁中涉及到的另一个进程就可以完成它的工作并释放锁，所以就不具备产生另一个死锁的条件了。）

## 4. 如何预防死锁

阻止死锁的途径就是避免满足死锁条件的情况发生，为此我们在开发的过程中需要遵循如下原则：

1. 尽量避免并发的执行涉及到修改数据的语句。
2. 要求每一个事务一次就将所有要使用到的数据全部加锁，否则就不允许执行。
3. 预先规定一个加锁顺序，所有的事务都必须按照这个顺序对数据执行封锁。如不同的过程在事务内部对对象的更新执行顺序应尽量保证一致。
4. 每个事务的执行时间不可太长，对程序段的事务可考虑将其分割为几个事务。在事务中不要求输入，应该在事务之前得到输入，然后快速执行事务。
5. 使用尽可能低的隔离级别。
6. 数据存储空间离散法。该方法是指采用各种手段，将逻辑上在一个表中的数据分散的若干离散的空间上去，以便改善对表的访问性能。主要通过将大表按行或者列分解为若干小表，或者按照不同的用户群两种方法实现。
7. 编写应用程序，让进程持有锁的时间尽可能短，这样其它进程就不必花太长的时间等待锁被释放。

## 5. 补充：

### (1) 死锁的概念：

如果一组进程中的每一个进程都在等待仅由该组进程中的其他进程才能引发的事件，那么改组进程是死锁的。

### (2) 死锁的常见表现：

死锁不仅会发生多个进程中，也会发生在一个进程中。

- 多进程死锁：有进程A，进程B，进程A拥有资源1，需要请求正在被进程B占有的资源2。而进程B拥有资源2，请求正在被进程A战友的资源1。两个进程都在等待对方释放资源后请求该资源，而相互僵持，陷入死锁。
- 单进程死锁：进程A拥有资源1，而它又在请求资源1，而它所请求的资源1必须等待该资源使用完毕得到释放后才可被请求。这样，就陷入了自己的死锁。

### (3) 产生死锁的原因：

- 进程推进顺序不当造成死锁。
- 竞争不可抢占性资源引起死锁。
- 竞争可消耗性资源引起死锁。

### (4) 死锁的四个必要条件（四个条件四者不可缺一）：

- 互斥条件。某段时间内，一个资源一次只能被一个进程访问。
- 请求和保持条件。进程A已经拥有至少一个资源，此时又去申请其他资源，而该资源又正在被进程使用，此时请求进程阻塞，但对自己已经获得的资源保持不放。
- 不可抢占资源。进程已获得的资源在未使用完不能被抢占，只能在自己使用完时由自己释放。
- 循环等待序列。存在一个循环等待序列P0P1P2……Pn，P0请求正在被进程P1占有的资源，P1请求正在被P2占有的资源……Pn正在请求被进程P0占有的资源。

### (5) 解除死锁的两种方法：

- 终止（或撤销）进程。终止（或撤销）系统中的一个或多个死锁进程，直至打破循环环路，使系统从死锁状态中解除出来。
- 抢占资源。从一个或多个进程中抢占足够数量的资源，分配给死锁进程，以打破死锁状态。

# 四. 锁优化

> 参考地址：  
> [《java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁》](https://blog.csdn.net/zqz_zqz/article/details/70233767)  

在早期版本的 JDK 中，主要使用 synchronized 作为并发锁，但 synchronized 是重量级锁，并不是所有场合下都适合使用，很多场景下都可以对其进行优化。在 JDK 1,6 之后，JVM 做了一定的锁优化，此外锁优化也是一种思想，我们可以某些开源代码锁的使用情况，理解作者对锁进行优化的思想，并合适的应用到自己的工程项目中。

## 4.1 JVM 锁优化

JVM 层面的锁优化见[《JVM 篇》](./JVM.md)。

## 4.2 锁优化思路

JVM 层面的锁优化是我们不能用代码控制的，我们最多只能从业务的场景出发进行判断，从而添加针对性的配置。但是在工程的开发过程中，我们可以借鉴上面某些加锁的思想，对我们自己业务代码中涉及同步的代码块进行优化。

### 4.2.1 减小锁粒度

**减小锁粒度**的主要思想，是将一个锁合理的**拆分多个逻辑上的锁**，降低锁的竞争程度。但是锁粒度不能无限拆，最多可以将锁。典型的应用就是 **ConcurrentHashMap** 的底层。  
关于 ConcurrentHashMap 的锁粒度减小的策略与实施，见[《数据结构与算法》篇](./数据结构与算法.md)。  
此外 LinkedBlockingQueue 也是类似的思想，从队列头入队，在队列尾出队，入队和出队使用不同的锁。  

## 4.2.2 锁粗化

**锁粗化**是指要增大锁的粒度。如果使用 JDK 1.7 之前的版本，如果有一个循环，且循环内的操作需要加锁，那么应该把锁放到循环外面，否则每次进出循环都会进入一次临界区，效率会很差。

## 4.2.3 读写分离

对于读多写少的场景，可以使用读写分离的思想，读的时候不加锁，写的时候加锁。  
典型示例：Redis 的主写从读、数据库的读写分离、Java 的 <code>CopyOnWriteArrayList, CopyOnWriteArraySet</code>。

# 五. 加锁的机制

> 参考网址：  
> [《mysql共享锁与排他锁》](https://www.cnblogs.com/boblogsbo/p/5602122.html)  

**共享锁与排他锁**：

- **共享锁 (S)**：对一条数据上共享锁，多个事务可以对该条数据共享同一个共享锁，但是只可以读，不可以写；
	- select 语句是默认不加锁的，也可以显式的加锁。添加共享锁的 select 语句是 select \* lock in share mode;
- **排他锁 (X)**：一个事务对一条数据加上排他锁，其他事物就不能获取该行的其他锁（包括其他的共享锁和排他锁）；
	- 注：这种理解是错误的：<font color=red>对一条数据上排他锁，该条数据对其他事务的读写都被拒绝。</font>
	- 在 InnoDB 中，使用 insert, update, delete 语句会自动加上共享锁；
	- 添加排他锁的 select 语句：select \* for update

共享锁与排他锁见上。

# 六. 多线程的六种状态？

线程有六种状态：**NEW, RUNNABLE(RUNNING), WAITING, TIME\_WAITING, BLOCKED, TERMINATED**。

![线程六种状态](https://images2018.cnblogs.com/blog/930824/201807/930824-20180715222029724-1669695888.jpg)

## 6.1 NEW

线程刚刚被创建的时候，即 new Thread()，且尚未执行 start() 方法的状态；

## 6.2 RUNNABLE / RUNNING

RUNNABLE(或称 READY) 与 RUNNING 是线程**已经准备执行**或**正在执行**的状态，是线程执行了 start() 方法之后状态。线程处于 RUNNABLE 或者是 RUNNING 状态，取决于 CPU 的调度，获取了 CPU 使用权的线程处于 RUNNING 状态，否则处于就绪状态 (RUNNABLE)。进入该状态的方法如下：

- **RUNNABLE -> RUNNING**: CPU 调度后，获取 CPU 的使用权；
- **RUNNING -> RUNNABLE**: 失去 CPU 的使用权，回到就绪状态；
- **NEW -> RUNNABLE**: thread.start() 方法调用；
- **WAITING -> RUNNABLE**: notify/notifyAll 方法调用；
- **TIME\_WAITING -> RUNNABLE**: notify/notifyAll 方法调用；

## 6.3 WAITING

WAITING 为等待状态，需要进行某些特定动作之后才能回到正常的运行状态，时间不确定
。进入该状态的方法如下：

- **RUNNABLE -> WAITING**: wait/join 方法调用；

## 6.4 TIME_WAITING

TIME\_WAITING 同样为等待状态，也需要进行某些特定动作才能回到正常运行状态。与 WAITING 方法不同的是 TIME\_WAITING 状态的时间是确定的。进入该状态的方法如下：

- **RUNNABLE -> TIME\_WAITING**: sleep 方法调用；

## 6.5 BLOCKED

BLOCKED 状态是在获取锁过程中被阻塞的状态，通常用于 synchronized, lock 的使用场景中。进入该状态的方法如下：

- **RUNNABLE -> BLOCKED**: synchronized, lock 阻塞；

## 6.6 TERMINATED

TERMINATED 状态标志着一个线程的结束，处于 TERMINATED 状态的线程不能再转变为其他状态。进入该状态的方法如下：

- 线程正常的运行完毕；
- 线程运行抛出异常；
- JVM Crash，所有线程全部结束；

# 七. CAS 原理
> 参考地址：  
> [《JAVA并发编程: CAS和AQS》](https://blog.csdn.net/u010862794/article/details/72892300)  
> [《面试必问的CAS，你懂了吗？》](https://blog.csdn.net/v123411739/article/details/79561458)  

**CAS (Compare And Swap)**，即比较并交换，是**解决多线程并行情况下使用锁造成性能损耗**的一种机制。在 JAVA 中，<code>sun.misc.Unsafe</code> 类提供了**硬件级别的原子操作**来实现 CAS。 java.util.concurrent 包下的大量类 (<code>AtomicInteger, AtomicBoolean, AtomicLong, ...</code> )都使用了这个 Unsafe.java 类的 CAS 操作。至于 Unsafe.java 的具体实现这里就不讨论了。下面以 AtomicInteger.java 的部分实现来大致讲解下这些原子类的实现。

> 注：**CAS 与 AQS 的关系与区别**：  
> CAS 是一种**解决并发问题的思想**，也就是**先比较后替换**，JUC 通过自旋执行 CAS 操作实现线程安全的状态更新。  
> AQS 是 Java 并发包的一个底层框架，是可重入锁 (ReentrantLock) 与共享锁（比如 CountDownLatch, CyclicBarrier 等）的基础。关于 Lock 与 AQS，Lock 面向用户，AQS 面向 Lock，也就是说 AQS 为各种 Lock 提供了底层的支持，AQS 的最核心原理之一就是利用 CAS 更新同步状态。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();

    // 初始 int 大小
    private volatile int value;
    // 省略了部分代码...

    // 返回旧值，并设置新值为 newValue
    public final int getAndSet(int newValue) {
        /**
         * 这里使用 for 循环不断通过 CAS 操作来设置新值
         * CAS 实现和加锁实现的关系有点类似乐观锁和悲观锁的关系
         */
        for (;;) {
            int current = get();
            if (compareAndSet(current, newValue))
                return current;
        }
    }

    // 原子的设置新值为 update, expect 为期望的当前的值
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    // 获取当前值 current，并设置新值为 current+1
    public final int getAndIncrement() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return current;
        }
    }

    // 此处省略部分代码，余下的代码大致实现原理都是类似的
}
```

核心系列方法 <code>compareAndSet</code> 包含三个操作数——内存位置 (V)、预期原值 (A) 和新值 (B)。CAS 有效地说明了“**我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则不做任何更改，只告诉我这个位置现在的值即可**。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。 

在竞争不是特别激烈的时候，使用该包下的原子操作性能比使用 synchronized 关键字的方式高效的多（查看 <code>getAndSet(int newValue)</code> 源码，可知如果资源竞争十分激烈的话，这个 for 循环可能换持续很久都不能成功跳出。不过这种情况更应该考虑降低资源竞争)。

> 注：通常使用 AtomicInteger，会用到它的 <code>getAndIncrement()</code> 方法作计数器。

CAS 最主要的运用，就是在 JUC 中的 <code>AbstractQueuedSynchronizer</code>，即 AQS，它是 Java 中多种锁实现的父类。该类核心**同步队列**的入队操作是一种乐观锁实现，多线程情况下对头节点、尾节点操作都有可能失效，失效后 CAS 会再次尝试，直到尝试成功。比如 <code>AbstractQueuedSynchronizer # enq(Node node)</code> 方法：

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

此外，CAS 仍然有三个缺点：

1. **循环时间开销大**
	- 从 <code>getAndAddInt()</code> 源码中可以看到，如果 CAS 执行返回 false，那么会一直执行尝试，如果 CAS 长时间不成功，可能会有比较大的 CPU 资源开销；
2. **只保证一个共享变量原子操作**
	- 对于多个共享变量，无法使用 CAS 方式保证操作原子性，此时应该使用锁 (synchronized, Lock) 保证原子性；
3. **ABA 问题**
	- **ABA 问题**：如果内存地址 V 上的值在读取时为 A，准备赋值的时候检查它的值也为 A，但也**不能保证在这期间没有被其他线程改变过**；
	- 在使用 CAS 之前要考虑好一致性的效果，是需要达到最终一致性，还是完全一致性。如果需要解决 ABA 问题，Java 中提供了 <code>AtomicStampedReference</code> / <code>AtomicMarkableReference</code> 来处理会发生 ABA 问题的场景，它们的主要思想是在对象中额外再增加一个版本号的标记，标识对象是否有过变更；此外也可以改用传统互斥同步。
4. **不适用于高并发场景**

# 八. AQS 原理

> 参考地址：  
> [《Java并发-AQS及各种Lock锁的原理》](https://blog.csdn.net/zhangdong2012/article/details/79983404)  
> [《JAVA并发编程: CAS和AQS》](https://blog.csdn.net/u010862794/article/details/72892300)  
> [《Java并发之AQS详解》](https://www.cnblogs.com/waterystone/p/4920797.html)

## 8.1 AQS 简介

**AQS (AbustactQueuedSynchronizer)** 是 Java 提供的底层同步工具类，主要思想是用一个 int 类型的变量表示**同步状态**，以及一个双链表形式的**同步队列**，并提供了一系列的 **CAS (Compare And Swap)** 操作来管理这个同步状态。  
AQS 的主要作用是为 Java 中的并发同步组件提供统一的底层支持，例如 <code>ReentrantLock</code>，<code>CountDownLatch</code> 就是基于 AQS 实现的，实现方法是通过继承 AQS 实现其模版方法，然后将子类作为同步组件的内部类。

## 8.2 AQS 基本方法

AQS 有若干基本方法：

```java
boolean tryAcquire(int arg);
boolean tryRelease(int arg);
int tryAcquireShared(int arg);
boolean tryReleaseShared(int arg);
boolean isHeldExclusively();
```

以上方法**不需要全部实现**，根据获取的锁的种类可以选择实现不同的方法。支持**独占（排他）锁**的同步器，比如 ReentrantLock，应该实现 **tryAcquire, tryRelease, isHeldExclusively** 方法；而作为**共享锁**的同步器，比如 CountDownLatch，应该实现 **tryAcquireShared, tryReleaseShared** 方法。

## 8.3 同步队列

同步队列是 AQS 很重要的组成部分，它是一个双端队列，遵循 FIFO 原则，主要作用是**存放在锁上阻塞的线程**。比如可重入锁 ReentrantLock 的 <code>lock()</code>, <code>unlock()</code> 方法，分别实现了线程挂起、释放，本质上就是将线程存入同步队列、弹出同步队列的操作。  

### 8.3.1 独占锁 - 获取锁

对于独占锁 (如 ReentrantLock)，需要实现 **tryAcquire, tryRelease, isHeldExclusively** 方法。当一个线程尝试获取锁时，如果已经被占用，那么当前线程就会被**构造成一个 Node 节点，加入到同步队列的尾部**。如下图所示：

![AQS Node 加入队列尾部](./pic/AQS_Node加入队列尾部.png)

线程尝试获取锁的操作，本质上就是将当前线程加入到同步队列尾部的操作。步骤如下：

1. 调用 AQS 的入口方法 <code>acquire(int arg)</code>；
2. 调用 AQS 的模板方法 <code>tryAcquire(int arg)</code> 尝试获取锁；如果获取锁成功，则不进入同步队列；
3. 如果尝试获取锁失败，则将当前线程构造成一个 Node 节点，通过 CAS 将其加入到同步队列尾部，然后该 Node 对应的线程进入**自旋状态**；
	- 自旋状态下，判断两个条件：
		- 同步队列的前驱节点 prev 是否为头结点 head；
		- <code>tryAcquire()</code> 是否获取成功；
	- 两个条件同时成立，则将当前线程节点设置为头结点；
	- 不同时成立，使用 <code>LockSupport.park(this)</code> 方法将当前线程挂起，等待 prev 节点的唤醒；

![AQS 队列 Node 状态](./pic/AQS_Node队列Node状态.png)

### 8.3.2 独占锁 - 释放锁

队列的头节点是成功获取锁的节点，对于独占锁 (如 ReentrantLock)，当头节点线程释放锁时，会唤醒后面的节点并释放当前头节点的引用。如下图所示：

![AQS Node 弹出队列头部](./pic/AQS_Node弹出队列头部.png)

线程释放锁的操作，本质上就是将当前线程从同步队列头部弹出的操作。步骤如下：

1. 调用 AQS 的入口方法 <code>release(int arg)</code>；
2. 调用 AQS 的模板方法 <code>tryRelease(int arg)</code> 尝试释放同步状态；
3. 将当前节点的下一个节点作为头结点 head；
4. 通过 <code>LockSupport.unpark(currentNode.next.thread)</code> 唤醒后继节点（即线程入队的步骤 3）；

### 8.3.3 共享锁 - 获取锁

<code>CountDownLatch</code>, <code>CyclicBarrier</code>, <code>Semaphore</code> 都属于**共享锁**的同步器，应该实现 **tryAcquireShared, tryReleaseShared** 方法。获取锁流程如下：

1. 调用 AQS 的入口方法 <code>acquireShared(int arg)</code>；
2. 进入 AQS 的模板方法 <code>tryAcquireShared(int arg)</code>，获取同步状态；
	- 返回值 >= 0，则说明同步状态有剩余，获取锁成功，直接返回；
	- 返回值 < 0，则说明获取同步锁失败，向队尾添加共享类型的 Node 节点 (Node.SHARED)，该 Node 对应的线程进入**自旋状态**；
3. 自旋状态下，判断两个条件：
		- 同步队列的前驱节点 prev 是否为头结点 head；
		- <code>tryAcquire()</code> 是否获取成功；
	- 两个条件同时成立，则将当前线程节点设置为头结点，**<font color=red>并唤醒所有后继节点</font>**，所有后继节点重新进入尝试获取锁的状态；
	- 不同时成立，使用 <code>LockSupport.park(this)</code> 方法将当前线程挂起，等待 prev 节点的唤醒；

需要注意的是，自旋状态下独占锁与共享锁判断的条件相同，但执行动作不同：独占锁只将当前节点设置为头结点，共享锁在此之外还唤醒所有的后继节点。

### 8.3.4 共享锁 - 释放锁

1. 调用 <code>releaseShared(int arg)</code> 模板方法释放同步状态；
2. 如果释放成功，则遍历整个队列，使用 <code>LockSupport.unpart(nextNode.thread)</code> 唤醒所有后继节点；

### 8.3.5 独占锁与共享锁的区别

1. 同步状态值
	- 独占锁的同步状态值 <code>state = 1</code>，即同一时刻只能有一个线程成功获取同步状态；
	- 共享锁的同步状态 <code>state \> 1</code>，取值由上层同步组件确定；此外，共享锁会出现多个线程同时成功获取同步状态的情况；
2. 头结点运行后操作
	- 独占锁的同步队列中，Node 节点运行完毕后，释放该节点后面一个后继节点（即直接后继节点）；
	- 共享锁的同步队列中，Node 节点运行完毕后，释放该节点后面所有后继节点；

## 8.4 AQS 实现原理

## 8.5 可重入锁 (ReentrantLock) 实现原理

**重入锁**指的是当前线程成功获取锁后，如果再次访问该临界区，则不会对自己产生互斥行为。Java 中的 <code>ReentrantLock</code> 和 <code>synchronized</code> 关键字都是可重入锁，synchronized 由 JVM 实现可重入性，ReentrantLock 的可重入性基于 AQS 实现。此外，ReentrantLock 还提供**公平锁**和**非公平锁**两种模式，默认创建非公平锁。  
重入锁的基本原理，是**判断上次获取锁的线程是否为当前线程**，如果是则可再次进入临界区，如果不是，则阻塞。由于 ReentrantLock 是基于 AQS 实现的，底层通过操作同步状态来获取锁。

### 8.5.1 非公平锁实现

下面看一下非公平锁的实现逻辑代码如下：

```java
    final boolean nonfairTryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 通过 AQS 获取同步状态
        int c = getState();
        // 同步状态为 0，说明临界区处于无锁状态，
        if (c == 0) {
            // 修改同步状态，即加锁
            if (compareAndSetState(0, acquires)) {
                // 将当前线程设置为锁的 owner
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果临界区处于锁定状态，且上次获取锁的线程为当前线程
        else if (current == getExclusiveOwnerThread()) {
            // 则递增同步状态
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

非公平锁是指在锁状态为可用时，所有等待该锁释放的线程有相同的权利争抢该锁，与线程等待时间长短无关。只要同步状态 <code>state = 0</code> 任意调用 **lock()** 方法，都有可能获取到锁。

> 注：在 compareAndSetState 方法中，使用了 CAS 的比较交换方法。关于 CAS 见其他部分的讲解。

### 8.5.2 公平锁实现

公平锁是指锁状态可用时，对于所有正在等待该锁释放的线程，按照等待时间进行排序，等待时间最长的线程获取该锁。公平锁与非公平锁的实现逻辑基本相同，逻辑不同的地方主要是在获取到线程状态 <code>state = 0</code> 时的处理。关键代码如下：

```java
    if (c == 0) {
        // 此处为公平锁的核心，即判断同步队列中当前节点是否有前驱节点
        if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
```

代码中的 <code>hasQueuedPredecessors()</code> 方法是关键区别，用来判断是否拥有前驱节点。对于公平锁来说，如果同步队列中拥有前驱节点，说明在该 Node 对应的线程之前，还有其他线程存入了同步队列中，那么就不满足获取该锁的条件。

## 8.6 读写锁 (ReadWriteReentrantLock) 实现原理

Java 提供了一个基于 AQS 到读写锁实现 ReentrantReadWriteLock，该读写锁到实现原理是：将同步变量 state **按照高 16 位和低 16 位**进行拆分，读锁为高 16 位，是共享锁；写锁是低 16 位，是独占锁。如下图所示。

![Java 读写锁划分方式](./pic/Java读写锁划分方式.png)

### 8.6.1 写锁

写锁是一个**独占锁**，获取锁的主要逻辑在 <code>ReentrantReadWriteLock # tryAcquire(int arg)</code> 中实现：

```java
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // Reentrant acquire
            setState(c + acquires);
            return true;
        }
        if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
```

1. 获取同步状态，同时从同步状态中分离出低 16 位的写锁状态；
2. 如果同步状态不为 0，说明当前状态下存在读锁或者写锁；
3. 如果存在读锁，那么不能获取写锁；这样是为了保证对读的可见性；
	- 存在读锁的判断：<code>c != 0 && w == 0</code>
4. 如果当前线程不是写锁的线程，不能获取写锁；
5. 上面的判断全部通过，则用 CAS 将锁同步状态进行修改，最后将当前线程设置为写锁的获取线程；
	- 修改同步状态，是通过修改同步状态低 16 位的写锁完成的；

写锁的释放逻辑与独占锁基本相同。代码如下：

```java
    protected final boolean tryRelease(int releases) {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
```

释放过程中，不断减少读锁的同步状态，直到读锁同步状态为 0 时，写锁完全被释放。

### 8.6.2 读锁

读锁是一个**共享锁**，获取读锁的步骤如下：

```java
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
        if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
    }
```

1. 获取当前同步状态；
2. 计算高 16 位读锁状态 r；
3. 异常判断：如果 <code>r + 1</code> 大于获取到读锁的最大值，则抛出异常；
4. 如果存在写锁，而且当前线程不是写锁的获取者，则获取读锁失败；
5. 如果上述所有判断都通过，则通过 CAS 重新设置读锁同步状态；

读锁的释放与写锁类似，不断的释放写锁状态，直到为 0，表示没有线程持有读锁。

# 九. Java里面 的同步锁了解吗？ CountDownLaunch和Cylicbarrior的区别，分别在什么场景下使用？

在java 1.5中，提供了一些非常有用的辅助类来帮助我们进行并发编程，比如CountDownLatch，CyclicBarrier和Semaphore，今天我们就来学习一下这三个辅助类的用法。

> 注：同步锁的原理见 AQS 部分；

## 1. CountDownLatch用法

CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。

CountDownLatch类只提供了一个构造器：

```java
//参数count为计数值
public CountDownLatch(int count) {  }
```
然后下面这3个方法是CountDownLatch类中最重要的方法：

```java
public void await() throws InterruptedException { };   //调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  //和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public void countDown() { };  //将count值减1
```

下面看一个例子大家就清楚CountDownLatch的用法了：

```java
public class Test {
     public static void main(String[] args) {   
         final CountDownLatch latch = new CountDownLatch(2);
 
         new Thread(){
             public void run() {
                 try {
                     System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                    Thread.sleep(3000);
                    System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                    latch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
             };
         }.start();
 
         new Thread(){
             public void run() {
                 try {
                     System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                     Thread.sleep(3000);
                     System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                     latch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
             };
         }.start();
 
         try {
             System.out.println("等待2个子线程执行完毕...");
            latch.await();
            System.out.println("2个子线程已经执行完毕");
            System.out.println("继续执行主线程");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
     }
}
```

执行结果：

```txt
线程Thread-0正在执行
线程Thread-1正在执行
等待2个子线程执行完毕...
线程Thread-0执行完毕
线程Thread-1执行完毕
2个子线程已经执行完毕
继续执行主线程
```

## 2. CyclicBarrier用法

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了。

CyclicBarrier类位于java.util.concurrent包下，CyclicBarrier提供2个构造器：

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
}
 
public CyclicBarrier(int parties) {
}
```

参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

然后CyclicBarrier中最重要的方法就是await方法，它有2个重载版本：

```java
public int await() throws InterruptedException, BrokenBarrierException { };
public int await(long timeout, TimeUnit unit)throws InterruptedException,BrokenBarrierException,TimeoutException { };
```

第一个版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；  
第二个版本是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。  
下面举几个例子就明白了：假若有若干个线程都要进行写数据操作，并且只有所有线程都完成写数据操作之后，这些线程才能继续做后面的事情，此时就可以利用CyclicBarrier了：

```java
public class Test {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier  = new CyclicBarrier(N);
        for(int i=0;i<N;i++)
            new Writer(barrier).start();
    }
    static class Writer extends Thread{
        private CyclicBarrier cyclicBarrier;
        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }
 
        @Override
        public void run() {
            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }catch(BrokenBarrierException e){
                e.printStackTrace();
            }
            System.out.println("所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```

执行结果：

```
线程Thread-0正在写入数据...
线程Thread-3正在写入数据...
线程Thread-2正在写入数据...
线程Thread-1正在写入数据...
线程Thread-2写入数据完毕，等待其他线程写入完毕
线程Thread-0写入数据完毕，等待其他线程写入完毕
线程Thread-3写入数据完毕，等待其他线程写入完毕
线程Thread-1写入数据完毕，等待其他线程写入完毕
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
```

从上面输出结果可以看出，每个写入线程执行完写数据操作之后，就在等待其他线程写入操作完毕。  
当所有线程线程写入操作完毕之后，所有线程就继续进行后续的操作了。  
如果说想在所有线程写入操作完之后，进行额外的其他操作可以为CyclicBarrier提供Runnable参数：

```java
public class Test {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier  = new CyclicBarrier(N,new Runnable() {
            @Override
            public void run() {
                System.out.println("当前线程"+Thread.currentThread().getName());   
            }
        });
 
        for(int i=0;i<N;i++)
            new Writer(barrier).start();
    }
    static class Writer extends Thread{
        private CyclicBarrier cyclicBarrier;
        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }
 
        @Override
        public void run() {
            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }catch(BrokenBarrierException e){
                e.printStackTrace();
            }
            System.out.println("所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```

运行结果：

```txt
线程Thread-0正在写入数据...
线程Thread-1正在写入数据...
线程Thread-2正在写入数据...
线程Thread-3正在写入数据...
线程Thread-0写入数据完毕，等待其他线程写入完毕
线程Thread-1写入数据完毕，等待其他线程写入完毕
线程Thread-2写入数据完毕，等待其他线程写入完毕
线程Thread-3写入数据完毕，等待其他线程写入完毕
当前线程Thread-3
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
```

从结果可以看出，当四个线程都到达barrier状态后，会从四个线程中选择一个线程去执行Runnable。

下面看一下为await指定时间的效果：

```java
public class Test {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier  = new CyclicBarrier(N);
 
        for(int i=0;i<N;i++) {
            if(i<N-1)
                new Writer(barrier).start();
            else {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                new Writer(barrier).start();
            }
        }
    }
    static class Writer extends Thread{
        private CyclicBarrier cyclicBarrier;
        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }
 
        @Override
        public void run() {
            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
                try {
                    cyclicBarrier.await(2000, TimeUnit.MILLISECONDS);
                } catch (TimeoutException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }catch(BrokenBarrierException e){
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+"所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```

执行结果：

```txt
线程Thread-0正在写入数据...
线程Thread-2正在写入数据...
线程Thread-1正在写入数据...
线程Thread-2写入数据完毕，等待其他线程写入完毕
线程Thread-0写入数据完毕，等待其他线程写入完毕
线程Thread-1写入数据完毕，等待其他线程写入完毕
线程Thread-3正在写入数据...
java.util.concurrent.TimeoutException
Thread-1所有线程写入完毕，继续处理其他任务...
Thread-0所有线程写入完毕，继续处理其他任务...
    at java.util.concurrent.CyclicBarrier.dowait(Unknown Source)
    at java.util.concurrent.CyclicBarrier.await(Unknown Source)
    at com.cxh.test1.Test$Writer.run(Test.java:58)
java.util.concurrent.BrokenBarrierException
    at java.util.concurrent.CyclicBarrier.dowait(Unknown Source)
    at java.util.concurrent.CyclicBarrier.await(Unknown Source)
    at com.cxh.test1.Test$Writer.run(Test.java:58)
java.util.concurrent.BrokenBarrierException
    at java.util.concurrent.CyclicBarrier.dowait(Unknown Source)
    at java.util.concurrent.CyclicBarrier.await(Unknown Source)
    at com.cxh.test1.Test$Writer.run(Test.java:58)
Thread-2所有线程写入完毕，继续处理其他任务...
java.util.concurrent.BrokenBarrierException
线程Thread-3写入数据完毕，等待其他线程写入完毕
    at java.util.concurrent.CyclicBarrier.dowait(Unknown Source)
    at java.util.concurrent.CyclicBarrier.await(Unknown Source)
    at com.cxh.test1.Test$Writer.run(Test.java:58)
Thread-3所有线程写入完毕，继续处理其他任务...
```

上面的代码在main方法的for循环中，故意让最后一个线程启动延迟，因为在前面三个线程都达到barrier之后，等待了指定的时间发现第四个线程还没有达到barrier，就抛出异常并继续执行后面的任务。

另外CyclicBarrier是可以重用的，看下面这个例子：

```java
public class Test {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier  = new CyclicBarrier(N);
 
        for(int i=0;i<N;i++) {
            new Writer(barrier).start();
        }
 
        try {
            Thread.sleep(25000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
 
        System.out.println("CyclicBarrier重用");
 
        for(int i=0;i<N;i++) {
            new Writer(barrier).start();
        }
    }
    static class Writer extends Thread{
        private CyclicBarrier cyclicBarrier;
        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }
 
        @Override
        public void run() {
            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
 
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }catch(BrokenBarrierException e){
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+"所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```

执行结果：

```txt
线程Thread-0正在写入数据...
线程Thread-1正在写入数据...
线程Thread-3正在写入数据...
线程Thread-2正在写入数据...
线程Thread-1写入数据完毕，等待其他线程写入完毕
线程Thread-3写入数据完毕，等待其他线程写入完毕
线程Thread-2写入数据完毕，等待其他线程写入完毕
线程Thread-0写入数据完毕，等待其他线程写入完毕
Thread-0所有线程写入完毕，继续处理其他任务...
Thread-3所有线程写入完毕，继续处理其他任务...
Thread-1所有线程写入完毕，继续处理其他任务...
Thread-2所有线程写入完毕，继续处理其他任务...
CyclicBarrier重用
线程Thread-4正在写入数据...
线程Thread-5正在写入数据...
线程Thread-6正在写入数据...
线程Thread-7正在写入数据...
线程Thread-7写入数据完毕，等待其他线程写入完毕
线程Thread-5写入数据完毕，等待其他线程写入完毕
线程Thread-6写入数据完毕，等待其他线程写入完毕
线程Thread-4写入数据完毕，等待其他线程写入完毕
Thread-4所有线程写入完毕，继续处理其他任务...
Thread-5所有线程写入完毕，继续处理其他任务...
Thread-6所有线程写入完毕，继续处理其他任务...
Thread-7所有线程写入完毕，继续处理其他任务...
```

从执行结果可以看出，在初次的4个线程越过barrier状态后，又可以用来进行新一轮的使用。而CountDownLatch无法进行重复使用。

## 3. Semaphore用法

Semaphore翻译成字面意思为 信号量，Semaphore可以控同时访问的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。  
Semaphore类位于java.util.concurrent包下，它提供了2个构造器：

```java
//参数permits表示许可数目，即同时可以允许多少线程进行访问
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {    //这个多了一个参数fair表示是否是公平的，即等待时间越久的越先获取许可
    sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
}
```

下面说一下Semaphore类中比较重要的几个方法，首先是acquire()、release()方法：

```java
//获取一个许可
public void acquire() throws InterruptedException {  }     
//获取permits个许可
public void acquire(int permits) throws InterruptedException { }    
//释放一个许可
public void release() { }          
//释放permits个许可
public void release(int permits) { }    
```

acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。  
release()用来释放许可。注意，在释放许可之前，必须先获获得许可。

这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```java
//尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire() { };    
//尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  
//尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits) { }; 
//尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; 
```

另外还可以通过availablePermits()方法得到可用的许可数目。  
下面通过一个例子来看一下Semaphore的具体使用：假若一个工厂有5台机器，但是有8个工人，一台机器同时只能被一个工人使用，只有使用完了，其他工人才能继续使用。那么我们就可以通过Semaphore来实现：

```java
public class Test {
    public static void main(String[] args) {
        int N = 8;            //工人数
        Semaphore semaphore = new Semaphore(5); //机器数目
        for(int i=0;i<N;i++)
            new Worker(i,semaphore).start();
    }
 
    static class Worker extends Thread{
        private int num;
        private Semaphore semaphore;
        public Worker(int num,Semaphore semaphore){
            this.num = num;
            this.semaphore = semaphore;
        }
 
        @Override
        public void run() {
            try {
                semaphore.acquire();
                System.out.println("工人"+this.num+"占用一个机器在生产...");
                Thread.sleep(2000);
                System.out.println("工人"+this.num+"释放出机器");
                semaphore.release();           
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

执行结果：

```txt
工人0占用一个机器在生产...
工人1占用一个机器在生产...
工人2占用一个机器在生产...
工人4占用一个机器在生产...
工人5占用一个机器在生产...
工人0释放出机器
工人2释放出机器
工人3占用一个机器在生产...
工人7占用一个机器在生产...
工人4释放出机器
工人5释放出机器
工人1释放出机器
工人6占用一个机器在生产...
工人3释放出机器
工人7释放出机器
工人6释放出机器
```

下面对上面说的三个辅助类进行一个总结：

1. CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：
	- CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；
	- 而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；
	- 另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。
2. Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。

# 

# 十. Java 的锁有哪些？可重入锁和不可重入锁的区别？

[《Java并发编程：Lock》](https://www.cnblogs.com/dolphin0520/p/3923167.html)  
[《java 锁 Lock接口详解》](https://www.cnblogs.com/myseries/p/10784076.html)   

[《[死磕 java同步系列之ReentrantLock源码解析（一）——公平锁、非公平锁]》](https://www.cnblogs.com/tong-yuan/p/ReentrantLock.html))

## 10.1 Java 锁的分类

锁的类型目前感觉可以分成两大类：synchronized 关键字，以及 Lock, ReadWriteLock 锁以及 Reentrant 为前缀修饰的实现类 (ReentrantLock, ReentrantReadWriteLock)；

其他角度来看，按照不同分类类型的锁：

- 实现方式：synchronized / Lock, ReadWriteLock 及其实现类；
- 可中断性：synchronized / Lock
- **公平性**：ReentrantLock 构造函数中，传入 boolean 值，可以控制公平性，默认非公平；公平锁按照锁的申请顺序分配锁，申请锁时间最长的线程在下一次会最早得到锁；非公平锁不对申请锁的时间进行保证，所以可能导致某个线程永远都获取不到锁；
- 可重入性：synchronized / ReentrantLock，如果一个线程已经获得了一个对象锁，此后该线程再请求进入被该对象锁的同步代码块时，由于该线程之前已经获取了这个对象锁，所以可以直接进入该锁的同步代码块；
- 读写性：ReadWriteLock；

> 可重入性的原理：参考《深入理解 Java 虚拟机》P391  
> 可重入锁和不可重入：参考[《Java不可重入锁和可重入锁理解》](https://blog.csdn.net/u012545728/article/details/80843595)  
> 
> 对于一个对象，进入<font color = red>对象锁</font>的代码域之后，线程对该锁进行计数。没有进入锁时，该锁的计数值为 0。第一个获取到该锁的线程获得该对象锁，此后每多一个线程对该锁进行申请，则计数值 +1。每当有一个线程释放了这个锁，则该锁对应的计数值 -1。直到这个锁的计数值重新回到 0，其他线程才可以获取到该锁的所有权。  
> 
> 我对于<font color=red>对象锁</font>的理解：
> 
> - 对于 synchronized，可以为 synchronized 方法的实例对象 this，或者 synchronized(object) 的 object，都是对象锁；
> - 对于 Lock，就是 lock 对象本身；

**不可重入锁**：以可重入的对立面理解即可：对于某个 Lock 的实现，如果该 Lock 被线程A锁住，**线程A的其他对象**想要获取该 Lock，但由于在此之前 Lock 已经被锁住，所以这里不能获取到该 Lock。总之感觉不可重入锁并不是一种合适的 Lock 的类型。

不可重入锁的代码实例如下：

```java
public class Lock {
    private boolean isLocked = false;
    public synchronized void lock() throws InterruptedException {
        while(isLocked) {    
            wait();
        }
        isLocked = true;
    }
    public synchronized void unlock() {
        isLocked = false;
        notify();
    }
}

public class Count {
    Lock lock = new Lock();
    public void print() {
        lock.lock();
        doAdd();
        lock.unlock();
    }
    public void doAdd() {
        lock.lock();
        //do something
        lock.unlock();
    }
}
```

Lock 接口主要有六个方法：

```java
// 获取锁
void lock();
// 获取锁，可中断
void lockInterruptibly() throws InterruptedException;
// 尝试获取锁；如果没有获取到，则返回 false；如果获取到则返回 true
boolean tryLock();
// 尝试在某段时间内获取锁，如果等待一段时间仍没有获取到则返回 false；
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
// 释放锁
void unlock();
// 条件锁
Condition newCondition();
```

## 10.2 公平锁与非公平锁

为什么ReentrantLock默认采用的是非公平模式？因为非公平模式效率比较高。非公平模式会在一开始就尝试两次获取锁，如果当时正好 state 的值为 0，它就会成功获取到锁，少了排队导致的阻塞/唤醒过程，并且减少了线程频繁的切换带来的性能损耗。  

同时非公平模式也存在弊端。非公平模式有可能会导致一开始排队的线程**一直获取不到锁**，导致线程饿死。  

公平锁与非公平锁在源码上的区别：

- **公平锁**：在尝试获取锁的时候，首先会判断 AQS 线程队列的头部是否为当前线程（队列的特性决定了公平性），如果**当前线程位于 AQS 线程队列的头部**，则说明当前线程等待的时间最长，当前线程有资格比较状态，然后才能获取到锁。
- **非公平锁**：尝试获取锁的时候不会判断 AQS 队列头部信息，直接进行比较状态并尝试获取锁。

# 十一. Lock 和 Synchronized 的区别？他们都是可重入锁吗？哪个效率更高？

Lock 和 synchronized 的区别：

对于 synchronized 关键字，与 Lock, ReadWriteLock 的相似性与区别，在于：

- **实现方式**：synchronized 关键字是通过 Java 语言的特性实现锁的特性的；Lock, ReadWriteLock 只是接口；
- **重入性**：synchronized, ReentrantLock, ReentrantReadWriteLock 都是重入锁；<font color=red>？？</font>
- **中断性**：synchronized 修饰的代码，在运行正常情况、异常阻塞的情况下都不能被中断；Lock 锁住的部分，在运行正常情况下不能被中断，但阻塞可以通过 interrupt 方法抛出中断异常而释放锁；
- **读写性**：synchronized 修饰的代码，无论读写，同一时刻都只能有一个线程读/写；ReadWriteLock 的实现类 ReentrantReadWriteLock 可以实现多个线程同时读 / 一个线程写（如果有一个线程A正在写，这种情况下其他对同一个文件进行读/写操作的线程都等待线程A把该 ReentrantReadWriteLock 释放）；
- **效率**：如果有大量线程对同一个资源竞争很激烈，synchronized 的并发性能很大程度上会弱于 Lock；

# 十二. Java 的代理

见 [《设计模式篇》](./设计模式.md)。

# 十三. 线程池的核心参数和基本原理？线程池的调优策略

# 十四. 简述线程池和并发工具有哪些？

# 十五. 负载均衡的原理

# 十六. 如何实现高并发环境下的削峰、限流？

流量削峰的由来主要是还是来自于互联网的业务场景，例如，马上即将开始的春节火车票抢购，大量的用户需要同一时间去抢购；以及大家熟知的阿里双11秒杀，短时间上亿的用户涌入，瞬间流量巨大（高并发），比如：200万人准备在凌晨12:00准备抢购一件商品，但是商品的数量缺是有限的100-500件左右。  
这样真实能购买到该件商品的用户也只有几百人左右，但是从业务上来说，秒杀活动是希望更多的人来参与，也就是抢购之前希望有越来越多的人来看购买商品。  
但是，在抢购时间达到后，用户开始真正下单时，秒杀的服务器后端缺不希望同时有几百万人同时发起抢购请求。  
我们都知道服务器的处理资源是有限的，所以出现峰值的时候，很容易导致服务器宕机，用户无法访问的情况出现。  
这就好比出行的时候存在早高峰和晚高峰的问题，为了解决这个问题，出行就有了错峰限行的解决方案。同理，在线上的秒杀等业务场景，也需要类似的解决方案，需要平安度过同时抢购带来的流量峰值的问题，这就是流量削峰的由来。

## 1. 怎样来实现流量削峰方案

削峰从本质上来说就是更多地延缓用户请求，以及层层过滤用户的访问需求，遵从“最后落地到数据库的请求数要尽量少”的原则。

### (1) 消息队列解决削峰

要对流量进行削峰，最容易想到的解决方案就是用消息队列来缓冲瞬时流量，把同步的直接调用转换成异步的间接推送，中间通过一个队列在一端承接瞬时的流量洪峰，在另一端平滑地将消息推送出去。  
消息队列中间件主要解决应用耦合，异步消息，流量削锋等问题。常用消息队列系统：目前在生产环境，使用较多的消息队列有 ActiveMQ、RabbitMQ、ZeroMQ、Kafka、MetaMQ、RocketMQ 等。  
在这里，消息队列就像“水库”一样，拦蓄上游的洪水，削减进入下游河道的洪峰流量，从而达到减免洪水灾害的目的。具体的消息队列MQ选型和应用场景可以参考我的往期文章：《高并发架构系列：详解分布式之消息队列的特点、选型、及应用场景》

## 2. 流量削峰漏斗：层层削峰

针对秒杀场景还有一种方法，就是对请求进行分层过滤，从而过滤掉一些无效的请求。分层过滤其实就是采用“漏斗”式设计来处理请求的，如下图所示：

![](https://upload-images.jianshu.io/upload_images/15069341-4275977ca5551477)

这样就像漏斗一样，尽量把数据量和请求量一层一层地过滤和减少了。

1. 分层过滤的核心思想通
	- 过在不同的层次尽可能地过滤掉无效请求；
	- 通过CDN过滤掉大量的图片，静态资源的请求。
	- 再通过类似Redis这样的分布式缓存，过滤请求等就是典型的在上游拦截读请求。
2. 分层过滤的基本原则
	- 对写数据进行基于时间的合理分片，过滤掉过期的失效请求。
	- 对写请求做限流保护，将超出系统承载能力的请求过滤掉。
	- 涉及到的读数据不做强一致性校验，减少因为一致性校验产生瓶颈的问题。
	- 对写数据进行强一致性校验，只保留最后有效的数据。
	- 最终，让“漏斗”最末端(数据库)的才是有效请求。例如：当用户真实达到订单和支付的流程，这个是需要数据强一致性的。

## 3. 总结

1. 对于秒杀这样的高并发场景业务，最基本的原则就是将请求拦截在系统上游，降低下游压力。如果不在前端拦截很可能造成数据库(mysql、oracle等)读写锁冲突，甚至导致死锁，最终还有可能出现雪崩等场景。
2. 划分好动静资源，静态资源使用CDN进行服务分发。
3. 充分利用缓存(redis等)：增加QPS，从而加大整个集群的吞吐量。
4. 高峰值流量是压垮系统很重要的原因，所以需要Kafka等消息队列在一端承接瞬时的流量洪峰，在另一端平滑地将消息推送出去。以上是就是流量削峰的详解。

# 十七. 高并发架构的设计思路

## 1. 前言

随着互联网的快速发展，很多传统行业都开始将原有的产品互联网化移动化，这其中就涉及到对原有系统的改造，因为之前大部分时间都是在传统银行工作所以对于原先的系统设计我们也有一个套路，类似传统的SSH、LAMP这种，但是随着技术的不断快速发展，互联网高并发的架构设计也有了新的模式，本文就介绍下基本的高并发设计模式。互联网大部分系统的设计采用本文的设计模式都是可以的，但是对于一些超高并发的特殊场景的系统还是需要根据具体业务场景单独去设计的，下面我们就对高并发设计模式进行讲解。

## 2. 分层

分层设计是企业应用系统中常见的设计模式，为了保证后续系统的可拆分和可复用，一般会将系统拆分为应用层、服务层、数据层，这也是传统的系统分层拆分模式。通过分层，可以更好的将一个庞大的软件系统切分为不同的部分，便于分工合作并行开发和维护。每层都是独立的，互相通过接口进行调用，各层可以根据业务情况的变化独立作出调整，只需要保证接口一致性即可。  
分层比较大的挑战就是合理规划和划分每一层的边界和接口，在实施过程中禁止跃层调用。对于分层架构不是一成不变的，一般来说在实际过程中，会根据具体的情况再细化分层，应用层可以分为视图层、业务逻辑层等，服务层也可以细分为数据接口适配层、逻辑处理层等。  
分层架构对网站支持高并发分布式的发展方向至关重要，所以在系统刚开始建的时候最好就要采取分层架构，这样后续分层会比较容易。

## 3. 分割

分层是对系统的横向的切分，分割则是对系统纵向的切分。系统越大，分割的就会越细，例如银行系统我们会切分为核心系统、账户系统、支付系统、现金管理系统、营销系统等等，基本上你在网银或者手机银行上看到的每个功能背后基本都是一个系统去支撑，例如我们常见的手机银行里的账户交易查询，就这么一个简单的交易查询功能，对于有一定规模的银行来说，都会单建一个交易查询系统提供全渠道的查询服务。  
按照这样将功能从业务层面分割以后，各个系统承载的压力自然也就下降下来，而且扩展性也会比较好。不过功能分割对于大型网站的分割粒度一般会比较小，对于小规模的公司来说一般没必要上来就分割成很多子系统，随着业务的发展再进行分割就可以。

## 4. 分布式

分布式由来已久，分布式意味着可以通过增加机器完成同样的功能，机器越多，cpu资源、内存、磁盘都是随着机器的增加而线性增加，机器越多能够处理的并发访问和数据量就越大，从而能为更多的用户提供服务。  
分布式架构需要考虑很多问题，并不是简单的加机器就可以了。实施分布式以后需要考虑用户会话如何管理，数据在分布式环境中如何保持数据一致性，分布式事务如何保证一致性，分布式的日志维护等都是需要考虑的问题，所以分布式设计要根据具体情况而来，不要为了分布式而分布式。
分布式方案有以下几种：  

1. 分布式应用和服务：将分层和分割后的应用分布式部署，这样可以提高网站性能和并发，加快开发和发布速度，还可以让不同应用交叉复用共同的应用，便于功能扩展。
2. 分布式静态资源：网站的静态资源独立分布式部署，并采用独立的域名，这就是动静分离。静态资源的分布式部署可以减少应用服务器的压力，使用独立域名可以加快加载速度，也可以降低静态资源服务器的压力。
3. 分布式数据和存储：大型网站处理的数据都是P级的，单台服务器无法提供这么大的存储空间，这么大的数据需要分布式存储。除了分布式文件存储，还有关系型数据库和Nosql都有分布式的部署方案。通过分布式能够提供海量的数据存储空间。
4. 分布式计算：对于很多后台批量处理的任务都是采用分布式计算方案，常用的有Hadoop和MapReduce分布式计算框架。分布式计算框架内容比较多。

## 5. 集群

分布式部署的应用和服务部署了很多机器后还需要将其集群化才能对外提供服务，多台服务器部署相同应用构成一个集群，通过负载均衡设备共同对外提供服务。  
当业务量不断增大以后，可以在集群中不断增加服务器就可以满足业务增长了，当一台服务器坏了也不会影响对外的服务提供，只是性能有所下降。

## 6. 缓存

缓存是高并发系统的杀手锏，在真正有高并发需求的系统一般会设计多级缓存来减少真正的服务计算，而是直接通过缓存提供服务。像微博、朋友圈这些高QPS的系统都是采用了多级缓存的架构方案。

1. CDN：CDN供应商一般都部署在距离用户最近的网络服务商，用户的请求总是先到网络服务商，在这里缓存网站的静态资源，可以就近将静态数据返回给用户。一般来说对于电商、社交应用、新闻门户或者视频网站这些高QPS网站，都建议使用CDN来环节源系统的压力。
2. 反向代理：反向代理属于网站前端架构的一部分，部署在网站的前端，当用户请求到达网站的数据中心时，最先访问到的就是反向代理服务器，这里将网站的静态资源缓存起来，这样不需要继续转发应用服务器，直接从反向代理缓存中返回就可以了。
3. 本地缓存：对于应用服务器被访问的高热点数据，应用程序可以在服务器内存中缓存这些热点数据，这样就不需要访问服务器磁盘或者数据库了，能够将内存中的热点数据直接返回。
4. 分布式缓存：很多系统数据量非常大，光靠本地缓存可能内存空间不够，这时候就可以考虑使用分布式缓存了，常用的分布式缓存有redis、memcache这些key-value分布式缓存，这些分布式缓存一般也都提供了集群版本，能够做到很好的高可用和高性能。

## 7. 异步

异步也是处理高并发的一把利器，也是处理解耦的手段之一，业务系统之间的消息传递不是同步调用，而是将一个业务操作分为多个阶段，每个阶段之间通过共享数据的方式异步执行。  
在单一服务器内部可通过多线程共享队列的方式实现异步，处在业务操作前面的线程将输出的内容写入队列，后面的处理线程从队列中取出数据进行处理；在分布式系统中，多个服务器集群通过分布式消息队列实现异步。  
对于简单的异步实现可以通过内存队列来实现，对于需要高可用和高并发的系统一般来说都依靠商用的消息队列中间件产品，Websphere MQ、Rabbit MQ、Rocket MQ等都是比较好的消息中间件产品。

## 8. 冗余

为了保证系统的高可用，我们一般会对服务器和数据都进行冗余备份，这样可以保证在服务器宕机的情况下系统依然可以保证提供服务。  
访问和负载很小的应用服务也必须部署至少两台服务器组成一个集群，就是为了通过冗余来实现高可用。数据库要定期进行全量备份，每天还要进行增量备份，防止数据库服务器出现异常导致数据丢失，对于实时的数据备份可以通过数据库自带的日志来进行恢复。

## 9. 总结

上面介绍的这些就是常用的一些高并发的设计模式，其中每一条都值得深入的研究，本文只是介绍了高并发设计时需要考虑的设计方案简介，可以通过这些方案缓解压力提高并发性能和高可用，每一点都可以单独去详细介绍下，后面我会再写几篇分别介绍下这些模式的具体应用方式。

# 十八. 生产者消费者模型的实现

## 1. 阻塞队列 BlockingQueue

实现如下：

```java
import java.util.concurrent.BlockingQueue;

public class Consumer implements Runnable {
    /*
        BlockingQueue put(e) 和 take() 这两个方法是带阻塞的。
     */
    BlockingQueue<String> queue;

    public Consumer(BlockingQueue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            String tmp = queue.take();
            System.out.println(Thread.currentThread().getName() + " have consumed a product from " + tmp);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

```java
import java.util.concurrent.BlockingQueue;

public class Producer implements Runnable {
    BlockingQueue<String> queue;

    public Producer(BlockingQueue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            String tmp = Thread.currentThread().getName();
            System.out.println(Thread.currentThread().getName() + " have made a product");
            queue.put(tmp);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class Main {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new LinkedBlockingQueue<String>(2);

        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);
        for (int i = 0; i < 5; i++) {
            new Thread(producer, "Producer" + (i + 1)).start();

            new Thread(consumer, "Consumer" + (i + 1)).start();
        }

    }
}
```

2. wait, notify 的方式实现

> 参考地址：  
> [《多线程的休息室WaitSet详细介绍》](https://www.cnblogs.com/ch-forever/p/10752499.html)  
> [《生产者消费者--BlockingQueue和wait、notify两种方式实现》](https://blog.csdn.net/weixin_40183884/article/details/83152825)

WAITTING 是多线程的一种状态，使用了 Object#wait(), LockSupport#park() 的方法都会进入 WAITTING 状态（放弃了对该对象锁的争夺权）。notify() 可以唤醒被 wait 的方法，使用了 Object#notify(), notifyAll(), LockSupport#unpark() 的方法可以把线程从 WAITTING 状态转到 RUNNABLE 状态。

实际上，所有对象都有一个**线程休息室 wait set**，用于存放调用了**该对象 wait 方法后进入了 WAITTING 状态的线程**。  
当调用该对象的 notify() 方法时，wait set 中会弹出一个线程，重新进入争夺该对象的锁；此外，Java 官方只定义了 waitset，却没有定义 waitset 的具体数据结构，所以根据 Java 虚拟机厂商的不同，waitset 可以使用 list, set 等不同的数据结构，所以线程从 waitset 中唤醒的顺序并不一定是 FIFO；  
当调用该对象的 notifyAll() 方法时，wait set 会将在其中的所有线程弹出，这些线程重新争夺该对象的锁。

```java
import java.util.LinkedList;
import java.util.List;

public class Consumer implements Runnable {
    private List<String> buffer = new LinkedList<String>();
    private Integer capacity;

    public Consumer(List buffer, Integer capacity) {
        this.buffer = buffer;
        this.capacity = capacity;
    }

    @Override
    public void run() {
        synchronized (capacity){
            try {
                if (buffer.isEmpty())
                    capacity.wait();
                String tmp = buffer.remove(0);
                System.out.println(Thread.currentThread().getName() + " have consume a product from " + tmp);
                capacity.notifyAll();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
import java.util.LinkedList;
import java.util.List;

public class Producer implements Runnable {
    private List<String> buffer = new LinkedList<String>();
    private Integer capacity;

    public Producer(List buffer, Integer capacity) {
        this.buffer = buffer;
        this.capacity = capacity;
    }

    @Override
    public void run() {
        synchronized (capacity){
            try {
                if (buffer.size() >= capacity)
                    capacity.wait();
                buffer.add(Thread.currentThread().getName());
                System.out.println(Thread.currentThread().getName() + " have made a product");
                capacity.notifyAll();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
import java.util.LinkedList;
import java.util.List;

public class Main {

    public static void main(String[] args) {
        Integer capacity = 5;
        List<String> buffer = new LinkedList<String>();
        Producer producer = new Producer(buffer, capacity);
        Consumer consumer = new Consumer(buffer, capacity);
        for (int i = 1; i < 5; i++) {
            new Thread(producer,"Producer" + i).start();
            new Thread(consumer,"Consumer" + i).start();
        }
    }
}
```



# 十九. Java中nio和io的区别？常用的类有哪些？

