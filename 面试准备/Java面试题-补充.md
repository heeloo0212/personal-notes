---
title: Java 面试题补充（基础 + 中间件）
tags: [面试, Java, MySQL, Redis, Spring, RabbitMQ, 补充]
created: 2026-07-22
---

# Java 面试题补充（基础 + 中间件，110+ 题）

> 配合 [[Java面试题]]（深度版）使用。本题库覆盖 Java 基础、MySQL、Redis、Spring/SpringBoot/Spring Cloud、RabbitMQ，每题附详细回答。

---

## 一、Java 基础（25 题）

### 1. == 和 equals 的区别？

- `==`：比较栈中值。基本类型比值，引用类型比内存地址（是否同一对象）。
- `equals`：Object 默认等价于 `==`；String/Integer 等已重写为比较内容。
- 重写 equals 必须同时重写 hashCode（HashMap/HashSet 依赖 hashCode 定位桶）。
- 规范：equals 相等 → hashCode 必相等；hashCode 相等 → equals 不一定相等。

### 2. String、StringBuilder、StringBuffer 区别？

- `String`：不可变（final char[]/byte[]），每次修改生成新对象，拼接低效。
- `StringBuilder`：可变，非线程安全，性能最好，单线程拼接首选。
- `StringBuffer`：可变，线程安全（synchronized），性能略差。
- JDK 9+ String 用 byte[] + coder（LATIN1/UTF16）省内存。
- 编译器对 `+` 拼接优化为 StringBuilder.append（循环内拼接每次新建 builder，应手动用 builder）。

### 3. String 为什么设计成不可变？

- 线程安全：多线程共享无需同步。
- 安全性：String 作类加载器参数、URL、文件路径，不可变防止被篡改。
- hashCode 缓存：不可变可缓存 hashCode，HashMap 查找快。
- 字符串常量池：可共享，节省内存。

### 4. String s = new String("abc") 创建了几个对象？

- 常量池已有 "abc"：堆中 1 个 String 对象。
- 常量池没有：常量池 1 个 + 堆 1 个 = 2 个。
- `s.intern()`：若常量池没有则把堆引用放入常量池（JDK 7+），返回常量池引用。

### 5. Integer 缓存机制？

- `Integer.valueOf` 对 -128~127 缓存（IntegerCache），返回同一对象。
- `Integer a = 127; Integer b = 127;` a == b 为 true；128 则 false。
- 自动装箱用 `valueOf`，拆箱用 `intValue`。
- 拆箱时若对象为 null 抛 NPE（常见坑：`Integer` 字段未赋值做比较）。

### 6. ArrayList 和 LinkedList 区别 + 扩容机制？

**区别**

| 维度 | ArrayList | LinkedList |
|---|---|---|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头插/中间插 | O(n) 移动 | O(1)（定位后） |
| 内存 | 连续，省 | 每节点多 prev/next |
| 适用 | 读多写少 | 频繁头尾增删 |

**ArrayList 扩容**

- 默认容量 10（首次 add 才分配）。
- 扩容 1.5 倍：`oldCapacity + (oldCapacity >> 1)`。
- `Arrays.copyOf` 复制到新数组。
- 预知大小用 `new ArrayList<>(cap)` 避免多次扩容。

### 7. HashMap 原理 + 扩容 + 线程不安全？

**1.8 结构**

- 数组 + 链表 + 红黑树。
- 哈希：`(h = key.hashCode()) ^ (h >>> 16)` 扰动，高 16 位参与运算。
- 定位：`(n-1) & hash`（n 是 2 的幂，等价取模更快）。
- 链表 ≥ 8 且数组 ≥ 64 树化；≤ 6 退化。

**扩容**

- 阈值 = capacity × loadFactor（默认 0.75）。
- 扩容 2 倍；元素重哈希：原位置 or 原位置 + oldCap（按 hash 最高位判断）。
- JDK 8 保留原链表顺序，避免 1.7 头插成环（死循环）。

**线程不安全**

- 并发 put 可能覆盖。
- 1.7 扩容头插成环 → get 死循环。
- 1.8 改尾插避免环，但仍可能丢数据。
- 用 `ConcurrentHashMap` 或 `Collections.synchronizedMap`。

### 8. HashMap 装载因子为什么是 0.75？

- 时间和空间折中：太小浪费空间，太大冲突多查询慢。
- 泊松分布下 0.75 让桶长 ≥ 8 概率极低（约 1e-8），配合树化阈值。
- 也可设 0.5/0.8 按场景调。

### 9. fail-fast 和 fail-safe 迭代器？

- fail-fast：遍历时被结构性修改（add/remove）抛 `ConcurrentModificationException`，靠 modCount 校验。ArrayList/HashMap 是。
- fail-safe：拷贝原集合遍历，不抛异常但可能看不到最新修改。CopyOnWriteArrayList/ConcurrentHashMap 是。
- 注意：fail-fast 不能 100% 保证，只是尽力检测。

### 10. Java 异常体系？

```
Throwable
 ├── Error（不该 catch：OOM、StackOverflow）
 └── Exception
      ├── RuntimeException（非受检，空指针/数组越界/类型转换）
      └── 其他（受检，必须 catch/throws：IOException/SQLException）
```

- 自定义异常继承 Exception（受检）或 RuntimeException（非受检）。
- try-with-resources 自动关资源（实现 AutoCloseable）。
- finally 不执行场景：`System.exit`、JVM 崩溃、当前线程被 kill、finally 内异常。

### 11. Java 反射 + 性能优化？

- `Class.forName` / `obj.getClass` / `Xxx.class` 获取 Class。
- `getDeclaredField/Method`（含 private）、`getField/Method`（public 含继承）。
- `setAccessible(true)` 突破访问控制。
- 性能：反射比直接调用慢 10-100 倍（JIT 后差距缩小）。
- 优化：缓存 Method/Field 对象；用 MethodHandle（JDK 7）或 LambdaMetafactory（JDK 8）生成动态调用，接近直接调用。
- 应用：Spring IoC、MyBatis、JSON 序列化、动态代理。

### 12. 动态代理 JDK vs CGLIB vs Javassist？

- JDK：接口 + InvocationHandler，`Proxy.newProxyInstance`，只能代理接口方法。
- CGLIB：生成子类，MethodInterceptor 拦截，不能代理 final；Spring AOP 默认。
- Javassist：直接操作字节码，性能介于，FastClass 索引优化。
- ByteBuddy（CGLIB 替代，Mockito/ByteBuddy 用）：更现代 API，Spring 6 已切换。

### 13. Java 泛型 + 类型擦除 + 通配符？

- 泛型编译期检查，运行期擦除（`List<String>` 运行时是 `List`）。
- 擦除导致：不能 `new T()`、不能 `T[].class`、不能 `instanceof List<String>`。
- 边界：`<T extends Comparable>` 上界（T 是 Comparable 子类）；`<? super T>` 下界。
- PECS 原则：Producer extends（读用 `<? extends T>`），Consumer super（写用 `<? super T>`）。

### 14. Lambda + 几大函数式接口 + Stream API？

**函数式接口**

- `Supplier<T>`：get()，产出。
- `Consumer<T>`：accept(t)，消费。
- `Function<T,R>`：apply(t)，转换。
- `Predicate<T>`：test(t)，断言。
- BiFunction/BiConsumer 等。

**Stream**

```java
list.stream()
    .filter(x -> x > 0)
    .map(x -> x * 2)
    .sorted()
    .distinct()
    .collect(Collectors.toList());
```

- 中间操作惰性（filter/map），终端操作触发（collect/forEach/count）。
- 并行流 `.parallel()` 用 ForkJoinPool.commonPool，注意线程安全和顺序性问题。
- 大数据用 forEachOrdered 保顺序；避免在并行流里修改共享状态。

### 15. Java IO/NIO/AIO？

