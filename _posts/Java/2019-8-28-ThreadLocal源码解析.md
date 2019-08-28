---
layout: post
title: ThreadLocal源码解析
tags: Java
---

------

------

[TOC]

# 1. 概述

简单来说，ThreadLocal是一个创建线程局部变量的类。为什么会需要这么一个类？我想一方面是线程需要访问线程外的变量，另一方面是自己不需要对其做更新，所以就存一个本地副本就好了。

所以ThreadLocal常常用于数据库连接池和Session管理

# 2. 使用方法

一个简单的使用例子：

```java
import java.util.Random;

/**
 * ThreadLocal测试
 */
public class ThreadLocalDemo implements Runnable
{
    private final static ThreadLocal<People> threadLocal = new ThreadLocal<People>(){
        @Override
        protected People initialValue(){
            return new People(101);
        }
    };

    public static void main(String[] agrs)
    {
        ThreadLocalDemo td = new ThreadLocalDemo();
        Thread t1 = new Thread(td);
        Thread t2 = new Thread(td);
        t1.start();
        t2.start();
        System.out.println("origin age is: "+threadLocal.get().getAge());
    }

    public void run()
    {
        // 获取当前线程的名字
        String currentThreadName = Thread.currentThread().getName();
        System.out.println(currentThreadName + " is running!");
        // 产生一个随机数并打印
        Random random = new Random();
        int age = random.nextInt(100);
        System.out.println("thread " + currentThreadName + " set age to:" + age);
        People people = threadLocal.get(); //从ThreadLocal中获取一个copy
        people.setAge(age);
        System.out.println("thread " + currentThreadName + " first read age is:"
                + people.getAge());
        try
        {
            Thread.sleep(500);
        }
        catch (InterruptedException ex)
        {
            ex.printStackTrace();
        }
        System.out.println("thread " + currentThreadName + " second read age is:"
                + people.getAge());

    }
}

class People
{
    private int age = 0;

    public People(){}
    public People(int age){
        this.age = age;
    }

    public int getAge()
    {
        return this.age;
    }

    public void setAge(int age)
    {
        this.age = age;
    }
}
```

某一次的运行结果如下：

```java
Thread-1 is running!
Thread-0 is running!
origin age is: 101
thread Thread-1 set age to:56
thread Thread-1 first read age is:56
thread Thread-0 set age to:46
thread Thread-0 first read age is:46
thread Thread-1 second read age is:56
thread Thread-0 second read age is:46
```

每次从ThreadLocal中获取的都是目标对象的一个copy，所以无论在本地线程中怎么修改，其原始值是不变的。

# 3. 源码解析

## 1. 属性

```java
    //每个ThreadLocal做hash时使用的一个变量
	private final int threadLocalHashCode = nextHashCode();

	//从零开始的自增的hashcode
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
	//threadLocalHashCode增量
    private static final int HASH_INCREMENT = 0x61c88647;
```

```java
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);//原子加
    }
```

上面三个属性和nextHashCode函数一起实现一个功能，就是为ThreadLocal设置一个threadLocalHashCode。而这个hashcode是计算map的key必须的一个值。

而HASH_INCREMENT上面那段英文注释，实际上是在解释为什么会用这个数，也就是达到的效果——配合map的大小为2的幂次，可以使hash后的key均匀的分布，减小冲突

## 2. 构造函数

```java
public ThreadLocal() {
}
```

只有一个无参构造函数，没啥好说的。

## 3. 静态内部类ThreadLocalMap

### 1. 概述

在说ThreadLocal的主要方法之前，先介绍一下其静态内部类ThreadLocalMap。其是ThreadLocal底层存储数据的数据结构。ThreadLocalMap是静态的。

```java
static class ThreadLocalMap {}
```

### 2. 属性

其有四个属性：

```java
//The initial capacity -- MUST be a power of two.
private static final int INITIAL_CAPACITY = 16;//map的初始大小
private Entry[] table;//真正存数据的地方
private int size = 0;//tables中entry数量
private int threshold; //触发table进行扩容的大小
```

Entry是WeakReference的子类，保存ThreadLoca的引用和value

