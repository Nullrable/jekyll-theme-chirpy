---
title: Thread-Per-Message 模式、Worker Thread 模式、生产者-消费者模式
author: nhsoft.lsd
date: 2025-11-18
categories: [Java]
pin: false
---


**Thread-Per-Message 模式**、**Worker Thread 模式**、**生产者-消费者模式** 是多线程中最基础且实用的分工设计模式，它们在任务分离、资源复用、吞吐优化方面各有优势。

---

## 一、Thread-Per-Message 模式（每个请求一个线程）

### 解释：

每个外部请求（如一个 socket 连接）由一个独立线程来处理。常见于早期 HTTP、TCP 服务实现中。

### 特点：

* 简单易理解，线程间无共享状态。
* 缺点：线程开销大，不能承载高并发请求。

---

### Java 示例：Socket 版 Echo Server（Thread-Per-Message）

```java
import java.io.*;
import java.net.*;

public class ThreadPerMessageSocketServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8888);
        System.out.println("Server started on port 8888...");

        while (true) {
            Socket clientSocket = serverSocket.accept(); // 每来一个请求
            new Thread(() -> handleClient(clientSocket)).start(); // 启动新线程处理
        }
    }

    private static void handleClient(Socket socket) {
        try (
            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true)
        ) {
            String line;
            while ((line = in.readLine()) != null) {
                System.out.println("Received from client: " + line);
                out.println("Echo: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try { socket.close(); } catch (IOException ignored) {}
        }
    }
}
```

---

## 二、Worker Thread 模式（线程池）

### 解释：

将多个任务分配给**固定数量的线程**去处理。每个线程会**重复执行不同的任务**，适用于高并发。

---

### Java 示例：

```java
import java.util.concurrent.*;

public class WorkerThreadPoolExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(3); // 固定3个工作线程

        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println(Thread.currentThread().getName() + " is processing task " + taskId);
                try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
            });
        }

        executor.shutdown();
    }
}
```

---

## 三、Producer-Consumer 模式（阻塞队列）

### 解释：

生产者线程生产任务放入队列，消费者线程从队列中取出任务执行，实现解耦、限流、异步。

---

### Java 示例（BlockingQueue）：

```java
import java.util.concurrent.*;

public class ProducerConsumerPattern {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);

        // 生产者
        new Thread(() -> {
            int i = 1;
            while (true) {
                try {
                    String task = "Task-" + i++;
                    queue.put(task);
                    System.out.println("Produced: " + task);
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();

        // 消费者
        new Thread(() -> {
            while (true) {
                try {
                    String task = queue.take();
                    System.out.println("Consumed: " + task);
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();
    }
}
```

---

## 三者对比总结：

| 模式                     | 分工机制        | 优点      | 缺点       | 适用场景         |
| ---------------------- | ----------- | ------- | -------- | ------------ |
| **Thread-Per-Message** | 每请求一个线程     | 简洁、无共享  | 高并发下资源耗尽 | 简单 socket 服务 |
| **Worker Thread**      | 固定线程 + 任务分配 | 线程重用、高效 | 调度逻辑稍复杂  | Web 服务/任务队列  |
| **Producer-Consumer**  | 队列缓冲 + 解耦   | 异步解耦、限流 | 控制复杂度高   | 日志处理、流水线任务   |



![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
