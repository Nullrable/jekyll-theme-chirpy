---
title: Java性能分析第四节 - Linux 下 JVM监控.
author: nhsoft.lsd
date: 2025-10-25
categories: [Java,Java性能分析]
pin: false
---

# 监控 与 性能分析
## 监控
监控通常是指一种在生产，质量评估或者开发环境中带有预防或主动性的活动。当应用负责人收到性能问题却没有足以定位问题的线索，首先会查看性能监控，随后性能
分析。
## 性能分析
性能分析是一种侵入方式收集运行性能数据的活动，他会影响应用的吞吐量或响应时间。性能分析是监控报警后的回应，关注并监控更加集中。

# JVM 监控
## 一、Linux 下 JVM 内存监控
Linux 下 常用 JVM 内存命令可以查看 **Java性能分析第三节 - Linux 下 内存监控**
这里主要介绍以下两种方式实现 JVM 监控

## 二、代码实现打印 JVM 监控
```java
  /**
     * 格式化字节
     */
    private static String formatBytes(long bytes) {
        if (bytes < 0) return "N/A";
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.2f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format("%.2f MB", bytes / (1024.0 * 1024));
        return String.format("%.2f GB", bytes / (1024.0 * 1024 * 1024));
    }

    /**
     * 打印内存信息
     */
    public static void printMemoryInfo() {
        MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();

        // 堆内存
        MemoryUsage heapMemory = memoryMXBean.getHeapMemoryUsage();
        System.out.println("=== 堆内存信息 ===");
        System.out.println("初始大小: " + formatBytes(heapMemory.getInit()));
        System.out.println("已使用: " + formatBytes(heapMemory.getUsed()));
        System.out.println("已提交: " + formatBytes(heapMemory.getCommitted()));
        System.out.println("最大值: " + formatBytes(heapMemory.getMax()));
        System.out.println("使用率: " + String.format("%.2f%%",
                (double) heapMemory.getUsed() / heapMemory.getMax() * 100));

        // 非堆内存
        MemoryUsage nonHeapMemory = memoryMXBean.getNonHeapMemoryUsage();
        System.out.println("\n=== 非堆内存信息 ===");
        System.out.println("已使用: " + formatBytes(nonHeapMemory.getUsed()));
        System.out.println("已提交: " + formatBytes(nonHeapMemory.getCommitted()));

        // 各个内存池详细信息
        System.out.println("\n=== 内存池详细信息 ===");
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        for (MemoryPoolMXBean pool : memoryPools) {
            MemoryUsage usage = pool.getUsage();
            System.out.println("\n内存池: " + pool.getName());
            System.out.println("  类型: " + pool.getType());
            System.out.println("  已使用: " + formatBytes(usage.getUsed()));
            System.out.println("  最大值: " + formatBytes(usage.getMax()));
            if (usage.getMax() > 0) {
                System.out.println("  使用率: " + String.format("%.2f%%",
                        (double) usage.getUsed() / usage.getMax() * 100));
            }
        }
    }

    /**
     * 打印 GC 信息
     */
    public static void printGCInfo() {
        System.out.println("\n=== GC 信息 ===");
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();

        for (GarbageCollectorMXBean gc : gcBeans) {
            System.out.println("\nGC 名称: " + gc.getName());
            System.out.println("  回收次数: " + gc.getCollectionCount());
            System.out.println("  回收耗时: " + gc.getCollectionTime() + " ms");
            System.out.println("  内存池: " + String.join(", ", gc.getMemoryPoolNames()));

            // 平均每次 GC 时间
            if (gc.getCollectionCount() > 0) {
                System.out.println("  平均每次耗时: " +
                        String.format("%.2f ms", (double) gc.getCollectionTime() / gc.getCollectionCount()));
            }
        }
    }
```

