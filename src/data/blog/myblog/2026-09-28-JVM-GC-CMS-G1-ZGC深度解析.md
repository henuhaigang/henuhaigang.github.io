---
layout:     post
title:      JVM GC 深度解析：CMS、G1、ZGC 原理、优劣与选型指南
pubDatetime: 2026-09-28T00:00:00.000Z
modDatetime: 2026-09-28T00:00:00.000Z
author:     henuhaigang
header-img: img/post-bg-tomcat.jpg
catalog: true
tags:
    - JVM
    - GC
    - CMS
    - G1
    - ZGC
    - 性能调优
    - Java
description: JVM垃圾回收器深度解析：从GC基础原理到CMS/G1/ZGC三大回收器的工作机制、源码级细节、优缺点、调优实战与选型决策，适合从入门到12年经验的开发者
---

# JVM GC 深度解析：CMS、G1、ZGC 原理、优劣与选型指南

## 一、为什么需要理解GC

### 1.1 GC对生产环境的影响

GC（Garbage Collection）是Java的核心机制之一，但也是生产环境性能问题的常见根源：

- **STW（Stop-The-World）**：GC期间所有应用线程暂停，直接影响响应时间
- **内存泄漏**：对象无法被回收，最终OOM
- **频繁Full GC**：老年代回收过于频繁，服务周期性卡顿
- **延迟毛刺**：GC暂停导致P99/P999延迟飙升

> **12年经验一句话**：GC调优不是玄学，而是基于对回收器原理的理解，结合业务场景（吞吐量 vs 延迟 vs 内存占用）做出合理选择。

### 1.2 GC调优的核心目标

| 目标 | 说明 | 适用场景 |
|------|------|----------|
| **高吞吐量** | GC时间占比低，应用运行时间长 | 批处理、离线计算 |
| **低延迟** | GC暂停时间短且可预测 | 在线服务、实时交易 |
| **低内存占用** | 堆外内存少，资源利用率高 | 容器化部署、内存受限环境 |

**三者不可兼得**，需要根据业务特点取舍。

### 1.3 GC调优的常见误区

| 误区 | 真相 |
|------|------|
| "堆内存越大越好" | 堆越大，单次GC时间越长，STW越久 |
| "GC次数越少越好" | 关注的是GC总时间和单次最长时间，而非次数 |
| "默认配置就是最优" | 默认配置适合大多数场景，但特殊场景需要调优 |
| "GC调优可以解决所有性能问题" | 很多性能问题是代码问题，GC调优治标不治本 |
| "Full GC一定是坏事" | 偶发的Full GC是正常的，频繁Full GC才是问题 |

---

## 二、GC基础：新人必读

### 2.1 堆内存结构

```
┌─────────────────────────────────────────────────────────┐
│                      Heap Memory                         │
│  ┌─────────────────────┐  ┌─────────────────────────┐   │
│  │      Young Gen       │  │       Old Gen            │   │
│  │  ┌─────┬─────┬────┐ │  │                         │   │
│  │  │Eden │ S0  │ S1 │ │  │    Tenured Space        │   │
│  │  │     │     │    │ │  │                         │   │
│  │  └─────┴─────┴────┘ │  └─────────────────────────┘   │
│  └─────────────────────┘                                │
└─────────────────────────────────────────────────────────┘
```

| 区域 | 说明 | 默认比例 |
|------|------|----------|
| **Eden** | 新对象分配区域 | 8 |
| **Survivor 0/1** | Minor GC后存活对象 | 各1 |
| **Old Gen** | 长期存活对象 | 2 |

### 2.2 对象生命周期与晋升

```
新对象 → Eden → Minor GC → Survivor → 年龄达标 → Old Gen → Full GC
                ↓
            (年龄+1)
```

**晋升规则**：
- 对象每经历一次Minor GC，年龄+1
- 年龄达到阈值（默认15）晋升到老年代
- 大对象直接进入老年代（`-XX:PretenureSizeThreshold`）
- 动态年龄判断：Survivor中相同年龄对象总和超过Survivor空间一半，该年龄及以上对象直接晋升

### 2.3 如何判断对象可回收

**引用计数法**（Java未采用）：
- 优点：实现简单，回收及时
- 缺点：无法解决循环引用

**可达性分析**（Java采用）：
- GC Roots：栈帧局部变量、静态变量、常量、JNI引用等
- 从GC Roots不可达的对象可回收

### 2.4 四种引用类型

