---
title: Java 面试题（深度版）
tags:
  - 面试
  - Java
  - 后端
  - 深度
created: 2026-07-19
---

# Java 面试题（深度版）

> 每题包含：核心原理 → 关键源码/细节 → 常见追问 → 代码示例/项目映射。面试官追问时可逐层展开。

---

## 一、JUC 并发

### 1. synchronized 与 ReentrantLock 的全部区别，以及锁升级全过程

**原理对比**

| 维度 | synchronized | ReentrantLock |
|---|---|---|
| 实现 | JVM 内置（monitorenter/monitorexit） | JDK API（AQS） |
| 释放 | 自动（异常也会释放） | 必须 `finally` 手动 `unlock()` |
| 中断 | 不可中断 | `lockInterruptibly()` 可响应中断 |
| 超时 | 不支持 | `tryLock(time)` |
| 公平 | 非公平 | 可选公平/非公平 |
| 条件 | 1 个 waitSet | 多个 `Condition`，可分组等待 |
| 锁状态 | 无法查询 | `isLocked()`/`getHoldCount()`/`hasQueuedThreads()` |

**synchronized 锁升级（HotSpot）**

1. 无锁：对象头 Mark Word 无线程 ID。
2. 偏向锁：首次进入时 CAS 写入线程 ID 到 Mark Word；后续同线程无 CAS，直接进入（虚拟偏向）。
3. 轻量级锁：第二个线程竞争 → 撤销偏向 → 在栈帧建 Lock Record，CAS 把 Mark Word 拷入并指向 Lock Record；自旋等待。
4. 重量级锁：自旋超阈值或多个线程同时自旋失败 → Mark Word 指向 monitor 对象（ObjectMonitor），线程进入 `_EntryList` 阻塞，owner 释放后唤醒。

**关键细节**

- 升级不可逆；偏向锁一旦撤销进入轻量级，就不会再回退。
- JDK 15（JEP 374）废弃偏向锁，JDK 18 默认关闭，面试提一句体现跟进。
- `hashCode()`/`wait()` 会强制膨胀到重量级锁（需要 monitor）。
- 锁消除（逃逸分析）+ 锁粗化（`append` 连续调用合并）是 JIT 优化。

**追问**

- Q：偏向锁为什么被废弃？ A：维护成本高、现代多核 CPU CAS 成本下降，收益变低。
- Q：自旋锁默认多少次？ A：JDK 6 自适应自旋（`-XX:PreBlockSpin` 早期固定，现由历史成功率动态决定）。

**代码**

```java
// ReentrantLock 公平 + 多 Condition
ReentrantLock lock = new ReentrantLock(true); // 公平
Condition notFull  = lock.newCondition();
Condition notEmpty = lock.newCondition();
lock.lockInterruptibly();
try { ... } finally { lock.unlock(); }
```

---

### 2. AQS 原理，ReentrantLock 如何基于 AQS 实现

**AQS 核心**

- `volatile int state`：同步状态，CAS 修改。
- CLH 变种的双向队列：`Node` 含 `prev/next/waitStatus/THREAD`。
- `Node.waitStatus`：CANCELLED=1、SIGNAL=-1（后继需唤醒）、CONDITION=-2、PROPAGATE=-3。
- 模板方法：`tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared` 由子类实现，`acquire/release` 由 AQS 排队与阻塞。

**ReentrantLock 非公平加锁**