## 三、通过 JMX 远程监控
启用 JMX 远程监控，需要在 JVM 启动参数中添加以下配置：
```shell
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -jar your-app.jar
```
代码实现
```java
/**
     * 连接远程 JVM
     */
    public static MBeanServerConnection connectToRemoteJVM(String host, int port)
            throws Exception {
        String url = "service:jmx:rmi:///jndi/rmi://" + host + ":" + port + "/jmxrmi";
        JMXServiceURL serviceURL = new JMXServiceURL(url);
        JMXConnector connector = JMXConnectorFactory.connect(serviceURL);
        return connector.getMBeanServerConnection();
    }

    /**
     * 从远程获取内存信息
     */
    public static void getRemoteMemoryInfo(MBeanServerConnection mbsc) throws Exception {
        ObjectName memoryBean = ObjectName.getInstance("java.lang:type=Memory");

        // 获取堆内存
        CompositeData heapMemory = (CompositeData) mbsc.getAttribute(memoryBean, "HeapMemoryUsage");
        long used = (Long) heapMemory.get("used");
        long max = (Long) heapMemory.get("max");

        System.out.println("远程 JVM 堆内存:");
        System.out.println("  已使用: " + formatBytes(used));
        System.out.println("  最大值: " + formatBytes(max));
        System.out.println("  使用率: " + String.format("%.2f%%", (double) used / max * 100));
    }

    /**
     * 从远程获取 GC 信息
     */
    public static void getRemoteGCInfo(MBeanServerConnection mbsc) throws Exception {
        ObjectName gcQuery = ObjectName.getInstance("java.lang:type=GarbageCollector,*");
        Set<ObjectName> gcNames = mbsc.queryNames(gcQuery, null);

        System.out.println("\n远程 JVM GC 信息:");
        for (ObjectName gcName : gcNames) {
            String name = (String) mbsc.getAttribute(gcName, "Name");
            Long count = (Long) mbsc.getAttribute(gcName, "CollectionCount");
            Long time = (Long) mbsc.getAttribute(gcName, "CollectionTime");

            System.out.printf("  %s: 次数=%d, 耗时=%dms%n", name, count, time);
        }
    }
```

## 四、-XX:+PrintGCDetails

`java -XX:+PrintGCDetails -jar your-application.jar `，**这里注意正式环境慎用**

```
[44.881s][info   ][gc,start    ] GC(3) Pause Young (Prepare Mixed) (G1 Evacuation Pause)
[44.881s][info   ][gc,task     ] GC(3) Using 9 workers of 9 for evacuation
[44.886s][info   ][gc,phases   ] GC(3)   Pre Evacuate Collection Set: 0.2ms
[44.886s][info   ][gc,phases   ] GC(3)   Merge Heap Roots: 0.1ms
[44.886s][info   ][gc,phases   ] GC(3)   Evacuate Collection Set: 3.9ms
[44.886s][info   ][gc,phases   ] GC(3)   Post Evacuate Collection Set: 0.6ms
[44.886s][info   ][gc,phases   ] GC(3)   Other: 0.1ms
[44.886s][info   ][gc,heap     ] GC(3) Eden regions: 510->0(549)
[44.886s][info   ][gc,heap     ] GC(3) Survivor regions: 5->10(65)
[44.886s][info   ][gc,heap     ] GC(3) Old regions: 8->8
[44.886s][info   ][gc,heap     ] GC(3) Humongous regions: 0->0
[44.886s][info   ][gc,metaspace] GC(3) Metaspace: 33276K(33920K)->33276K(33920K) NonClass: 28938K(29312K)->28938K(29312K) Class: 4338K(4608K)->4338K(4608K)
[44.886s][info   ][gc          ] GC(3) Pause Young (Prepare Mixed) (G1 Evacuation Pause) 521M->16M(1024M) 5.087ms
[44.886s][info   ][gc,cpu      ] GC(3) User=0.02s Sys=0.01s Real=0.01s
```

**基本信息**

| 字段 | 数值 | 含义解释 | 数值解读 |
|------|------|---------|---------|
| **触发原因** | G1 Evacuation Pause | G1疏散暂停 | 正常的对象复制/疏散过程 |

---

**并行工作线程**

| 字段 | 数值 | 含义解释 | 数值解读 |
|------|------|---------|---------|
| **Using X workers of Y** | 9 / 9 | 使用了9个线程，总共9个可用 | 🟢 线程利用率100%，并行度最大化 |

---

**GC各阶段耗时详解**

| 阶段 | 耗时 | 含义解释 | 数值解读 |
|------|------|---------|---------|
| **Pre Evacuate Collection Set** | 0.2ms | 疏散前准备：选择要回收的区域 | 选择工作很快，开销小 |
| **Merge Heap Roots** | 0.1ms | 合并堆根引用：整合GC根对象 | 根对象数量少，处理快速 |
| **Evacuate Collection Set** | 3.9ms | 疏散回收集：复制存活对象到新区域 | 🔴 **主要耗时**（占76%），存活对象较多 |
| **Post Evacuate Collection Set** | 0.6ms | 疏散后处理：更新引用、清理工作 | 后续处理开销合理 |
| **Other** | 0.1ms | 其他杂项时间 | 其他开销可忽略 |