- BIO：一连接一线程，阻塞，`ServerSocket.accept()` 阻塞。
- NIO：多路复用，Channel + Buffer + Selector，一个线程管多连接（Netty 基础）。
- AIO（NIO.2）：异步回调，Linux 用 epoll 模拟，Windows IOCP 原生；Netty 曾支持后移除（Linux 效果不理想）。
- 零拷贝：`FileChannel.transferTo` 用 sendfile。

### 16. 强软弱虚引用？

- 强引用：`Object o = new Object()`，普通引用，GC 不回收。
- 软引用（SoftReference）：内存不足才回收，适合缓存（图片缓存）。
- 弱引用（WeakReference）：下次 GC 必回收，ThreadLocalMap 的 key、WeakHashMap。
- 虚引用（PhantomReference）：随时回收，get 永远 null，配合 ReferenceQueue 跟踪对象被回收时机（资源清理）。

### 17. final/finally/finalize 区别？

- `final`：修饰类（不可继承）、方法（不可重写）、变量（不可重新赋值）。
- `finally`：try 块必执行（除特殊情况）。
- `finalize`：Object 方法，GC 回收前调用，JDK 9 废弃，不可靠（不保证执行时机/一定执行），用 Cleaner 替代。

### 18. 序列化机制 + serialVersionUID？

- `Serializable` 标记接口，默认序列化所有非 transient 字段。
- `transient` 不序列化；`static` 不序列化（属于类）。
- `serialVersionUID` 标记版本，反序列化比对，不一致抛 InvalidClassException；不写则编译器按类结构生成（改类即变）。
- 替代：JSON（Jackson/Fastjson）、Protobuf（跨语言、紧凑）、Kryo（高性能 Java）。
- 安全：反序列化漏洞（Fastjson 远程代码执行），用白名单 `ObjectInputFilter`（JDK 9+）。

### 19. Java 8-17 新特性？

- 8：Lambda、Stream、Optional、接口默认/静态方法、新日期 API（LocalDate/Instant/Duration）、CompletableFuture。
- 9：模块化 Jigsaw、JShell、私有接口方法、`List.of` 不可变集合。
- 10：`var` 局部类型推断。
- 11：HTTP Client、String API（strip/isBlank/lines）、var 用于 Lambda。
- 14：Switch 表达式（yield）、Records（预览）。
- 16：Records 正式、密封类（Sealed）。
- 17 LTS：密封类、模式匹配 instanceof、Records、强封装 JDK 内部 API。

### 20. Object 类有哪些方法？

- `equals`/`hashCode`：成对重写。
- `toString`：默认 `类名@十六进制hash`。
- `getClass`：反射入口，final。
- `clone`：浅拷贝，需实现 Cloneable，深拷贝要自己处理引用字段或序列化。
- `wait`/`notify`/`notifyAll`：需持有对象 monitor（synchronized 块内）。
- `finalize`：废弃。

### 21. 深拷贝 vs 浅拷贝？

- 浅拷贝：复制基本类型 + 引用地址，引用对象共享。
- 深拷贝：递归复制引用对象，完全独立。
- 实现：实现 Cloneable 递归 clone；序列化/反序列化（JSON、Kryo）；拷贝构造器。
- 集合的 `clone()` 是浅拷贝。

### 22. 内部类四种 + 访问限制？

- 静态内部类：不依赖外部实例，可访问外部静态成员。
- 成员内部类：持有外部实例引用，可访问外部所有成员；创建需 `outer.new Inner()`。
- 局部内部类：方法内定义，访问方法局部变量需 final（或事实 final）。
- 匿名内部类：实现接口/继承类的就地实例化，Lambda 简化单方法场景。

### 23. Java 中的 SPI 机制？

- Service Provider Interface：约定接口，第三方提供实现。
- `META-INF/services/接口全名` 文件写实现类全名。
- `ServiceLoader.load(接口.class)` 加载。
- 应用：JDBC Driver、SLF4J、Dubbo SPI（增强版，支持 IOC/AOP）、Spring Boot SPI。
- 与 Spring 区别：SPI 是 JDK 原生，无依赖注入；Spring Boot 自动装配是 SPI + 条件注解的扩展。

### 24. 注解的本质 + 元注解？

- 注解本质是继承 Annotation 接口的接口，编译后 JVM 在运行期通过反射读取（RetentionPolicy）。
- `@Retention`：SOURCE（编译期丢弃，如 @Override）、CLASS（class 文件，默认）、RUNTIME（运行期反射可读）。
- `@Target`：可标注位置（TYPE/METHOD/FIELD...）。
- `@Documented`/`@Inherited`/`@Repeatable`。
- 处理：运行期反射；编译期 Annotation Processor（Lombok/MapStruct）。

### 25. JVM 字节码层面理解方法调用？

- 方法调用指令：
  - `invokevirtual`：虚方法（动态分派，运行期查虚方法表）。
  - `invokeinterface`：接口方法（查 itable）。
  - `invokespecial`：构造器、private、super 调用（静态分派）。
  - `invokestatic`：静态方法。
  - `invokedynamic`：动态调用点（Lambda、String 拼接 JDK 9+）。
- 静态分派：编译期确定（重载，按静态类型）。
- 动态分派：运行期确定（重写，按实际类型查虚方法表）。

---

## 二、MySQL（25 题）

### 26. InnoDB 与 MyISAM 区别？为什么选 InnoDB？

| 维度 | InnoDB | MyISAM |
|---|---|---|
| 事务 | 支持 | 不支持 |
| 锁 | 行锁 + 表锁 | 表锁 |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | redo 恢复 | 易损坏 |
| 聚簇索引 | 是 | 非聚簇（索引/数据分离） |
| 全文索引 | 5.6+ 支持 | 支持 |

- InnoDB 行锁并发高、事务安全，是 5.5 后默认引擎，OLTP 必选。
- MyISAM 适合只读/插入为主的简单场景，已逐渐被淘汰。

### 27. InnoDB 三大日志：redo/undo/binlog？

- **redo log**（InnoDB 引擎层）：物理日志，记录「页的修改」，循环写，崩溃恢复保证持久性（crash-safe）。WAL 机制：先写 redo 再刷盘。
- **undo log**（InnoDB 引擎层）：逻辑日志，记录「反向操作」，用于回滚 + MVCC。
- **binlog**（Server 层）：逻辑日志，记录「SQL/行变更」，追加写，用于主从复制 + 数据恢复。

**两阶段提交（redo + binlog）**

1. 写 undo log（prepare）。
2. 写 redo log（prepare 状态）。
3. 写 binlog。
4. redo log 改 commit 状态。

保证 redo 与 binlog 一致，主从复制不丢/不错乱。

### 28. binlog 三种格式？

- `STATEMENT`：记 SQL，小但函数（now/uuid）+ 不确定语句导致主从不一致。
- `ROW`：记每行变更（前/后镜像），大但精确，主从一致（默认 5.7+）。
- `MIXED`：一般 STATEMENT，不确定时用 ROW。

### 29. MySQL 主从复制原理？

1. 主库写 binlog。
2. 从库 IO 线程拉取 binlog 写 relay log。
3. 从库 SQL 线程回放 relay log。
- 异步复制：主不等从，性能高但可能丢。
- 半同步复制：主等至少一个从收到 binlog 才返回，平衡。
- 组复制（MGR）：基于 Paxos，强一致多主。
- 延迟原因：从库单线程回放（5.7 并行复制基于组提交/写集合，8.0 并行更激进）、大事务、网络。

### 30. 读写分离 + 主从延迟解决方案？

- 读写分离：主写从读，ShardingSphere/MyCat 代理或应用路由。
- 延迟问题：
  - 关键写后立即读走主（强制路由主）。
  - 半同步复制降低延迟。
  - 从库并行复制。
  - 业务用缓存兜底刚写的数据。

### 31. 索引分类 + 联合索引最左前缀？

**类型**

- 主键索引（聚簇）、唯一索引、普通索引、联合索引、全文索引、前缀索引（字符串前 N 字符）。
- 覆盖索引：查询字段都在索引中，不回表。

**联合索引 `(a,b,c)`**

