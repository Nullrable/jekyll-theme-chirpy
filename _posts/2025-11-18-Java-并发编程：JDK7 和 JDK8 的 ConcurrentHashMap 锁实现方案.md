---
title: 并发编程：JDK7 和 JDK8 的 ConcurrentHashMap 锁实现方案
author: nhsoft.lsd
date: 2025-11-18
categories: [Java,并发编程]
pin: false
---


## 一、背景与目标

`ConcurrentHashMap` 是 Java 并发包中的高性能线程安全 Map，在 **JDK7 与 JDK8** 里，其底层实现机制发生了 **重大变革**。

主要目标：

* 支持高并发读写；
* 保证线程安全；
* 尽量减少锁竞争，提升吞吐性能。

---

## 二、JDK7：分段锁（Segment）机制

### 1. 结构概览

```text
ConcurrentHashMap
   └── Segment[0] ─→ HashEntry[]
   └── Segment[1] ─→ HashEntry[]
   └── ...
   └── Segment[N-1]
```

* 整体由多个 `Segment` 组成（默认 16 个），每个 `Segment` 是一个小型的 `HashMap`，继承 `ReentrantLock`；
* 每个 `Segment` 内部包含 `HashEntry[]`；
* 每次操作只锁定对应的 `Segment`，从而实现**分段加锁**，提高并发度。

### 2. 加锁粒度

* 每个 Segment 独立加锁：写操作锁定 1 个 Segment；
* 读操作使用 volatile + final 字段实现弱一致性，无需加锁。

### 3. 核心源码片段（put）

```java
final Segment<K,V> s = segmentFor(hash);
s.lock(); // 加锁 Segment
try {
    s.put(...)
} finally {
    s.unlock();
}
```

---

## 三、JDK8：取消 Segment，采用节点级别同步 + CAS

### 1. 新结构（简化）如下：

```text
ConcurrentHashMap
   └── Node[] table; // 数组 + 链表/红黑树
```

* 完全去掉了 `Segment`，直接使用 `Node[] table`；
* 每个桶内是链表或红黑树；
* 核心是对每个 **桶（bin）进行细粒度控制**。

### 2. 锁机制

* 写操作使用 `synchronized` 锁住**桶头节点**；
* 并发扩容、put 操作时大量使用 `CAS` + 自旋；
* 读操作使用 volatile 字段，无锁。

### 3. 核心源码片段（put）

```java
if (casTabAt(tab, i, null, new Node(...))) {
    // CAS 插入成功，无需加锁
} else {
    synchronized (f) {
        // 锁住桶头节点，进行链表插入
    }
}
```

---

## 四、JDK7 vs JDK8 对比总结表

| 项目    | JDK7（Segment）               | JDK8（Node + CAS）       |
| ----- | --------------------------- | ---------------------- |
| 核心结构  | Segment\[16] + HashEntry\[] | Node\[] + 链表/红黑树       |
| 加锁方式  | 锁住 Segment（ReentrantLock）   | 锁桶头（synchronized）+ CAS |
| 并发读   | 无锁，volatile + final 保证      | 无锁，volatile 保证         |
| 并发写   | 锁一个 Segment（最多16个并发）        | 精确到桶 + CAS，自旋扩容        |
| 扩容    | 全表加锁迁移                      | 多线程协作迁移（transferIndex） |
| 红黑树优化 | ❌ 无                         | ✅ 桶内链表长度超过 8 转红黑树      |
| 内存占用  | 多 Segment 和锁，略高             | 更紧凑（去 Segment）         |

---

## 五、JDK8 优势详解

* **更高并发性**：无 Segment 限制，可支持比 JDK7 更高的并发；
* **更低内存占用**：去除了大量 `Segment` 对象；
* **红黑树优化**：链表长度过长自动转红黑树，防止哈希碰撞攻击；
* **扩容过程并发化**：多个线程可一起迁移桶，提高扩容效率。

---

## 六、图示结构对比

### JDK7 分段锁结构图：

```text
┌─────────────┐
│ConcurrentMap│
└─────┬───────┘
      ↓
 ┌─────────────┐
 │Segment[16]  │   ← 分段锁数组，每个 Segment 是一个小 HashMap
 └─────────────┘
      ↓
 ┌─────────────┐
 │ HashEntry[] │
 └─────────────┘
```

### JDK8 精细锁结构图：

```text
┌─────────────┐
│ConcurrentMap│
└─────┬───────┘
      ↓
 ┌─────────────┐
 │   Node[]    │   ← 每个桶可为链表或红黑树，无 Segment
 └─────────────┘
      ↓
链表/红黑树 + synchronized + CAS 控制访问
```

---

## 七、典型使用建议

| 场景              | 推荐版本          |
| --------------- | ------------- |
| 高并发、大量写操作       | JDK8 及以上      |
| 低 JDK 环境（≤JDK7） | 可用，但有并发上限     |
| 高并发读，少量写        | 两者都可用，JDK8 更优 |

---

## 八、简单性能对比测试建议（可选）

可以构造多线程同时 `put` 的场景，对比 JDK7 与 JDK8 在写压力下的吞吐差异。JDK8 通常能支撑更大的并发线程数，且内存占用更低。

---

## 九、结语
JDK7 使用的是基于分段锁（Segment）的粗粒度并发控制方式；<br>JDK8 重构为基于数组 + CAS + synchronized 的精细化控制结构，整体性能和可扩展性更强；<br>JDK8 的设计更加符合现代硬件架构，适合大规模并发场景
