---
layout: post
title: 关于volatile、MESI、内存屏障、#Lock
tags: Java
---

***

***

\# 说明：此篇文章转载自[关于volatile、MESI、内存屏障、#Lock](https://www.jianshu.com/p/6745203ae1fe)，仅作为个人记录

## 1. 可见性和MESI

### 1.1 内存模型

在JVM的内存模型中，每个线程有自己的工作内存。Java线程借助了底层操作系统线程实现，一个JVM线程对应一个操作系统，线程的工作内存其实是CPU寄存器和高速缓存的抽象。现代处理器的缓存分为三级，由每一个CPU核共享的L1,L2Cache，以及所有CPU核共享的L3Cache组成。每个Cache，实际上是有很多缓存行（我理解为数据存取单元）组成的。

![1567127857357](D:\git\YunWang.github.io\_posts\Java\assets\1567127857357.png)

### 1.2 缓存一致性和MESI

缓存一致性协议给缓存行（通常为64字节）定义了四个状态：修改（modified）、独占（exclusive）、共享（share）、失效（invalid），用来描述该缓存行是否被多处理器共享，是否被修改。所以缓存一致性协议也称作MESI协议。

1. 修改（Modified）：仅当前处理器拥有该缓存行，并且缓存行被修改过，一定时间会写回主存，写回成功状态会变成S
2. 独享（Exclusive）：仅当前处理器拥有该缓存行，并且没有修改过，是最新值。
3. 共享（Share）：有多个处理器拥有该缓存行，每个处理器都没有修改过缓存，是最新值
4. 失效（Invalid）：有多个处理器拥有该缓存行，该缓存行被其他处理器修改过，不是最新值，需要读取主存上最新值

协议如下：

1. 一个处于M状态的缓存行，必须时刻监听所有试图读取该缓存行对应的主存地址的操作，如果监听到，则必须在此操作执行前把其缓存行中的数据写回CPU，写回成功状态变为S
2. 一个处于S状态的缓存行，必须时刻监听使该缓存行无效或独享该缓存行的请求，如果监听到，则必须把其缓存行状态设置为I
3. 一个处于E状态的缓存行，必须时刻监听其他试图读取该缓存行对应的主存地址的操作，如果监听到，则必须把其缓存行状态设置为S
4. 当CPU读取数据时，如果其缓存行状态是I，则需要从主存中读取，并把自己状态设置为S；如果不是I，则可以直接读取缓存中的值，但在此之前，必须要等待其他CPU的监听结果，如果其他CPU也有该数据的缓存且状态是M，则需要等待其缓存更新到内存之后，再读取。
5. 当CPU需要写数据时，只有在其缓存行是M或E的时候才行，否则需要发出特殊的RFO指令（Read of Ownership，一种总线事务），通知其他CPU置缓存无效I，这种情况下性能开销相对较大，在写入完成后，修改状态为M。

![1567133307369](D:\git\YunWang.github.io\_posts\Java\assets\1567133307369.png)

这个图的含义就是当一个core持有一个cacheline的状态为Y时,其它core对应的cacheline应该处于状态X, 比如地址 0x00010000 对应的cacheline在core0上为状态M, 则其它所有的core对应于0x00010000的cacheline都必须为I , 0x00010000 对应的cacheline在core0上为状态S, 则其它所有的core对应于0x00010000的cacheline 可以是S或者I 。

MESI状态转移

![1567136575804](D:\git\YunWang.github.io\_posts\Java\assets\1567136575804.png)

1.触发事件

| 触发事件                 | 描述                       |
| :----------------------- | :------------------------- |
| 本地读取（Local read）   | 本地cache读取本地cache数据 |
| 本地写入（Local write）  | 本地cache写入本地cache数据 |
| 远端读取（Remote read）  | 其他cache读取本地cache数据 |
| 远端写入（Remote write） | 其他cache写入本地cache数据 |

2.cache分类：
前提：所有的cache共同缓存了主内存中的某一条数据。