- 最左前缀：能命中 `a`、`a,b`、`a,b,c`；`a,c` 部分命中（c 走不到索引但 ICP 过滤）。
- 范围字段后失效：`a=1 and b>5 and c=3` 中 c 用不到。
- 排序：`order by b,c` 配合 `where a=?` 可走索引；`order by c` 用不到。

### 32. 聚簇索引 vs 二级索引 + 回表？

- 聚簇索引：叶子存完整行，按主键组织，一张表一个。
- 二级索引：叶子存主键值，查询非索引字段需回表（用主键查聚簇索引）。
- 覆盖索引避免回表：`select a,b from t where a=?`（(a,b) 联合索引）。
- InnoDB 主键建议自增整型，避免页分裂 + 索引体积小。

### 33. 索引失效场景？

- 函数运算 `where date(t)=...`、`where f(a)=1`。
- 隐式类型转换：`where a='1'`（a 是 varchar 反而失效？varchar vs int 字符串字段被数字比较才失效；字段是字符串但用数字查会触发隐式转换导致全表）。
- `like '%abc'`（左模糊）；`like 'abc%'` 可走索引。
- `!=`/`<>`/`not in`/`is null`（视统计信息）。
- `or` 两侧不全有索引。
- 联合索引违反最左前缀。
- 优化器认为全表更快（小表/统计信息不准）→ `analyze table` 修复。

### 34. EXPLAIN 各列含义？

| 列 | 含义 |
|---|---|
| id | 查询序号，越大越先执行 |
| select_type | SIMPLE/PRIMARY/SUBQUERY/DERIVED |
| type | 访问类型：const > eq_ref > ref > range > index > ALL |
| possible_keys | 可能用的索引 |
| key | 实际用的索引 |
| key_len | 索引使用字节数（判断联合索引用了几列） |
| ref | 哪个列或常量与 key 比较 |
| rows | 估算扫描行 |
| filtered | 过滤后比例 |
| Extra | Using index（覆盖）/filesort（额外排序）/temporary（临时表，差） |

### 35. 慢查询优化流程？

1. 开慢查询日志 `slow_query_log=on`、`long_query_time=1`。
2. `EXPLAIN` 看 type/key/rows/Extra。
3. 加合适索引（联合索引按区分度从高到低排）。
4. 消除 `Using filesort`/`Using temporary`：调整索引或改写 SQL。
5. 大分页游标 `where id > last_id limit n`。
6. `analyze table` 更新统计信息。
7. 避免索引失效写法。
8. 拆大事务、批量操作分批。

### 36. 大表分页优化？

- 深分页 `limit 1000000, 10` 扫描 100w 行再丢。
- 延迟关联：先 `select id from t where ... limit 1000000,10` 走覆盖索引，再 join 取详情。
- 游标分页：`where id > last_id order by id limit 10`（要求连续 id + 排序字段）。
- 业务限制最大页数。
- ES 二级索引 + 主库回取详情。

### 37. InnoDB 行锁、间隙锁、Next-Key Lock？

- Record Lock：锁单行。
- Gap Lock：锁间隙（不锁记录），防插入，RR 下解决幻读。
- Next-Key Lock = Record + Gap（左开右闭区间）。
- 唯一索引等值命中退化为 Record Lock（无间隙）。
- 插入意向锁：插入前获间隙锁失败则等待。
- 意向锁（IS/IX）：表级提示，加速判断表锁冲突。

### 38. 死锁排查 + 避免？

- 死锁：两个事务互相持有对方需要的锁。
- `SHOW ENGINE INNODB STATUS` 看最近死锁记录。
- `innodb_deadlock_detect=on` 自动检测回滚代价小的事务。
- 避免：
  - 固定加锁顺序（按主键升序）。
  - 事务尽量短。
  - 大事务拆小。
  - 用 RC 隔离 + 业务幂等替代部分锁。
  - 减少行锁范围（走索引，否则锁升级到表）。

### 39. 事务四大特性 ACID？

- 原子性（Atomicity）：undo log 回滚。
- 一致性（Consistency）：业务约束，由 A/I/D 保证。
- 隔离性（Isolation）：锁 + MVCC。
- 持久性（Durability）：redo log + doublewrite。

### 40. MVCC 原理 + ReadView？

- 行隐藏列 `DB_TRX_ID`（最近修改事务 id）、`DB_ROLL_PTR`（undo 链）。
- ReadView：`creator_trx_id`、`min_trx_id`（最小活跃）、`max_trx_id`（下一个分配）、`active_list`。
- 可见性：等于 creator 可见；小于 min 可见；大于等于 max 不可见；中间看是否在活跃列表。
- RC：每次 SELECT 新 ReadView。
- RR：事务第一次 SELECT 生成，后续复用。
- 优势：读不加锁，写不阻塞读（快照读）。

### 41. 当前读 vs 快照读？

- 快照读：普通 `select`，走 MVCC，不加锁，读历史版本。
- 当前读：`select ... for update`、`update`、`delete`、`insert`，读最新 + 加锁。
- RR 下快照读防不可重复读，当前读 + Next-Key Lock 防幻读。

### 42. MySQL 8 新特性？

- 隐藏索引（invisible index）：调试索引，不真正删除。
- 降序索引：`index(col desc)` 真正降序存储。
- 函数索引：`index((upper(name)))`。
- 通用 CTE（`WITH ... AS`）、窗口函数（`ROW_NUMBER`/`RANK`/`LAG`/`LEAD`）。
- 原子 DDL（DDL 失败不留残留）。
- 自增主键持久化（5.7 重启可能重置）。
- JSON 增强、角色管理。

### 43. 窗口函数 + CTE 用法？

**窗口函数**

```sql
SELECT name, dept, salary,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) rk
FROM emp;
-- ROW_NUMBER 不重复，RANK 跳号，DENSE_RANK 不跳号
-- LAG(salary,1,0) 上行，LEAD 下行
```

**CTE**

```sql
WITH high_salary AS (
  SELECT * FROM emp WHERE salary > 10000
)
SELECT dept, COUNT(*) FROM high_salary GROUP BY dept;
-- 递归 CTE：树形/层级查询
```

### 44. SQL 连接类型 JOIN？

- INNER JOIN：交集。
- LEFT JOIN：左全 + 右匹配，右无 null。
- RIGHT JOIN：右全。
- FULL JOIN：并集（MySQL 不支持，用 union left+right）。
- CROSS JOIN：笛卡尔积。
- SELF JOIN：自连接（同一表两别名）。
- 自然 JOIN：自动按同名列连接，少用。
- 执行：嵌套循环（NLJ）/ 哈希 join（8.0.18+）/ 排序合并。

### 45. MySQL 性能监控工具？

- `SHOW PROCESSLIST` / `SHOW FULL PROCESSLIST`：当前连接。
- `SHOW STATUS LIKE 'Innodb_%'`：引擎状态。
- `performance_schema`：等待事件、锁、IO 等细粒度。
- `sys` 库：友好视图（`sys.innodb_lock_waits`）。
- 慢查询日志 + pt-query-digest 分析。
- Prometheus + mysqld_exporter + Grafana 监控。

### 46. 分库分表后唯一 ID 怎么生成？

- UUID：无序，B+ 树插入差。
- 雪花算法（Snowflake）：时间戳 + 机器 + 序列号，趋势递增，时钟回拨问题。
- 号段模式（美团 Leaf）：DB 批量取号，本地缓存。
- Redis `INCR`：简单但依赖 Redis。
- Zookeeper/etch 持久顺序节点。
- 数据库自增 + 步长（分库后各库不同初始 + 步长）。

### 47. 数据库连接池 HikariCP 为什么快？

- 字节码精简，方法少。
- 用 FastList 替代 ArrayList，避免范围检查。
- ConcurrentBag 无锁获取连接（thread-local + 共享队列）。
- 严格管理连接生命周期，避免泄漏。
- 参数：`maximumPoolSize`、`minimumIdle`、`connectionTimeout`、`idleTimeout`、`maxLifetime`。
- 经验：池大小 = `(核心数 × 2) + 磁盘数`（Hikari 官方），不是越大越好（上下文切换 + DB 端连接压力）。