| 引用类型 | 回收时机 | 典型用途 |
|----------|----------|----------|
| **强引用** | 永不回收（除非置null） | 普通对象 `Object obj = new Object()` |
| **软引用** | 内存不足时回收 | 缓存 `SoftReference<T>` |
| **弱引用** | 下次GC必回收 | `WeakHashMap`、ThreadLocal |
| **虚引用** | 随时可回收，仅用于感知回收 | 堆外内存管理 `PhantomReference` |

### 2.5 GC Roots详解

GC Roots是可达性分析的起点，包括：

| 类型 | 说明 | 示例 |
|------|------|------|
| **栈帧局部变量** | 方法内的局部变量 | `Object obj = new Object()` |
| **静态变量** | 类的静态字段 | `static Object obj` |
| **常量** | 静态final字段 | `static final Object obj` |
| **JNI引用** | Native代码引用的Java对象 | JNI回调中的对象 |
| **活跃线程** | 正在运行的线程对象 | `Thread.currentThread()` |
| **类加载器** | 已加载的类加载器 | `ClassLoader` |
| **基本类型对应的Class** | 基本类型的Class对象 | `int.class` |

### 2.6 对象分配策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **指针碰撞** | 内存规整，移动指针分配 | 标记-整理、复制算法 |
| **空闲列表** | 内存不规整，从列表分配 | 标记-清除 |
| **TLAB** | 线程本地分配缓冲区 | 多线程环境，减少竞争 |
| **栈上分配** | 对象不逃逸，栈上分配 | 逃逸分析优化 |

---

## 三、GC算法基础

### 3.1 标记-清除（Mark-Sweep）

```
标记阶段：从GC Roots标记所有存活对象
清除阶段：回收未标记对象

优点：实现简单
缺点：内存碎片、效率随对象数量下降
```

### 3.2 标记-复制（Mark-Copy）

```
将内存分为两块，每次只用一块
GC时复制存活对象到另一块，清空当前块

优点：无碎片、分配快（指针碰撞）
缺点：内存利用率50%、存活率高时效率低
应用：新生代（Eden + Survivor）
```

### 3.3 标记-整理（Mark-Compact）

```
标记阶段：标记存活对象
整理阶段：存活对象向一端移动，清理边界外内存

优点：无碎片、内存利用率高
缺点：整理阶段耗时、需要移动对象
应用：老年代（CMS的备选方案）
```

### 3.4 分代收集（Generational Collection）

```
基于"弱分代假说"：绝大多数对象朝生夕灭
新生代：复制算法（存活率低）
老年代：标记-清除或标记-整理（存活率高）
```

### 3.5 增量收集与并发收集

| 方式 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| **增量收集** | 分多次完成GC | 单次STW短 | 总GC时间长 |
| **并发收集** | GC与用户线程并发 | STW短 | CPU开销大、实现复杂 |

---

## 四、CMS（Concurrent Mark Sweep）

### 4.1 概述

CMS是JDK 1.4引入的老年代收集器，**以获取最短回收停顿时间为目标**，是互联网早期（JDK 5-8时代）的主流选择。

> **现状**：JDK 9标记为废弃（Deprecated），JDK 14移除。但理解CMS对理解G1/ZGC的演进至关重要。

### 4.2 工作流程（4个阶段）

```
┌─────────────────────────────────────────────────────────────┐
│                    CMS 执行流程                              │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐ │
│  │ 初始标记  │ → │ 并发标记  │ → │ 重新标记  │ → │ 并发清除│ │
│  │ (STW)    │    │ (并发)   │    │ (STW)    │    │ (并发) │ │
│  │ 极短     │    │ 耗时最长  │    │ 较短     │    │ 并发  │ │
│  └──────────┘    └──────────┘    └──────────┘    └────────┘ │
│                                                             │
│  仅标记GC Roots      遍历整个对象图      修正并发标记期间    回收垃圾  │
│  直接关联的对象      但可与用户线程并发   用户线程变动的对象   与用户并发│
└─────────────────────────────────────────────────────────────┘
```

**阶段详解**：

| 阶段 | 是否STW | 耗时 | 说明 |
|------|---------|------|------|
| **初始标记** | 是 | 极短 | 仅标记GC Roots直接关联的对象 |
| **并发标记** | 否 | 最长 | 从GC Roots遍历整个对象图，与用户线程并发 |
| **重新标记** | 是 | 较短 | 修正并发标记期间用户线程导致变动的对象 |
| **并发清除** | 否 | 较长 | 清除垃圾对象，与用户线程并发 |

### 4.3 CMS的核心参数