本地cache:指当前cpu的cache。
触发cache:触发读写事件的cache。
其他cache:指既除了以上两种之外的cache。
注意：本地的事件触发 本地cache和触发cache为相同。

上图的切换解释：

|       状态        |                         触发本地读取                         |                         触发本地写入                         |                         触发远端读取                         |                         触发远端写入                         |
| :---------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| **M状态（修改）** |             本地cache:M  触发cache:M 其他cache:I             |             本地cache:M  触发cache:M 其他cache:I             | 本地cache:M→E→S 触发cache:I→S 其他cache:I→S 同步主内存后修改为E独享,同步触发、其他cache后本地、触发、其他cache修改为S共享 | 本地cache:M→E→S→I 触发cache:I→S→E→M 其他cache:I→S→I 同步和读取一样,同步完成后触发cache改为M，本地、其他cache改为I |
| **E状态（独享）** |             本地cache:E 触发cache:E 其他cache:I              | 本地cache:E→M 触发cache:E→M 其他cache:I 本地cache变更为M,其他cache状态应当是I（无效） | 本地cache:E→S 触发cache:I→S 其他cache:I→S 当其他cache要读取该数据时，其他、触发、本地cache都被设置为S(共享) | 本地cache:E→S→I 触发cache:I→S→E→M 其他cache:I→S→I 当触发cache修改本地cache独享数据时时，将本地、触发、其他cache修改为S共享.然后触发cache修改为独享，其他、本地cache修改为I（无效），触发cache再修改为M |
|  **S状态(共享)**  |             本地cache:S 触发cache:S 其他cache:S              | 本地cache:S→E→M 触发cache:S→E→M 其他cache:S→I  当本地cache修改时，将本地cache修改为E,其他cache修改为I,然后再将本地cache为M状态 |             本地cache:S 触发cache:S 其他cache:S              | 本地cache:S→I 触发cache：S→E→M 其他cache:S→I 当触发cache要修改本地共享数据时，触发cache修改为E（独享）,本地、其他cache修改为I（无效）,触发cache再次修改为M(修改) |
| **I状态（无效）** | 本地cache:I→S或者I→E 触发cache:I→S或者I →E 其他cache:E、M、I→S、I 本地、触发cache将从I无效修改为S共享或者E独享，其他cache将从E、M、I 变为S或者I |  本地cache:I→S→E→M 触发cache:I→S→E→M 其他cache:M、E、S→S→I   |           既然是本cache是I，其他cache操作与它无关            |           既然是本cache是I，其他cache操作与它无关            |

#### MESI优化和他们引入的问题

缓存的一致性消息传递是要时间的，这就使其切换时会产生延迟。当一个缓存被切换状态时其他缓存收到消息完成各自的切换并且发出回应消息这么一长串的时间中CPU都会等待所有缓存响应完成。可能出现的阻塞都会导致各种各样的性能问题和稳定性问题。

#### CPU切换状态阻塞解决-存储缓存（Store Bufferes）

比如你需要修改本地缓存中的一条信息，那么你必须将I（无效）状态通知到其他拥有该缓存数据的CPU缓存中，并且等待确认。等待确认的过程会阻塞处理器，这会降低处理器的性能。应为这个等待远远比一个指令的执行时间长的多。

#### Store Bufferes

为了避免这种CPU运算能力的浪费，Store Bufferes被引入使用。处理器把它想要写入到主存的值写到缓存，然后继续去处理其他事情。当所有失效确认（Invalidate Acknowledge）都接收到时，数据才会最终被提交。
这么做有两个风险

#### Store Bufferes的风险

第一、就是处理器会尝试从存储缓存（Store buffer）中读取值，但它还没有进行提交。这个的解决方案称为Store Forwarding，它使得加载的时候，如果存储缓存中存在，则进行返回。
第二、保存什么时候会完成，这个并没有任何保证。

```
value = 3；

void exeToCPUA(){
  value = 10;
  isFinsh = true;
}
void exeToCPUB(){
  if(isFinsh){
    //value一定等于10？！
    assert value == 10;
  }
}
```