### 48. count(*)/count(1)/count(列) 区别？

- `count(*)`：统计行数，含 null，MySQL 8 优化（InnoDB 走最小索引扫描）。
- `count(1)`：类似 `count(*)`，几乎无差异。
- `count(列)`：统计该列非 null 数，慢（要解析列）。
- 大表 count 估算用 `show table status` 或维护计数表。

### 49. MySQL 中 JSON 类型 + 使用场景？

- 5.7+ 原生 JSON 类型，二进制存储，可索引（虚拟列 + 索引）。
- 函数：`JSON_EXTRACT`、`JSON_SET`、`JSON_ARRAY`、`JSON_OBJECT`、`->`、`->>`。
- 场景：半结构化数据、动态字段、配置存储。
- 不适合大量查询/统计（性能弱于规范化表）。

### 50. 千万级大表如何优化？

- 索引优化（联合索引 + 覆盖索引）。
- 分库分表（取模/范围）。
- 冷热分离（热数据 SSD + 冷数据归档）。
- 读写分离 + 从库分担读。
- ES 二级索引支撑复杂检索。
- 大字段拆分（垂直分表）。
- 归档历史数据（按时间分区 + 定时迁历史库）。
- 缓存（Redis）兜底热点查询。

---

## 三、Redis（20 题）

### 51. Redis 五大数据结构底层实现？

| 结构 | 编码（小数据） | 编码（大数据） |
|---|---|---|
| String | int（数字）/ embstr（≤44字节）/ raw | raw（SDS） |
| List | listpack（7.0）/ ziplist | quicklist |
| Hash | listpack/ziplist | hashtable |
| Set | intset（纯整数）/ listpack | hashtable |
| ZSet | listpack/ziplist | skiplist + hashtable |

- 转换阈值：listpack 节点数/单元素大小超限转大结构。
- 压缩编码省内存，访问 O(n)；超限转哈希/跳表，访问 O(1)/O(logN)。

### 52. ZSet 为什么用跳表而不是红黑树？

- 跳表实现简单（链表 + 多级索引），红黑树复杂。
- 范围查询（`ZRANGE`）跳表链表天然顺序遍历，红黑树需中序遍历。
- 内存换性能：跳表多层指针，空间约 1.5x；红黑树无额外指针但实现难。
- 跳表 + hashtable 双结构：hashtable O(1) 单点查，跳表 O(logN) 范围查。

### 53. Redis 过期策略 + 内存淘汰策略？

**过期**

- 惰性删除：访问时检查过期才删。
- 定期删除：每隔一段时间随机抽 key 检查删除（`hz` 频率）。
- 二者结合，平衡 CPU 和内存。

**淘汰（maxmemory-policy）**

- `noeviction`：不淘汰，写报错。
- `allkeys-lru`/`volatile-lru`：LRU（所有/设过期的）。
- `allkeys-lfu`/`volatile-lfu`：LFU（4.0+，频率）。
- `allkeys-random`/`volatile-random`：随机。
- `volatile-ttl`：优先淘汰快过期的。
- Redis LRU 是近似 LRU（采样 5 个比最近最少用），LFU 用 Morris 计数器衰减。

### 54. Redis 持久化 RDB/AOF/混合 + fork 影响？

（详见深度版 Q25，补充：）

- fork 大实例耗时（拷贝页表），停顿可达秒级，生产监控 `info stats latest_fork_usec`。
- 写多时 COW 触发频繁页拷贝，内存可能 2x。
- 大实例建议用 `SYNC` 子进程或 AOF 重写分时段。
- AOF 重写期间磁盘 IO + 内存双倍，注意错峰。

### 55. Redis 主从复制 + 全量/增量？

- 全量：主 `bgsave` 生成 RDB + 缓存期间的写命令 → 发给从 → 从加载 RDB + 回放命令。
- 增量：从记录 offset，主在 backlog（环形缓冲）找 offset 后的命令发送。
- 全量触发：初次/断线太久 backlog 丢失/主从 offset 不一致。
- 复制积压缓冲区：`repl-backlog-size`，越大越抗断线。

### 56. Redis 哨兵 Sentinel？

- 监控主从 + 自动故障转移 + 通知客户端。
- 主观下线（SDOWN）：单哨兵 ping 超时。
- 客观下线（ODOWN）：过半哨兵确认。
- 选举 leader 哨兵 → 选最优从晋升 → 通知其他从 + 客户端。
- 至少 3 节点奇数防脑裂。
- 客户端订阅哨兵 +sentinel 获取新主地址。

### 57. Redis Cluster vs Sentinel 选型？

- Sentinel：单主多从，主写从读，故障转移；适合数据量小、单主够用。
- Cluster：分片（16384 槽）+ 多主多从，水平扩展；适合大数据量、高并发写。
- Cluster 限制：跨槽事务/多键操作需 hashtag；`SELECT` 不可用；pub/sub 跨节点需 hashtag。

### 58. 缓存一致性策略：Cache Aside / Read Through / Write Through / Write Behind？

- Cache Aside（最常用）：读先查缓存，未命中查 DB 回写；写先更 DB 再删缓存。
- Read Through：缓存层封装 DB 读取。
- Write Through：写同时写缓存 + DB（同步）。
- Write Behind：写只写缓存，异步刷 DB（性能高但丢数据风险）。
- 延迟双删：写时先删缓存 → 更 DB → 延迟再删缓存，防旧数据回写。

### 59. 缓存与 DB 一致性 + 延迟双删 + Canal？

- 标准做法：更新 DB 后删缓存（而非更新缓存，避免并发覆盖）。
- 延迟双删：删 → 更新 DB → 延迟删，覆盖并发读回写。
- 终极方案：订阅 binlog（Canal）异步删缓存，最终一致 + 解耦。
- 强一致场景：分布式锁 + 串行化，牺牲性能。

### 60. 布隆过滤器 + 布谷鸟过滤器？

- 布隆：k 个 hash 映射到 bit 数组；查询全 1 才可能存在（假阳性不漏报）；不能删。
- 参数：误判率 p、元素数 n → 计算 bit 数 m 和 hash 数 k。
- 布谷鸟：两个 hash + 两个表，冲突踢出已有元素；支持删除、查询更快、空间更优。
- Redisson 内置 BloomFilter；RedisBloom 模块提供 CF/TopK/CountMin。

### 61. Redis 分布式锁的几种实现对比？

- `SETNX + EXPIRE`：两条命令非原子（旧陷阱），应用 `SET key val NX PX 30000`。
- value 用唯一 UUID：释放时 Lua 校验避免删别人锁。
- Redisson：可重入 + 看门狗 + 公平/红锁。
- RedLock：多实例过半数加锁，争议大（时钟漂移、GC STW）。
- 强一致场景用 Zookeeper/etcd（CP），Redis 是 AP 牺牲一致性换可用性。

### 62. Redis 大 key/热 key 处理？

**大 key**

- 定义：string > 10KB，list/hash/set/zset 元素多。
- 危害：阻塞（单线程）、网络阻塞、集群迁移卡顿、内存倾斜。
- 检测：`redis-cli --bigkeys`、`memory usage key`。
- 处理：拆分（按字段/分片）、压缩、异步删除 `UNLINK`。

**热 key**

- 危害：单节点 QPS 高、热点带宽。
- 检测：`hotkeys`、`MONITOR`、proxy 统计。
- 处理：多副本读分散、本地缓存（Caffeine）、按 key 分片到多节点。

### 63. Redis Pipeline + 事务 + Lua？

- Pipeline：客户端批量发命令，一次网络往返，减少 RTT，但非原子。
- 事务（MULTI/EXEC）：命令入队后顺序执行，不被打断；`WATCH` 乐观锁（CAS）；**不支持回滚**（某命令出错其余仍执行）。
- Lua：`EVAL` 单线程原子执行整脚本，适合复杂原子操作（限流、分布式锁、扣库存）。

