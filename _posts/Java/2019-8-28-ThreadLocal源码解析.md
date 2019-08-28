---
layout: post
title: ThreadLocal源码解析
tags: Java
---

------

------

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
        return nextHashCode.getAndAdd(HASH_INCREMENT);
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

1. 新创建一个ThreadLocalMap：

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

2. 从父线程继承ThreadLocalMap：

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

3. 总的来说
   1. ThreadLocalMap既可以直接创建一个，也可以从父线程继承
   2. 底层是数组实现Map
   3. 长度为2的幂次，负载因子是2/3
   4. hash函数是取threadLocalHashCode的后log(Capacity)位
   5. 解决冲突的方式是开放定址法

### 4. 核心函数

1. 获取Map中的数据

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

2. 往Map中加数据

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

### 1. Question1

从名字可以看出，ThreadLocalMap是一个Map，但是ThreadLocalMap并没有实现Map接口，而是独立实现的一个Map。

### 2. Question

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

# 参考

[ThreadLocal 和神奇的 0x61c88647](https://blog.csdn.net/important0534/article/details/50706774)