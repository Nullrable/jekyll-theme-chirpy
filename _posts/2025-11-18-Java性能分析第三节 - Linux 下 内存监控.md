---
title: Java性能分析第二节 - Linux 下 内存监控.
author: nhsoft.lsd
date: 2025-10-25
categories: [Java性能分析]
pin: false
---

# 一、基本信息和工具
- 服务器配置 阿里云上ECS服务器，配置为*2核4G*为例
- 压测工具 `wrk`, 下载见 [wrk 下载和安装.md](2025-11-18-wrk 下载和安装.md)
- 压测脚本 `wrk -t 2 -c 500 -d 60s --latency http://localhost:8080/mem?count=1000`
- Java版本：JDK21，启动命令：`java -jar -Xms1024m -Xmx1024m -XX:NativeMemoryTracking=summary demo1.jar`
- 接口实现
```java
    @GetMapping("/mem")
public String mem(@RequestParam(value = "count", defaultValue = "1") int count) {

  for (int i = 0; i < count; i++) {
    byte[] b = new byte[1024 * 1024];
    if (i == 1000) {
      try {
        // 模拟响应慢
        Thread.sleep(1000 * 20);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    }
  }
  return "ok";
}
```

# 二、Linux 内存监控工具

`free -h`
```shell
              total        used        free      shared  buff/cache   available
Mem:          3.5Gi       1.6Gi       1.7Gi       2.0Mi       425Mi       1.9Gi
Swap:            0B          0B          0B
```

| 列 | 值 | 含义 |
|------|------|------|
| **total** | 3.5 GiB | 物理内存总量 |
| **used** | 1.6 GiB | 已使用内存（包括应用+buff/cache） |
| **free** | 1.7 GiB | 完全空闲的内存 |
| **shared** | 1.0 MiB | 共享内存（tmpfs等） |
| **buff/cache** | 407 MiB | 缓冲区和页面缓存 |
| **available** | 1.9 GiB | **可用内存**（最重要！）available ≈ free + buff/cache(可回收部分) |

其他实用命令
```shell
# 人性化显示
free -h

# 持续监控（每 2 秒刷新）
free -h -s 2

# 显示总计（包括 swap）
free -h -t

# 以 MB 为单位
free -m
```

vmstat (虚拟内存统计)
```
# 每 2 秒输出一次，共 10 次
vmstat 2 10

# 显示活跃/非活跃内存
vmstat -a

# 显示内存详细统计
vmstat -s

#### **输出示例**

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
1  0      0 1740000 50000 357000    0    0    10    20  100  200  5  2 93  0  0
```

**字段解读：**

| 分类 | 字段 | 含义 |
|------|------|------|
| **memory** | free | 空闲内存 (KB) |
| | buff | 缓冲区内存 (KB) |
| | cache | 页面缓存 (KB) |
| **swap** | si | 从 swap 换入内存 (KB/s) |
| | so | 换出到 swap (KB/s) |
| **io** | bi | 块读入 (blocks/s) |
| | bo | 块写出 (blocks/s) |

**关键指标：**
```bash
# 如果 si、so 持续不为 0，说明发生了内存交换（性能杀手）
si > 0 或 so > 0  →  内存不足，正在使用 swap

# 理想状态
si = 0, so = 0  →  内存充足
```

# 三、Linux JVM监控工具
`jps -mlv` 查看JVM进程ID

`jhsdb jmap --pid 45458 --heap`， 这个注意在不同JDK版本下，命令有所差异，注意识别

```shell
[root@iZm5eir5ome0diow93n29fZ ~]# jhsdb jmap --pid 45458 --heap
Attaching to process ID 45458, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 21.0.7+6-LTS

using thread-local object allocation.
Garbage-First (G1) GC with 2 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 1073741824 (1024.0MB) # -Xmx指定，启动是 -Xmx1024m
   NewSize                  = 1363144 (1.2999954223632812MB)
   MaxNewSize               = 643825664 (614.0MB) # NewRatio 参数在 G1 垃圾回收器下不再严格生效。这个值 MaxNewSize = MaxHeapSize × G1MaxNewSizePercent= 1024 MB × 60%  = 614.4 MB ≈ 614 MB
   OldSize                  = 5452592 (5.1999969482421875MB)
   NewRatio                 = 2 # 新生代与老年代的比例为 1:2，
   SurvivorRatio            = 8
   MetaspaceSize            = 22020096 (21.0MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB # 元空间最大大小:无限制
   G1HeapRegionSize         = 1048576 (1.0MB) # 可以看到下方 regions  = 1024。 MaxHeapSize = 1024.0MB。所以每个 region = 1MB

Heap Usage:
G1 Heap:
   regions  = 1024
   capacity = 1073741824 (1024.0MB)
   used     = 653005008 (622.7541046142578MB)
   free     = 420736816 (401.2458953857422MB)
   60.815830528736115% used
G1 Young Generation:
Eden Space:
   regions  = 590
   capacity = 675282944 (644.0MB)
   used     = 618659840 (590.0MB)
   free     = 56623104 (54.0MB)
   91.61490683229813% used
Survivor Space:
   regions  = 0
   capacity = 1048576 (1.0MB)
   used     = 35352 (0.03371429443359375MB)
   free     = 1013224 (0.9662857055664062MB)
   3.371429443359375% used
G1 Old Generation:
   regions  = 35
   capacity = 397410304 (379.0MB)
   used     = 34309816 (32.72039031982422MB)
   free     = 363100488 (346.2796096801758MB)
   8.63334836934676% used
```

