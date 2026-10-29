

---
layout:     post
title:      synchronized(this)、synchronized(类名.class) 和 synchronized(lock) 记录
pubDatetime: 2026-10-29T00:59:04.866Z
modDatetime: 2026-10-29T00:59:04.866Z
author:     henuhaigang
header-img: img/post-bg-algorithm.jpg
catalog: true
tags:
    - 算法
    - LeetCode
    
description: java锁
---
---

#  `synchronized(this)`、`synchronized(类名.class)` 和 `synchronized(lock)` 记录




我们把 `synchronized(this)`、`synchronized(类名.class)` 和 `synchronized(lock)` 放在一起，从 **锁的本质、锁定对象、并发范围、JVM 底层、实际场景、常见陷阱** 六个方面彻底梳理。

这三个写法在语法上很相似，但它们锁定的对象不同，因此决定了哪些线程会互相阻塞。

先记住最重要的一句话：

**`synchronized` 决定如何获取和释放监视器，而 `this`、`Class` 对象和 `lock` 决定大家竞争的是哪一把锁。**

# 一、先看三种写法的本质区别

## 1. `synchronized(this)`

```java
public class Counter {

    public void increment() {
        synchronized (this) {
            // 临界区
        }
    }
}
```

`this` 表示当前对象实例。

例如：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

那么：

```java
synchronized (c1) {
    // 临界区 A
}
```

与：

```java
synchronized (c2) {
    // 临界区 B
}
```

使用的不是同一个锁。

因此，两个线程分别锁住 `c1` 和 `c2` 时，**不会仅仅因为执行相同的代码，就互相阻塞**。

但如果两个线程锁定同一个实例：

```java
synchronized (c1) {
    // 临界区 A
}
```

以及：

```java
synchronized (c1) {
    // 临界区 B
}
```

它们就会竞争同一个监视器。

### 特点

- 锁的粒度：实例级别。
- 锁的范围：同一个实例。
- 不同实例：默认不互斥。
- 典型用途：保护实例字段、实例内部状态。
- 主要风险：外部代码可以获得实例引用，进而影响该实例上的锁竞争。

---

## 2. `synchronized (类名.class)`

```java
public class Counter {

    public void increment() {
        synchronized (Counter.class) {
            // 临界区
        }
    }
}
```

`Counter.class` 是代表 `Counter` 这个运行时类的 `Class` 对象。

假设：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

无论线程通过 `c1` 还是 `c2` 调用方法，只要它们进入的代码都锁定同一个 `Counter.class` 对象，就会竞争同一把锁。

例如：

```java
public class Counter {

    public void methodA() {
        synchronized (Counter.class) {
            System.out.println("A");
        }
    }

    public void methodB() {
        synchronized (Counter.class) {
            System.out.println("B");
        }
    }
}
```

即使：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

线程 A 调用：

```java
c1.methodA();
```

线程 B 调用：

```java
c2.methodB();
```

两个线程仍然需要竞争同一个类锁。

### 特点

- 锁的粒度：特定运行时类对应的 `Class` 对象。
- 锁的范围：所有使用同一个 `Class` 对象的同步代码。
- 不同实例：也会互斥。
- 典型用途：保护类级别的共享状态。
- 主要风险：锁的影响范围较大，而且其他代码也可以锁定这个公开可获得的 `Class` 对象。

注意：这里的“类级别”不是指所有类加载器加载的同名类。不同类加载器可以加载出不同的 `Class` 对象，因此可能对应不同的类锁。

---

## 3. `synchronized(lock)`

```java
public class Counter {

    private int count;

    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
}
```

这里的 `lock` 是专门创建的一个普通 Java 对象。

它没有特殊的加锁能力。它之所以能用于 `synchronized`，是因为 Java 允许使用任何非 `null` 对象引用作为内置锁的目标。

关键在于：

```java
private final Object lock = new Object();
```

- `private`：限制外部代码直接访问这个字段。
- `final`：避免这个字段被重新赋值为另一个对象。
- `new Object()`：提供一个独立的对象身份。