```bash
# 启用CMS
-XX:+UseConcMarkSweepGC

# 老年代使用阈值（默认68%，JDK6后92%）
-XX:CMSInitiatingOccupancyFraction=70

# 启用CMSInitiatingOccupancyFraction必须配合此参数
-XX:+UseCMSInitiatingOccupancyOnly

# 并发GC线程数（默认 (ParallelGCThreads+3)/4）
-XX:ConcGCThreads=4

# Full GC后是否进行内存整理（默认true）
-XX:+UseCMSCompactAtFullCollection

# 多少次Full GC后进行一次整理（默认0，即每次）
-XX:CMSFullGCsBeforeCompaction=1

# 并行Full GC线程数
-XX:ParallelCMSThreads=4
```

### 4.4 CMS的三大缺点

#### 缺点1：内存碎片（Memory Fragmentation）

```
并发清除后：
┌─────────────────────────────────────────┐
│ ████    ████        ████    ████        │
│ 存活   垃圾       存活   垃圾          │
│ 对象   空间       对象   空间          │
└─────────────────────────────────────────┘
问题：无法分配大对象，触发Full GC
```

**解决方案**：`-XX:+UseCMSCompactAtFullCollection`（Full GC时整理）

#### 缺点2：浮动垃圾（Floating Garbage）

```
并发清除阶段：
用户线程继续运行 → 产生新垃圾 → 本次GC无法回收
必须等待下一次GC

解决方案：预留空间（CMSInitiatingOccupancyFraction）
```

#### 缺点3：Concurrent Mode Failure

```
场景：并发清除期间，老年代空间不足
后果：触发Serial Old收集器（单线程Full GC）
       → 长时间STW → 服务不可用

触发条件：
1. 老年代空间不足（浮动垃圾过多）
2. 大对象分配失败
3. 内存碎片导致无法分配
```

### 4.5 CMS调优实战

```bash
# 典型CMS配置（JDK 8）
-XX:+UseConcMarkSweepGC
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSInitiatingOccupancyFraction=70
-XX:+UseCMSCompactAtFullCollection
-XX:CMSFullGCsBeforeCompaction=1
-XX:+CMSParallelInitialMarkEnabled
-XX:+CMSScavengeBeforeRemark
-XX:+ParallelRefProcEnabled
```

**调优要点**：
1. `CMSInitiatingOccupancyFraction`：设置过低→频繁Full GC；过高→Concurrent Mode Failure
2. `CMSScavengeBeforeRemark`：重新标记前先做Minor GC，减少重新标记时间
3. `ParallelRefProcEnabled`：并行处理引用对象，减少重新标记时间

### 4.6 CMS源码级细节

#### 4.6.1 并发标记的三色标记法

```
白色：未访问（GC后仍为白色 → 可回收）
灰色：已访问但引用的对象未完全扫描
黑色：已访问且引用的对象已完全扫描

并发标记过程：
1. GC Roots标记为灰色
2. 取出灰色对象，标记为黑色，其引用的对象标记为灰色
3. 重复步骤2，直到没有灰色对象
4. 剩余白色对象可回收
```

#### 4.6.2 写屏障维护并发标记

```
用户线程修改引用时：
ObjectA.field = ObjectB

写屏障（伪代码）：
void writeBarrier(Object obj, Object value) {
    if (isMarking && isWhite(value)) {
        markGray(value);  // 标记为灰色
    }
    obj.field = value;
}
```

---

## 五、G1（Garbage First）

### 5.1 概述

G1是JDK 7引入的收集器，**JDK 9成为默认GC**。它开创了"面向局部收集"和"可预测停顿时间"的设计思路，是**大堆内存（6GB+）的首选**。

> **设计目标**：在延迟可控的情况下，尽可能提高吞吐量，同时简化GC调优。

### 5.2 G1的核心创新：Region化

```
传统分代：
┌─────────────────────────────────────────┐
│  Young Gen (连续)  │  Old Gen (连续)     │
└─────────────────────────────────────────┘

G1 Region化：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ E  │ S  │ O  │ O  │ H  │ E  │ O  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ O  │ H  │ E  │ O  │ S  │ O  │ H  │ E  │
└────┴────┴────┴────┴────┴────┴────┴────┘

E = Eden    S = Survivor    O = Old    H = Humongous
```

**Region特点**：
- 每个Region大小1-32MB（2的幂次）
- 逻辑上分代，物理上不连续
- 大对象（>Region 50%）进入Humongous Region
- 最多约2048个Region（堆大小/Region大小）