---

**堆内存各区域变化**

| 区域 | 变化 (前→后) | 容量 | 含义解释 | 数值解读 |
|------|-------------|------|---------|---------|
| **Eden regions** | 510 → 0 | (549) | 新对象分配区，GC后清空 | 🟢 回收了510个区域，回收率100% |
| **Survivor regions** | 5 → 10 | (65) | 存活对象暂存区 | 🟡 存活对象翻倍，说明有较多对象晋升 |
| **Old regions** | 8 → 8 | - | 老年代区域 | 🟢 老年代暂未增长，压力不大 |
| **Humongous regions** | 0 → 0 | - | 大对象区（> 50%区域大小的对象） | 🟢 无大对象，内存碎片风险低 |

**区域数量说明：**
- 每个region默认大小通常为 1-32MB（取决于堆大小）
- 括号内数字(549)、(65)为该区域类型的最大容量

---

 **Metaspace元空间（类元数据）**

| 区域 | 变化 | 容量 | 含义解释 | 数值解读 |
|------|------|------|---------|---------|
| **Metaspace总量** | 33276K → 33276K | (33920K) | 类元数据总空间 | 🟢 无变化，已加载类稳定（使用率98%） |
| **NonClass** | 28938K → 28938K | (29312K) | 非类元数据（常量池等） | 使用率99%，接近满 |
| **Class** | 4338K → 4338K | (4608K) | 类元数据 | 使用率94%，正常范围 |

⚠️ **注意**：Metaspace使用率很高（98%），如果持续增长可能导致Full GC

---

**总体GC效果摘要**

| 指标 | 数值 | 含义解释 | 数值解读 |
|------|------|---------|---------|
| **堆内存变化** | 521M → 16M | GC前后堆使用量 | 🟢 **回收了505M（97%）**，效果极佳 |
| **堆总容量** | (1024M) | 堆最大可用内存 | 当前使用率仅1.6%，空间充足 |
| **总暂停时长** | 5.087ms | 应用停顿的实际时间 | 🟢 < 10ms，对用户无感知 |

---

**CPU资源使用**

| CPU指标 | 数值 | 含义解释 | 数值解读 |
|---------|------|---------|---------|
| **User Time** | 0.02s | 用户态CPU时间（GC线程执行时间） | 9个线程共消耗0.02s CPU |
| **System Time** | 0.01s | 内核态CPU时间（系统调用） | 系统调用开销小 |
| **Real Time** | 0.01s | 实际挂钟时间（墙钟时间） | 实际暂停0.01s = 10ms |

**CPU利用率计算：**
```
CPU利用率 = (User + System) / Real = (0.02 + 0.01) / 0.01 = 300%
```
**解读**：多线程并行效果好，9个线程让CPU利用率达到300%（3个核心满负荷）

## 五、OutOfMemoryError: Metaspace
Metaspace 内存溢出，一般是有循环生成代理类，或者频繁加载大量第三方库导致的。

```java
public String testMetapace() {
  System.out.println("开始 Metaspace 溢出测试...");
  System.out.println("当前 MaxMetaspaceSize: " +
    Runtime.getRuntime().maxMemory() / 1024 / 1024 + "MB");

  List<Object> proxyList = new ArrayList<>();
  int count = 0;
  try {
    while (true) {
      // 创建新的 ClassLoader，每次都创建新的类
      ClassLoader classLoader = Demo1Application.class.getClassLoader();

      // 创建动态代理
      Object proxy = createDynamicProxy(classLoader);
      proxyList.add(proxy);

      count++;
      if (count % 1000 == 0) {
        System.out.println("已创建代理类数量: " + count);
        printMemoryInfo();
      }
    }
  } catch (Exception e) {
    System.err.println("\n=== 发生 OutOfMemoryError ===");
    System.err.println("错误信息: " + e.getMessage());
    System.err.println("总共创建了 " + count + " 个代理类");
    printMemoryInfo();
    e.printStackTrace();
  }
  return "Metaspace 溢出测试完成";
}

/**
 * 创建动态代理对象
 */
private static Object createDynamicProxy(ClassLoader classLoader) {
  return Proxy.newProxyInstance(
    classLoader,
    new Class<?>[]{TestInterface.class},
    new TestInvocationHandler()
  );
}
```