### 3. 构造函数

#### 1. 新创建一个ThreadLocalMap：

```java
//firstKey是ThreadLocal自身，firstValue是存的值
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];//Map初始容量为16
    //这里是hash函数，取threadLocalHashCode的后四位，随着map扩容，是取后log(Capacity)位
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

hash函数是，取当前ThreadLocal的threadLocalHashCode的后log(Capacity)位i，其中Capacity保证是2的次幂。因为初始值INIT_CAPACITY为16，而且每次都是2倍扩容。setThreshold函数如下：

```java
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

也就是说，负载因子是2/3。

#### 2. 从父线程继承ThreadLocalMap：

```java
//@param parentMap the map associated with parent thread.
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();//获取key
            if (key != null) {
                Object value = key.childValue(e.value);//实际就是value
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);//计算hash值
                while (table[h] != null)
                    h = nextIndex(h, len);//开放定址法解决冲突
                table[h] = c;
                size++;
            }
        }
    }
}
```

其中nextIndex函数如下：

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

另外，childValue函数如下

```java
/**
 * Method childValue is visibly defined in subclass
 * InheritableThreadLocal, but is internally defined here for the
 * sake of providing createInheritedMap factory method without
 * needing to subclass the map class in InheritableThreadLocal.
 * This technique is preferable to the alternative of embedding
 * instanceof tests in methods.
 */
T childValue(T parentValue) {
    throw new UnsupportedOperationException();
}
```

按照注释中说的，这个函数会在子类中定义，之所以在这里也定义，是为了creatInheritedMap函数的便利，而不需要将InheritableThreadLocal类编入ThreadLocalMap中(纯属个人理解加翻译，如果有错误麻烦指正，感激不尽)

而在InheritableThreadLocal的childValue函数中，直接返回parentValue，如下：

```java
protected T childValue(T parentValue) {
    return parentValue;
}
```



#### 3. 总的来说

1. ThreadLocalMap既可以直接创建一个，也可以从父线程继承
2. 底层是数组实现Map
3. 长度为2的幂次，负载因子是2/3
4. hash函数是取threadLocalHashCode的后log(Capacity)位
5. 解决冲突的方式是开放定址法

### 4. 核心函数

#### 1. 获取Map中的数据

```java
//根据key获取entry
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);//对key计算hash值
    Entry e = table[i];//获取hash值位置上的entry
    if (e != null && e.get() == key)//如果entry不为空，且entry保存的ThreadLocal就是key，则返回
        return e;
    else //否则存在冲突，key对应的entry被保存到其他位置
        return getEntryAfterMiss(key, i, e);
}
```

其中，当首次不命中，则需要调用下面的函数处理

```java
//key是threadLocal，i是key的hash值，e是table[i]的值
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();//获取e中的key
        if (k == key)//如果是key，返回
            return e;
        if (k == null)//如果是null，则删除i上的entry，并调整i后的entry，重新hash一次
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);//根据开放地址法，找下一个地址
        e = tab[i];
    }
    return null;
}
```

getEntryAfterMiss函数思路很简单，就是从i开始，往后找key的entry，如果找到就返回，找不到返回null，中间遇到entry中的key为null时，删除该项，并调整i后的entry

其中，当entry中的key为null时，调用下面的函数

```java
//staleSlot是需要删除的entry的index
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //如果key不为空，则对其rehash
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

#### 2. 往Map中加数据

```java
//ThreadLocal对象为key
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        //存在key，则只需更新值
        if (k == key) {
            e.value = value;
            return;
        }

        //key为null，则检查是否已经存在，存在就更新，不存在就新创建一个entry，并触发垃圾回收
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    //清除无用数据后，size仍然大于threshold，则进行rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

其中，replaceStaleEntry函数如下：

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    //查找staleSlot前面是不是也有无用数据
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    //从staleSlot开始查找后面是否有key对应的entry
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        //如果存在，则将key-value换到staleSlot的位置，并触发垃圾回收
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    //如果没有找到key,则创建一个entry，存到staleSlot上
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    //检查是否有无用数据，触发垃圾回收
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

