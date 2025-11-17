# Sentinel 滑动时间窗口算法深度解析

> 深入剖析 Sentinel 的核心限流算法及其相比传统算法的优势

---

## 📋 目录

1. [四种限流算法对比](#1-四种限流算法对比)
2. [滑动时间窗口算法原理](#2-滑动时间窗口算法原理)
3. [Sentinel LeapArray 核心实现](#3-sentinel-leaparray-核心实现)
4. [源码级实现剖析](#4-源码级实现剖析)
5. [性能优化与最佳实践](#5-性能优化与最佳实践)
6. [实战案例与代码示例](#6-实战案例与代码示例)

---

## 1. 四种限流算法对比

### 1.1 固定窗口（Fixed Window）算法

**原理：**

```
时间窗口固定，统计窗口内的请求数

窗口 1 [00:00 - 00:01)  计数器 = 0
窗口 2 [00:01 - 00:02)  计数器 = 0
窗口 3 [00:02 - 00:03)  计数器 = 0
...

每个请求到来：
1. 判断当前时间属于哪个窗口
2. 窗口计数器 +1
3. 如果计数器 > 阈值，拒绝请求
4. 窗口结束后，计数器重置为 0
```

**实现代码：**

```java
public class FixedWindowLimiter {
    private final long windowSizeMs;  // 窗口大小（毫秒）
    private final int maxRequests;    // 最大请求数

    private long windowStart;         // 当前窗口开始时间
    private int counter;              // 当前窗口计数器

    public boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 计算当前窗口的开始时间
        long currentWindowStart = (now / windowSizeMs) * windowSizeMs;

        // 如果进入新窗口，重置计数器
        if (currentWindowStart != windowStart) {
            windowStart = currentWindowStart;
            counter = 0;
        }

        // 判断是否超过阈值
        if (counter < maxRequests) {
            counter++;
            return true;
        }

        return false;
    }
}
```

**可视化示例：**

```
限流规则：每分钟最多 100 个请求

场景：临界问题
┌────────────────┬────────────────┐
│  窗口 1 (0-60s)│  窗口 2 (60-120s)│
├────────────────┼────────────────┤
│                │                │
│        55s → 100 req ✅         │
│        59s → (已达上限)         │
│                │ 61s → 100 req ✅│
│                │ 65s → (已达上限)│
└────────────────┴────────────────┘

问题：在 55s-65s 这 10 秒内，实际通过了 200 个请求！
（是限流阈值的 2 倍）

原因：两个窗口的边界处，允许短时间内突发大量流量
```

**优缺点：**

| 优点 | 缺点 |
|------|------|
| ✅ 实现简单 | ❌ **临界问题**：窗口边界流量突刺 |
| ✅ 内存占用小 | ❌ 无法精确控制流量 |
| ✅ 性能高 | ❌ 窗口切换时瞬时流量可能超过限制的 2 倍 |

---

### 1.2 滑动窗口（Sliding Window）算法

**原理：**

```
将时间窗口划分为多个小格子（Bucket），窗口随时间滑动

整体窗口：1 分钟（60 秒）
划分为：60 个小格子，每个 1 秒

┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│10│20│15│18│22│19│25│30│28│26│  ← 每个格子统计 1 秒内的请求数
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
 ↑                              ↑
 窗口开始                       窗口结束

当前时间：T
统计范围：[T-60s, T) 的所有格子的请求总数

时间推移 1 秒后：
- 最旧的格子（T-60s）被丢弃
- 新增一个格子（T）
- 窗口向右滑动 1 格
```

**实现伪代码：**

```java
public class SlidingWindowLimiter {
    private final long windowSizeMs;      // 窗口大小：60000ms (1分钟)
    private final int bucketCount;        // 格子数量：60
    private final int maxRequests;        // 最大请求数：100

    private final Bucket[] buckets;       // 存储格子的数组

    // 格子类
    class Bucket {
        long windowStart;   // 格子的起始时间戳
        int counter;        // 格子内的请求计数
    }

    public boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 1. 计算当前请求属于哪个格子
        int index = (int)((now / bucketSizeMs) % bucketCount);
        Bucket bucket = buckets[index];

        // 2. 如果格子过期（超过窗口大小），重置格子
        if (now - bucket.windowStart >= windowSizeMs) {
            bucket.windowStart = (now / bucketSizeMs) * bucketSizeMs;
            bucket.counter = 0;
        }

        // 3. 统计整个窗口内的请求总数
        int totalCount = 0;
        for (int i = 0; i < bucketCount; i++) {
            Bucket b = buckets[i];
            // 只统计有效的格子（在窗口范围内）
            if (now - b.windowStart < windowSizeMs) {
                totalCount += b.counter;
            }
        }

        // 4. 判断是否超过阈值
        if (totalCount < maxRequests) {
            bucket.counter++;
            return true;
        }

        return false;
    }
}
```

**可视化示例：**

```
限流规则：每 60 秒最多 100 个请求
格子大小：1 秒

时间线演示：
T=0s    窗口范围 [-60s, 0s)
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│ 2│ 3│ 1│ 2│ 5│ 4│ 3│ 2│ 6│ 5│  总计：33 ✅ 通过
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
 ↑                              ↑
-60s                            0s

T=1s    窗口范围 [-59s, 1s)
   ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
   │ 3│ 1│ 2│ 5│ 4│ 3│ 2│ 6│ 5│ 4│  总计：35 ✅ 通过
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
    ↑                              ↑
  -59s                             1s

窗口滑动 → 丢弃最旧的格子（-60s），新增格子（1s）

优势：解决了固定窗口的临界问题！
```

**优缺点：**

| 优点 | 缺点 |
|------|------|
| ✅ **解决临界问题**：平滑限流 | ❌ 内存占用较大（需要存储多个格子） |
| ✅ 精度高（取决于格子数量） | ❌ 实现相对复杂 |
| ✅ 统计数据丰富（可查看历史趋势） | ❌ 每次判断需要遍历所有格子 |

---

### 1.3 漏桶（Leaky Bucket）算法

**原理：**

```
请求像水一样流入漏桶，以固定速率流出

        请求到来
           ↓
    ┌──────────────┐
    │              │
    │   ████████   │  ← 漏桶（队列）
    │   ████████   │     容量固定
    │   ██████     │
    │              │
    └──────┬───────┘
           ↓
      固定速率流出
      (如每秒 10 个)

规则：
1. 请求到来时，加入漏桶队列
2. 如果漏桶已满，拒绝请求（溢出）
3. 以固定速率从漏桶取出请求处理

特点：流出速率恒定，可以削峰填谷
```

**实现代码：**

```java
public class LeakyBucketLimiter {
    private final int capacity;          // 漏桶容量
    private final int leakRate;          // 流出速率（个/秒）

    private int water;                   // 当前水量
    private long lastLeakTime;           // 上次漏水时间

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 1. 计算从上次漏水到现在，应该漏掉多少水
        long elapsedMs = now - lastLeakTime;
        int leaked = (int)(elapsedMs * leakRate / 1000);

        // 2. 更新水量
        water = Math.max(0, water - leaked);
        lastLeakTime = now;

        // 3. 判断漏桶是否已满
        if (water < capacity) {
            water++;  // 请求加入漏桶
            return true;
        }

        return false;  // 漏桶已满，拒绝请求
    }
}
```

**可视化示例：**

```
漏桶容量：100 个请求
流出速率：10 个/秒

时间线演示：
T=0s    桶内水量：0
        ↓ 突发 50 个请求
        桶内水量：50 ✅ 全部接受

T=1s    流出 10 个
        桶内水量：40

T=2s    流出 10 个
        桶内水量：30
        ↓ 突发 80 个请求
        桶内水量：100 (接受 70 个，拒绝 10 个 ❌)

T=3s    流出 10 个
        桶内水量：90

优势：平滑处理突发流量，输出速率恒定
缺点：无法应对短时间的合理突发（如秒杀场景）
```

**优缺点：**

| 优点 | 缺点 |
|------|------|
| ✅ 流量平滑：输出速率恒定 | ❌ 无法应对突发流量需求 |
| ✅ 削峰填谷：保护下游系统 | ❌ 漏桶满时，新请求全部拒绝 |
| ✅ 实现简单 | ❌ 可能导致请求排队等待时间过长 |

---

### 1.4 令牌桶（Token Bucket）算法

**原理：**

```
以固定速率向桶中放入令牌，请求需要获取令牌才能通过

        固定速率生成令牌
        (如每秒 10 个)
              ↓
    ┌──────────────┐
    │  🪙 🪙 🪙 🪙  │  ← 令牌桶
    │  🪙 🪙 🪙 🪙  │     容量固定
    │  🪙 🪙        │
    └──────────────┘
           ↑
      请求获取令牌
      (消耗令牌)

规则：
1. 令牌以固定速率生成（如每秒 10 个）
2. 令牌桶有最大容量（如 100 个）
3. 请求到来时，尝试获取 1 个令牌
4. 如果有令牌，消耗令牌，请求通过
5. 如果无令牌，拒绝请求

特点：允许一定程度的突发流量
```

**实现代码：**

```java
public class TokenBucketLimiter {
    private final int capacity;          // 令牌桶容量
    private final int refillRate;        // 令牌生成速率（个/秒）

    private int tokens;                  // 当前令牌数
    private long lastRefillTime;         // 上次补充令牌时间

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 1. 计算应该补充多少令牌
        long elapsedMs = now - lastRefillTime;
        int tokensToAdd = (int)(elapsedMs * refillRate / 1000);

        // 2. 补充令牌（不超过容量）
        if (tokensToAdd > 0) {
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTime = now;
        }

        // 3. 尝试消耗 1 个令牌
        if (tokens > 0) {
            tokens--;
            return true;
        }

        return false;
    }
}
```

**可视化示例：**

```
令牌桶容量：100 个
生成速率：10 个/秒

时间线演示：
T=0s    令牌数：100 (满桶)
        ↓ 突发 120 个请求
        通过 100 个 ✅，拒绝 20 个 ❌
        令牌数：0

T=1s    生成 10 个令牌
        令牌数：10
        ↓ 5 个请求
        通过 5 个 ✅
        令牌数：5

T=2s    生成 10 个令牌
        令牌数：15

优势：允许短时间突发（最多 100 个），比漏桶灵活
```

**优缺点：**

| 优点 | 缺点 |
|------|------|
| ✅ **允许突发流量**（桶满时） | ❌ 实现稍复杂 |
| ✅ 灵活性高 | ❌ 突发流量可能对下游造成压力 |
| ✅ 广泛应用（Guava RateLimiter） | ❌ 需要额外线程定期生成令牌 |

---

### 1.5 四种算法对比总结

| 算法 | 精确度 | 突发处理 | 内存占用 | 实现复杂度 | 适用场景 |
|------|--------|----------|----------|-----------|----------|
| **固定窗口** | ⭐⭐ | ❌ 临界问题 | 极小 | 简单 | 低精度限流 |
| **滑动窗口** | ⭐⭐⭐⭐⭐ | ✅ 平滑限流 | 中等 | 中等 | **推荐：精确限流** |
| **漏桶** | ⭐⭐⭐ | ❌ 流量平滑 | 小 | 简单 | 消息队列、流量整形 |
| **令牌桶** | ⭐⭐⭐⭐ | ✅ 允许突发 | 小 | 中等 | API 限流、突发场景 |

**Sentinel 选择滑动窗口的原因：**

1. ✅ **精度最高**：可以精确控制每秒 QPS
2. ✅ **统计数据丰富**：可以查看实时监控、历史趋势
3. ✅ **无临界问题**：窗口平滑滑动，无流量突刺
4. ✅ **易于扩展**：可以统计多种指标（QPS、RT、异常数等）

---

## 2. 滑动时间窗口算法原理

### 2.1 Sentinel 滑动窗口设计

**核心概念：**

```
统计窗口：1 秒（1000ms）
样本窗口：500ms（默认）
样本数量：2 个

┌────────────────────────────────────────────────────┐
│              1 秒统计窗口（Interval）                │
├────────────────────┬───────────────────────────────┤
│  样本窗口 1 (500ms)│  样本窗口 2 (500ms)            │
│  WindowWrap        │  WindowWrap                   │
│  ┌──────────────┐ │ ┌──────────────┐              │
│  │ windowStart  │ │ │ windowStart  │              │
│  │ windowLength │ │ │ windowLength │              │
│  │ MetricBucket │ │ │ MetricBucket │              │
│  │  - pass: 10  │ │ │  - pass: 15  │              │
│  │  - block: 2  │ │ │  - block: 1  │              │
│  │  - rt: 45ms  │ │ │  - rt: 38ms  │              │
│  └──────────────┘ │ └──────────────┘              │
└────────────────────┴───────────────────────────────┘

当前 QPS = (10 + 15) / 1s = 25
```

**关键术语：**

| 术语 | 英文 | 说明 | 示例 |
|------|------|------|------|
| 统计窗口 | Interval | 整体时间窗口 | 1000ms（1秒） |
| 样本窗口 | Sample Window | 最小统计单元 | 500ms |
| 样本数量 | Sample Count | 样本窗口数量 | 2 个 |
| 窗口包装 | WindowWrap | 封装样本窗口的对象 | 包含起始时间、长度、统计数据 |
| 指标桶 | MetricBucket | 存储统计数据 | pass, block, rt, exception 等 |

---

### 2.2 LeapArray 数据结构

**LeapArray 是 Sentinel 滑动窗口的核心实现：**

```java
/**
 * LeapArray（跳跃数组）
 *
 * 原理：环形数组 + 时间戳判断
 *
 * @param <T> 窗口统计数据类型（如 MetricBucket）
 */
public class LeapArray<T> {

    // 样本窗口长度（毫秒）
    protected int windowLengthInMs;

    // 样本窗口数量
    protected int sampleCount;

    // 统计窗口长度（毫秒）= windowLengthInMs * sampleCount
    protected int intervalInMs;

    // 存储样本窗口的数组（环形）
    protected final AtomicReferenceArray<WindowWrap<T>> array;

    /**
     * 窗口包装类：封装样本窗口
     */
    public static class WindowWrap<T> {
        private final long windowLengthInMs;  // 窗口长度
        private long windowStart;              // 窗口起始时间戳
        private T value;                       // 统计数据（MetricBucket）

        // 判断窗口是否在指定时间内
        public boolean isTimeInWindow(long timeMillis) {
            return windowStart <= timeMillis && timeMillis < windowStart + windowLengthInMs;
        }
    }
}
```

**环形数组原理：**

```
数组大小 = sampleCount = 2

索引计算公式：index = (timeMillis / windowLengthInMs) % sampleCount

示例：windowLengthInMs = 500ms, sampleCount = 2

时间戳         计算过程                               索引
0ms       → (0 / 500) % 2 = 0 % 2                 → 0
500ms     → (500 / 500) % 2 = 1 % 2               → 1
1000ms    → (1000 / 500) % 2 = 2 % 2              → 0  (复用)
1500ms    → (1500 / 500) % 2 = 3 % 2              → 1  (复用)
2000ms    → (2000 / 500) % 2 = 4 % 2              → 0  (复用)

环形数组：
┌─────┬─────┐
│  0  │  1  │
└─────┴─────┘
  ↑           ← 当时间推进时，循环复用数组位置
  └───────────┘
```

---

### 2.3 滑动窗口核心流程

**流程图：**

```
请求到来
    ↓
1. 计算当前时间戳所属的样本窗口索引
   index = (currentTimeMillis / windowLengthInMs) % sampleCount
    ↓
2. 获取该索引位置的 WindowWrap
   WindowWrap<T> windowWrap = array.get(index)
    ↓
3. 判断窗口是否有效
   ├─ 如果为 null → 创建新窗口
   ├─ 如果窗口起始时间 = 当前窗口 → 直接使用
   └─ 如果窗口过期（太旧）→ 重置窗口
    ↓
4. 更新统计数据
   windowWrap.value().addPass(1)  // 通过数 +1
    ↓
5. 判断是否限流
   long totalPass = sum(所有有效窗口的 pass)
   if (totalPass >= threshold) {
       return false;  // 限流
   }
    ↓
6. 请求通过
   return true;
```

---

## 3. Sentinel LeapArray 核心实现

### 3.1 核心方法：currentWindow()

**作用：获取当前时间戳对应的样本窗口**

```java
/**
 * 获取当前时间戳对应的窗口
 *
 * @param timeMillis 当前时间戳
 * @return 当前窗口
 */
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }

    // 1. 计算当前时间戳所属的窗口索引
    int idx = calculateTimeIdx(timeMillis);

    // 2. 计算当前窗口的起始时间
    long windowStart = calculateWindowStart(timeMillis);

    /*
     * 3. 从数组中获取该索引位置的窗口
     *
     * 可能的情况：
     * (1) 窗口为 null（从未使用过）
     * (2) 窗口的起始时间 = windowStart（正好是当前窗口）
     * (3) 窗口的起始时间 < windowStart（窗口过期，需要重置）
     * (4) 窗口的起始时间 > windowStart（不可能，时间不会倒流）
     */

    while (true) {
        WindowWrap<T> old = array.get(idx);

        // 情况 (1)：窗口为 null，创建新窗口
        if (old == null) {
            WindowWrap<T> window = new WindowWrap<>(
                windowLengthInMs,
                windowStart,
                newEmptyBucket(timeMillis)  // 创建空的统计数据
            );

            // 使用 CAS 操作保证线程安全
            if (array.compareAndSet(idx, null, window)) {
                return window;  // 创建成功，返回新窗口
            } else {
                // CAS 失败，说明其他线程已经创建了窗口，重新循环
                Thread.yield();
            }
        }
        // 情况 (2)：窗口起始时间正好匹配，直接使用
        else if (windowStart == old.windowStart()) {
            return old;
        }
        // 情况 (3)：窗口过期（起始时间太旧），需要重置
        else if (windowStart > old.windowStart()) {
            // 使用锁保证只有一个线程重置窗口
            if (updateLock.tryLock()) {
                try {
                    // 重置窗口数据
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // 获取锁失败，让出 CPU 时间片，重新循环
                Thread.yield();
            }
        }
        // 情况 (4)：窗口起始时间 > 当前计算的起始时间（不应该发生）
        else {
            // 这种情况理论上不应该发生，除非时钟回拨
            // 等待一段时间，让时间追上
            return new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}

/**
 * 计算时间戳对应的窗口索引
 *
 * @param timeMillis 时间戳
 * @return 数组索引
 */
private int calculateTimeIdx(long timeMillis) {
    // 公式：(timeMillis / windowLengthInMs) % sampleCount
    long timeId = timeMillis / windowLengthInMs;
    return (int)(timeId % array.length());
}

/**
 * 计算窗口的起始时间
 *
 * @param timeMillis 时间戳
 * @return 窗口起始时间
 */
protected long calculateWindowStart(long timeMillis) {
    // 公式：(timeMillis / windowLengthInMs) * windowLengthInMs
    return timeMillis - timeMillis % windowLengthInMs;
}

/**
 * 重置窗口数据
 */
private WindowWrap<T> resetWindowTo(WindowWrap<T> windowWrap, long startTime) {
    // 重置窗口起始时间
    windowWrap.resetWindowStart(startTime);
    // 重置统计数据（清零）
    windowWrap.value().reset();
    return windowWrap;
}
```

---

### 3.2 核心方法：values()

**作用：获取当前时间窗口内所有有效的样本窗口**

```java
/**
 * 获取当前时间范围内的所有有效窗口
 *
 * @param timeMillis 当前时间戳
 * @return 有效窗口列表
 */
public List<WindowWrap<T>> values(long timeMillis) {
    if (timeMillis < 0) {
        return new ArrayList<>();
    }

    List<WindowWrap<T>> result = new ArrayList<>(array.length());

    // 遍历数组中的所有窗口
    for (int i = 0; i < array.length(); i++) {
        WindowWrap<T> windowWrap = array.get(i);

        // 跳过空窗口
        if (windowWrap == null) {
            continue;
        }

        // 判断窗口是否在有效范围内
        // 有效范围：[timeMillis - intervalInMs, timeMillis)
        if (isWindowDeprecated(timeMillis, windowWrap)) {
            // 窗口过期，跳过
            continue;
        }

        result.add(windowWrap);
    }

    return result;
}

/**
 * 判断窗口是否过期
 *
 * @param timeMillis 当前时间戳
 * @param windowWrap 窗口
 * @return true = 过期, false = 有效
 */
public boolean isWindowDeprecated(long timeMillis, WindowWrap<T> windowWrap) {
    // 窗口结束时间 = windowStart + windowLengthInMs
    long windowEnd = windowWrap.windowStart() + windowWrap.windowLength();

    // 如果窗口结束时间 <= (当前时间 - 统计窗口大小)，则窗口过期
    return windowEnd <= (timeMillis - intervalInMs);
}
```

**可视化示例：**

```
当前时间：T = 1500ms
统计窗口：intervalInMs = 1000ms
样本窗口：windowLengthInMs = 500ms
样本数量：sampleCount = 2

有效范围：[T - intervalInMs, T) = [500ms, 1500ms)

数组状态：
┌─────────────────┬─────────────────┐
│    索引 0       │    索引 1        │
├─────────────────┼─────────────────┤
│ windowStart:    │ windowStart:     │
│   1000ms        │   1500ms         │
│                 │                  │
│ 窗口结束:        │ 窗口结束:         │
│   1500ms        │   2000ms         │
│                 │                  │
│ 是否有效:        │ 是否有效:         │
│   ✅ 有效        │   ✅ 有效         │
│   (1500 > 500)  │   (2000 > 500)   │
└─────────────────┴─────────────────┘

如果时间推进到 T = 2000ms：
有效范围：[1000ms, 2000ms)

┌─────────────────┬─────────────────┐
│    索引 0       │    索引 1        │
├─────────────────┼─────────────────┤
│ windowStart:    │ windowStart:     │
│   1000ms        │   1500ms         │
│                 │                  │
│ 是否有效:        │ 是否有效:         │
│   ❌ 过期        │   ✅ 有效         │
│   (1500 = 1000) │   (2000 > 1000)  │
│   (临界，算过期)  │                  │
└─────────────────┴─────────────────┘
```

---

### 3.3 统计数据结构：MetricBucket

**作用：存储单个样本窗口内的统计数据**

```java
/**
 * 指标桶：存储各种统计指标
 */
public class MetricBucket {

    // 使用 LongAdder 实现高性能并发计数
    // LongAdder 相比 AtomicLong 在高并发下性能更好

    private final LongAdder pass = new LongAdder();      // 通过数
    private final LongAdder block = new LongAdder();     // 阻塞数
    private final LongAdder exception = new LongAdder(); // 异常数
    private final LongAdder rt = new LongAdder();        // 总响应时间
    private final LongAdder success = new LongAdder();   // 成功数

    /**
     * 增加通过数
     */
    public void addPass(int count) {
        pass.add(count);
    }

    /**
     * 增加响应时间
     */
    public void addRT(long rt) {
        this.rt.add(rt);
    }

    /**
     * 获取通过数
     */
    public long pass() {
        return pass.sum();
    }

    /**
     * 获取平均响应时间
     */
    public long avgRt() {
        long successCount = success.sum();
        if (successCount == 0) {
            return 0;
        }
        return rt.sum() / successCount;
    }

    /**
     * 重置所有统计数据
     */
    public void reset() {
        pass.reset();
        block.reset();
        exception.reset();
        rt.reset();
        success.reset();
    }
}
```

---

## 4. 源码级实现剖析

### 4.1 完整的限流判断流程

**从请求进入到限流判断的完整调用链：**

```
请求进入
    ↓
SentinelWebInterceptor（拦截器）
    ↓
Entry entry = SphU.entry(resourceName)
    ↓
ProcessorSlotChain（处理器链）
    ├─ NodeSelectorSlot（选择节点）
    ├─ ClusterBuilderSlot（构建集群统计）
    ├─ LogSlot（日志记录）
    ├─ StatisticSlot（统计数据）← 更新滑动窗口
    │     ↓
    │     currentWindow().addPass(1)
    ├─ FlowSlot（流量控制）  ← 判断是否限流
    │     ↓
    │     FlowRuleChecker.checkFlow()
    │     ↓
    │     统计窗口内的 QPS
    │     sum(values().map(w -> w.value().pass()))
    │     ↓
    │     if (totalPass >= threshold) → 抛出 FlowException
    ├─ DegradeSlot（熔断降级）
    └─ SystemSlot（系统保护）
```

**StatisticSlot 核心代码：**

```java
public class StatisticSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, Object... args)
            throws Throwable {

        try {
            // 1. 继续执行下一个 Slot（如 FlowSlot 进行限流判断）
            fireEntry(context, resourceWrapper, node, count, args);

            // 2. 如果没有被限流，增加通过数
            node.increaseThreadNum();
            node.addPassRequest(count);  // ← 调用 LeapArray.currentWindow().addPass()

            // 3. 记录其他统计信息（如来源、上下文等）
            // ...

        } catch (BlockException e) {
            // 4. 如果被限流，增加阻塞数
            node.increaseBlockQps(count);
            throw e;
        }
    }

    @Override
    public void exit(Entry entry, int count, Object... args) {
        // 5. 请求完成时，记录响应时间、成功数等
        DefaultNode node = (DefaultNode)entry.getCurNode();

        if (entry.getError() == null) {
            long rt = TimeUtil.currentTimeMillis() - entry.getCreateTime();
            node.addRtAndSuccess(rt, count);
            node.decreaseThreadNum();
        }

        // ...
    }
}
```

**FlowSlot 核心代码：**

```java
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, Object... args)
            throws Throwable {

        // 1. 检查流控规则
        checkFlow(resourceWrapper, context, node, count);

        // 2. 如果通过流控，继续执行下一个 Slot
        fireEntry(context, resourceWrapper, node, count, args);
    }

    void checkFlow(ResourceWrapper resource, Context context,
                   DefaultNode node, int count) throws BlockException {

        // 获取该资源的所有流控规则
        List<FlowRule> rules = FlowRuleManager.getRules(resource.getName());

        if (rules != null) {
            for (FlowRule rule : rules) {
                // 逐个检查规则
                if (!canPassCheck(rule, context, node, count)) {
                    // 不通过，抛出 FlowException
                    throw new FlowException(rule.getLimitApp(), rule);
                }
            }
        }
    }
}
```

**FlowRuleChecker 核心代码：**

```java
public class FlowRuleChecker {

    public static boolean passCheck(FlowRule rule, Context context,
                                     DefaultNode node, int acquireCount) {

        // 1. 获取限流指标（QPS、线程数等）
        String limitApp = rule.getLimitApp();

        // 2. 根据流控模式选择统计节点
        Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
        if (selectedNode == null) {
            return true;
        }

        // 3. 调用具体的流控策略
        return rule.getRater().canPass(selectedNode, acquireCount);
    }
}

/**
 * QPS 流控策略（基于滑动窗口）
 */
public class DefaultController implements TrafficShapingController {

    private final double count;  // QPS 阈值

    @Override
    public boolean canPass(Node node, int acquireCount) {
        // 1. 获取当前窗口的通过 QPS
        long currentQps = node.passQps();  // ← 调用 LeapArray.values() 统计

        // 2. 判断是否超过阈值
        if (currentQps + acquireCount > count) {
            return false;  // 限流
        }

        return true;  // 通过
    }
}
```

**Node.passQps() 实现：**

```java
public class StatisticNode implements Node {

    // 滑动窗口统计数据
    private transient Metric rollingCounterInSecond;

    @Override
    public long passQps() {
        // 获取当前秒级窗口的通过数
        return rollingCounterInSecond.pass();
    }
}

/**
 * 滑动窗口指标
 */
public class ArrayMetric implements Metric {

    private final LeapArray<MetricBucket> data;

    @Override
    public long pass() {
        // 1. 获取当前有效的所有样本窗口
        List<WindowWrap<MetricBucket>> windows = data.values();

        // 2. 累加所有窗口的通过数
        long pass = 0;
        for (WindowWrap<MetricBucket> window : windows) {
            pass += window.value().pass();
        }

        return pass;
    }

    @Override
    public void addPass(int count) {
        // 获取当前时间戳的窗口，并增加通过数
        WindowWrap<MetricBucket> wrap = data.currentWindow();
        wrap.value().addPass(count);
    }
}
```

---

### 4.2 滑动窗口实战示例

**场景：限制 QPS 为 10**

```
规则配置：
- 资源名：/api/order
- 限流阈值：10 QPS
- 统计窗口：1000ms
- 样本窗口：500ms
- 样本数量：2

时间线模拟：
T=0ms
┌───────────────┬───────────────┐
│  窗口 0 (0ms) │  窗口 1 (未创建)│
│  pass: 0      │               │
└───────────────┴───────────────┘

T=100ms  → 5 个请求到来
         → currentWindow(100).addPass(5)
         → 计算索引：(100/500)%2 = 0
         → 使用窗口 0
┌───────────────┬───────────────┐
│  窗口 0 (0ms) │  窗口 1 (未创建)│
│  pass: 5      │               │
└───────────────┴───────────────┘
总 QPS = 5 ✅ 通过

T=200ms  → 6 个请求到来
         → currentWindow(200).addPass(6)
         → 索引：(200/500)%2 = 0
┌───────────────┬───────────────┐
│  窗口 0 (0ms) │  窗口 1 (未创建)│
│  pass: 11     │               │
└───────────────┴───────────────┘
总 QPS = 11 ❌ 限流！(超过阈值 10)

T=550ms  → 8 个请求到来
         → currentWindow(550).addPass(8)
         → 索引：(550/500)%2 = 1
         → 创建窗口 1
┌───────────────┬───────────────┐
│  窗口 0 (0ms) │  窗口 1 (500ms)│
│  pass: 11     │  pass: 8      │
└───────────────┴───────────────┘
有效窗口：[0ms-500ms), [500ms-1000ms)
总 QPS = 11 + 8 = 19 ❌ 限流！

T=1050ms → 5 个请求到来
          → currentWindow(1050).addPass(5)
          → 索引：(1050/500)%2 = 0
          → 窗口 0 过期，重置窗口
┌───────────────┬───────────────┐
│ 窗口 0 (1000ms)│ 窗口 1 (500ms)│
│  pass: 5      │  pass: 8      │
└───────────────┴───────────────┘
有效窗口：[500ms-1000ms), [1000ms-1500ms)
总 QPS = 8 + 5 = 13 ❌ 限流！

T=1550ms → 3 个请求到来
          → 索引：(1550/500)%2 = 1
          → 窗口 1 过期，重置窗口
┌───────────────┬───────────────┐
│ 窗口 0 (1000ms)│ 窗口 1 (1500ms)│
│  pass: 5      │  pass: 3      │
└───────────────┴───────────────┘
有效窗口：[1000ms-1500ms), [1500ms-2000ms)
总 QPS = 5 + 3 = 8 ✅ 通过
```

---

## 5. 性能优化与最佳实践

### 5.1 Sentinel 滑动窗口的性能优化

**1. 环形数组复用**

```java
// 不需要频繁创建新数组，只需重置窗口数据
// 内存占用固定，不会随时间增长

// 数组大小 = sampleCount（如 2 个）
// 无论运行多久，内存占用不变
```

**2. LongAdder 高性能计数**

```java
// 相比 AtomicLong，LongAdder 在高并发下性能更好
// 原理：分段计数，减少 CAS 竞争

private final LongAdder pass = new LongAdder();

// 添加操作（高并发友好）
pass.add(1);

// 获取总和（适合低频读取）
long total = pass.sum();
```

**3. CAS 无锁操作**

```java
// 创建窗口时使用 CAS 保证线程安全
if (array.compareAndSet(idx, null, window)) {
    return window;
}

// 避免使用重量级锁（synchronized）
// 提高并发性能
```

**4. 懒加载窗口**

```java
// 窗口只在需要时才创建
// 不会预先分配所有窗口，节省内存

if (old == null) {
    WindowWrap<T> window = new WindowWrap<>(...);
    array.compareAndSet(idx, null, window);
}
```

---

### 5.2 配置优化建议

**样本窗口数量选择：**

| 场景 | 样本数量 | 样本窗口大小 | 精度 | 内存占用 |
|------|----------|--------------|------|----------|
| **低精度限流** | 2 | 500ms | 一般 | 极小 |
| **标准限流** | 4 | 250ms | 较高 | 小 |
| **高精度限流** | 10 | 100ms | 高 | 中等 |
| **秒杀场景** | 20 | 50ms | 极高 | 较大 |

**配置示例：**

```java
// 方式一：代码配置（不推荐，精度固定）
FlowRule rule = new FlowRule();
rule.setResource("myResource");
rule.setCount(100);  // QPS 阈值
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);

// 方式二：动态配置（推荐，可调整精度）
// 通过 Sentinel Dashboard 或配置中心动态调整

// 样本数量默认值（Sentinel 源码）
// SampleCountProperty.SAMPLE_COUNT = 2（秒级窗口）
// SampleCountProperty.SAMPLE_COUNT = 60（分钟级窗口）
```

---

### 5.3 监控与调优

**实时监控指标：**

```java
// 通过 MetricBucket 可以获取丰富的监控数据

public class MetricBucket {
    long pass;        // 通过数
    long block;       // 阻塞数
    long exception;   // 异常数
    long rt;          // 总响应时间
    long success;     // 成功数
}

// 计算指标：
// 1. QPS（每秒查询率）= pass / 时间窗口（秒）
// 2. 限流率 = block / (pass + block)
// 3. 平均响应时间 = rt / success
// 4. 异常率 = exception / success
```

**Sentinel Dashboard 监控：**

```
访问 http://localhost:8080（Sentinel 控制台）

可以查看：
- 实时 QPS（通过 / 阻塞）
- 响应时间曲线
- 限流规则配置
- 集群流控状态
```

---

## 6. 实战案例与代码示例

### 6.1 Spring Cloud Alibaba 集成 Sentinel

**1. 添加依赖**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

**2. 配置文件**

```yaml
spring:
  application:
    name: sentinel-demo
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080  # Sentinel 控制台地址
        port: 8719                 # 客户端通信端口

      # 启用 Sentinel（默认 true）
      enabled: true

      # 饥饿加载（推荐开启，避免首次请求失败）
      eager: true
```

**3. 使用 @SentinelResource 注解**

```java
@RestController
public class OrderController {

    /**
     * 使用 @SentinelResource 注解标记资源
     *
     * @param orderId 订单 ID
     * @return 订单详情
     */
    @GetMapping("/order/{orderId}")
    @SentinelResource(
        value = "getOrder",           // 资源名称
        blockHandler = "handleBlock", // 限流处理方法
        fallback = "handleFallback"   // 异常降级方法
    )
    public String getOrder(@PathVariable String orderId) {
        // 模拟业务逻辑
        return "订单详情：" + orderId;
    }

    /**
     * 限流处理方法
     *
     * 注意：方法签名必须与原方法一致，且多一个 BlockException 参数
     */
    public String handleBlock(String orderId, BlockException ex) {
        return "系统繁忙，请稍后再试！（限流）";
    }

    /**
     * 异常降级方法
     *
     * 注意：方法签名必须与原方法一致，且多一个 Throwable 参数
     */
    public String handleFallback(String orderId, Throwable ex) {
        return "服务异常，请稍后再试！（降级）";
    }
}
```

**4. 代码配置限流规则**

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        // 规则 1：QPS 限流
        FlowRule rule1 = new FlowRule();
        rule1.setResource("getOrder");
        rule1.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 模式
        rule1.setCount(10);  // QPS 阈值：10
        rules.add(rule1);

        // 规则 2：并发线程数限流
        FlowRule rule2 = new FlowRule();
        rule2.setResource("createOrder");
        rule2.setGrade(RuleConstant.FLOW_GRADE_THREAD);  // 线程数模式
        rule2.setCount(5);  // 最大并发线程数：5
        rules.add(rule2);

        // 加载规则
        FlowRuleManager.loadRules(rules);
    }
}
```

**5. 通过 Sentinel Dashboard 配置规则**

```
步骤：
1. 启动 Sentinel Dashboard
   java -jar sentinel-dashboard.jar

2. 访问 http://localhost:8080
   用户名/密码：sentinel/sentinel

3. 调用一次接口，触发懒加载
   curl http://localhost:8080/order/123

4. 在控制台左侧找到应用 sentinel-demo

5. 点击"流控规则" → "新增流控规则"
   - 资源名：getOrder
   - 阈值类型：QPS
   - 单机阈值：10

6. 测试限流效果
   使用 JMeter 或 ab 压测工具
   ab -n 1000 -c 20 http://localhost:8080/order/123
```

---

### 6.2 自定义滑动窗口实现

**简化版 LeapArray 实现：**

```java
/**
 * 简化版滑动窗口限流器
 *
 * 用于理解滑动窗口原理
 */
public class SimpleSlidingWindowLimiter {

    // 样本窗口数组
    private final WindowWrap[] array;

    // 样本窗口长度（毫秒）
    private final int windowLengthInMs;

    // 样本数量
    private final int sampleCount;

    // QPS 阈值
    private final int threshold;

    public SimpleSlidingWindowLimiter(int windowLengthInMs, int sampleCount, int threshold) {
        this.windowLengthInMs = windowLengthInMs;
        this.sampleCount = sampleCount;
        this.threshold = threshold;
        this.array = new WindowWrap[sampleCount];
    }

    /**
     * 尝试获取令牌
     *
     * @return true = 通过, false = 限流
     */
    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 1. 获取当前窗口
        WindowWrap currentWindow = currentWindow(now);

        // 2. 统计所有有效窗口的请求数
        int totalCount = 0;
        for (WindowWrap window : array) {
            if (window != null && !isWindowDeprecated(now, window)) {
                totalCount += window.counter;
            }
        }

        // 3. 判断是否超过阈值
        if (totalCount < threshold) {
            currentWindow.counter++;  // 增加计数
            return true;              // 通过
        }

        return false;  // 限流
    }

    /**
     * 获取当前时间戳对应的窗口
     */
    private WindowWrap currentWindow(long timeMillis) {
        // 计算索引
        int idx = (int)((timeMillis / windowLengthInMs) % sampleCount);

        // 计算窗口起始时间
        long windowStart = (timeMillis / windowLengthInMs) * windowLengthInMs;

        WindowWrap window = array[idx];

        // 如果窗口为 null 或过期，创建/重置窗口
        if (window == null || window.windowStart != windowStart) {
            window = new WindowWrap(windowStart, windowLengthInMs);
            array[idx] = window;
        }

        return window;
    }

    /**
     * 判断窗口是否过期
     */
    private boolean isWindowDeprecated(long timeMillis, WindowWrap window) {
        long windowEnd = window.windowStart + window.windowLength;
        long intervalInMs = windowLengthInMs * sampleCount;
        return windowEnd <= (timeMillis - intervalInMs);
    }

    /**
     * 窗口包装类
     */
    static class WindowWrap {
        long windowStart;      // 窗口起始时间
        int windowLength;      // 窗口长度
        int counter;           // 计数器

        WindowWrap(long windowStart, int windowLength) {
            this.windowStart = windowStart;
            this.windowLength = windowLength;
            this.counter = 0;
        }
    }

    /**
     * 测试方法
     */
    public static void main(String[] args) throws InterruptedException {
        // 创建限流器：窗口 1 秒，2 个样本，阈值 10
        SimpleSlidingWindowLimiter limiter = new SimpleSlidingWindowLimiter(500, 2, 10);

        // 模拟请求
        for (int i = 0; i < 20; i++) {
            boolean result = limiter.tryAcquire();
            System.out.printf("第 %2d 个请求：%s\n", i + 1, result ? "✅ 通过" : "❌ 限流");
            Thread.sleep(50);  // 每 50ms 一个请求
        }

        // 预期结果：
        // 前 10 个请求通过（500ms 内）
        // 第 11-20 个请求部分限流（取决于时间窗口滑动）
    }
}
```

---

## 📚 总结

### 核心要点回顾

1. **四种限流算法对比**：
   - 固定窗口：实现简单，但有临界问题
   - **滑动窗口：精度高，平滑限流**（Sentinel 选择）
   - 漏桶：流量平滑，但无法应对突发
   - 令牌桶：允许突发，灵活性高

2. **Sentinel 滑动窗口优势**：
   - ✅ 解决临界问题
   - ✅ 精度可配置（通过样本数量调整）
   - ✅ 统计数据丰富（QPS、RT、异常数等）
   - ✅ 性能优化（环形数组、LongAdder、CAS）

3. **LeapArray 核心原理**：
   - 环形数组存储样本窗口
   - 时间戳计算索引（取模复用）
   - 窗口过期自动重置
   - 线程安全（CAS 无锁）

4. **性能优化建议**：
   - 样本数量：标准场景 2-4 个，高精度场景 10-20 个
   - 使用 LongAdder 替代 AtomicLong
   - 懒加载窗口，节省内存
   - 监控关键指标（QPS、限流率、响应时间）

5. **实战建议**：
   - 优先使用 Sentinel Dashboard 动态配置规则
   - 开启饥饿加载（eager: true）
   - 使用 @SentinelResource 注解标记资源
   - 配置 blockHandler 和 fallback 处理限流和降级

---

## 🔗 参考资料

- [Sentinel 官方文档](https://sentinelguard.io/zh-cn/)
- [Sentinel GitHub 仓库](https://github.com/alibaba/Sentinel)
- [Spring Cloud Alibaba Sentinel 集成](https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel)
- [滑动窗口算法详解](https://github.com/alibaba/Sentinel/wiki/%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6)

---

**文档版本：** 1.0
**最后更新：** 2025-01-15
**适用版本：** Sentinel 1.8+ / Spring Cloud Alibaba 2022.x