### 64. Redis 限流（令牌桶）Lua 实现？

```lua
-- KEYS[1]=key, ARGV[1]=容量, ARGV[2]=速率, ARGV[3]=now(ms), ARGV[4]=请求量
local capacity=tonumber(ARGV[1])
local rate=tonumber(ARGV[2])
local now=tonumber(ARGV[3])
local req=tonumber(ARGV[4])
local data=redis.call("hmget",KEYS[1],"tokens","ts")
local tokens=tonumber(data[1]) or capacity
local ts=tonumber(data[2]) or now
local delta=math.max(0, now-ts)*rate/1000
tokens=math.min(capacity, tokens+delta)
if tokens < req then return 0 end
tokens=tokens-req
redis.call("hmset",KEYS[1],"tokens",tokens,"ts",now)
redis.call("pexpire",KEYS[1],60000)
return 1
```

### 65. Redis Stream + 消息队列？

- 5.0 新增，类似 Kafka，支持持久化、消费组、ACK。
- `XADD` 写、`XREAD` 读、`XREADGROUP` 消费组读、`XACK` 确认、`XPENDING` 未 ack、`XCLAIM` 转移。
- 优势：原生 Redis 无需额外组件；劣势：吞吐不如专业 MQ、无完善分区机制。
- 适合轻量级消息、事件流。

### 66. Redis 6 多线程 + RESP3？

- 6.0 IO 多线程：网络读写/协议解析多线程，命令执行仍单线程。
- 开启：`io-threads 4`、`io-threads-do-reads yes`。
- RESP3 协议：支持更多类型（map/set/push）、客户端缓存（`CLIENT TRACKING`）。
- ACL：6.0 用户权限隔离。

### 67. Redis 集群扩缩容 + 数据迁移？

- `redis-cli --cluster add-node` 加节点。
- `reshard` 迁移 slot：源节点 `MIGRATE` 单 key 到目标。
- 大 key 迁移注意耗时；建议低峰操作。
- 客户端 MOVED/ASK 自动重定向。
- 节点间 Gossip 传播集群拓扑。

### 68. Redis 内存优化技巧？

- 合理编码：小 hash/list/zset 用 ziplist/listpack。
- 键名缩短（业务前缀规范）。
- 用 hash 聚合多个字段（比多个 string 省）。
- 设置过期时间，避免冷数据堆积。
- 大对象压缩（gzip/snak e）。
- LRU/LFU 淘汰策略。
- 分片分散到多节点。

### 69. Redis 应用：排行榜/计数器/分布式锁/限流？

- 排行榜：ZSet `ZADD`/`ZREVRANGE`。
- 计数器：`INCR`/`INCRBY`，点赞/浏览量。
- 分布式锁：Redisson。
- 限流：令牌桶 Lua / 固定窗口 / 滑动窗口（ZSet 时间戳）。
- 去重：Set 或 HyperLogLog（近似 UV）。
- 地理位置：GEO（基于 ZSet）。

### 70. HyperLogLog + Geo？

- HyperLogLog：基数估算，12KB 估 12 亿去重，误差 0.81%，UV 统计。
- Geo：基于 ZSet + GeoHash，`GEOADD`/`GEORADIUS`/`GEODIST`，附近的人/店。
- BitMap：位图，签到/在线状态，1 亿用户约 12MB。

---

## 四、Spring（15 题）

### 71. Spring IoC 理解 + 容器启动流程？

- IoC：控制反转，对象创建/依赖由容器管理，解耦。
- DI：依赖注入，容器把依赖注入对象（构造器/setter/字段）。
- 启动：
  1. `BeanDefinitionReader` 读 XML/注解生成 `BeanDefinition`。
  2. `BeanFactoryPostProcessor` 修改 BeanDefinition（如 PropertyPlaceholder）。
  3. 实例化 Bean（反射）。
  4. 属性填充 + 初始化。
  5. 注册到单例池。

### 72. Bean 生命周期完整步骤？

1. 实例化（`createBeanInstance`）。
2. 属性填充（`populateBean`，处理 `@Autowired`）。
3. Aware 接口回调（`BeanNameAware`/`BeanFactoryAware`/`ApplicationContextAware`）。
4. `BeanPostProcessor.postProcessBeforeInitialization`（`@PostConstruct` 在此）。
5. `InitializingBean.afterPropertiesSet` / `init-method`。
6. `BeanPostProcessor.postProcessAfterInitialization`（AOP 代理在此生成）。
7. 使用。
8. `@PreDestroy` / `DisposableBean.destroy` / `destroy-method`。

### 73. 三级缓存解决循环依赖（详解）？

- 一级 `singletonObjects`：完整单例。
- 二级 `earlySingletonObjects`：早期半成品（可能已代理）。
- 三级 `singletonFactories`：ObjectFactory，按需生成早期引用。
- A 依赖 B、B 依赖 A：
  1. A 实例化 → 三级放 A 的 ObjectFactory。
  2. A 填充 B → 创建 B → B 填充 A → 三级 A.getObject() → 提前暴露（AOP 时生成代理）→ 放二级。
  3. B 完成放一级。
  4. A 拿到 B 完成初始化 → 放一级。
- 三级而非二级：决定是否提前生成代理（AOP），保证循环里拿到的也是代理对象。
- 不能解决：构造器循环依赖、多例循环依赖、`@Async` 类内调用（代理时机不同）。

### 74. Spring 事务传播行为 + 失效场景？

（深度版 Q13 已详述，补充：）

- `@Transactional` 标注在接口上：JDK 代理生效，CGLIB 不生效。
- `@Transactional` 加在 private 方法：AOP 不拦截。
- 异常被 catch 又抛同样异常：可生效；catch 后不抛则不回滚。
- 数据源多时事务管理器配置错误：跨数据源失效。

### 75. Spring AOP 术语 + 通知类型？

- JoinPoint：连接点（方法执行）。
- Pointcut：切点（匹配规则）。
- Advice：通知（Before/After/Around/AfterReturning/AfterThrowing）。
- Aspect：切面（切点 + 通知）。
- Weaving：织入（编译期 AspectJ / 运行期 Spring AOP）。
- Target：被代理对象。
- Proxy：代理对象。

### 76. @Autowired vs @Resource vs @Inject？

- `@Autowired`（Spring）：按类型注入，多实现配合 `@Qualifier` 按名；可标构造器/字段/setter。
- `@Resource`（JSR-250）：默认按名，找不到再按类型；`name`/`type` 属性。
- `@Inject`（JSR-330）：按类型，配合 `@Named` 按名。
- 推荐：构造器注入（不可变 + 显式依赖 + 易测），字段注入不推荐。

### 77. @Component/@Service/@Repository/@Controller 区别？

- 语义上分层标记，本质都是 `@Component`。
- `@Repository`：额外做异常转换（`PersistenceExceptionTranslationPostProcessor`）。
- `@Controller`：Spring MVC 控制器识别。
- `@Service`：业务层标记，无额外功能。
- 扫描：`@ComponentScan` 按包路径 + 过滤器。

### 78. Spring 事件机制 ApplicationEvent？

- 发布：`applicationContext.publishEvent(new MyEvent(...))`。
- 监听：`@EventListener` 或实现 `ApplicationListener`。
- 同步默认；`@Async` + `@EnableAsync` 异步。
- Spring 4.2+ 支持任意事件对象（不必继承 ApplicationEvent）。
- 用途：解耦（订单完成发事件 → 通知/积分/统计分别监听）。

### 79. Spring 扩展点有哪些？

- `BeanFactoryPostProcessor`：BeanDefinition 加载后修改（改属性/注册新 bean）。
- `BeanDefinitionRegistryPostProcessor`：注册 BeanDefinition（MyBatis Mapper 扫描）。
- `BeanPostProcessor`：Bean 实例化前后处理（AOP）。
- `InstantiationAwareBeanPostProcessor`：实例化前后 + 属性填充前后。
- `ApplicationContextAware`/`EnvironmentAware` 等 Aware 回调。
- `SmartInitializingSingleton`：所有单例初始化完成回调。
- `ApplicationListener`：事件监听。