## 方法一：jmap -histo

只看前 20 个占用最大的类
`jmap -histo:live 45458 | head -n 25`

```
num     #instances         #bytes  class name (module)
-------------------------------------------------------
1:         52133        5007752  [B (java.base@21.0.7)
2:           457        1706872  [C (java.base@21.0.7)
3:         49399        1185576  java.lang.String (java.base@21.0.7)
4:          7644         908800  java.lang.Class (java.base@21.0.7)
5:         26392         844544  java.util.concurrent.ConcurrentHashMap$Node (java.base@21.0.7)
6:          6646         584848  java.lang.reflect.Method (java.base@21.0.7)
7:          8112         528600  [Ljava.lang.Object; (java.base@21.0.7)
8:          4499         349712  [I (java.base@21.0.7)
9:          7693         307720  java.util.LinkedHashMap$Entry (java.base@21.0.7)
10:          3586         307048  [Ljava.util.HashMap$Node; (java.base@21.0.7)
11:           351         278320  [Ljava.util.concurrent.ConcurrentHashMap$Node; (java.base@21.0.7)
12:          8033         257056  java.util.HashMap$Node (java.base@21.0.7)
13:          3660         234240  java.util.LinkedHashMap (java.base@21.0.7)
14:         13789         220624  java.lang.Object (java.base@21.0.7)
15:          7003         178832  [Ljava.lang.Class; (java.base@21.0.7)
16:          3625         174000  java.lang.invoke.MemberName (java.base@21.0.7)
17:          1568         123496  [Ljdk.internal.vm.FillerElement; (java.base@21.0.7)
18:          2597         103880  java.lang.invoke.MethodType (java.base@21.0.7)
19:          2153          86120  java.lang.ref.SoftReference (java.base@21.0.7)
20:          2090          83600  sun.util.locale.LocaleObjectCache$CacheEntry (java.base@21.0.7)
21:          2598          83136  jdk.internal.util.WeakReferenceKey (java.base@21.0.7)
22:          1651          79248  java.util.HashMap (java.base@21.0.7)
23:          1576          75648  org.apache.tomcat.util.buf.ByteChunk
```

其他命令
```
# 只看前 20 个占用最大的类
jmap -histo:live 45458 | head -n 25

# 过滤特定包的类
jmap -histo:live 45458 | grep "com.example"

# 按实例数排序（需要处理）
jmap -histo:live 45458 | sort -k2 -nr | head -n 20

# 按内存大小排序
jmap -histo:live 45458 | sort -k3 -nr | head -n 20
```

## 方法二：jcmd (推荐，JDK 8+)

```
# 查看堆直方图
jcmd 45458 GC.class_histogram

# 只看活跃对象
jcmd 45458 GC.class_histogram | head -n 30

# 查看更详细的堆信息
jcmd 45458 VM.native_memory summary
```


## 方法三：Eclipse MAT (内存泄漏分析神器)
```
# 1. 生成 heap dump
jmap -dump:live,format=b,file=heap.hprof 45458

# 2. 下载 Eclipse MAT
# https://www.eclipse.org/mat/

# 3. 打开 heap.hprof 文件

# 4. 查看关键报告
- Leak Suspects (泄漏嫌疑)
- Dominator Tree (支配树)
- Histogram (直方图)


### **查看对象占用**

Histogram 视图：
- 右键类名 -> List objects -> with incoming references
- 可以看到该类所有实例及引用链
```

![img.png](../assets/img/nhsoft_lsd/img_12321321.png)

## 方法四：jstat 监控
实时监控 GC 和内存（每秒刷新） jstat -gcutil 45458 1000
``` S0     S1     E      O      M     CCS    YGC     YGCT     FGC    FGCT     CGC    CGCT       GCT
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802
     -      -   0.47   3.70  97.69  92.67    712     2.676     3     0.122     2     0.004     2.802

堆内存使用率（百分比）
S0: Survivor 0 区使用率 = - (空，G1中可能不显示)
S1: Survivor 1 区使用率 = - (空，G1中可能不显示)
E: Eden 区使用率 = 0.47% (非常低)
O: Old Gen (老年代) 使用率 = 3.70% (很健康)
M: Metaspace 使用率 = 97.69% (⚠️ 接近满了，需要关注)
CCS: Compressed Class Space 使用率 = 92.67% (较高)

GC 统计信息
YGC: Young GC 次数 = 712 次
YGCT: Young GC 总耗时 = 2.676 秒
FGC: Full GC 次数 = 3 次
FGCT: Full GC 总耗时 = 0.122 秒
CGC: Concurrent GC 次数 = 2 次 (G1 特有)
CGCT: Concurrent GC 总耗时 = 0.004 秒
GCT: 总 GC 耗时 = 2.802 秒

# 查看堆内存统计
jstat -gccapacity 45458

# 查看新生代对象晋升情况
jstat -gcnew 45458 1000
```

## 方法五：发生 OutOfMemoryError 时自动生成 Dump 文件
在 JVM 启动参数中添加： -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumpfile.hprof