其中，清除无用数据的函数cleanSomeSlot如下：

```java
/**
	* Heuristically scan some cells looking for stale entries.
    * This is invoked when either a new element is added, or
    * another stale one has been expunged. It performs a
    * logarithmic number of scans, as a balance between no
    * scanning (fast but retains garbage) and a number of scans
    * proportional to number of elements, that would find all
    * garbage but would cause some insertions to take O(n) time.
    */
//清理i后面的slot
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);//每次检查log(len)的slot，如果都没有为null的，则结束
    return removed;
}
```

如其注释，之所以检查log(len)的长度，是权衡了不做垃圾回收和遍历所有元素导致插入操作为O(n)两种情况。

#### 3. 删除Map数据

```java
//key是ThreadLocal
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);//Hash值
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();//将指向ThreadLocal的指针设为null，方法继承于Reference
            expungeStaleEntry(i);//触发垃圾回收
            return;
        }
    }
}
```

删除数据，就是将ThreadLocal设为null，然后触发一次垃圾回收

#### 4. rehash

```java
//只在set函数中会触发rehash，要求size大于等于threshold
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    //垃圾回收之后，size仍然大于Capacity的一半(粗略计算)，则扩容。
    if (size >= threshold - threshold / 4)
        resize();
}
```

其中，全Map垃圾回收函数如下：

```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

而，扩容函数resize如下：

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;//2倍扩容
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```



### 5. 注意点

#### 1. Point1

从名字可以看出，ThreadLocalMap是一个Map，但是ThreadLocalMap并没有实现Map接口，而是独立实现的一个Map。

#### 2. Point2

虽然ThreadLocalMap是static的，但是并非所有的ThreadLocal共享一个ThreadLocalMap，而是一个线程维护一个ThreadLocalMap，该线程内的所有ThreadLocal都存在同一个ThreadLocalMap中，这样就不存在共享问题了。

```java
import java.util.Random;

/**
 * ThreadLocal测试
 *
 */
public class ThreadLocalDemo implements Runnable
{
    public static void main(String[] agrs)
    {
        ThreadLocalDemo td = new ThreadLocalDemo();
        Thread t1 = new Thread(td);
        t1.start();
        ThreadLocal<People> threadLocal = new ThreadLocal<People>();
        if(threadLocal.get()==null){
            System.out.println("Thread["+Thread.currentThread().getName()+"],ThreadLocalMap is Empty!");
        }else{
            System.out.println("Thread["+Thread.currentThread().getName()+"],"+threadLocal.get().getAge());
        }
    }

    public void run()
    {
          ThreadLocal<People> threadLocal1 = new ThreadLocal<People>(){
              @Override
              protected People initialValue(){
                  return new People(new Random().nextInt(100));
              }
          };
          ThreadLocal<Animal> threadLocal2 = new ThreadLocal<Animal>(){
              @Override
              protected Animal initialValue(){
                  return new Animal("Monkey");
              }
          };
          People peo = threadLocal1.get();
          Animal ani = threadLocal2.get();
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+peo.getAge());
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+ani.getType());
    }
}

class Animal{
    private String type;
    public Animal(){};
    public Animal(String type){
        this.type = type;
    }
    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
}

```

结果是：

```java
Thread[main],ThreadLocalMap is Empty!
Thread[Thread-0],59
Thread[Thread-0],Monkey

```

可见main线程中ThreadLocalMap是空的，而Thread-0中的ThreadLocalMap存在两个ThreadLocal。

## 4. 核心函数

### 1. 获取ThreadLocal数据副本

流程图如下：

![1566963195135](D:\git\YunWang.github.io\_posts\Java\assets\1566963195135.png)

源码如下，从get函数开始

```java
//获取当前线程中的ThreadLocal的副本
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);//获取ThreadLocalMap
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);//获取当前ThreadLocal对应的value
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();//返回初始值
}
```

其中，getMap函数就是返回Thread中的ThreadLocal变量

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

可以看到，ThreadLocalMap每个线程都维护一个。

另外，setInitialValue函数是设置初始值