这样就可以将内部同步机制与 `Counter` 实例本身分离。

例如：

```java
Counter c1 = new Counter();

synchronized (c1) {
    // 锁定 c1 本身
}
```

这不会直接阻塞：

```java
c1.increment();
```

因为 `increment()` 锁定的是 `c1.lock`，不是 `c1`。

但如果其他代码获得了同一个 `lock` 对象的引用，它仍然可以竞争这把锁。因此，封装锁对象非常重要。

### 特点

- 锁的粒度：由你选择的对象决定。
- 锁的范围：所有使用同一个 `lock` 对象的同步代码。
- 不同锁对象：默认不互斥。
- 典型用途：保护内部共享状态、隔离外部锁竞争。
- 主要风险：锁对象泄露、锁对象被替换，或者不同方法使用了不同的锁来保护同一份状态。

---

# 二、三种锁的对比总表

| 对比维度 | `synchronized(this)` | `synchronized(Counter.class)` | `synchronized(lock)` |
|---|---|---|---|
| 锁定对象 | 当前实例 | `Counter` 的 `Class` 对象 | `lock` 指向的对象 |
| 锁的范围 | 同一个实例 | 同一个运行时类对应的 `Class` 对象 | 同一个锁对象 |
| 不同实例之间是否互斥 | 通常不互斥 | 使用同一 `Class` 对象时互斥 | 取决于是否共享同一个锁对象 |
| 是否需要创建额外锁对象 | 不需要 | 不需要 | 通常需要 |
| 能否用于保护实例字段 | 可以 | 可以，但会扩大竞争范围 | 可以 |
| 能否用于保护静态共享状态 | 可以，但不能仅靠它实现跨实例互斥 | 可以 | 可以，前提是所有相关访问都使用同一把锁 |
| 外部能否直接获得锁对象 | 通常能获得实例引用 | 通常能通过类名获取 | `private` 锁字段可以限制直接访问 |
| 能否控制多个资源使用不同的锁 | 可以，但需要不同实例或其他锁对象 | 通常不适合细分多个资源 | 可以创建多个独立锁对象 |
| 典型场景 | 保护实例内部状态 | 保护类级共享状态 | 封装内部同步机制 |

注意，表格中的“通常”很重要：真正决定互斥关系的，不是变量名字，也不是代码写在哪里，而是线程最终获取的是否为同一个对象的监视器。

---

# 三、用一个例子彻底理解：三个对象、三把锁

假设我们有下面的类：

```java
public class Demo {

    private final Object lock = new Object();

    public void lockThis() {
        synchronized (this) {
            System.out.println("锁住当前实例");
        }
    }

    public void lockClass() {
        synchronized (Demo.class) {
            System.out.println("锁住 Class 对象");
        }
    }

    public void lockObject() {
        synchronized (lock) {
            System.out.println("锁住内部 lock");
        }
    }
}
```

创建两个实例：

```java
Demo d1 = new Demo();
Demo d2 = new Demo();
```

此时存在以下几个锁对象：

```text
d1 ──────────────────── 实例锁 A
d2 ──────────────────── 实例锁 B

Demo.class ───────────── 类锁 C

d1.lock ──────────────── 内部锁 D
d2.lock ──────────────── 内部锁 E
```

在这个例子中：

- `d1` 和 `d2` 是两个不同的实例。
- `Demo.class` 是它们对应的同一个运行时类的 `Class` 对象。
- `d1.lock` 和 `d2.lock` 是分别在两个实例创建时初始化的不同对象。

因此，锁的竞争关系如下。

| 线程 A | 线程 B | 是否会因为同一把锁而互斥 |
|---|---|---|
| `synchronized(d1)` | `synchronized(d1)` | 会 |
| `synchronized(d1)` | `synchronized(d2)` | 不会 |
| `synchronized(Demo.class)` | `synchronized(Demo.class)` | 会 |
| `synchronized(d1)` | `synchronized(Demo.class)` | 不会 |
| `synchronized(d1.lock)` | `synchronized(d1.lock)` | 会 |
| `synchronized(d1.lock)` | `synchronized(d2.lock)` | 不会 |
| `synchronized(d1)` | `synchronized(d1.lock)` | 不会 |