### 5.3 G1的工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                    G1 执行流程                               │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐   │
│  │ 初始标记  │ → │ 并发标记  │ → │    混合回收           │   │
│  │ (STW)    │    │ (并发)   │    │ (Mixed GC)           │   │
│  │ 借用Minor│    │ 遍历对象图│    │ 回收整个Young + 部分Old│  │
│  └──────────┘    └──────────┘    └──────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 可选：Full GC（G1失败时退化为Serial Old）              │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**阶段详解**：

| 阶段 | 是否STW | 说明 |
|------|---------|------|
| **初始标记** | 是 | 标记GC Roots直接关联对象，借用Minor GC的STW |
| **并发标记** | 否 | 从GC Roots遍历对象图，计算存活率 |
| **最终标记** | 是 | 处理SATB（Snapshot-At-The-Beginning）记录的引用变化 |
| **筛选回收** | 是 | 根据回收价值排序，选择Region进行回收 |

### 5.4 G1的核心机制

#### 5.4.1 SATB（Snapshot-At-The-Beginning）

```
并发标记开始时：
┌─────────────────────────────────────────┐
│ 对象A → 对象B → 对象C                   │
└─────────────────────────────────────────┘

并发标记期间，用户线程修改：
┌─────────────────────────────────────────┐
│ 对象A → 对象C（B被断开）                 │
└─────────────────────────────────────────┘

SATB保证：
- 标记开始时存活的对象，本次GC不回收
- 新分配的对象标记为存活
- 被断开的对象B：如果标记开始时存活，本次不回收（浮动垃圾）
```

#### 5.4.2 记忆集（Remembered Set / RSet）

```
问题：如何判断老年代对象引用新生代对象？
解决：每个Region维护一个RSet，记录"谁引用了我"

┌─────────┐         ┌─────────┐
│ Old R1  │ ──────► │ Young R2│
│         │  引用    │  RSet   │
└─────────┘         └─────────┘

RSet更新：通过写屏障（Write Barrier）在引用赋值时维护
```

#### 5.4.3 回收价值排序

```
G1根据每个Region的：
1. 存活对象比例（越低越好）
2. 回收耗时预估
3. 用户设定的目标停顿时间

选择"垃圾最多"的Region优先回收 → "Garbage First"名称由来
```

### 5.5 G1的核心参数

```bash
# 启用G1（JDK 9+默认）
-XX:+UseG1GC

# 目标停顿时间（默认200ms）
-XX:MaxGCPauseMillis=200

# Region大小（1-32MB，2的幂次）
-XX:G1HeapRegionSize=16m

# 老年代使用阈值（默认45%，触发并发标记）
-XX:InitiatingHeapOccupancyPercent=45

# 混合回收时，老年代Region回收比例（默认10%）
-XX:G1MixedGCCountTarget=8

# 堆内存预留比例（默认10%，防止to-space溢出）
-XX:G1ReservePercent=10

# 大对象阈值（默认Region的50%）
-XX:G1HeapWastePercent=5

# 并发标记线程数
-XX:ConcGCThreads=4

# 并行GC线程数
-XX:ParallelGCThreads=8
```

### 5.6 G1的优缺点

**优点**：
- 可预测的停顿时间（`-XX:MaxGCPauseMillis`）
- 无内存碎片（Region间复制）
- 大堆内存表现优异（6GB+）
- 自动调优，参数简单

**缺点**：
- 小堆内存（<4GB）不如CMS
- 内存占用高（RSet、卡表等额外开销约10-20%）
- 并发标记阶段占用CPU
- 混合回收时STW时间可能超预期

### 5.7 G1调优实战

```bash
# 典型G1配置（JDK 11+，4C8G）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=40
-XX:G1ReservePercent=15
-XX:+ParallelRefProcEnabled
-XX:+AlwaysPreTouch

# 大堆配置（32GB+）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=32m
-XX:InitiatingHeapOccupancyPercent=35
-XX:G1MixedGCCountTarget=16
```

**调优要点**：
1. `MaxGCPauseMillis`：设置过小→回收频率高；过大→单次回收时间长
2. `G1HeapRegionSize`：大对象多→调大；小对象多→调小
3. `InitiatingHeapOccupancyPercent`：过低→频繁并发标记；过高→Full GC风险
4. 避免显式设置`-Xmn`，让G1自动管理新生代大小

### 5.8 G1源码级细节

#### 5.8.1 G1的Region状态转换