### 80. Spring MVC 请求处理流程？

1. `DispatcherServlet.doDispatch`。
2. `HandlerMapping` 找到 `HandlerExecutionChain`（Controller + 拦截器）。
3. `HandlerAdapter` 适配执行 Controller。
4. 拦截器 preHandle。
5. Controller 执行返回 `ModelAndView`。
6. 拦截器 postHandle。
7. `ViewResolver` 解析 View 渲染。
8. 拦截器 afterCompletion。
- REST 用 `@RestController`，返回对象经 `HttpMessageConverter`（Jackson）转 JSON，跳过 ViewResolver。

### 81. @RequestMapping 处理 + 参数解析？

- `RequestMappingHandlerMapping` 启动时扫描 `@RequestMapping` 注册到 `HandlerMethod`。
- 参数解析：`HandlerMethodArgumentResolver`，如 `@RequestParam`/`@PathVariable`/`@RequestBody`/`@RequestHeader`。
- 返回值处理：`HandlerMethodReturnValueHandler`，对象 → `@ResponseBody` → Jackson。
- `@Valid` + `MethodArgumentNotValidException` 校验。

### 82. Spring 拦截器 vs Servlet 过滤器 vs AOP？

| 项 | 触发 | 范围 |
|---|---|---|
| Filter | Servlet 容器 | 所有请求（含静态） |
| Interceptor | Spring MVC | Controller 请求 |
| AOP | Spring Bean 方法 | 任意 Bean 方法 |
- Filter 在 DispatcherServlet 前；Interceptor 在 Handler 前后；AOP 在方法前后。
- 鉴权/日志/跨域多用 Interceptor；通用过滤（编码/安全头）用 Filter；业务横切用 AOP。

### 83. Spring SpEL 用法？

- 表达式语言，运行期求值。
- `@Value("#{'${list}'.split(',')}")`、`@Value("#{systemProperties['user.home']}")`。
- `@Cacheable(key = "#user.id")`、`@PreAuthorize("hasRole('ADMIN')")`。
- 支持：属性访问、方法调用、集合投影/筛选、正则、类型 `T(java.lang.Math)`。

### 84. Spring 多环境 Profile + 外部化配置？

- `@Profile("dev")` 标 Bean/配置类，`spring.profiles.active=dev` 激活。
- 配置优先级：命令行参数 > 系统属性 > 环境变量 > application-{profile}.yml > application.yml。
- `@PropertySource` 加载额外配置文件。
- 配置中心（Nacos/Apollo）运行期动态刷新（`@RefreshScope`）。

### 85. SpringFactoriesLoader + 自动配置加载？

- 读 `META-INF/spring.factories`，按 key 加载实现类。
- SpringBoot 自动装配基于此（2.7+ 改 `AutoConfiguration.imports`）。
- 启动时加载并按 `@Conditional` 过滤。
- 自定义 starter 同样机制。

---

## 五、SpringBoot（15 题）

### 86. SpringBoot 启动流程详解？

1. `SpringApplication.run`。
2. 创建 `SpringApplication` 实例：推断 Web 类型、读 `spring.factories` 的初始化器/监听器。
3. `run()`：
   - `SpringApplicationRunListener` 发布 starting。
   - 准备 Environment（加载 application.yml/命令行/环境变量）。
   - 创建 ApplicationContext（Servlet/Web/Reactive）。
   - `prepareContext`：注册主类、执行初始化器。
   - `refreshContext`：核心，触发自动配置 + Bean 实例化。
   - 发布 started/ready 事件。
4. 内嵌 Tomcat/Jetty/Netty 启动。

### 87. @SpringBootApplication 组成 + 自动装配链路？

- = `@SpringBootConfiguration`（配置类）+ `@EnableAutoConfiguration` + `@ComponentScan`。
- 自动装配：`AutoConfigurationImportSelector` → `META-INF/spring/...AutoConfiguration.imports` → `@ConditionalOnXxx` 过滤 → 注册生效的配置类。

### 88. SpringBoot 条件注解有哪些？

- `@ConditionalOnClass`/`@ConditionalOnMissingClass`：类存在/不存在。
- `@ConditionalOnBean`/`@ConditionalOnMissingBean`：Bean 存在/不存在。
- `@ConditionalOnProperty`：配置项满足（prefix/name/havingValue/matchIfMissing）。
- `@ConditionalOnWebApplication`/`@ConditionalOnNotWebApplication`。
- `@ConditionalOnExpression`（SpEL）、`@ConditionalOnJava`、`@ConditionalOnResource`。
- 自定义：实现 `Condition` + `@Conditional`。

### 89. SpringBoot 内嵌容器 + 切换？

- 默认 Tomcat；换 Jetty/Undertow/Netty（WebFlux）。
- 排除 Tomcat 依赖 + 引入对应 starter。
- 配置：`server.port`、`server.tomcat.max-threads`、`server.tomcat.accept-count`。
- Undertow 性能好内存低；Netty 配 WebFlux 响应式。
- 调优：max-threads（默认 200）、max-connections、accept-count、min-spare-threads。

### 90. SpringBoot 配置加载顺序 + yml/profile？

优先级（高→低）：
1. 命令行参数。
2. `SPRING_APPLICATION_JSON`。
3. ServletConfig/ServletContext。
4. JNDI。
5. Java 系统属性。
6. OS 环境变量。
7. `RandomValuePropertySource`。
8. jar 包外 `application-{profile}.yml`。
9. jar 包内 `application-{profile}.yml`。
10. jar 包外 `application.yml`。
11. jar 包内 `application.yml`。
12. `@PropertySource`。
13. 默认值。

### 91. SpringBoot Actuator + 端点？

- 生产级监控：`/actuator/health`、`/info`、`/metrics`、`/env`、`/loggers`、`/beans`、`/mappings`、`/threaddump`、`/heapdump`。
- 暴露：`management.endpoints.web.exposure.include=*`，安全限制只暴露 health/info。
- 整合 Prometheus：`micrometer-registry-prometheus`，`/actuator/prometheus` 出指标。
- 健康检查自定义 `HealthIndicator`。

### 92. SpringBoot 自定义 Starter 步骤？