试想一下开始执行时，CPU A保存着finished在E(独享)状态，而value并没有保存在它的缓存中。（例如，Invalid）。在这种情况下，value会比finished更迟地抛弃存储缓存。完全有可能CPU B读取finished的值为true，而value的值不等于10。

**即isFinsh的赋值在value赋值之前。**

这种在可识别的行为中发生的变化称为重排序（reordings）。注意，这不意味着你的指令的位置被恶意（或者好意）地更改。

它只是意味着其他的CPU会读到跟程序中写入的顺序不一样的结果。

#### 存储缓存Store Buffer

也就是常说的写缓存，当处理器修改缓存时，把新值放到存储缓存中，处理器就可以去干别的事了，把剩下的事交给存储缓存。

#### 失效队列Invalidate Queues

执行失效也不是一个简单的操作，它需要处理器去处理。另外，存储缓存（Store Buffers）并不是无穷大的，所以处理器有时需要等待失效确认的返回。这两个操作都会使得性能大幅降低。为了应付这种情况，引入了失效队列。它们的约定如下：

- 对于所有的收到的Invalidate请求，Invalidate Acknowlege消息必须立刻发送
- Invalidate并不真正执行，而是被放在一个特殊的队列中，在方便的时候才会去执行。
- 处理器不会发送任何消息给所处理的缓存条目，直到它处理Invalidate。

即便是这样处理器已然不知道什么时候优化是允许的，而什么时候并不允许。
干脆处理器将这个任务丢给了写代码的人。这就是内存屏障（Memory Barriers）。

### 1.3 MESI和CAS关系

在x86架构上，CAS被翻译为”lock cmpxchg...“，当两个core同时执行针对同一地址的CAS指令时,其实他们是在试图修改每个core自己持有的Cache line,

> 假设两个core都持有相同地址对应cacheline,且各自cacheline 状态为S, 这时如果要想成功修改,就首先需要把S转为E或者M, 则需要向其它core invalidate 这个地址的cacheline,则两个core都会向**ring bus**发出 invalidate这个操作, 那么在ringbus上就会根据特定的设计协议仲裁是core0,还是core1能赢得这个invalidate, 胜者完成操作, 失败者需要接受结果, invalidate自己对应的cacheline,再读取胜者修改后的值, 回到起点.

对于我们的CAS操作来说, 其实锁并没有消失,只是转嫁到了ring bus的总线仲裁协议中. 而且大量的多核同时针对一个地址的CAS操作会引起反复的互相invalidate 同一cacheline, 造成pingpong效应, 同样会降低性能（参考[9]）。**当然如果真的有性能问题，我觉得这可能会在ns级别体现了,一般的应用程序中使用CAS应该不会引起性能问题**

## 2. 指令重排和内存屏障

### 2.1 指令重排

现代CPU的速度越来越快，为了充分的利用CPU，在编译器和CPU执行期，都可能对指令重排。举个例子：

```
LDR R1, [R0];//操作1
ADD R2, R1, R1;//操作2
ADD R3, R4, R4;//操作3
```

上面这段代码，如果操作1如果发生cache miss，则需要等待读取内存外存。看看有没有能优先执行的指令，操作2依赖于操作1，不能被优先执行，操作3不依赖1和2，所以能优先执行操作3。
 JVM的[JSR-133](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)规范中定义了as-if-serial语义，即compiler, runtime, and hardware三者需要保证在单线程模型下程序不会感知到指令重排的影响。

在并发模型下，重排序还是可能会引发问题，比较经典的就是“单例模式失效”问题

```java
public class Singleton {
  private static Singleton instance = null;

  private Singleton() { }

  public static Singleton getInstance() {
     if(instance == null) {
        synchronzied(Singleton.class) {
           if(instance == null) {
               instance = new Singleton();  //
           }
        }
     }
     return instance;
   }
}
```

上面这段代码，初看没问题，但是在并发模型下，可能会出错,那是因为instance= new Singleton()并非一个原子操作，它实际上下面这三个操作：