```
Eden Region:
  分配对象 → 满 → Minor GC → 存活对象复制到Survivor → 清空

Survivor Region:
  存放存活对象 → Minor GC → 年龄达标晋升到Old → 清空

Old Region:
  存放长期存活对象 → 并发标记 → 混合回收 → 清空

Humongous Region:
  存放大对象 → 并发标记 → 混合回收 → 清空
```

#### 5.8.2 G1的并发标记周期

```
1. 初始标记（STW）：标记GC Roots直接关联对象
2. 并发标记：遍历对象图，计算存活率
3. 最终标记（STW）：处理SATB队列
4. 筛选回收（STW）：选择Region，复制存活对象
```

#### 5.8.3 G1的混合回收（Mixed GC）

```
混合回收 = Minor GC + 部分Old Region回收

回收流程：
1. 回收所有Young Region（Eden + Survivor）
2. 根据回收价值排序，选择部分Old Region
3. 复制存活对象到新Region
4. 清空旧Region

关键参数：
- G1MixedGCCountTarget：混合回收次数（默认8）
- G1OldCSetRegionThresholdThreshold：单次混合回收Old Region上限
```

---

## 六、ZGC（Z Garbage Collector）

### 6.1 概述

ZGC是JDK 11引入的**低延迟**收集器，JDK 15转正，JDK 21引入分代ZGC。**目标：TB级堆内存，暂停时间<1ms，且堆越大暂停时间越稳定**。

> **设计哲学**：牺牲少量吞吐量（约15%），换取极致的低延迟。

### 6.2 ZGC的核心创新

#### 6.2.1 染色指针（Colored Pointers）

```
传统GC：对象头存储标记信息
ZGC：指针本身存储GC信息（64位系统）

64位指针布局：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 0  │ 0  │ 0  │ M0 │ M1 │ R  │ F  │
└────┴────┴────┴────┴────┴────┴────┴────┘
  42-44位    │    │    │    │
             │    │    │    └─ Finalizable（是否需finalize）
             │    │    └────── Remapped（是否已重映射）
             │    └─────────── Marked1（标记位1）
             └──────────────── Marked0（标记位0）

优势：
1. 无需访问对象头即可判断对象状态
2. 指针自愈：GC后无需更新对象，只需更新指针
3. 支持并发重映射
```

#### 6.2.2 读屏障（Load Barrier）

```java
// 用户代码
Object obj = field.get();  // 读取引用

// ZGC插入的读屏障（伪代码）
Object obj = field.get();
if (isBadColor(obj)) {     // 检查指针颜色
    obj = loadBarrier(obj); // 慢路径：修正指针
}
```

**作用**：
- 并发标记阶段：保证对象图遍历的正确性
- 并发转移阶段：自动修正引用，无需STW

#### 6.2.3 并发转移（Concurrent Relocation）

```
传统GC：STW阶段移动对象，更新所有引用
ZGC：并发移动对象，通过读屏障自动修正引用

转移过程：
1. 选择转移集（Collection Set）
2. 并发转移对象到新Region
3. 读屏障自动修正指向旧地址的引用
4. 旧Region清空后可复用
```

### 6.3 ZGC的工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                    ZGC 执行流程                              │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ 并发标记  │ → │ 并发预备  │ → │ 并发转移  │               │
│  │ (并发)   │    │ 重分配   │    │ (并发)   │               │
│  │          │    │ (STW极短)│    │          │               │
│  └──────────┘    └──────────┘    └──────────┘               │
│                                                             │
│  遍历对象图        选择转移集        移动对象+修正引用         │
│  标记存活对象      准备新Region      读屏障自动修正            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 可选：Full GC（极少触发，如内存严重不足）              │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**关键特性**：
- **并发标记**：与用户线程并发，无STW
- **并发预备重分配**：STW极短（<1ms），仅选择转移集
- **并发转移**：对象移动与用户线程并发，读屏障自动修正

### 6.4 ZGC的核心参数

```bash
# 启用ZGC（JDK 15+）
-XX:+UseZGC

# 分代ZGC（JDK 21+，推荐）
-XX:+UseZGC -XX:+ZGenerational

# 最大堆内存
-Xmx16g

# 并发GC线程数（默认CPU核数的1/8）
-XX:ConcGCThreads=2

# 软最大堆（ZGC会尝试保持堆大小低于此值）
-XX:SoftMaxHeapSize=12g

# 大对象阈值（默认Medium Page的3/4）
-XX:ZAllocationSpikeTolerance=2

# 启用大对象支持（JDK 23+）
-XX:+ZUncommit
```

