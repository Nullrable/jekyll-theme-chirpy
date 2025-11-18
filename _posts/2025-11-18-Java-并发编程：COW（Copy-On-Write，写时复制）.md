---
title: 并发编程：COW（Copy-On-Write，写时复制）
author: nhsoft.lsd
date: 2025-11-18
categories: [Java,并发编程]
pin: false
---

**COW（Copy-On-Write，写时复制）** 是一种常用的无锁设计思想，适用于读多写少的场景。

---

## 一、COW（写时复制）概念

### 原理：

* 多个线程**读取共享数据时，使用同一份数据副本**，不会发生冲突；
* 当某个线程**需要修改数据时**，它**复制一份数据副本**进行修改，修改完成后再将修改后的副本替换为共享数据；
* 读线程看到的始终是**旧的、稳定的数据副本**；
* 写操作的替换过程通常使用 **CAS** 或类似机制保证可见性和原子性。

---

## 二、COW 应用场景

### 1. Java 中的典型实现：

#### `java.util.concurrent.CopyOnWriteArrayList`

适用于“读多写少”场景，避免了加锁带来的性能瓶颈。

---

## 三、CopyOnWriteArrayList 原理简析

### 核心字段：

```java
private transient volatile Object[] array;
```

* 读操作：直接读取 `array`；
* 写操作：复制一份数组，对副本修改，然后通过 CAS 或 `synchronized` 替换 `array`。

### 示例代码：

```java
List<String> cowList = new CopyOnWriteArrayList<>();
cowList.add("A");
cowList.add("B");

// 多线程读取不会加锁
cowList.forEach(System.out::println);

// 写操作底层做了拷贝和替换
cowList.remove("A");
```

---

## 四、CopyOnWriteArrayList 的写操作原理（源码简要）

以 `add()` 为例：

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();          // 获取当前数组
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1); // 拷贝副本
        newElements[len] = e;
        setArray(newElements);                  // 替换引用
        return true;
    } finally {
        lock.unlock();
    }
}
```

虽然 `add()` 使用了锁，但这是**局部短暂锁**，不会影响读线程，也不会阻塞其他线程的读操作，性能仍远优于全局锁。

---

## 五、COW 的优点与缺点

| 优点                 | 缺点             |
| ------------------ | -------------- |
| 读操作无锁，性能极高（适合读多写少） | 每次写都复制数组，开销大   |
| 不会产生读写冲突或数据不一致     | 不适合频繁写入或大数据量结构 |
| 线程安全，不需要额外同步       | 内存消耗高（需要临时副本）  |

---

## 六、自定义简化 COW 实现（Java）

```java
class MyCowList<E> {
    private volatile Object[] array = new Object[0];

    public E get(int index) {
        return (E) array[index];
    }

    public synchronized void add(E e) {
        Object[] oldArray = array;
        Object[] newArray = Arrays.copyOf(oldArray, oldArray.length + 1);
        newArray[oldArray.length] = e;
        array = newArray; // 原子替换
    }

    public int size() {
        return array.length;
    }
}
```

---

## 七、适用场景

* 缓存、配置读取
* 事件监听器集合（读多写少）
* 快照式数据结构（如日志收集）

---

