---

---

***

***

## 1. 概述

PriorityQueue通过二叉小顶堆实现。实现Queue接口。Queue是与List和Set同级的接口。

PriorityQueue不允许放入null元素



## 2. 核心方法

### 1. 插入元素

add()和offer()，都可以想Queue中插入元素，不同是两者在插入失败时的处理不同：

1. add函数在插入失败时，抛出异常
2. offer函数在插入失败时，会返回false

### 2. 获取队首元素

element()和peek()，都可以获取权值最小的元素，区别是两者在操作失败时的处理不同：

1. element函数会抛出异常
2. peek函数会返回null

### 3. 删除队首元素

remove()和poll()，都可以删除权值最小的元素，也就是队首的元素，区别同样是失败时的处理不同：

1. remove函数会抛出异常
2. poll函数会返回null