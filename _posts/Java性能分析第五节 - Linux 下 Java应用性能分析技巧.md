---
title: Java性能分析第五节 - Linux 下 Java应用性能分析技巧.
author: nhsoft.lsd
date: 2025-10-30
categories: [Java性能分析]
pin: false
---

# 性能优化机会
## 使用更加高效的算法
对应用程序进行性能优化，往往采用更加高效算法或数据结构，以减少 CPU调用，内存使用，更短的执行路径实现业务功能
## 减少锁竞争
对共享资源的竞争会导致程序性能无法随着线程数和CPU增加，提升程序性能。我们代码中应该减少锁竞争频率，缩短是有锁的时间。该类典型的现象是CPU利用率很低，但是接口响应很慢
## 为算法生成更有效的代码
这个主要是JIT 生成更加有效的机器代码。开发者层面：程序员写出更“适合优化”的算法结构

# 一、减少锁竞争
## Collections.synchronizedMap(hashMap) VS ConcurrentHashMap

### 性能对比表格（1线程+10000容量场景）

| 操作类型               | ConcurrentHashMap       | Hashtable               | synchronizedMap         | ConcurrentHashMap优势倍数 |
|--------------------|-------------------------|-------------------------|-------------------------|--------------------------|
| ComputeIfAbsent    | 215,630,824.98          | -（无数据）             | 11,079,032.86           | 约20倍                   |
| Read               | 525,187,332.14          | 11,027,254.78           | 13,240,371.76           | 约40倍                   |
| Write     | 56,801,031.95           | 8,239,291.86            | 6,186,892.10            | 约9倍                    |
| ReadWriteMix | 450,591,900.49      | 10,316,611.54           | 8,956,227.98            | 约44倍                   |


### 性能对比表格（8线程+10000容量场景）
| 操作类型          | ConcurrentHashMap       | Hashtable               | synchronizedMap         | 性能倍数（CHM/其他） |
|-------------------|-------------------------|-------------------------|-------------------------|---------------------|
| ComputeIfAbsent   | 212,978,748.52          | -（无数据）             | 7,876,830.53            | 约27倍              |
| Read              | 500,711,085.59          | 15,940,258.85           | 11,815,974.16           | 约31-42倍           |
| Write             | 58,165,131.07           | 7,874,771.86            | 7,802,218.14            | 约7.4倍             |
| ReadWriteMix      | 442,338,345.34          | 10,031,357.21           | 9,719,126.53            | 约44-46倍           |

这里有synchronizedMap 两个 Write，ReadWriteMix，这个有上升，这个应该是测试误差导致的偶然结果（基准测试的 Error 值较大）。实际中，多线程写操作的锁竞争更激烈，性能必然低于单线程。

## 全局 Random VS ThreadLocalRandom

Random.next() 代码实现

```java
    private final AtomicLong seed;
    protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));
        return (int)(nextseed >>> (48 - bits));
    }
```
可以看到，如果所有线程共享一个Rando实例，会通过CAS保证线程安全，高并发下线程竞争激烈，CAS 重试频繁，导致性能损耗

而 `ThreadLocalRandom` 每个线程持有独立的种子变量（存储在 Thread 对象中），无竞争，且实例通过 current() 方法获取，无需频繁创建。
优势：兼顾线程安全和低开销，避免了 CAS 竞争和实例重复创建的问题。

那大家会问，如果使用局部 Random 实例呢？可以看下 Random 初始化代码
```java
    private static long seedUniquifier() {
        // L'Ecuyer, "Tables of Linear Congruential Generators of
        // Different Sizes and Good Lattice Structure", 1999
        for (;;) {
            long current = seedUniquifier.get();
            long next = current * 1181783497276652981L;
            if (seedUniquifier.compareAndSet(current, next))
                return next;
        }
    }
```
可以看到初始化一个Random 实例成本比较高（需初始化种子，可能涉及系统时间或随机源调用），如果一个线程里频繁创建会导致额外的性能开销（尤其是短生命周期任务）。当然，如果一个线程生命周期里只有初始化一次，那跟
`ThreadLocalRandom` 效果相同