1. 创建 `xxx-spring-boot-starter` 模块（依赖 `xxx-spring-boot-autoconfigure`）。
2. 写 `XxxAutoConfiguration` + `@ConditionalOnXxx`。
3. `XxxProperties` 用 `@ConfigurationProperties` 绑定前缀。
4. `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 写配置类全名。
5. 使用方引入 starter 即生效。

### 93. SpringBoot 热部署方案？

- `spring-boot-devtools`：类路径变化触发重启（类加载器分层，只重载应用类，快）。
- JRebel：商业，热更新更彻底。
- IDE 调试热替换：方法体内改可热替换，结构改不行。
- 生产不用热部署，靠滚动发布。

### 94. SpringBoot 打包 + 外部化配置部署？

- `spring-boot-maven-plugin` 打 fat jar，`java -jar` 启动。
- 内嵌 Tomcat，无需外部容器。
- 外部配置：`--server.port=8080` 命令行、`SPRING_PROFILES_ACTIVE` 环境变量、外置 `application.yml`（同目录/config 目录）。
- Docker：fat jar + Dockerfile + 多阶段构建。
- 优雅停机：`server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase`。

### 95. SpringBoot 日志体系？

- 默认 Logback（`spring-boot-starter-logging`）。
- 切换 Log4j2：排除默认 starter + 引 `spring-boot-starter-log4j2`。
- 配置 `logback-spring.xml`（用 `<springProfile>` 区分环境）。
- 默认级别 INFO；`logging.level.com.xxx=DEBUG`。
- 结构化日志：Logstash encoder 输出 JSON 喂 ELK。

### 96. SpringBoot 异步 + 定时 + 缓存注解？

- `@EnableAsync` + `@Async`：异步执行，需外部线程池（默认 SimpleAsyncTaskExecutor 每次新线程，生产要自定义）。
- `@EnableScheduling` + `@Scheduled`（cron/fixedRate/fixedDelay）：单线程默认，自定义 `TaskScheduler` 多线程。
- `@EnableCaching` + `@Cacheable`/`@CachePut`/`@CacheEvict`：缓存抽象，配 RedisCacheManager。

### 97. SpringBoot 异常处理 + 全局异常？

- `@ControllerAdvice` + `@ExceptionHandler`：全局捕获，返回统一错误结构。
- `ErrorController`：处理未捕获的，默认 `/error` 跳转。
- `ResponseEntityExceptionHandler`：处理 Spring MVC 标准异常。
- RESTful 统一返回 `{code, msg, data}` + 业务异常枚举。

### 98. SpringBoot 配置类 vs XML？

- 优先 `@Configuration` + `@Bean`（类型安全、IDE 支持）。
- `@Import` 引入其他配置类。
- `@ImportResource` 兼容老 XML。
- `@ConfigurationProperties` 绑定配置前缀到 POJO。

### 99. SpringBoot 跨域配置？

- `@CrossOrigin` 方法/类级别。
- 全局 `WebMvcConfigurer.addCorsMappings`：
```java
registry.addMapping("/**").allowedOrigins("*").allowedMethods("*").allowCredentials(true);
```
- 注意 `allowedOrigins=*` 与 `allowCredentials=true` 冲突，用 `allowedOriginPatterns`。
- 网关层统一处理跨域更合适。

### 100. SpringBoot 健康检查 + 优雅停机？

- `/actuator/health`：DB/Redis/Disk/MQ 探活。
- 自定义 `HealthIndicator` 返回 UP/DOWN。
- K8s liveness/readiness 探针接 `health/liveness`、`health/readiness`。
- 优雅停机：`server.shutdown=graceful`，Spring 接收 SIGTERM 后拒绝新请求 + 等待进行中请求完成（超时强制停）。

---

## 六、Spring Cloud（15 题）

### 101. Spring Cloud 各组件全景 + 替代方案？

| 能力 | 主流 | 替代 |
|---|---|---|
| 注册中心 | Nacos | Eureka/Consul/ZK |
| 配置中心 | Nacos | Apollo/ConfigCloud |
| 网关 | Gateway | Zuul/APISIX |
| 远程调用 | OpenFeign | Dubbo/gRPC |
| 熔断限流 | Sentinel | Hystrix/Resilience4j |
| 链路追踪 | SkyWalking | Zipkin/Jaeger |
| 分布式事务 | Seata | 消息最终一致 |
| 负载均衡 | Spring Cloud LoadBalancer | Ribbon（已停） |

### 102. Nacos vs Eureka vs Consul？

| 维度 | Nacos | Eureka | Consul |
|---|---|---|---|
| 一致性 | AP+CP | AP | CP（Raft） |
| 配置中心 | 是 | 否 | 是 |
| 健康检查 | 心跳+主动 | 心跳 | TCP/HTTP/脚本 |
| 多数据中心 | 支持 | 弱 | 支持 |
| 推送方式 | UDP+长轮询/gRPC | 客户端拉 | 长轮询 |

- Eureka 2.x 停更，社区转向 Nacos/Consul。
- Nacos 国内最流行，注册+配置一体。

### 103. Nacos 注册中心 AP/CP 切换？

- 临时实例（默认）：AP（Distro 协议），客户端心跳，节点间最终一致。
- 永久实例：CP（Raft），服务端主动探测，强一致。
- `spring.cloud.nacos.discovery.ephemeral=false` 切永久。
- 选型：服务发现一般 AP（可用性优先）；强一致要求（如配置）用 CP。

### 104. Nacos 配置中心动态刷新原理？

- 客户端长轮询（默认 29.5s 超时），服务端有变更立即返回。
- 客户端本地快照兜底。
- `@RefreshScope`：配置变更时销毁重建该 Bean，注入新值。
- `@ConfigurationProperties` 自动刷新；`@Value` 需 `@RefreshScope` 才刷新。
- Nacos 2.x 改 gRPC 长连接，推送更实时。

### 105. OpenFeign 原理 + 超时/重试陷阱？

（深度版 Q17 已详述，补充：）

- Feign 默认 `HttpURLConnection`，无连接池，生产换 OkHttp/Apache HttpClient。
- 配置：
```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 2000
        readTimeout: 5000
  okhttp:
    enabled: true
```
- 重试与限流叠加放大风险，写接口禁用重试。
- Hystrix/Sentinel 整合做熔断降级。

### 106. Gateway vs Zuul？

- Zuul 1.x：Servlet 同步阻塞，停更。
- Zuul 2.x：Netty 异步，社区少。
- Gateway：基于 WebFlux + Reactor，异步非阻塞，性能好。
- 功能：路由、断言、过滤器、限流、熔断、协议转换。
- 选型新项目用 Gateway。

### 107. Gateway 路由 + 自定义过滤器？

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user
          uri: lb://user-service
          predicates:
            - Path=/user/**
            - Header=X-Req, \d+
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
```

- 自定义 `GlobalFilter` 实现鉴权、日志、TraceId 传递。
- `Ordered` 控制过滤器顺序。
- 限流用 `RequestRateLimiter`（Redis 令牌桶）或 Sentinel。

### 108. Sentinel vs Hystrix vs Resilience4j？

| 维度 | Hystrix | Sentinel | Resilience4j |
|---|---|---|---|
| 状态 | 停更 | 维护中 | 维护中 |
| 限流 | 弱 | 强（QPS/线程/热点） | 一般 |
| 熔断 | 是 | 是 | 是 |
| 降级 | 是 | 是 | 是 |
| 系统自适应 | 否 | 是（load/cpu） | 否 |
| 控制台 | Dashboard | 强控制台 | 弱 |
| 编程模型 | 注解 | 注解+API | 函数式 |

- 新项目国内多用 Sentinel。

### 109. Sentinel 限流规则 + 熔断策略？

- 流控规则：QPS/并发线程，单机/集群；直接/关联/链路。
- 熔断：慢调用比例（RT 阈值 + 比例 + 时间窗）、异常比例、异常数。
- 热点参数限流：按参数值限流（商品 id）。
- 系统规则：Load/CPU/RT/线程/入口 QPS 自适应。
- 来源：控制台推 + 数据源（Nacos/Apollo）持久化。

### 110. Seata 分布式事务模式？

- AT：自动二阶段，基于 undo 自动回滚，业务无感知，适合大部分场景。全局锁保证隔离。
- TCC：Try-Confirm-Cancel，业务实现三接口，性能高但侵入大。
- SAGA：长事务编排，补偿事务，适合长流程。
- XA：强一致，性能差。
- 角色：TC（事务协调器，独立部署）+ TM（事务管理器，发起方）+ RM（资源管理器，参与方）。

### 111. 分布式事务最终一致方案？

- 本地消息表：业务 + 消息同事务写本地库 → 定时扫描发 MQ → 消费方幂等处理。
- 事务消息（RocketMQ）：半消息 + 回查，保证本地事务与消息发送一致。
- Saga 补偿：失败时调补偿接口。
- 对账兜底：定时核对各服务数据一致性。
- 选型：强一致用 Seata AT；高可用最终一致用消息 + 对账。

### 112. 链路追踪 SkyWalking vs Zipkin？

- SkyWalking：字节码增强无侵入，Java agent，自动埋点；存储 ES/H2/MySQL；功能全（拓扑/告警/性能）。
- Zipkin：需手动埋点（Brave）或 Sleuth 自动；轻量；功能简单。
- Jaeger：云原生，OpenTelemetry 标准。
- TraceId 贯穿：网关生成 → HTTP/RPC header 传递 → MDC 输出日志。

### 113. 负载均衡 Ribbon/LoadBalancer 算法？

- 轮询 RoundRobin（默认）、随机 Random、加权响应时间 WeightedResponseTime、最少连接 BestAvailable、重试 Retry。
- Spring Cloud LoadBalancer（替代 Ribbon）默认轮询，可自定义 `ReactLoadBalancer`。
- 自定义：实现 `IRule`/`ReactorServiceInstanceLoadBalancer`。