### 6.5 ZGC的优缺点

**优点**：
- 极低延迟：暂停时间<1ms，且与堆大小无关
- 超大堆支持：最大16TB堆内存
- 无内存碎片：并发转移自动整理
- 吞吐量损失小：约15%（JDK 21分代ZGC后更小）

**缺点**：
- 吞吐量略低于G1（约10-15%）
- 内存占用高：需要额外空间存储染色指针元数据
- 小堆内存优势不明显
- JDK 11-14为非分代ZGC，Full GC代价大

### 6.6 ZGC调优实战

```bash
# 典型ZGC配置（JDK 21+，分代ZGC）
-XX:+UseZGC
-XX:+ZGenerational
-Xmx16g
-XX:SoftMaxHeapSize=12g
-XX:ConcGCThreads=4
-XX:+AlwaysPreTouch
-XX:+DisableExplicitGC

# 低延迟场景（暂停时间优先）
-XX:+UseZGC
-XX:+ZGenerational
-Xmx8g
-XX:SoftMaxHeapSize=6g
-XX:ConcGCThreads=2
-XX:ZAllocationSpikeTolerance=1.5
```

**调优要点**：
1. `SoftMaxHeapSize`：设置为Xmx的75-80%，给ZGC预留空间
2. `ConcGCThreads`：过多影响业务线程，过少影响回收效率
3. 避免显式`System.gc()`，使用`-XX:+DisableExplicitGC`
4. 分代ZGC（JDK 21+）显著降低Full GC频率

### 6.7 ZGC源码级细节

#### 6.7.1 染色指针的状态转换

```
Marked0 / Marked1：标记位，交替使用
  - 00：未标记
  - 01：Marked0（第一次标记）
  - 10：Marked1（第二次标记）
  - 11：已标记（最终状态）

Remapped：重映射位
  - 0：未重映射（指向旧地址）
  - 1：已重映射（指向新地址）

Finalizable：finalize位
  - 0：无需finalize
  - 1：需要finalize
```

#### 6.7.2 读屏障的实现

```java
// ZGC读屏障（伪代码）
Object loadBarrier(Object obj) {
    if (isColored(obj)) {
        if (isMarked(obj)) {
            return obj;  // 已标记，直接返回
        }
        if (isRemapped(obj)) {
            return obj;  // 已重映射，直接返回
        }
        // 慢路径：修正指针
        return slowPath(obj);
    }
    return obj;
}
```

#### 6.7.3 并发转移的实现

```
1. 选择转移集（Collection Set）
2. 并发转移对象到新Region
3. 读屏障自动修正指向旧地址的引用
4. 旧Region清空后可复用

关键数据结构：
- 转移表（Forwarding Table）：记录对象转移映射
- 读屏障：自动修正引用
```

---

## 七、三大收集器对比

### 7.1 核心指标对比

| 特性 | CMS | G1 | ZGC |
|------|-----|-----|-----|
| **目标** | 最短停顿 | 可预测停顿 | 极低延迟 |
| **算法** | 标记-清除 | 标记-整理（Region间复制） | 染色指针+并发转移 |
| **内存碎片** | 有（需整理） | 无 | 无 |
| **STW时间** | 10-100ms | 10-200ms | <1ms |
| **堆大小** | <4GB | 4GB-64GB | 8GB-16TB |
| **吞吐量** | 中 | 高 | 中高（损失~15%） |
| **内存开销** | 低 | 中（10-20%） | 中高 |
| **CPU开销** | 中 | 中 | 中高 |
| **JDK版本** | 5-14（已移除） | 7+（9+默认） | 11+（15+转正） |
| **适用场景** | 已淘汰 | 通用首选 | 低延迟/大堆 |

### 7.2 停顿时间对比

```
堆大小：8GB

CMS:  ████████████████████████████████████████ 50-200ms
G1:   ████████████████ 20-100ms
ZGC:  █ <1ms

堆大小：64GB

CMS:  ████████████████████████████████████████████████████████ 200-1000ms
G1:   ████████████████████████ 50-200ms
ZGC:  █ <1ms（几乎不变）

堆大小：512GB

CMS:  不支持
G1:   ████████████████████████████████ 100-500ms
ZGC:  █ <1ms（几乎不变）
```

### 7.3 吞吐量对比

```
吞吐量（越高越好）：

CMS:  ████████████████████████████ 85-90%
G1:   ████████████████████████████████ 90-95%
ZGC:  ██████████████████████████ 80-85%（JDK 21分代ZGC后提升）
```

---

