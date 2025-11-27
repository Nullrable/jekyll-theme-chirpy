---
title: Java 并发编程 -ThreadPoolExecutor
author: nhsoft.lsd
date: 2025-11-14
categories: [Java, 并发编程]
pin: false
---

ThreadPoolExecutor 线程池中的核心线程**不会自动销毁**，而是会根据线程池的配置（核心参数）和状态，在特定条件下被**回收（销毁）** 或**复用**，核心原则是：**尽量复用线程以减少创建/销毁开销，仅在空闲时间过长或线程池收缩时销毁线程**。


### 关键结论
1. 核心线程（corePoolSize）：默认情况下（`allowCoreThreadTimeOut=false`），核心线程会**一直存活**，即使空闲也不会销毁；
2. 非核心线程（超过 corePoolSize 的线程）：当空闲时间超过 `keepAliveTime` 时，会被自动销毁；
3. 特殊配置：若设置 `allowCoreThreadTimeOut=true`，核心线程也会在空闲超过 `keepAliveTime` 后被销毁（最终线程池可能变为 0 个线程）。


### 详细拆解：线程的“生命周期”与销毁条件
ThreadPoolExecutor 的线程创建、复用、销毁逻辑，完全由以下核心参数控制：
| 参数                | 作用                                                                 |
|---------------------|----------------------------------------------------------------------|
| corePoolSize        | 核心线程数（线程池长期维持的最小线程数）                             |
| maximumPoolSize     | 最大线程数（线程池允许创建的最多线程数）                             |
| keepAliveTime       | 非核心线程的空闲存活时间（空闲超过此时间则销毁）                     |
| allowCoreThreadTimeOut | 是否允许核心线程超时销毁（默认 false，核心线程永不销毁）             |
| workQueue           | 任务队列（核心线程满后，任务先入队，队列满后才创建非核心线程）       |


#### 1. 线程的创建时机
线程池不会一开始就创建所有核心线程，而是**懒加载**（按需创建）：
- 提交第一个任务时，创建第一个核心线程；
- 持续提交任务，直到线程数达到 `corePoolSize`（核心线程满）；
- 核心线程满 + 任务队列满，再提交任务时，创建非核心线程（直到达到 `maximumPoolSize`）。


#### 2. 线程的复用逻辑
线程池的核心价值是“线程复用”：线程执行完一个任务后，不会销毁，而是会循环从任务队列（workQueue）中获取下一个任务执行，直到队列中再无任务可执行（进入空闲状态）。


#### 3. 线程的销毁条件（重点）
只有当线程进入“空闲状态”，且满足以下条件时，才会被销毁：
##### （1）非核心线程的销毁（默认必触发）
当线程数 > `corePoolSize`（即该线程是非核心线程），且空闲时间超过 `keepAliveTime` 时，线程会被自动销毁，直到线程数降至 `corePoolSize`。

示例：
corePoolSize=2，maximumPoolSize=5，keepAliveTime=60s
- 提交 5 个任务：线程池创建 5 个线程（2 核心 + 3 非核心）；
- 所有任务执行完后，3 个非核心线程进入空闲状态；
- 空闲 60s 后，3 个非核心线程被销毁，仅保留 2 个核心线程待命。

##### （2）核心线程的销毁（需手动开启）
默认情况下（`allowCoreThreadTimeOut=false`），核心线程即使长期空闲也不会销毁（一直存活待命）。
若需要核心线程也能超时销毁，可通过 `threadPool.setAllowCoreThreadTimeOut(true)` 开启，此时：
所有线程（核心 + 非核心）的空闲时间超过 `keepAliveTime` 后，都会被销毁，最终线程池的线程数可能变为 0。

**注意**：开启后 `keepAliveTime` 对核心线程生效，需确保 `keepAliveTime` 配置合理（避免核心线程频繁创建/销毁）。

##### （3）线程池关闭时的销毁（强制销毁）
当调用线程池的 `shutdown()` 或 `shutdownNow()` 方法时，线程池会进入“关闭状态”，此时会：
- `shutdown()`：等待所有已提交的任务执行完毕后，销毁所有线程；
- `shutdownNow()`：立即中断正在执行的任务（需任务支持中断），并销毁所有线程。


### 代码示例：验证线程销毁逻辑
```java
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolDestoryDemo {
    public static void main(String[] args) throws InterruptedException {
        // 配置：核心2，最大5，空闲超时10秒，队列容量10
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
                2, 5, 10, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(10)
        );

        // 提交6个任务（核心2满 + 队列10未满，实际仅创建2个核心线程）
        for (int i = 0; i < 6; i++) {
            threadPool.submit(() -> {
                System.out.println("线程 " + Thread.currentThread().getName() + " 执行任务");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            });
        }

        // 任务执行完毕后，查看线程数（此时核心线程空闲，不会销毁）
        Thread.sleep(1000);
        System.out.println("任务执行完后，线程池活跃线程数：" + threadPool.getActiveCount()); // 输出 0（无活跃任务），但线程数仍为2

        // 等待15秒（超过keepAliveTime=10秒，核心线程默认不销毁）
        Thread.sleep(15000);
        System.out.println("空闲15秒后，线程池活跃线程数：" + threadPool.getActiveCount()); // 仍为0，线程数保持2

        // 开启核心线程超时销毁
        threadPool.allowCoreThreadTimeOut(true);
        // 再等待15秒（核心线程空闲超时，会被销毁）
        Thread.sleep(15000);
        System.out.println("开启核心线程超时后，活跃线程数：" + threadPool.getActiveCount()); // 输出 0，线程数可能变为0

        // 关闭线程池（强制销毁所有剩余线程）
        threadPool.shutdown();
    }
}
```


### 总结
ThreadPoolExecutor 的线程**不会自动销毁**，而是：
- 核心线程：默认“常驻”，空闲不销毁；开启 `allowCoreThreadTimeOut=true` 后，空闲超时销毁；
- 非核心线程：空闲超时（`keepAliveTime`）后自动销毁；
- 线程池关闭时，所有线程会被强制销毁。

这种设计的目的是**平衡性能和资源消耗**：复用线程减少创建/销毁开销，通过超时销毁避免空闲线程占用过多内存。

![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
