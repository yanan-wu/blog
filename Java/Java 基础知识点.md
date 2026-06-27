# 高并发

## AQS
AQS 提供了一个基于 FIFO 等待队列的框架，用于实现阻塞锁和相关的同步器。它使用一个 volatile int 类型的 state 表示同步状态，通过内置的 CLH 队列管理等待线程。


AQS 的三大核心组件：

**同步状态（state）**，用 volatile修饰，保证可见性（不同同步器对 state 含义不同：
ReentrantLock：0 表示未锁定，>0 表示锁定次数
Semaphore：剩余许可证数量
CountDownLatch：剩余需要等待的线程数）；

**CLH 等待队列**，是一种公平、FIFO（先入先出）的自旋锁队列算法，基于链表的自旋锁/等待队列抽象模型，用来管理多线程争抢锁时的排队、阻塞与唤醒；

**条件队列**，每个 Condition 对象维护一个独立的等待队列，await()将线程移入条件队列，signal()将线程从条件队列移到同步队列

AQS 通过 volatile int state 表示同步状态，使用 CLH 双向队列管理等待线程，采用模板方法模式让子类实现 tryAcquire/tryRelease 等钩子方法，统一了入队、阻塞、唤醒等并发控制逻辑；ReentrantLock、Semaphore、CountDownLatch 等同步器都是基于 AQS 实现的。

## 容器
### HashMap
JDK 1.8 的 HashMap 采用 数组 + 链表 + 红黑树​ 的结构。

 hash 值计算，(h = key.hashCode()) ^ (h >>> 16) 将高位信息混合到低位，减少哈希碰撞概率。

### ConcurrentHashMap
ConcurrentHashMap 是 Java 并发包中的线程安全哈希表，其数据结构在不同 JDK 版本中有显著差异。目前最常用的是 JDK 8 版本。
整体结构：

```
┌─────────────────────────────────────────────────────────────┐
│                    ConcurrentHashMap                        │
├─────────────────────────────────────────────────────────────┤
│  Node<K,V>[] table    ← 哈希桶数组（核心）                   │
│  Node<K,V>[] nextTable ← 扩容时用的新数组                    │
│  volatile int sizeCtl  ← 控制标志（容量/扩容/初始化）         │
│  volatile long baseCount ← 元素总数（近似）                   │
│  CounterCell[] counterCells ← 辅助计数（避免竞争）            │
└─────────────────────────────────────────────────────────────┘
```

**核心数据结构 = 数组 + 链表 + 红黑树 + CAS + synchronized**

ConcurrentHashMap JDK 8 采用 Node 数组 + 链表/红黑树的数据结构，通过 CAS 初始化/插入空桶 + synchronized 锁桶头节点实现并发安全，扩容时多线程协助迁移；相比 JDK 7 的 Segment 分段锁，锁粒度更细、内存更小、并发度更高。

---


#### JDK 7 vs JDK 8 数据结构对比

| 维度 | JDK 7 | JDK 8 |
|---|---|---|
| **数据结构** | Segment 数组 + HashEntry 数组 | Node 数组 + 链表/红黑树 |
| **锁粒度** | Segment 级别（分段锁） | 桶级别（单个桶头节点） |
| **锁机制** | ReentrantLock | CAS + synchronized |
| **查询优化** | 无 | 链表 → 红黑树（≥8） |
| **扩容** | 单线程扩容 | 多线程协助扩容 |
| **内存占用** | 较大（Segment 额外对象） | 较小 |

---



#### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_CAPACITY` | 16 | 默认初始容量 |
| `LOAD_FACTOR` | 0.75 | 扩容因子（固定，不可调） |
| `TREEIFY_THRESHOLD` | 8 | 链表转红黑树阈值 |
| `UNTREEIFY_THRESHOLD` | 6 | 红黑树转链表阈值 |
| `MIN_TREEIFY_CAPACITY` | 64 | 链表转红黑树的最小数组长度 |
| `MAXIMUM_CAPACITY` | 1 << 30 | 最大容量 |
| `NCPU` | Runtime.availableProcessors() | CPU 核数，影响扩容步长 |

---

#### 面试高频问题

**1. ConcurrentHashMap JDK 8 为什么用 synchronized 代替 ReentrantLock？**

- synchronized 在 JDK 6 后做了大量优化（偏向锁、轻量级锁、锁粗化等）
- synchronized 不需要手动释放锁，代码更简洁
- 锁竞争激烈时，synchronized 会升级为重量级锁，性能与 ReentrantLock 相当

**2. 扩容时其他线程能 put 吗？**

可以。遇到 ForwardingNode 的线程会协助扩容（helpTransfer），扩容完成后继续 put。

**3. 为什么链表转红黑树的阈值是 8？**

基于泊松分布计算：在理想随机哈希下，链表长度达到 8 的概率极低（约 0.00000006,1 亿分之 6），说明发生了严重的哈希冲突，此时转为红黑树提升性能。

**4. size() 是怎么实现的？**

通过 `baseCount` + `CounterCell[]` 累加计数，避免每次修改都竞争一个变量。sumCount() 不加锁遍历所有 CounterCell 求和。size是近似值。

**5.ConcurrentHashMap computeIfAbsent()死锁的原因及解决方案**

ConcurrentHashMap.computeIfAbsent()在 JDK 8 中因桶锁内执行回调且可能递归调用同一 CHM 导致死锁；JDK 8 中 ConcurrentHashMap.computeIfAbsent()对目标桶加 synchronized且在锁内执行 mappingFunction。若回调中再次操作同一个 CHM（尤其涉及不同桶的交叉调用），可能造成死锁——线程 A 持有桶 1 锁等待桶 2 锁，线程 B 持有桶 2 锁等待桶 1 锁。JDK 9 通过缩小锁范围和检测嵌套调用修复了此问题。最佳实践是：不在 mappingFunction 中操作同一个 CHM，改用 get+ putIfAbsent两步操作，或升级 JDK 9+，或使用 Caffeine 等专业缓存库。


# 高可用
## RocketMQ 

#### 面试高频问题
**1.RocketMQ 怎么保证一致性?**

RocketMQ 的一致性保障分三层：生产端不丢消息、Broker 端持久化可靠、消费端不丢不重复，最终达到端到端的最终一致性（At-Least-Once + 幂等 = 可达成 Exactly-Once）
- 生产端 → Broker（消息不丢），同步发送（同步判断Broker ACK） + 重试
- Broker 端刷盘 + 主从复制， 同步刷盘 + 同步复制（出从复制）
- Broker → 消费端（不丢 + 不重复消费）。消费成功后，再提交offset；消费失败，重试消费；消费幂等保证。

# JVM 

#### 面试高频问题
**1.JVM 年轻代晋升老年代的几种情况？**

1、正常晋升（年龄阈值）。对象每经过一次Minor GC → 年龄 +1，年龄 > MaxTenuringThreshold（默认 15）→ 晋升老年代。

2、动态年龄判断。JVM 并不一定等到 MaxTenuringThreshold才晋升，而是根据 Survivor 区空间动态调整：Minor GC 后，计算 Survivor 中所有对象年龄从小到大累加，当累加值 > Survivor 区 50% 时，将 >= 该年龄的对象全部晋升老年代。

3、Survivor 区放不下。Minor GC 后，存活对象大小 > Survivor 区大小，放不下的对象直接晋升老年代。

4、大对象直接分配到老年代。大对象阈值可设置（默认 0，即不限制）
-XX:PretenureSizeThreshold=1m   # 超过 1MB 的对象直接进入老年代