## 八、选型决策指南

### 8.1 决策流程图

```
开始
  │
  ▼
堆内存 < 4GB？
  ├─ 是 → G1（或Parallel GC）
  │
  └─ 否 → 延迟敏感（P99 < 10ms）？
           ├─ 是 → 堆内存 > 16GB？
           │        ├─ 是 → ZGC
           │        └─ 否 → ZGC 或 G1
           │
           └─ 否 → 吞吐量优先？
                    ├─ 是 → G1 或 Parallel GC
                    └─ 否 → G1（通用首选）
```

### 8.2 场景化推荐

| 场景 | 推荐GC | 理由 |
|------|--------|------|
| **微服务（<4GB）** | G1 | 自动调优，参数简单 |
| **Web应用（4-16GB）** | G1 | 平衡延迟与吞吐量 |
| **大数据/缓存（16-64GB）** | G1 或 ZGC | G1成熟稳定，ZGC低延迟 |
| **实时交易（低延迟）** | ZGC | 暂停时间<1ms |
| **批处理/离线计算** | Parallel GC | 吞吐量优先 |
| **JDK 21+新项目** | 分代ZGC | 低延迟+高吞吐量 |
| **遗留系统（JDK 8）** | G1 | CMS已废弃，G1更优 |

### 8.3 版本化建议

| JDK版本 | 推荐GC | 备注 |
|---------|--------|------|
| JDK 8 | G1 | CMS已废弃，G1更稳定 |
| JDK 11 | G1 | ZGC实验性，生产慎用 |
| JDK 17 | G1 或 ZGC | ZGC已转正 |
| JDK 21+ | 分代ZGC | 低延迟+高吞吐量 |

---

## 九、GC日志与监控

### 9.1 开启GC日志

```bash
# JDK 8
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=20M

# JDK 9+
-Xlog:gc*:file=/path/to/gc.log:time,level,tags:filecount=5,filesize=20M
```

### 9.2 GC日志分析工具

| 工具 | 特点 | 适用场景 |
|------|------|----------|
| **GCEasy** | 在线分析，可视化 | 快速分析 |
| **GCViewer** | 开源，功能全面 | 深度分析 |
| **JProfiler** | 商业，集成IDE | 开发调试 |
| **Arthas** | 在线诊断 | 生产环境 |

### 9.3 关键监控指标

```bash
# 使用jstat监控GC
jstat -gcutil <pid> 1000 10

# 输出说明
S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
0.00  95.23  67.45  45.67  98.12  96.34   1234   12.345    12   1.234   13.579

# 关键指标：
# O (Old Gen使用率) > 70% → 关注
# FGC (Full GC次数) 增长快 → 内存泄漏或配置不当
# GCT (GC总时间) / 运行时间 > 10% → GC开销过大
```

---

## 十、生产问题排查实战

### 10.1 频繁Full GC

**现象**：FGC次数快速增长，服务周期性卡顿

**排查步骤**：
```bash
# 1. 查看GC日志，确认Full GC原因
grep "Full GC" gc.log | tail -20

# 2. 常见原因：
# - Metadata GC Threshold：Metaspace不足
# - Ergonomics：堆空间不足
# - System.gc()：显式调用
# - Allocation Failure：内存分配失败

# 3. 分析堆内存
jmap -histo:live <pid> | head -30

# 4. 导出堆快照
jmap -dump:format=b,file=heap.hprof <pid>
```

**解决方案**：
- 增大堆内存：`-Xmx`
- 增大Metaspace：`-XX:MaxMetaspaceSize`
- 禁用显式GC：`-XX:+DisableExplicitGC`
- 检查内存泄漏：MAT分析堆快照

### 10.2 内存泄漏

**现象**：Old Gen使用率持续增长，Full GC后不下降

**排查步骤**：
```bash
# 1. 确认Old Gen趋势
jstat -gcutil <pid> 5000 20

# 2. 导出堆快照
jmap -dump:format=b,file=heap.hprof <pid>

# 3. MAT分析
# - Dominator Tree：查看最大对象
# - Leak Suspects：自动检测泄漏
# - Path to GC Roots：查看引用链
```

**常见泄漏源**：
- ThreadLocal未清理
- 静态集合类持续增长
- 连接/流未关闭
- 监听器未注销

### 10.3 GC暂停时间过长

**现象**：P99延迟高，GC日志显示长暂停