```
memory =allocate();    //1：分配对象的内存空间
ctorInstance(memory);  //2：初始化对象
instance =memory;     //3：设置instance指向刚分配的内存地址
```

上面操作2依赖于操作1，但是操作3并不依赖于操作2，所以JVM是可以针对它们进行指令的优化重排序的，经过重排序后如下：

```
memory =allocate();    //1：分配对象的内存空间
instance =memory;     //3：instance指向刚分配的内存地址，此时对象还未初始化
ctorInstance(memory);  //2：初始化对象
```

可以看到指令重排之后，instance指向分配好的内存放在了前面，而这段内存的初始化被排在了后面。在多线程场景下，可能A线程执行到了3，B线程发现已经不为空就返回继续执行，就会出错。

在java里面volatile可以防止重排，当然还有另外一个作用即内存可见性，这个知道的人还应该比较普遍，就不说了

### 2.2 内存屏障

硬件层的内存屏障分为两种：Load Barrier 和 Store Barrier即读屏障和写屏障。内存屏障有两个作用：

> 1.阻止屏障两侧的指令重排序；
>  2.强制把写缓冲区/高速缓存中的脏数据等写回主内存，让缓存中相应的数据失效。

**写屏障 Store Memory Barrier(a.k.a. ST, SMB, smp_wmb)是一条告诉处理器在执行这之后的指令之前，应用所有已经在存储缓存（store buffer）中的保存的指令。**

**读屏障Load Memory Barrier (a.k.a. LD, RMB, smp_rmb)是一条告诉处理器在执行任何的加载前，先应用所有已经在失效队列中的失效操作的指令。**

```java
void executedOnCpu0() {
    value = 10;
    //在更新数据之前必须将所有存储缓存（store buffer）中的指令执行完毕。
    storeMemoryBarrier();
    finished = true;
}
void executedOnCpu1() {
    while(!finished);
    //在读取之前将所有失效队列中关于该数据的指令执行完毕。
    loadMemoryBarrier();
    assert value == 10;
}
```



在JSR规范中定义了4种内存屏障：

- LoadLoad屏障：（指令Load1; LoadLoad; Load2），在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- LoadStore屏障：（指令Load1; LoadStore; Store2），在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：（指令Store1; StoreStore; Store2），在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- StoreLoad屏障：（指令Store1; StoreLoad; Load2），在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。**它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能** 

对于volatile关键字，按照规范会有下面的操作：

- 在每个volatile写入之前，插入一个StoreStore，写入之后，插入一个StoreLoad
- 在每个volatile读取之前，插入LoadLoad，之后插入LoadStore

然而，对于程序员来说简直是一个灾难。不想和平台耦合我们要跨平台。Write One,Run Everywhere!
幸好java解决了这个问题，至于如何解决的请关注JMM(JavaMemoryMode)与物理内存相爱相杀。

## 3. happends-before

结合前面的两点，再看happends-before就比较好理解了。因为光说可见性和重排很难联想到happends-before。这个点在并发编程里还是非常重要的，再详细记录下：

- 1.**Each action in a thread happens-before every subsequent action in that thread** 
- 2.An unlock on a monitor happens-before every subsequent lock on that monitor.
- 3.**A write to a volatile field happens-before every subsequent read of that volatile** 
- 4.A call to start() on a thread happens-before any actions in the started thread.
- 5.All actions in a thread happen-before any other thread successfully returns from a join() on
   that thread.
- 6.**If an action a happens-before an action b, and b happens before an action c, then a happensbefore c**

## 4. \#Local

再往下挖一层，会发现volatile关键字，转换成指令以后，会有一个#lock前缀...原来以为会有相应的内存屏障指令，说好的内存屏障的那些呢？
后来参考了资料[11]以及其他一些文章以后才了解到，任何带有lock前缀的指令以及CPUID等指令都有内存屏障的作用。

## 5. 参考

[CPU缓存一致性协议MESI](https://www.cnblogs.com/yanlong300/p/8986041.html)