### 114. 微服务拆分原则？

- 单一职责：按业务能力拆。
- 高内聚低耦合：服务内强相关，服务间松散。
- 数据库独占：每服务独立库，避免共享。
- 接口稳定：API 版本化，向后兼容。
- 拆分粒度：过细增加运维成本，过粗失去意义。
- DDD 限界上下文指导拆分。

### 115. 微服务常见问题 + 解决？

- 服务发现 → 注册中心。
- 配置管理 → 配置中心。
- 远程调用 → Feign + 负载均衡。
- 熔断限流 → Sentinel。
- 链路追踪 → SkyWalking。
- 分布式事务 → Seata/消息。
- 网关统一入口 → Gateway。
- 日志聚合 → ELK + TraceId。
- 监控告警 → Prometheus + Grafana。

---

## 七、RabbitMQ（15 题）

### 116. RabbitMQ 整体架构 + 核心概念？

- Producer → Exchange → Binding → Queue → Consumer。
- Exchange：接收消息按类型路由。
- Binding：exchange 与 queue 的绑定规则（routing key）。
- Queue：消息存储，FIFO。
- Channel：连接内的轻量虚拟连接，复用 TCP。
- VirtualHost：逻辑隔离。

### 117. 四种 Exchange 类型？

- `direct`：精确匹配 routing key。
- `fanout`：广播到所有绑定队列（忽略 key）。
- `topic`：模式匹配（`*` 单词，`#` 多个），如 `user.*.created`。
- `headers`：按消息头属性匹配（性能差，少用）。

### 118. 队列属性 + 持久化？

- `durable`：队列持久化（重启不丢）。
- `exclusive`：独占（连接断自动删，仅当前连接）。
- `autoDelete`：无消费者后自动删。
- `x-message-ttl`：消息 TTL。
- `x-expires`：队列空闲超时删。
- `x-max-length`/`x-max-length-bytes`：长度/容量限制（溢出策略 `x-overflow`）。
- `x-dead-letter-exchange`/`x-dead-letter-routing-key`：死信路由。
- 消息持久化：`deliveryMode=2`。

### 119. 消息可靠投递三段（详解）？

**生产者**

- `confirm` 模式：Broker 持久化后回 ack/nack。
- `mandatory` + ReturnListener：路由不到队列时回调。
- 持久化 exchange/queue + deliveryMode=2。
- 备份交换器 `alternate-exchange` 兜底。

**Broker**

- 镜像队列（旧）/ 仲裁队列 Quorum（3.8+，基于 Raft 推荐）。

**消费者**

- 手动 ack；处理完再 ack。
- 异常 nack + requeue=false → 死信。
- 幂等：唯一键 + Redis/DB 去重。

### 120. 幂等设计 + 重复消费？

- 业务唯一键（msgId/业务id）+ Redis setnx + TTL。
- DB 唯一索引。
- 状态机校验（订单状态流转）。
- 乐观锁版本号。
- Token 机制（前端先取 token，提交校验）。
- 实战：消费前 `SETNX mq:{msgId}`，存在则 ack 跳过。

### 121. 死信队列 + 触发 + 应用？

触发：
1. `nack/reject` + `requeue=false`。
2. TTL 过期。
3. 队列长度超限。

应用：
- 失败重试 N 次后转死信人工处理。
- 延迟队列（TTL + DLX）。
- 异常订单隔离分析。

绑定：
```java
args.put("x-dead-letter-exchange", "dlx");
args.put("x-dead-letter-routing-key", "key");
```

### 122. 延迟队列实现方案对比？

| 方案 | 优 | 缺 |
|---|---|---|
| TTL+DLX | 原生支持 | 队头阻塞（先入先过期） |
| 单条 expiration | 灵活 | 同样按队头检查 |
| `rabbitmq_delayed_message_exchange` 插件 | 准确 | 插件依赖 |
| Redis Zset 时间戳 | 简单 | 需轮询 |
| 时间轮/Quartz | 高性能 | 自实现复杂 |
| RocketMQ 延迟消息 | 成熟 | 换 MQ |

### 123. 镜像队列 vs 仲裁队列？

- 镜像队列（classic + ha-mode）：主从复制，主写从同步，主宕选举；非强一致，可能丢消息。
- 仲裁队列 Quorum（3.8+）：基于 Raft，多数写成功才 ack，强一致 + 持久化默认；性能略低但更可靠。
- 推荐：新项目用 Quorum，放弃 mirror。

### 124. RabbitMQ 集群 + 节点类型？

- 集群：多节点共享元数据（exchange/binding/user），队列默认只在单节点（除非镜像/仲裁）。
- 节点类型：disc（磁盘，持久化元数据）、ram（内存，仅缓存，需配 disc 节点）。
- 集群发现：`cluster_nodes` 配置或 `join_cluster`。
- 网络分区处理：`pause_minority`（少数派暂停）/`autoheal`（多数派胜出）。

### 125. RabbitMQ 流控 + prefetch？

- 流控：基于 credit 机制，进程内多级反压；消费者慢导致 Broker 内存涨触发流控阻塞生产者。
- `prefetch_count`：单消费者未 ack 上限，控制流量。
- Spring：
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10
        concurrency: 5
        max-concurrency: 20
        acknowledge-mode: manual
```

### 126. 消费顺序消费 + 并发矛盾？

- 单队列单消费者保证顺序。
- 多消费者并发破坏顺序。
- 折中：按业务 key 路由到多队列（每队列单消费者），队列内顺序、整体并发。
- 或单队列 + 业务 key 分桶到多线程，桶内顺序处理。

### 127. RabbitMQ 与 Kafka 对比 + 选型？

| 维度 | RabbitMQ | Kafka |
|---|---|---|
| 模型 | 队列推 | 分区拉 |
| 协议 | AMQP | 自研 |
| 吞吐 | 万级 | 百万级 |
| 路由 | 灵活 | 简单 |
| 顺序 | 单队列 | 分区内 |
| 延迟 | μs | ms |
| 持久化 | 内存+磁盘 | 顺序写盘 |
| 适用 | 业务消息/任务 | 流式/日志/大数据 |

### 128. RabbitMQ 消息积压怎么处理？

- 临时扩消费者（注意数据库/下游压力）。
- 增加 prefetch + 并发。
- 修复消费逻辑（卡住/慢查询）。
- 临时关闭非核心消费，专注积压队列。
- 极端：消息转储到别的队列/文件，事后补处理。
- 监控 queue depth 告警提前发现。

### 129. RabbitMQ 事务 vs confirm？

- 事务（`txSelect`/`txCommit`）：同步阻塞，吞吐低，少用。
- confirm：异步 ack/nack，性能高，生产推荐。
- 消费端事务：手动 ack 模式，业务处理完再 ack。

### 130. RabbitMQ 高可用方案？

- 集群 + 仲裁队列：节点宕少数仍可用，强一致。
- 负载均衡（HAProxy/Nginx）前置，客户端连 LB。
- 配合 Sentinel/Keepalived 防 LB 单点。
- 监控：queue 深度、消费速率、连接数、磁盘水位。
- 磁盘水位告警（默认 50% 阻塞发布）。

---

## 附：高频追问速查

- **集合**：HashMap 扩容 / ConcurrentHashMap 分段锁演进 / ArrayList vs LinkedList
- **JVM**：GC 调优 / 类加载 / 内存泄漏排查
- **MySQL**：MVCC / 索引失效 / 分库分表 / 死锁
- **Redis**：分布式锁 / 缓存一致性 / 大 key 热 key / 持久化
- **Spring**：循环依赖 / 事务失效 / AOP 代理
- **SpringBoot**：自动装配 / 启动流程 / Actuator
- **Spring Cloud**：Nacos / Sentinel / Seata / Gateway
- **RabbitMQ**：可靠投递 / 死信 / 延迟 / 幂等