**排查步骤**：
```bash
# 1. 查看GC暂停时间
grep "Pause" gc.log | awk '{print $NF}' | sort -n | tail -20

# 2. 分析暂停原因
# - 对象晋升过快：增大Survivor或调整晋升阈值
# - 大对象分配：调整Region大小或PretenureSizeThreshold
# - 引用处理慢：开启ParallelRefProcEnabled
```

**解决方案**：
- G1：调整`MaxGCPauseMillis`、`G1HeapRegionSize`
- ZGC：增大`ConcGCThreads`、调整`SoftMaxHeapSize`
- 通用：增大堆内存、优化对象分配

---

## 十一、面试高频问题

### 11.1 基础问题

**Q：CMS和G1的区别？**
- CMS：标记-清除，有碎片，已废弃
- G1：Region化，标记-整理，可预测停顿，JDK 9+默认

**Q：G1如何避免Full GC？**
- 及时并发标记（IHOP控制）
- 混合回收（Mixed GC）
- 预留空间（G1ReservePercent）
- 避免大对象（Humongous Region）

**Q：ZGC如何实现极低延迟？**
- 染色指针：指针存储GC信息
- 读屏障：并发修正引用
- 并发转移：对象移动与用户线程并发

### 11.2 进阶问题

**Q：G1的RSet如何维护？**
- 写屏障：引用赋值时维护
- 卡表：512字节粒度
- 并发 refinement 线程：批量处理

**Q：ZGC的染色指针如何实现？**
- 64位系统：指针高位存储标记
- 多重映射：不同视角访问同一内存
- 自愈：读屏障自动修正

**Q：如何选择GC？**
- 堆大小：小堆G1，大堆ZGC
- 延迟：低延迟ZGC，高吞吐Parallel
- 版本：JDK 8用G1，JDK 21+用分代ZGC

### 11.3 深入问题

**Q：CMS的并发标记如何保证正确性？**
- 三色标记法：白、灰、黑
- 写屏障：维护标记正确性
- 重新标记：修正并发期间的变动

**Q：G1的SATB如何工作？**
- 并发标记开始时记录对象图快照
- 写屏障记录引用变化
- 最终标记处理SATB队列

**Q：ZGC的读屏障性能开销如何？**
- 读屏障在每次引用读取时执行
- 快路径：检查指针颜色，无额外开销
- 慢路径：修正指针，开销较大
- 总体开销：约5-10%

---

## 十二、总结与最佳实践

### 12.1 选型总结

```
┌─────────────────────────────────────────────────────────────┐
│                    GC选型速查                                │
│                                                             │
│  JDK 8：  G1（CMS已废弃）                                    │
│  JDK 11： G1（默认）或 ZGC（实验性）                          │
│  JDK 17： G1 或 ZGC（已转正）                                │
│  JDK 21+：分代ZGC（低延迟+高吞吐量）                          │
│                                                             │
│  小堆（<4GB）：G1                                            │
│  中堆（4-16GB）：G1                                          │
│  大堆（16-64GB）：G1 或 ZGC                                  │
│  超大堆（>64GB）：ZGC                                        │
│  低延迟（P99<10ms）：ZGC                                     │
│  高吞吐量：Parallel GC                                       │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 调优最佳实践

1. **先监控后调优**：开启GC日志，分析后再调整
2. **避免过度调优**：G1/ZGC自动调优能力强，优先使用默认配置
3. **关注业务指标**：P99延迟、吞吐量、错误率
4. **压测验证**：调优后必须压测验证
5. **逐步调整**：每次只改一个参数，观察效果

### 12.3 通用配置模板

```bash
# G1通用配置（JDK 11+）
-XX:+UseG1GC
-Xms4g -Xmx4g
-XX:MaxGCPauseMillis=200
-XX:+ParallelRefProcEnabled
-XX:+AlwaysPreTouch
-XX:+DisableExplicitGC
-Xlog:gc*:file=/var/log/gc.log:time,level,tags:filecount=5,filesize=20M

# ZGC通用配置（JDK 21+）
-XX:+UseZGC
-XX:+ZGenerational
-Xms8g -Xmx8g
-XX:SoftMaxHeapSize=6g
-XX:+AlwaysPreTouch
-XX:+DisableExplicitGC
-Xlog:gc*:file=/var/log/gc.log:time,level,tags:filecount=5,filesize=20M
```

---

> **更新于 2026-09-28** | 持续补充中...
>
> 这篇文章是对JVM GC的系统性梳理，从基础原理到生产实践，适合不同经验水平的开发者。配合《Tomcat 架构原理与性能调优深度解析》中的JVM调优章节一起阅读效果更佳。