这里的“不会”是指不会因为这两处同步使用了同一个监视器而互斥。它们仍可能因其他锁、线程调度、资源争用等原因产生等待。

另外，如果两个方法都使用 `synchronized(this)`，但一个方法通过 `d1` 调用、另一个通过 `d2` 调用，那么它们锁定的不是同一个对象。

**理解 Java 内置锁，最重要的是跟踪锁对象的引用，而不是只看方法名。**

---

# 四、一个非常容易踩坑的问题：实例方法锁和类锁能不能互斥？

例如：

```java
public class Counter {

    private int count;

    public synchronized void increment() {
        count++;
    }

    public static synchronized void reset() {
        // 重置静态状态
    }
}
```

有人可能会认为：既然它们都在 `Counter` 类中，那么执行时应该互斥。

其实不是。

实例同步方法：

```java
public synchronized void increment()
```

使用当前实例，也就是 `this`。

静态同步方法：

```java
public static synchronized void reset()
```

使用 `Counter.class` 对象。

它们是两把不同的锁。

因此，即使线程 A 正在执行：

```java
counter.increment();
```

线程 B 也可以同时执行：

```java
Counter.reset();
```

只要没有其他同步关系阻止它们。

如果你需要这两个方法互斥，就必须明确设计共同的锁。

例如：

```java
public class Counter {

    private static final Object LOCK = new Object();

    private int count;

    public void increment() {
        synchronized (LOCK) {
            count++;
        }
    }

    public static void reset() {
        synchronized (LOCK) {
            // 重置状态
        }
    }
}
```

现在两个方法都竞争同一个 `LOCK`。

但这里还有一个重要的设计问题：如果 `count` 是每个实例独立拥有的状态，却让所有实例共享一把静态锁，就可能造成不必要的竞争。

因此，**只有在业务确实需要跨实例互斥时，才应该考虑使用静态共享锁。**

---

# 五、JVM 底层有什么区别？

这部分与你之前研究的 `monitorenter`、`monitorexit` 直接相关。

先看代码：

```java
synchronized (this) {
    // A
}
```

```java
synchronized (Demo.class) {
    // B
}
```

```java
synchronized (lock) {
    // C
}
```

对于同步代码块，编译器会生成相应的字节码，典型形式包括：

```text
monitorenter
    ...
monitorexit
```

这三种写法在同步语义上没有本质区别。

区别主要在于：**执行 `monitorenter` 时，操作数栈上提供的是哪个对象引用。**

- `this`：当前实例引用。
- `Demo.class`：对应的 `Class` 对象引用。
- `lock`：锁字段当前指向的对象引用。

JVM 根据该对象执行监视器进入操作。

如果一个线程已经持有这个监视器，再次进入时可以重入；如果其他线程持有它，则当前线程需要等待获取。

所以不能认为：

- `this` 使用一种特殊的锁算法。
- `.class` 使用另一种特殊的锁算法。
- `new Object()` 使用第三种锁算法。

它们使用的是 Java 内置锁机制，区别主要是锁定的对象身份。

## 5.1 三种写法对应的字节码有什么不同？

例如：

```java
public void lockThis() {
    synchronized (this) {
        work();
    }
}
```

```java
public void lockClass() {
    synchronized (Demo.class) {
        work();
    }
}
```

```java
public void lockObject() {
    synchronized (lock) {
        work();
    }
}
```

从概念上看：

| Java 表达式 | 获取对象引用的来源 | 进入的监视器 |
|---|---|---|
| `this` | 当前实例引用 | 当前实例关联的监视器 |
| `Demo.class` | 类字面量对应的 `Class` 对象 | 该 `Class` 对象关联的监视器 |
| `lock` | 读取字段获得的对象引用 | 该对象关联的监视器 |