# 二、适当的数据结构大小

## StringBuilder 或 StringBuffer

可以看下 append 方法实现
```java
    public AbstractStringBuilder append(String str) {
        if (str == null) {
            return appendNull();
        }
        int len = str.length();
        ensureCapacityInternal(count + len);
        putStringAt(count, str);
        count += len;
        return this;
    }


    private void ensureCapacityInternal(int minimumCapacity) {
      // overflow-conscious code
      int oldCapacity = value.length >> coder;
      //这里会判断是否需要扩容
      if (minimumCapacity - oldCapacity > 0) {
        //扩容的代价：数组复制（Arrays.copyOf()）是耗时操作，尤其是数据量大时，多次扩容会显著降低性能。
        value = Arrays.copyOf(value,
          newCapacity(minimumCapacity) << coder);
      }
    }
```
可以看到如果在没有指定大小的情况下，频繁的 append 会导致多次扩容，导致显著降低性能。 类似的还有 Java Collections

## Java Collections

### ArrayList.add
```java
    private void add(E e, Object[] elementData, int s) {
      if (s == elementData.length)
        elementData = grow();
      elementData[s] = e;
      size = s + 1;
    }
    private Object[] grow(int minCapacity) {
        int oldCapacity = elementData.length;
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            int newCapacity = ArraysSupport.newLength(oldCapacity,
                    minCapacity - oldCapacity, /* minimum growth */
                    oldCapacity >> 1           /* preferred growth */);
            return elementData = Arrays.copyOf(elementData, newCapacity);
        } else {
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }
```
ArrayList 底层是动态数组，默认初始容量为 10，当元素数量超过当前容量时，会触发扩容机制（新容量 = 旧容量 ×1.5），并伴随数组拷贝（Arrays.copyOf）。这一过程会消耗额外时间和内存。
**容量设置的核心逻辑**
- 预估元素数量 N：若能明确知道会存入 N 个元素，直接将初始容量设为 N。
- 无法精确预估：若元素数量浮动，可设为预估最大值的 1.1~1.2 倍，避免因少量超出再次扩容。
- 禁止盲目设大：若仅存 5 个元素却设初始容量为 1000，会造成 99.5% 的内存浪费，且无性能收益。

### Map（以 HashMap 为例）

HashMap 底层是 “数组 + 链表 / 红黑树”，其扩容机制与负载因子（默认 0.75）强相关。当元素数量（size）> 初始容量 × 负载因子时，会触发扩容（新容量 = 旧容量 ×2），同样伴随数组拷贝和哈希重计算。

# 三、增加并行性
现代的CPU架构是多核，多硬件执行线程技术摆到程序员面前，这意味着我们可以利用更多的CPU资源做更多的工作。举例来说，parallelStream 进行并行处理服务健康检查
```java
    public void heartbeat() {
        //这个是个provider用的，consumer 不需要的
        heartBeatExecutor.scheduleWithFixedDelay(() -> {
            String leaderUrl = leader();

            serviceMetaMap.keySet().parallelStream().forEach(instance -> {


                List<ServiceMeta> serviceMetas = serviceMetaMap.get(instance);
                String services = serviceMetas.stream().map(ServiceMeta::toPath).collect(Collectors.joining(","));

                RequestBody body = RequestBody.create(JSON_MEDIA, JSON.toJSONString(instance));
                Request req = new Request.Builder().url(leaderUrl + "/heartbeat?services=" + services).post(body).build();
                try (Response response = okHttpClient.newCall(req).execute()) {
                    log.info(" ====>>>> heartbeat success service = {}, response: {}", services, response.body().string());
                } catch (IOException e) {
                    log.error(" ====>>>> heartbeat failed service = {}", services);
                }
            });

        }, 5, 5, TimeUnit.SECONDS);
    }
```