```java
private T setInitialValue() {
    T value = initialValue();//调用初始化函数
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);//创建一个Map
    return value;
}
```

先设置一个初值，然后将ThreadLocal与value存到Map中。如果Map为空，则创建一个Map

其中，initialValue函数如下

```java
protected T initialValue() {
    return null;
}
```

是的，你没看错，就是直接返回null。所以一般，在创建ThreadLocal变量时，会重写这个函数。

另外一个，创建Map的函数如下：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

createMap函数也很简单，就是调用ThreadLocalMap的构造函数，给当前线程的threadLocals变量赋值。

#### 2. 设置ThreadLocal对应的值

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

set函数也很简单了，先获取Map，为null的话，新建一个，否则就将ThreadLocal和value添加道ThreadLocalMap中。

#### 3. 删除ThreadLocal

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

首先获取当前线程的ThreadLocalMap，然后删除ThreadLocal对应的entry

#### 4. 从父进程继承ThreadLocalMap

```java
/**
 * Factory method to create map of inherited thread locals.
 * Designed to be called only from Thread constructor.
 */
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

在线程的构造函数中调用这个函数，从父进程中继承ThreadLocalMap。

下面是对继承ThreadLocalMap的测试

```java
/**
 * ThreadLocal测试
 *
 */
public class ThreadLocalDemo implements Runnable
{
    private ThreadLocal<People> threadLocal1;
    private InheritableThreadLocal<Animal> threadLocal2;

    public ThreadLocalDemo(ThreadLocal threadLocal1,ThreadLocal threadLocal2){
        this.threadLocal1 = threadLocal1;
        this.threadLocal2 = (InheritableThreadLocal<Animal>) threadLocal2;
    }

    public static void main(String[] agrs)
    {
        ThreadLocal<People> threadLocal1 = new ThreadLocal<People>(){
            @Override
            protected People initialValue(){
                return new People(50);
            }
        };
        InheritableThreadLocal<Animal> threadLocal2 = new InheritableThreadLocal<Animal>(){
            @Override
            protected Animal initialValue(){
                return new Animal("Monkey");
            }
        };
        threadLocal1.set(new People(100));
        threadLocal2.set(new Animal("Horse"));
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+threadLocal1.get().getAge());
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+threadLocal2.get().getType());
        ThreadLocalDemo td = new ThreadLocalDemo(threadLocal1,threadLocal2);
        Thread t1 = new Thread(td);
        t1.start();
    }

    public void run()
    { 
        People peo = threadLocal1.get();
        Animal ani = threadLocal2.get();
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+peo.getAge());
        System.out.println("Thread["+Thread.currentThread().getName()+"],"+ani.getType());
    }
}
```

结果为

```java
Thread[main],100
Thread[main],Horse
Thread[Thread-0],50
Thread[Thread-0],Horse
```

从上面的测试例子，可以看出，Thread中维护了一个ThreadLocalMap和一个InheritableThreadLocal。ThreadLocalMap是不能继承的，而InheritableThreadLocal是可以从父进程中继承的(实际上，在线程的多种创建方式中，除了通过继承AccessControlContext创建的线程外，默认都是会继承父进程的InheritableThreadLocal的)。在创建的时候会调用createInheritedMap函数，从父进程继承InheritableThreadLocal。

# 4. 总结

1. 一个线程一个ThreadLocalMap，一个ThreadLocalMap可以保存多个ThreadLocal与Value的键值对
2. 线程的ThreadLocalMap，是不可以继承的；而InheritableThreadLocal是可以从父进程继承的。
3. ThreadLocal可以用来实现线程安全，只需将非线程安全的对象使用ThreadLocal封装之后，就会变得线程安全
4. ThreadLocal还用来实现单个线程上下文信息存储，比如交易id

# 参考

[ThreadLocal 和神奇的 0x61c88647](https://blog.csdn.net/important0534/article/details/50706774)

[[InheritableThreadLocal类原理简介使用 父子线程传递数据详解 多线程中篇（十八）]](https://www.cnblogs.com/noteless/p/10448283.html#)

[理解Java中的ThreadLocal](https://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/)