它们最终都会落实为对特定对象监视器的同步操作。

对于 `synchronized` 方法，情况稍有不同：实例同步方法通常通过 `ACC_SYNCHRONIZED` 方法标志表达同步语义，静态同步方法也使用该标志，但其锁定目标不同。JVM 在方法调用和返回过程中处理监视器的进入与退出，而不是要求方法体内一定出现显式的 `monitorenter` 和 `monitorexit`。

---

# 六、应该如何选择？给你一套实战规则

不要死记“实例用 `this`，静态用 `.class`，内部用 `Object`”。这只是初步规则，真正应该从共享状态和同步边界出发。

## 场景一：保护当前对象的实例字段

```java
public class Account {

    private int balance;

    public synchronized void deposit(int amount) {
        balance += amount;
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

**优先考虑实例锁。**

因为账户余额属于当前实例，而相关方法都使用同一个实例的监视器。

如果不希望外部代码锁定账户对象影响内部方法，则可以考虑专门的内部锁。

## 场景二：保护静态共享状态

```java
public class IdGenerator {

    private static long id;

    public static long nextId() {
        synchronized (IdGenerator.class) {
            return ++id;
        }
    }
}
```

类锁能够让所有通过同一个 `IdGenerator.class` 对象获取锁的代码互斥。

也可以使用专门的静态锁：

```java
public class IdGenerator {

    private static final Object LOCK = new Object();

    private static long id;

    public static long nextId() {
        synchronized (LOCK) {
            return ++id;
        }
    }
}
```

第二种方式可以把同步对象与 `Class` 对象分离，降低其他代码意外竞争类锁的可能性。

## 场景三：一个类有多个独立资源

```java
public class ResourceManager {

    private final Object userLock = new Object();
    private final Object orderLock = new Object();

    public void updateUser() {
        synchronized (userLock) {
            // 更新用户资源
        }
    }

    public void updateOrder() {
        synchronized (orderLock) {
            // 更新订单资源
        }
    }
}
```

如果这两项操作不需要共同维护一个跨资源的不变量，就可以使用不同的锁。

这样更新用户与更新订单时，不会仅仅因为使用了同一个监视器而互相阻塞。

不过，如果一个业务操作同时需要修改用户和订单状态，就需要重新考虑多个锁的获取顺序，以及如何保证跨资源状态的一致性。

## 场景四：只需要一个简单原子操作

```java
private final java.util.concurrent.atomic.AtomicInteger count =
        new java.util.concurrent.atomic.AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

对于单个计数器的原子递增，通常不需要为了这个操作专门设计一把 `synchronized` 锁。

但如果需要保证多个操作组成一个整体，例如“检查库存并扣减库存”，仍然需要考虑整个业务过程的原子性。

---

# 七、最后给你一个架构师视角的总结

这三种写法并不是三种不同的锁机制，而是**三种不同的锁对象选择方式**。

```java
synchronized (this)
```

含义是：

> 当前实例上的相关操作需要竞争同一个实例监视器。

```java
synchronized (Demo.class)
```

含义是：

> 使用同一个运行时类对应的 `Class` 对象作为共同监视器。

```java
synchronized (lock)
```

含义是：

> 使用指定的对象作为监视器，并可以通过封装控制这把锁的使用范围。

实际做设计时，可以按以下顺序思考：

1. **保护的是什么共享状态？** 是实例字段、静态字段，还是多个资源？
2. **哪些操作必须互斥？** 同一实例、所有实例，还是某几个特定操作？
3. **是否允许外部代码影响内部锁竞争？** 如果不希望如此，可以考虑 `private final Object lock`。
4. **锁的范围是否过大？** 不必要的全局锁或类锁会造成额外竞争。
5. **是否需要更高级的同步能力？** 如果需要超时、公平策略或多个条件队列，可以考虑 `ReentrantLock`。

**最值得记住的一点：锁的设计首先是共享状态和并发边界的设计，其次才是选择 `this`、`.class` 还是 `new Object()` 的语法问题。**