```java
// NonfairSync.tryAcquire
final boolean nonfairTryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) { // 抢占式 CAS
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;              // 重入
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

- 公平锁区别：`c==0` 时先 `hasQueuedPredecessors()` 判断队列有人则不抢。
- 释放：`state` 减到 0 才真正释放，唤醒 `head.next`（`unparkSuccessor`）。

**追问**

- Q：为什么 CLH 要改成双向？ A：取消节点需要让前驱找到新的后继，取消可能发生在队中，双向便于回溯。
- Q：AQS 入队前为什么先尝试两次 CAS 加锁？ A：减少队列操作开销，无竞争时直接成功。
- Q：ReentrantReadWriteLock 如何用一个 state 表示两个锁？ A：高 16 位共享（读），低 16 位独占（写）。

---

### 3. 线程池 7 参数 + 提交流程 + 拒绝策略 + 合理配置 + 常见陷阱

**7 参数**

`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`。

**提交流程（`execute` 源码要点）**

```
workerCountOf < corePoolSize        → addWorker(command, true)
否则入队列 workQueue.offer(command)  → 成功则返回
失败且 workerCountOf < maximumPoolSize → addWorker(command, false)
否则 reject(command)                 → 走拒绝策略
```

注意：**队列满**才会创建非核心线程（不是先到 max 再入队），这是最常被问错的点。

**拒绝策略**

- `AbortPolicy`：抛 `RejectedExecutionException`（默认）。
- `CallerRunsPolicy`：让提交线程自己跑，天然反压 + 降低提交速率。
- `DiscardPolicy`：静默丢弃。
- `DiscardOldestPolicy`：丢队列最老任务，再 `execute`。

**配置经验**

- CPU 密集：`N+1`；IO 密集：`N × (1 + 等待/计算)`，或 Brian Goetz 公式 `N × U × (1 + W/C)`。
- 实际：先压测取 QPS/P99 平衡点，配合动态参数（美团 `Hippo4j` / `Dynamic-TP`）。
- 队列选型：有界 `ArrayBlockingQueue` 防止 OOM；`SynchronousQueue` 配合 `CachedThreadPool`；`PriorityBlockingQueue` 任务带优先级。

**陷阱**

- `Executors.newFixedThreadPool` 用 `LinkedBlockingQueue`（无界）→ OOM 风险，阿里规约禁用。
- `CachedThreadPool` max = `Integer.MAX_VALUE` → 创建过多线程 OOM。
- `allowCoreThreadTimeOut(true)` 让核心线程也超时回收。

**追问**

- Q：线程池如何保证线程不退出？ A：核心线程 `take()` 阻塞，非核心线程 `poll(keepAliveTime)` 超时返回 null 退出。
- Q：线程池满了但队列还没满为什么是入队而不是创建新线程？ A：设计上优先复用线程 + 限制并发，避免线程数膨胀。

---

### 4. volatile 的语义、内存屏障、与 happens-before

**两语义**

1. 可见性：写后立即刷新主存，读强制从主存加载（JMM 变量刷新）。
2. 有序性：禁止重排序，靠内存屏障（StoreStore/StoreLoad/LoadLoad/LoadStore）。
3. **不保证原子性**：`i++` 仍是读-改-写三步。

**底层（x86）**

- 写 volatile：`lock addl $0,0(%esp)`（锁总线/缓存行）+ StoreLoad 屏障。
- 读 volatile：LoadLoad + LoadStore 屏障。
- 缓存一致性靠 MESI 协议 + Store Buffer + Invalidate Queue。

**happens-before 8 条**

程序顺序、监视器锁、volatile、线程 start、线程终止、线程中断、对象初始化完成、传递性。

**追问**

- Q：`volatile` 数组能保证可见性吗？ A：对**引用**可见，但对元素读写不保证，需 `AtomicIntegerArray`。
- Q：DCL 单例为什么要 volatile？ A：`new` 是「分配→初始化→赋值」，可能重排成「分配→赋值→初始化」，别人拿到未初始化对象。volatile 禁止重排。

```java
public class Singleton {
    private static volatile Singleton instance;
    public static Singleton get() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```

---

### 5. ThreadLocal 原理、内存泄漏、InheritableThreadLocal

**结构**

- 每个 `Thread` 持有 `ThreadLocalMap`。
- `ThreadLocalMap` 的 Entry 是 `WeakReference<ThreadLocal>` + value。
- key 弱引用：`ThreadLocal` 实例无强引用时 key 被 GC，value 仍被 Entry 强引用 → 泄漏。

**泄漏条件**

线程长期存活（线程池）+ ThreadLocal 实例被回收 → Entry.key=null 但 value 还在。

**防范**

- 用完 `remove()`（finally 里）。
- ThreadLocal 设为 `static final`，让 key 不被回收，value 也能被访问清理。
- `set/get/remove` 时会顺带清理 key==null 的 Entry（启发式），但不可依赖。

**InheritableThreadLocal**

- 子线程能继承父线程的值，靠 `Thread.init` 时复制 parent 的 map。
- **线程池场景失效**（线程复用，只在创建时继承一次）→ 用阿里 `TransmittableThreadLocal`（TTL）。

**项目映射**：风控平台用 ThreadLocal 存请求 traceId，线程池场景换 TTL。

---

### 6. ConcurrentHashMap 1.7 vs 1.8，分段锁与 CAS+synchronized

**1.7**

- `Segment[]` + `HashEntry[]`，Segment 继承 ReentrantLock。
- 默认 16 段，并发度 = 段数，扩容只扩段内。

**1.8**

- `Node[]` + 链表/红黑树，锁单个桶 `Node`（`synchronized` 锁头节点）。
- 写：空桶 CAS 插入；非空 synchronized 锁头节点。
- 树化阈值：链表 ≥ 8 且数组 ≥ 64；退化阈值 ≤ 6。
- `size`：`baseCount` + `CounterCell[]` 减少 CAS 竞争，最终累加。

**追问**

- Q：为什么 1.8 改 synchronized？ A：锁粒度更细（桶）、JDK 6 后 synchronized 优化好、节省内存（不用 Segment）。
- Q：为什么树化阈值 8？ A：泊松分布，负载因子 0.75 下桶长 ≥ 8 概率约 1e-8，红黑树空间换时间。
- Q：`get` 全程不加锁？ A：Node.val/vol `volatile`，next 指针 volatile，读可见；扩容时 `ForwardingNode` 转发到新表。

---

## 二、JVM

### 7. JVM 运行时数据区与对象内存布局

**数据区**

| 区域 | 线程 | 内容 | OOM 类型 |
|---|---|---|---|
| 堆 | 共享 | 对象/数组 | `OutOfMemoryError: Java heap space` |
| 方法区/元空间 | 共享 | 类元信息、常量池（1.7 移到堆）、JIT 代码 | `Metaspace` OOM |
| 栈 | 私有 | 栈帧（局部变量表、操作数栈、动态链接、返回地址） | `StackOverflowError` / OOM |
| 本地方法栈 | 私有 | native 调用 | 同上 |
| PC 寄存器 | 私有 | 当前指令地址 | 唯一不会 OOM |

- 直接内存：NIO `ByteBuffer.allocateDirect`，靠 `-XX:MaxDirectMemorySize`。

**对象布局**

- 对象头：Mark Word（锁状态、hash、GC 年龄）+ Klass Pointer（压缩指针 4 字节）+ 数组长度（数组才有）。
- 实例数据：字段重排，相同宽度放一起，父类在前。
- 对齐填充：8 字节对齐。
- 工具：`jol`（Java Object Layout）打印。

---

### 8. 对象创建全过程 + 指针碰撞 vs 空闲列表

1. 类加载检查（常量池类符号 → 是否加载/解析/初始化）。
2. 分配内存：
   - 指针碰撞：内存规整（Serial/ParNew），指针向前移动对象大小。
   - 空闲列表：内存碎片（CMS），维护可用块列表，找到合适块分配。
3. 内存空间初始化零值（不含对象头）。
4. 设置对象头（Mark Word、Klass Pointer、hash、GC 年龄）。
5. 执行 `<init>` 构造方法。

**线程安全分配**：CAS + 失败重试，或 TLAB（每线程预分配一块）。

---

### 9. GC 算法与收集器全谱

**算法**

- 标记-清除：碎片；适合老年代（CMS）。
- 复制：浪费空间；适合新生代存活少（Eden:S0:S1 = 8:1:1）。
- 标记-整理：移动对象，无碎片但 STW 久。
- 分代收集：新生代复制，老年代标记清除/整理。

**收集器**

| 收集器 | 区域 | 算法 | 特点 |
|---|---|---|---|
| Serial / Serial Old | 新/老 | 复制/整理 | 单线程，client 模式 |
| ParNew | 新 | 复制 | Serial 多线程版，配 CMS |
| Parallel Scavenge / Old | 新/老 | 复制/整理 | 吞吐量优先，`UseParallelGC` JDK 8 默认 |
| CMS | 老 | 标记-清除 | 低停顿，4 阶段（初始标记-并发标记-重新标记-并发清除），碎片化、Concurrent Mode Failure |
| G1 | 全堆 | Region + 标记-整理 | 停顿可控，混合回收，`-XX:MaxGCPauseMillis`，JDK 9 默认 |
| ZGC / Shenandoah | 全堆 | 染色指针/读屏障 | <10ms 停顿，TB 级堆 |

**G1 细节**

- 堆分 2048 个 Region（1-32MB），Eden/Survivor/Old/Humongous 动态。
- `Pause Prediction Model` 选回收集合（CSet），满足停顿目标。
- RememberedSet 记录跨 Region 引用，避免全堆扫描。
- Mixed GC：回收年轻代 + 部分老年代；Full GC 兜底（Serial Old，应避免）。

**ZGC 细节**

- 染色指针（64 位地址高位存 Marked0/Marked1/Remapped/Finalizable）。
- 读屏障：每次读对象时检查并转发，并发整理不停应用线程。
- 多重映射（mmap）让染色地址指向同一物理内存。

**追问**

- Q：CMS 什么时候退化 Serial Old？ A：Concurrent Mode Failure（并发阶段老年代满）或 Promotion Failed（晋升失败）。
- Q：G1 怎么解决跨代引用？ A：Remembered Set + 卡表（Card Table，512 字节粒度）。

---

### 10. GC 调优实战 + 线上排查流程

**调参常用**

```
-Xms4g -Xmx4g                  # 堆大小一致，避免动态扩缩抖动
-Xmn1g                         # 新生代
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-Xlog:gc*,gc+heap=debug:file=/log/gc.log:time,level,tags:filecount=10,filesize=50M   # JDK 9+
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/log/heap.hprof
```

**排查高 CPU**

```bash
top -Hp <pid>                 # 找高 CPU 线程 TID
printf "%x" <TID>             # 转 nid
jstack <pid> | grep <hex> -A 30   # 看线程栈
```

**排查内存泄漏**

```bash
jmap -histo:live <pid> | head -20     # 大对象 top
jmap -dump:live,format=b,file=h.bin <pid>
jstat -gcutil <pid> 1000            # 看 F 频率
```

MAT 看 Dominator Tree 找 GC Root 引用链。

**常见根因**

- 静态集合无限 add。
- ThreadLocal 未 remove（线程池）。
- 连接/流未关闭（finally close）。
- 监听器/回调未注销。
- `Integer` 缓存以外的缓存被强引用。

**项目映射**：产能统计系统 Kafka 消费堆积，调 `fetch.max.bytes` + 消费并发 + 监控 lag，发现是处理逻辑有同步阻塞调用 DB，改异步后下降。

---

### 11. 类加载机制 + 双亲委派 + 打破场景

**加载 7 阶段**：加载 → 验证 → 准备（静态变量赋零值）→ 解析（符号引用→直接引用）→ 初始化（`<cinit>` 赋真实值）→ 使用 → 卸载。

**双亲委派**

`ClassLoader.loadClass`：先 `parent.loadClass`，失败再 `findClass`。保证核心类（`java.*`）由 Bootstrap 加载，防止用户类冒充。

**打破场景**

| 场景 | 方式 | 原因 |
|---|---|---|
| Tomcat | 每个应用独立 `WebappClassLoader`，先自己后父 | 应用隔离、同 Tomcat 多应用 |
| JDBC | `Thread.contextClassLoader` | Bootstrap 加 `DriverManager`，但驱动在 classpath |
| OSGi | 网状委派 | 模块化 |
| 自定义 | 重写 `loadClass` | 想先自己加载 |

**Tomcat 细节**

- `CommonClassLoader` → `CatalinaClassLoader` → `SharedClassLoader` → `WebappClassLoader`（每应用一个）。
- Webapp 先查自己 `WEB-INF/classes|lib`，再委派父，违反双亲委派（避免应用类被父加载）。
- 但 `java.*` 仍走 Bootstrap，保证核心。

---

## 三、Spring

### 12. SpringBoot 自动装配完整链路

1. `@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。
2. `@EnableAutoConfiguration` → `@Import(AutoConfigurationImportSelector.class)`。
3. `selectImports` 调 `getAutoConfigurationEntry`：
   - 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（2.7+）。
   - 旧版 `META-INF/spring.factories`。
   - 去重 + `AutoConfigurationImportFilter` 过滤。
4. 每个配置类用 `@ConditionalOnXxx` 决定是否生效：
   - `@ConditionalOnClass` 类存在
   - `@ConditionalOnMissingBean` 容器没有该 Bean
   - `@ConditionalOnProperty` 配置项满足
   - `@ConditionalOnWebApplication` 是 Web 应用
5. `@AutoConfigureOrder`/`@AutoConfigureBefore`/`@AutoConfigureAfter` 控制顺序。

**自定义 starter**

```
my-starter
 ├─ META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
 │      写入 com.xxx.XxxAutoConfiguration
 └─ XxxAutoConfiguration 用 @ConditionalOnClass 等
```

**追问**

- Q：`@ConditionalOnMissingBean` 怎么判断？ A：`BeanTypeRegistry` + `ConfigurableListableBeanFactory`，启动早期用 `ConfigurationClassParser` 阶段的 bean 名集合。

---

### 13. Spring 事务失效场景 + 传播机制 + 原理

**失效**

1. 方法非 `public`（AOP 默认只代理 public）。
2. 同类自调用（`this.xxx()` 不走代理）。解决：注入自身 `@Lazy`、`AopContext.currentProxy()`、拆 Bean。
3. 异常被 catch 吞。
4. 抛受检异常但 `@Transactional` 默认只回滚 `RuntimeException`/`Error`，需 `rollbackFor = Exception.class`。
5. `final`/`static` 方法（无法代理）。
6. 传播 `NOT_SUPPORTED`/`NEVER`/`SUPPORTS` 无事务。
7. 数据库引擎不支持事务（MyISAM）。
8. 多线程跨方法调用（不同线程不同连接）。

**传播（7 种）**

| 行为 | 含义 |
|---|---|
| REQUIRED | 有就加入，无就新建（默认） |
| REQUIRES_NEW | 总是新建，挂起当前 |
| NESTED | 有就嵌套（保存点），无就新建 |
| SUPPORTS | 有就加入，无就非事务 |
| NOT_SUPPORTED | 非事务，挂起当前 |
| MANDATORY | 必须有，否则抛异常 |
| NEVER | 不能有，否则抛异常 |

**原理**

- `TransactionInterceptor.invoke` → `createTransactionIfNecessary` → `PlatformTransactionManager.getTransaction` → 执行 → `commit`/`rollback`。
- JDBC：`DataSourceTransactionManager` 维护 `ConnectionHolder`，绑定到 `ThreadLocal`，`autoCommit=false`，方法结束统一 commit。
- AOP：CGLIB/JDK 代理 + `TransactionAttributeSource` 解析注解。

---

### 14. AOP 两种代理 + 切点表达式 + 实战

**JDK 动态代理**

- 接口 + `InvocationHandler`，生成 `$Proxy0` implements 接口。
- 反射调用，性能稍差（早期）；JDK 8 后优化。
- 只能代理接口方法。

**CGLIB**

- 生成目标子类，`MethodInterceptor` 拦截。
- 不能代理 `final` 类/方法。
- 字节码生成，第一次稍慢但运行快。

**Spring 选型**

- 有接口默认 JDK 代理（旧版）；SpringBoot 2.x 默认 `proxyTargetClass=true`（CGLIB）。
- `@EnableAspectJAutoProxy(proxyTargetClass=true)` 强制 CGLIB。

**切点**

```java
@Pointcut("execution(* com.xxx.service..*.*(..))")     // 方法级别
@Pointcut("@annotation(com.xxx.MyLog)")                // 注解
@Pointcut("bean(userService)")                          // Bean 名
```

**通知**

```java
@Around("pointcut()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long s = System.currentTimeMillis();
    Object r = pjp.proceed();
    log.info("{} cost {}ms", pjp.getSignature(), System.currentTimeMillis()-s);
    return r;
}
```

**追问**

- Q：同类调用为什么 AOP 失效？ A：`this` 是目标对象本身，不是代理；代理只在外部调用时进入。
- Q：`@Async` 也是 AOP，同类调用同样失效。 A：是的，所有基于 AOP 的注解都失效。

---

### 15. Spring Bean 生命周期（含循环依赖）

**完整生命周期**

1. 实例化（`createBeanInstance`）。
2. 属性填充（`populateBean`，处理 `@Autowired`）。
3. 初始化（`initializeBean`）：
   - `BeanNameAware`/`BeanFactoryAware`/`ApplicationContextAware`
   - `BeanPostProcessor.postProcessBeforeInitialization`
   - `InitializingBean.afterPropertiesSet` / `@PostConstruct` / `init-method`
   - `BeanPostProcessor.postProcessAfterInitialization`（AOP 代理在此生成）
4. 使用。
5. 销毁：`@PreDestroy` / `DisposableBean` / `destroy-method`。

**三级缓存解决单例循环依赖**

- `singletonObjects`（一级）：成品单例。
- `earlySingletonObjects`（二级）：早期半成品（已实例化未初始化）。
- `singletonFactories`（三级）：ObjectFactory，用于生成早期引用（可能被代理）。

**流程**（A 依赖 B，B 依赖 A）

1. 创建 A，实例化后把 A 的 ObjectFactory 放三级缓存。
2. 填充 B 时创建 B，B 填充 A：从三级拿到 A 的 ObjectFactory → `getObject` 提前暴露（若需代理则提前生成代理）→ 放二级 → 返回 A 的早期引用。
3. B 完成放一级。
4. A 继续填充 B（一级有 B）→ A 完成放一级。

**为什么需要三级**

- 三级缓存存 ObjectFactory 而非直接引用：决定是否需要提前生成代理（AOP 时调用 `getEarlyBeanReference`），保证循环依赖中拿到的也是代理。
- 不能解决构造器循环依赖（实例化阶段就互相需要）。
- 不能解决多例循环依赖。

---

## 四、SpringCloud

### 16. Nacos 注册 + 配置双能力原理

**注册中心**

- 临时实例：客户端 5s 心跳，15s 标记不健康，30s 摘除。
- 永久实例：服务端主动 HTTP 探测。
- 一致性：
  - AP（Distro 协议）：临时实例，每个节点负责一部分数据，最终一致，节点间 gossip 同步。
  - CP（Raft）：永久实例，强一致，选举 leader。
- 客户端订阅：UDP push + 定时拉取兜底。

**配置中心**

- 客户端：`long polling` 29.5s 超时，服务端有变更立即返回，否则到点返回空。
- 本地快照：`${user.home}/nacos/config/fixed-${ip}/`，断网兜底。
- 命名空间 + Group + DataId 三级隔离。

**追问**

- Q：Nacos 2.x 改了什么？ A：gRPC 长连接替代 HTTP 长轮询，连接复用、推送更轻。
- Q：Nacos 集群最少几台？ A：3 台（Raft 过半数），生产推荐 3/5。

---

### 17. OpenFeign 调用全流程 + 超时重试陷阱

**流程**

1. `@EnableFeignClients` 扫描 `@FeignClient` 接口。
2. `FeignClientFactoryBean` 生成 JDK 代理。
3. 调用 → `ReflectiveFeign` → `SynchronousMethodHandler`。
4. `RequestTemplate` 构造请求 → `Client`（默认 `HttpURLConnection`，可换 OkHttp/Apache HttpClient）→ 负载均衡器选实例（`LoadBalancer`，旧版 Ribbon）。
5. `Decoder` 解析响应。

**超时**

- `feign.client.config.<name>.connectTimeout=2000`、`readTimeout=5000`。
- Spring Cloud LoadBalancer 默认无重试；Ribbon 旧版有 `MaxAutoRetries`、`MaxAutoRetriesNextServer`。

**陷阱**

- 重试叠加：Feign 重试 × LoadBalancer 重试 → 调用放大 N 倍，写接口慎用。
- GET 默认可重试，POST 默认不可（避免重复写）。

**集成 Sentinel**

```yaml
feign:
  sentinel:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 2000
        readTimeout: 5000
```

---

### 18. Gateway 核心模型 + 限流实现

**模型**

- Route = ID + URI + Predicate + Filter。
- Predicate：`After/Before/Between/Cookie/Header/Host/Method/Path/Query/RemoteAddr/Weight`。
- Filter：`AddRequestHeader/StripPrefix/Retry/SetStatus`，自定义 `GlobalFilter`/`GatewayFilter`。

**工作流**

`DispatcherHandler` → `RoutePredicateHandlerMapping` 匹配路由 → `FilteringWebHandler` 执行过滤器链 → 代理到下游。

**限流**

```java
spring:
  cloud:
    gateway:
      routes:
        - id: limit
          uri: lb://svc
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100   # 令牌/秒
                redis-rate-limiter.burstCapacity: 200   # 桶容量
                key-resolver: "#{@ipKeyResolver}"
```

- 实现：`RedisRateLimiter` Lua 脚本令牌桶，原子操作 + Lua 保证并发安全。
- 自定义 `KeyResolver`：按 IP / 用户 / 接口限流。
- 接 Sentinel：`@SentinelGatewayFilter`，规则中心化动态推。

---

## 五、MyBatis / MyBatis-Plus

### 19. MyBatis 一级/二级缓存 + 原理 + 陷阱

**一级缓存**

- `SqlSession` 级别，`PerpetualCache` 内 `HashMap`。
- 同一 session 内同查询返回同一对象（除非 `flushCache`）。
- `update/delete/insert` 清空整个 session 缓存。
- Spring 整合时每个请求一个 SqlSession（默认每次方法调用），一级缓存几乎失效。

**二级缓存**

- Mapper（namespace）级别，跨 session。
- 实体需 `Serializable`；写操作清整个 namespace 缓存。
- 多表 join 时缓存脏数据风险（A join B，B 改了 A 的缓存没失效）。
- 分布式下需接 Redis：`<cache type="org.mybatis.caches.redis.RedisCache"/>`。

**追问**

- Q：一级缓存失效场景？ A：不同 session、`flushCache`、`commit`、`localCacheScope=STATEMENT`。
- Q：为什么 Spring 下一级缓存几乎没用？ A：每次 Mapper 调用创建/复用不同 SqlSession，缓存不复用。

---

### 20. MyBatis-Plus 插件原理 + 实战

**插件机制**

- `Interceptor` 拦截 `Executor.query/update`、`StatementHandler.prepare`、`ResultSetHandler`。
- 基于 JDK 动态代理层层包装目标。
- `InnerInterceptor` 链（分页、乐观锁、防止全表操作）。

**分页**

```java
@Bean
public MybatisPlusInterceptor interceptor() {
    MybatisPlusInterceptor i = new MybatisPlusInterceptor();
    i.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    i.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return i;
}

// 用法
Page<User> p = userMapper.selectPage(new Page<>(1, 10), wrapper);
```

- 原理：`PaginationInnerInterceptor.beforeQuery` 改写 SQL 加 `limit`，并执行 `count(*)`。
- 大数据量 count 慢：`optimizeJoin=true` 优化 join 子查询；或手写 count。

**乐观锁**

```java
@Version
private Integer version;
// update user set ..., version=version+1 where id=? and version=?
```

**逻辑删除**

```java
@TableLogic
private Integer deleted;   // 0 未删 1 已删
```

---

## 六、Redis

### 21. Redis 快的根本原因 + 单线程模型 + 6.0 多线程

- 内存操作；单线程命令执行避免锁与上下文切换；IO 多路复用 epoll。
- 数据结构：SDS（O(1) length）、跳表（zset，平均 O(logN)，并发友好不用重平衡）、压缩列表（小 zset/hash/list）、quicklist（list，链表 + ziplist 节点）、intset（纯整数 set）。
- 6.0 多线程：仅网络读写/协议解析多线程，**命令执行仍单线程**，避免数据竞争 + 保留简单模型。
- pipeline 减少网络 RTT；Lua 保证原子批操作。

**追问**

- Q：为什么不用多线程执行命令？ A：内存操作瓶颈是网络/内存带宽，不是 CPU；多线程要加锁，反而慢。
- Q：6.0 多线程怎么开？ A：`io-threads 4`、`io-threads-do-reads yes`。

---

### 22. 缓存击穿/穿透/雪崩 + 布隆过滤器

**击穿**（热点 key 过期瞬间大量请求打到 DB）

- 互斥锁（Redisson）重建；逻辑过期（value 内带 expire，后台异步重建）；永不过期 + 主动更新。
- 限流降级兜底。

**穿透**（查不存在的 key）

- 缓存空值（短 TTL 5min），防恶意打同一不存在 key。
- 布隆过滤器：前置过滤，可能误判存在（假阳性），不漏报。
- 布隆原理：k 个 hash 映射到 bit 数组；查询全为 1 才可能存在；不能删除（需 Counting Bloom Filter）。

**雪崩**（大量 key 同时过期 / Redis 宕机）

- TTL 加随机（`base + random(0, 300s)`）。
- 多级缓存（本地 Caffeine + Redis + DB）。
- Redis 高可用集群 + 限流降级 + 熔断。

**追问**

- Q：布隆过滤器怎么扩容？ A：标准布隆不能扩容，需重建；或用 Scalable Bloom、Cuckoo Filter（支持删除）。

---

### 23. Redisson 分布式锁全细节

**加锁 Lua**

```lua
-- key=锁名 field=UUID:threadId value=重入次数
if (exists(key) == 0) then
    hset(key, field, 1); pexpire(key, 30000); return nil;
end;
if (hexists(key, field) == 1) then
    hincrby(key, field, 1); pexpire(key, 30000); return nil;
end;
return ttl(key);   -- 失败返回剩余时间
```

**看门狗**

- 加锁成功启动 `Timeout` 任务，每 10s（`lockWatchdogTimeout/3`）续期到 30s。
- 客户端宕机则不再续期，到期自动释放，防死锁。
- **注意**：`leaseTime` 显式设置则**不启动看门狗**，到点强制释放。

**释放**

- Lua 判断 field 再 `hincrby -1`，归零 `del`。
- 重入减计数，幂等。

**高可用**

- 主从切换可能丢锁：主写入未同步到从即宕机，从提升后锁丢失。RedLock 在多个独立实例过半加锁，但有争议（Martin Kleppmann 批评）。
- 生产实践：主从 + 业务幂等兜底，或用 Zookeeper/etcd 强一致锁。

**追问**

- Q：业务执行超过 30s 怎么办？ A：看门狗续期；或自己分段加锁；或拆分任务。
- Q：锁被别人释放怎么办？ A：field 含 UUID + threadId，释放时校验。

---

### 24. Redis Cluster 槽位 + 扩容 + 客户端路由

- 16384 槽，`CRC16(key) % 16384`。
- 节点间 Gossip 交换槽位分布。
- 客户端：键不在本节点返回 `MOVED <slot> <ip:port>` → 客户端更新路由表重试。
- 迁移中：源节点返回 `ASK` → 临时跳转不更新路由（迁移完成后才是 `MOVED`）。
- 批量（`mget`/pipeline）必须同槽，用 `{tag}` 让相关 key 落同槽：`user:{1000}:info`、`user:{1000}:order`。
- 扩容：`redis-cli --cluster reshard` 迁移 slot；`--cluster add-node` 加节点。

**追问**

- Q：为什么是 16384 不是 65536？ A：作者权衡：集群节点 ≤ 1000，心跳包大小 = 槽位/8 = 2KB，65536 会到 8KB 太大。

---

### 25. Redis 持久化 RDB/AOF/混合

**RDB**

- `bgsave` fork 子进程，COW（copy-on-write）写时复制，父进程继续服务。
- 体积小、恢复快；宕机丢数据 = 距上次快照时间窗。
- 大量写时父进程大量页 COW，内存可能翻倍。

**AOF**

- 追加命令；`appendfsync always`（每条，最安全最慢）/ `everysec`（默认，最多丢 1s）/ `no`。
- AOF 重写：fork 子进程，按当前内存状态生成最小命令集，期间新命令同时写 AOF 缓冲和重写缓冲，完成后追加。
- 文件大但恢复慢。

**混合（4.0+）**

- AOF 重写时基线用 RDB 全量 + 后续 AOF 增量，兼顾恢复速度与数据安全。
- `aof-use-rdb-preamble yes`。

---

## 七、RabbitMQ

### 26. 消息可靠投递三段 + confirm + return

**生产者**

- `channel.confirmSelect()` 开启 confirm，Broker 写入持久化队列后回 `ack`，否则 `nack`。
- 持久化：exchange `durable=true`、queue `durable=true`、message `deliveryMode=2`。
- `mandatory=true` + ReturnListener：路由不到队列时回调，避免消息丢失。
- 备份交换器（`alternate-exchange`）兜底。

**Broker**

- 镜像队列（旧版）/ 仲裁队列（Quorum Queue，3.8+，基于 Raft，推荐）保证高可用。

**消费者**

- 手动 `ack`；处理完再 `basicAck`。
- 异常 `basicNack` + `requeue=false` → 死信队列；或重试 N 次后转死信。
- 幂等：业务唯一键 + Redis `setnx`/DB 唯一索引。

**幂等实现**

```java
String msgId = msg.getMessageProperties().getMessageId();
if (!redisTemplate.opsForValue().setIfAbsent("mq:" + msgId, "1", 24, HOURS)) {
    return; // 已处理
}
// 业务...
```

---

### 27. 死信队列 + 延迟队列实现

**死信触发**

1. `basicNack/reject` 且 `requeue=false`。
2. 消息 TTL 过期（队列 `x-message-ttl` 或单条 `expiration`）。
3. 队列长度超限 `x-max-length`。

**DLX 绑定**

```java
Map<String,Object> args = Map.of("x-dead-letter-exchange", "dlx", "x-dead-letter-routing-key", "key");
channel.queueDeclare("biz.queue", true, false, false, args);
```

**延迟队列**

- TTL + DLX：消息进 biz.queue 设 TTL，过期转 DLX，消费者订阅 DLX 即延迟消费。
- 陷阱：队列 TTL 是按队头消息过期，后续消息再等；单条 expiration 不会按单条过期（需后入先出场景用插件）。
- 推荐：`rabbitmq_delayed_message_exchange` 插件，或 Redis Zset 时间戳，或专业 RocketMQ/TimeWheel。

---

### 28. 削峰 + 限流 + 预取

- 削峰：上游全量入队，消费者按能力拉。
- `basic.qos prefetchCount`：单消费者未 ack 上限，控制流量。
- 消费并发：Spring `setConcurrentConsumers` / `setMaxConcurrentConsumers`。
- 配合 Sentinel 限流；时间窗口/令牌桶。

**令牌桶 vs 漏桶**

- 令牌桶（Guava `RateLimiter`）：匀速产生令牌，桶满丢弃；允许突发（攒令牌）。
- 漏桶：恒定速率出水，超出排队或丢弃；强平滑。
- 网关限流多用令牌桶；下游保护用漏桶。

---

### 29. 幂等设计的几种方案 + 选型

| 方案 | 适用 | 缺点 |
|---|---|---|
| 唯一键约束 | DB 写入 | 不适用更新 |
| 乐观锁版本 | 更新 | 高并发失败重试多 |
| Redis setnx + TTL | 通用 | Redis 故障短暂失效 |
| 状态机校验 | 流程性强 | 业务耦合 |
| Token 机制 | 前端防重复提交 | 需先取 token |

**项目映射**：IOT 设备上报带 `msgId`，消费前 `SETNX mq:msgId 24h`，重复直接 ack 跳过。

---

## 八、MySQL

### 30. InnoDB 索引 B+ 树 + 聚簇/二级索引

**B+ 树**

- 非叶子只存索引，单页 ~1000 指针，3 层支持千万级数据。
- 叶子有序双向链表，范围扫描高效。

**聚簇索引**

- 叶子存完整行，按主键组织；一张表一个；默认主键，无主键用唯一非空，再无则隐式 `row_id`。
- 二级索引叶子存主键值（不是行指针），需**回表**。

**为什么主键建议自增**

- 顺序插入避免页分裂；UUID 随机插入频繁分裂、碎片、写放大。

**追问**

- Q：为什么不用 B 树？ A：B 树非叶子也存数据，扇出小、树高、IO 多；B+ 叶子链表范围快。
- Q：为什么不用红黑树/跳表？ A：磁盘 IO 代价高，需矮胖结构；B+ 每页 16KB 适配页大小。

---

### 31. 联合索引 + 最左前缀 + 索引下推 + 覆盖索引

`(a,b,c)`

- 命中：`a`、`a,b`、`a,b,c`、`a,b,c`（等值+范围都算）；`a,c` 部分（a 走索引，c 不走，但 ICP 过滤）。
- 范围字段后失效：`a=1 and b>5 and c=3`，c 用不到索引。

**索引下推（ICP，5.6+）**

- 旧版：存储引擎按索引取行 → server 层过滤 c，回表多次。
- ICP：在存储引擎用 c 过滤（c 在联合索引里），减少回表。

**覆盖索引**

- 查询字段都在索引里，不需回表：`select a,b,c from t where a=1`。
- `Extra: Using index`。

**索引失效**

- 函数运算 `where f(a)=1`；隐式转换 `where a='1'`（a 是 int 反而失效，`a=1` 才命中）；`like '%abc'`；`!=`/`<>`（统计低除外）；`or` 两侧不全有索引；`is null` 视情况。

---

### 32. 慢 SQL 排查 + EXPLAIN 各列

```sql
EXPLAIN select ...
```

| 列 | 关注点 |
|---|---|
| type | system > const > eq_ref > ref > range > index > ALL（至少 range，最好 ref） |
| key | 实际用的索引 |
| key_len | 索引使用长度，判断联合索引用了几列 |
| rows | 估算扫描行 |
| Extra | `Using index`（覆盖，好）；`Using where`（server 过滤）；`Using filesort`（额外排序，坏）；`Using temporary`（临时表，坏） |

**优化步骤**

1. 是否走索引（type、key）。
2. 行数是否过大（rows、统计信息 `analyze table`）。
3. 排序/分组是否用了索引（filesort/temporary）。
4. 大分页游标 `where id > last_id order by id limit n`。
5. 必要时强制/忽略索引 `force index/ignore index`。

**项目映射**：产能统计按产线 + 时间范围查询，加 `(line_id, create_time)` 联合索引，把 `Using filesort` 消掉。

---

### 33. 事务隔离 + MVCC + 锁

**四个隔离**

| 级别 | 脏读 | 不可重复读 | 幻读 |
|---|---|---|---|
| 读未提交 | ✓ | ✓ | ✓ |
| 读已提交（RC） | ✗ | ✓ | ✓ |
| 可重复读（RR，InnoDB 默认） | ✗ | ✗ | ✗（Next-Key Lock） |
| 串行化 | ✗ | ✗ | ✗ |

**MVCC**

- 行隐藏列：`DB_TRX_ID`（最近修改事务 id）、`DB_ROLL_PTR`（undo 链）。
- ReadView：`creator_trx_id`、`min_trx_id`、`max_trx_id`、`active_trx_list`。
- 可见性判断：
  - `trx_id == creator`：自己改的可见。
  - `trx_id < min`：已提交可见。
  - `trx_id >= max`：未来事务不可见。
  - `min <= trx_id < max`：在活跃列表里不可见，走 undo 找上一版本。
- RC：每次 SELECT 生成新 ReadView（看到最新提交）。
- RR：事务第一次 SELECT 生成，之后复用（看到的快照固定）。

**锁**

- 记录锁 Record Lock、间隙锁 Gap Lock、Next-Key Lock = Record + Gap（RR 防幻读）。
- 插入意向锁、意向共享/排他锁（表级提示）。
- 唯一索引等值命中：退化为记录锁（无间隙）。

**追问**

- Q：RR 怎么解决幻读？ A：Next-Key Lock 锁住间隙，其他事务插不进来；MVCC 快照读也看不到新行。
- Q：快照读 vs 当前读？ A：普通 select 快照读（MVCC）；`for update`/`lock in share mode`/update/delete 当前读（加锁看最新）。

---

### 34. 分库分表 + ShardingSphere + 跨库问题

**拆分**

- 垂直：按业务/字段，库按服务，表按字段（热/冷拆）。
- 水平：取模、范围、一致性 hash、基因法（避免后续拆分迁移）。

**跨库问题**

- 跨库 join：应用层组装、冗余字段、ES 宽表。
- 跨库事务：Seata AT（自动二阶段）/TCC（业务补偿）/SAGA；或最终一致（本地消息表 + MQ/事务消息）。
- 分布式 ID：雪花算法、号段（Leaf）、UUID（无序）。
- 分页深翻：流式归并、游标、二级索引（ES）。

**ShardingSphere**

- Sharding-JDBC（嵌入）/ Sharding-Proxy（独立代理）。
- 配置 `shardingRule` 表规则、`actual-data-nodes`、分库分表策略。
- 广播表（字典）、绑定表（关联表按相同 key 分）。

**项目映射**：产能统计按产线 id 取模分表 16 张，冷热数据按月归档到历史库。

---

## 九、消息与异步

### 35. Kafka vs RabbitMQ 全面对比 + 选型

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| 模型 | 分区 + 拉模式 | 队列 + 推模式 |
| 协议 | 自研 TCP | AMQP/MQTT/STOMP |
| 吞吐 | 百万/秒 | 万级/秒 |
| 顺序 | 分区内有序 | 单队列有序 |
| 路由 | 简单（topic/pattern） | 灵活（exchange/binding） |
| 延迟 | ms 级 | μs 级 |
| 持久化 | 顺序写盘 + 零拷贝 | 内存 + 磁盘 |
| 适用 | 日志、流式、大数据 | 业务消息、任务分发 |

**Kafka 高吞吐原因**

- 顺序写盘（机械盘也快）。
- 零拷贝 `sendfile`（页缓存 → 网卡）。
- 分区并行、批量发送、压缩（snappy/lz4/zstd）。
- 消费者组分区分配，水平扩展。

**Kafka 顺序消费**

- 同 key 进同分区保证顺序；业务键（订单 id）做 key。
- 单分区单消费者；并发消费需按 key 分桶到不同线程，每桶内顺序。

**项目映射**：产能统计每秒万级数据点，Kafka 顺序写 + 分区并行 + 消费者组。

---

### 36. Kafka 消息丢失与重复

**生产端丢失**

- `acks=0`：不等确认，网络丢则丢。
- `acks=1`：leader 写入即确认，leader 宕机未同步则丢。
- `acks=all(-1)` + `min.insync.replicas>=2`：leader + 副本都写入才确认，最安全。

**Broker**

- 副本机制，`unclean.leader.election.enable=false` 防止未同步副本当 leader 丢数据。

**消费端**

- 自动提交 offset 可能消费到一半未处理就提交 → 丢。
- 手动提交：处理完再 `commitSync`；幂等防重复。

**精确一次（EOS）**

- 幂等生产者（`enable.idempotence=true`，单分区去重）。
- 事务（`transactional.id`）跨分区原子。
- 消费端 `read-process-commit` 配合。

---

## 十、工程实战

### 37. Netty 线程模型 + 零拷贝 + 粘包

**Reactor 主从**

- BossGroup（1 个线程）`select` accept，把 channel 注册到 worker。
- WorkerGroup（N 个）轮询 IO 事件并处理。
- 业务重可单独抽出业务线程池（`addLast(handler, bizGroup)`），避免阻塞 IO 线程。

**零拷贝**

- `FileRegion`：`sendfile` 系统调用，文件 → 网卡不经用户态。
- `CompositeByteBuf`：逻辑合并多个 ByteBuf 无内存拷贝。
- `Unpooled.wrappedBuffer`、`ByteBuf.slice` 共享内存。
- `recvBuf` 直接内存（堆外），减少一次内核→用户拷贝（但要手动释放防泄漏）。

**粘包/拆包**

- `FixedLengthFrameDecoder` 固定长度。
- `LineBasedFrameDecoder` 换行分隔。
- `DelimiterBasedFrameDecoder` 自定义分隔符。
- `LengthFieldBasedFrameDecoder` 长度字段（最常用，自定义协议头）。

**追问**

- Q：Netty 解决空轮询 bug？ A：`Selector` 空轮询后重建新的 Selector，把 channel 迁移过去。

---

### 38. ELK 日志链路 + TraceId 串联

**架构**

- Filebeat（轻量采集）→ Kafka（削峰）→ Logstash（解析/Grok/geoip）→ ES（存储 + 倒排）→ Kibana（查询/可视化）。
- 或 Filebeat → ES 直出（7.x 后 ingest node）。

**TraceId**

- 网关生成 TraceId 放 MDC + HTTP header。
- 各服务从 header 取放 MDC（Logback `%X{traceId}` 输出）。
- 线程池用 TTL 传递。
- 跨服务串联：Spring Cloud Sleuth / SkyWalking 自动注入。

**ES 优化**

- 索引按天/服务切分，`index.lifecycle` 滚动。
- 字段类型映射，关键字段 `keyword`（不分词）。
- `refresh_interval=30s`（实时性换性能）。
- 冷热分离（SSD 热数据 + HDD 冷）。

---

### 39. Quartz vs XXL-JOB vs ElasticJob

| 项 | Quartz | XXL-JOB | ElasticJob |
|---|---|---|---|
| 部署 | 嵌入 | 中心 + 执行器 | 中心 + 执行器 |
| 集群 | DB 锁 | 注册中心注册 | ZK/Nacos |
| 调度可视化 | 无 | 有（强） | 有 |
| 分片 | 弱 | 广播分片 | 分片 |
| 适用 | 简单 | 通用业务 | 大数据量分片 |

**XXL-JOB 防重复**

- 中心调度 + 执行器注册 + DB 锁任务实例。
- 分片广播：`ShardingUtil.getShardingVo`，按 `id % total` 处理自己分片。
- 幂等业务兜底。

---

### 40. 项目综合：IOT 平台高并发数据上报设计

**接入**

- 设备 MQTT/TCP，Netty 长连接，鉴权 token，连接限流。
- 协议解析（微调后的 GLM4-9B）把报文转结构化字段。

**削峰**

- Kafka/RabbitMQ 异步落库，消费者限速。
- Redis 缓存设备最新状态（热数据），订阅推送。

**存储**

- 时序数据 InfluxDB/TDengine 或 MySQL 时间分表。
- 冷数据归档 OSS + Hive。

**联动**

- LiteFlow 编排规则引擎：事件触发 → 条件判断 → 动作下发（开门/告警）。
- 规则热更新 + 版本管理。

**监控**

- ELK 日志 + Prometheus + Grafana（连接数、消息 TPS、消费 lag）。
- 告警：设备离线、消费堆积、解析失败率。

---

### 41. Java 转 AI 的优势 + 风控平台 Java 侧贡献

**优势**

- 微服务、高并发、消息、缓存、可观测性 → 把模型真正落地成可靠系统。
- IOT 行业 + 数据 → 垂直领域微调/RAG。
- 既懂后端又懂部署，端到端交付。

**风控平台 Java 侧**

- 用户/角色/菜单/订阅管理：SpringCloud + MyBatis-Plus + RBAC。
- 鉴权：Spring Security + JWT（无状态）。
- 异步：RabbitMQ 爬虫任务分发 + 死信兜底；Redis 任务状态 + 限流。
- 审计：AOP 切面记录操作日志，敏感字段脱敏。
- 结果落 MySQL + ES 全文检索。
