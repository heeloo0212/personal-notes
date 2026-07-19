---
title: Java 面试题
tags: [面试, Java, 后端]
created: 2026-07-19
---

# Java 方向面试题

> 基于简历技能栈与项目经验定制。每题给出参考答案要点，便于快速复习。

---

## 一、Java 核心（JUC / 线程池 / 锁 / JVM / GC / 类加载）

### 1. 说说你了解的 JUC 同步工具，分别适用什么场景？
- `synchronized`：JVM 内置锁，重量级时升级为 monitor；JDK 6 后引入偏向锁/轻量级锁/自旋优化，多数场景性能足够。
- `ReentrantLock`：可中断、可超时、公平/非公平、支持多个 `Condition`，适合复杂等待/通知。
- `ReentrantReadWriteLock` / `StampedLock`：读多写少优化，`StampedLock` 乐观读性能更高但用法复杂。
- `Semaphore`：限流、资源许可。
- `CountDownLatch` / `CyclicBarrier`：等待 N 个任务完成 / 多线程到达屏障。
- `ConcurrentHashMap` / `CopyOnWriteArrayList` / `BlockingQueue`：高并发容器。
- `CompletableFuture`：异步编排、任务组合。

### 2. 线程池核心参数有哪些？如何合理设置？
- 七参数：`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`。
- 提交流程：核心线程 → 阻塞队列 → 非核心线程 → 拒绝策略。
- CPU 密集型：核心数 +1；IO 密集型：核心数 × (1 + 等待/计算比)，可用 `Runtime.getRuntime().availableProcessors()`。
- 拒绝策略：`AbortPolicy`（默认抛异常）、`CallerRunsPolicy`（回退调用线程，可做反压）、`DiscardPolicy`、`DiscardOldestPolicy`。
- 优先用 `ThreadPoolExecutor` 显式构造，避免 `Executors` 默认队列/线程数导致 OOM。

### 3. synchronized 的锁升级过程？
- 无锁 → 偏向锁（线程 ID 写入对象头）→ 轻量级锁（CAS 自旋 + Mark Word 栈帧锁记录）→ 重量级锁（monitor，进入等待队列）。
- 升级不可逆；竞争激烈时自旋开销大，应直接走重量级锁。
- JDK 15 后偏向锁被废弃，面试可点一下新版本变化。

### 4. volatile 和 synchronized 的区别？happens-before 语义？
- `volatile`：保证可见性 + 禁止指令重排序，不保证原子性；底层内存屏障。
- `synchronized`：原子性 + 可见性 + 有序性。
- happens-before：解锁 先于 后续加锁；`volatile` 写 先于 后续读；线程 start 先于 其第一个动作；传递性。

### 5. JVM 运行时数据区划分？哪些线程共享，哪些私有？
- 共享：堆、方法区（元空间）。
- 私有：程序计数器、虚拟机栈、本地方法栈。
- JDK 8 后永久代被元空间（本地内存）取代，降低 OOM 概率。

### 6. 对象的创建过程与内存布局？
- 类加载检查 → 分配内存（指针碰撞 / 空闲列表）→ 内存空间初始化零值 → 设置对象头 → `<init>` 执行。
- 布局：对象头（Mark Word + 类型指针）+ 实例数据 + 对齐填充。

### 7. 你了解哪些 GC 算法与收集器？
- 算法：标记-清除（碎片）、复制（新生代）、标记-整理（老年代）、分代收集。
- 收集器：Serial、ParNew、Parallel Scavenge、CMS（并发标记清除、碎片）、G1（Region、停顿可控）、ZGC/Shenandoah（低延迟）。
- G1：`-XX:MaxGCPauseMillis` 控制停顿，混合回收；ZGC：染色指针、并发整理，<10ms。

### 8. 什么情况下会触发 Full GC？如何排查？
- 老年代不足、元空间不足、`System.gc()`、CMS 并发失败（Concurrent Mode Failure）退化 Serial Old。
- 排查：`jstat -gcutil`、`jmap -histo`、`jmap -dump` + MAT 分析大对象/内存泄漏，看是否存在 `final` 引用、静态集合未释放、ThreadLocal 未 remove。

### 9. 类加载机制与双亲委派，为什么要打破？
- 加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载。
- 双亲委派：自底向上检查，自顶向下加载，保证核心类不被篡改。
- 打破场景：Tomcat（web 应用隔离）、SPI（`Thread.contextClassLoader`）、OSGi、JDBC。
- Tomcat：每个 webapp 独立 ClassLoader，先加载自己 classpath 再委托父，违反双亲委派。

### 10. JVM 调优用过哪些参数？项目中怎么定位线上问题？
- 堆：`-Xms -Xmx -Xmn`；元空间 `-XX:MetaspaceSize`；GC 日志 `-Xlog:gc*`（JDK 9+）。
- 工具：`arthas`（`watch`/`trace`/`dashboard`）、`jstack` 看死锁、`jmap` 看堆、`top -Hp` 找高 CPU 线程 → `printf "%x"` → 对应 nid。
- 实际项目（产能统计系统）：Kafka 消费堆积时调大 `fetch.max.bytes`、并发线程数、监控 lag。

---

## 二、Spring 全家桶

### 11. SpringBoot 自动装配原理？
- `@SpringBootApplication` → `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → 读取 `META-INF/spring.factories`（2.7+ 改为 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）。
- 配合 `@ConditionalOnClass` / `@ConditionalOnMissingBean` / `@ConditionalOnProperty` 实现按需装配。

### 12. Spring 事务失效的常见场景？
- 方法非 `public`；同类内自调用（绕过 AOP 代理）；`@Transactional` 注解加在被 `final`/`static` 修饰方法上；异常被 catch 吞掉；默认只回滚 `RuntimeException`，受检异常需 `rollbackFor`；事务传播 `NOT_SUPPORTED`/`NEVER`；多线程跨方法调用。
- 解决：注入自身代理（`AopContext.currentProxy()`）或拆分 Bean。

### 13. AOP 的实现原理？JDK 动态代理与 CGLIB 区别？
- Spring 默认目标有接口用 JDK 动态代理（基于接口），无接口用 CGLIB（子类继承）。
- `proxyTargetClass = true` 强制 CGLIB。
- SpringBoot 2.x 起默认 CGLIB。
- 切点、通知（前置/后置/环绕/异常/返回）、织入时机（运行时）。

### 14. SpringCloud 各组件及替代方案？
- 注册中心：Nacos / Eureka / Consul。
- 配置中心：Nacos / Apollo。
- 网关：Spring Cloud Gateway（路由 + 过滤器 + 限流）/ Zuul。
- 远程调用：OpenFeign（声明式 HTTP）/ Dubbo（RPC）。
- 熔断限流：Sentinel / Hystrix / Resilience4j。
- 链路追踪：SkyWalking / Zipkin。

### 15. Nacos 既能做注册中心又能做配置中心，原理区别？
- 注册中心：临时实例心跳（5s），永久实例服务端主动探测；CP（Raft，一致性）+ AP（Distro，可用）双模式。
- 配置中心：长轮询（Long Polling，29.5s 超时）拉取变更 + 本地快照兜底。

### 16. OpenFeign 调用超时/重试怎么配置？如何调试？
- `feign.client.config.default.connectTimeout/readTimeout`；`ribbon.ConnectTimeout/ReadTimeout`（旧版）。
- 重试：`Retryer.Default`；超时与重试叠加可能放大调用压力，生产慎用。
- 日志级别：`NONE/BASIC/HEADERS/FULL`，配合 `Logger.Level.BASIC` 调试。

### 17. Gateway 的核心概念与限流实现？
- `Route`（路由 = 断言 + 过滤器 + 目标 URI）、`Predicate`（Path/Header/Query/Time）、`Filter`（全局 `GlobalFilter`）。
- 限流：基于 `RequestRateLimiterGatewayFilterFactory` + Redis Lua 令牌桶；`@SentinelGatewayFilter` 接入 Sentinel。

### 18. Spring Security 认证授权流程？
- `FilterChainProxy` → `UsernamePasswordAuthenticationFilter` → `AuthenticationManager` → `Provider` → `UserDetailsService` → 返回 `Authentication` → `SecurityContextHolder`。
- 授权：`AccessDecisionManager` / `AuthorizationManager`（新版）结合 `@PreAuthorize`、`hasRole`。
- JWT 集成：自定义 `JwtAuthenticationFilter`，无状态会话 `SessionCreationPolicy.STATELESS`。

---

## 三、MyBatis / MyBatis-Plus

### 19. MyBatis 的一级缓存和二级缓存？
- 一级缓存：`SqlSession` 级别，默认开启，`update/commit/close` 失效。
- 二级缓存：Mapper（namespace）级别，需开启 `cacheEnabled` + `<cache/>`，跨 session 共享；分布式下需对接 Redis。
- MyBatis-Plus 二级缓存可换 `MybatisPlus二级缓存` 实现 + Redis。

### 20. MyBatis-Plus 的乐观锁/逻辑删除/分页插件？
- 乐观锁：`@Version` + `OptimisticLockerInnerInterceptor`，更新时 `where version = ?`。
- 逻辑删除：`@TableLogic`，`logic-delete-field` 配置，查询自动过滤。
- 分页：`PaginationInnerInterceptor`，配置方言，`IPage<T>` 返回总数。

---

## 四、Redis

### 21. Redis 为什么这么快？
- 内存操作；单线程 IO 多路复用（epoll）；无锁避免上下文切换；6.0 后多线程处理网络读写，命令仍单线程；高效数据结构（SDS、跳表、压缩列表、quicklist）。

### 22. 缓存击穿、穿透、雪崩及解决方案？
- 击穿：热点 key 过期 → 互斥锁/逻辑过期/永不过期 + 异步重建。
- 穿透：查不存在的数据 → 布隆过滤器 / 缓存空值（短 TTL）。
- 雪崩：大量 key 同时过期或 Redis 宕机 → TTL 随机化 + 多级缓存 + 集群高可用 + 限流降级。

### 23. Redisson 分布式锁的实现？看门狗机制？
- 基于 Lua + Redis Hash（key = 锁名，field = UUID:threadId，value = 重入计数）。
- 加锁成功启动 watchdog（默认 10s 续期到 30s，每 10s 续一次），避免业务未完成锁过期。
- 释放走 Lua 原子：判断 UUID 再删。
- 高可用：RedLock（多节点过半数），但有争议；生产多用主从 + 合理超时。

### 24. Redis Cluster 的槽位与扩容？
- 16384 槽，节点间 Gossip 通信；key 用 CRC16 计算 slot。
- 客户端 MOVED/ASK 重定向；扩缩容时迁移 slot，`redis-cli --cluster reshard`。
- 集群下批量操作需 `{}` hashtag 保证同槽。

### 25. Redis 持久化 RDB vs AOF？
- RDB：快照、体积小、恢复快，可能丢数据；`bgsave` fork 子进程 COW。
- AOF：追加命令，更安全；`appendfsync` always/everysec/no；AOF 重写压缩。
- 4.0 后混合持久化：RDB 全量 + AOF 增量。

---

## 五、RabbitMQ

### 26. 消息可靠投递的三个环节？
- 生产者：`confirm` 模式 + 持久化 exchange/queue + 消息 `deliveryMode=2`。
- Broker：队列持久化、镜像队列/仲裁队列保证高可用。
- 消费者：手动 `ack`，异常 `nack` 重投或进死信；幂等去重（业务唯一键 + Redis/DB）。

### 27. 死信队列的触发条件？项目里怎么用？
- 触发：消息被 `reject/nack` 且 `requeue=false`；TTL 过期；队列长度超限。
- 项目场景：风控平台订单超时关闭、重试 N 次仍失败转入人工处理队列。

### 28. 削峰如何实现？令牌桶与漏桶区别？
- 削峰：上游全量入队，消费者按速率限流拉取（`prefetch` + 消费并发）。
- 令牌桶：允许突发，按速率补令牌；漏桶：匀速出水，平滑流量。
- RabbitMQ 自身靠 `prefetch_count` 限流，再外接 Sentinel/Guava RateLimiter。

### 29. 如何保证消费幂等？
- 业务唯一键 + Redis `setnx` / 数据库唯一索引；乐观锁版本号；状态机校验。
- IOT 平台举例：设备上报 `msgId`，消费前 `SETNX msgId`，存在即跳过。

---

## 六、MySQL

### 30. 索引数据结构为什么用 B+ 树？
- B+ 树非叶子只存索引，单页能放更多 key，扇出大、树矮，磁盘 IO 少。
- 叶子节点有序链表，范围查询高效；查询稳定 O(logN)。
- 对比 Hash 不能范围；B 树叶子和非叶子都存数据，扇出小。

### 31. 联合索引最左前缀与索引下推？
- `(a,b,c)` 能命中：`a`、`a,b`、`a,b,c`；`a,c` 部分命中（c 走不到索引，但 ICP 在引擎层过滤）。
- 索引下推（ICP，5.6+）：在存储引擎用索引列过滤，减少回表次数。

### 32. 慢 SQL 排查思路？
- 开慢查询日志（`slow_query_log` + `long_query_time`）。
- `EXPLAIN` 看 `type`（最好 ref/range 以上）、`key`、`rows`、`Extra`（`Using filesort`/`Using temporary` 需优化）。
- 避免索引失效：函数运算、隐式类型转换、`!=`/`LIKE '%x'`、`OR` 两侧不全有索引。
- 大分页用游标 `where id > last_id limit n`。

### 33. 事务隔离级别与 MVCC？
- 读未提交、读已提交、可重复读（默认）、串行化。
- MVCC：隐藏列 `trx_id`/`roll_ptr` + undo log 版本链 + ReadView 判断可见性。
- RR 在事务首次读时生成 ReadView；RC 每次读都生成 → 不可重复读现象。
- InnoDB 在 RR 下用 Next-Key Lock 解决幻读。

### 34. 分库分表方案与跨库问题？
- 垂直：按业务/字段拆；水平：取模、范围、一致性 hash。
- 跨库 join 用应用层组装或冗余字段；跨库事务用 Seata AT/TCC 或最终一致（本地消息表/事务消息）。
- 分页深翻：用 ShardingSphere 的流式归并或 ES 二级索引。
- 项目（产能统计）：按产线维度分表，时间维度冷热分离。

---

## 七、项目实战 / 综合

### 35. 你在 IOT 平台里如何设计设备数据高并发上报？
- 设备端 MQTT/TCP，接入层 Netty 长连接，鉴权 + 限流。
- 中间 Kafka/RabbitMQ 削峰异步落库；Redis 缓存最新状态 + 订阅推送。
- LiteFlow 编排场景联动规则，事件触发 → 动作下发。
- 数据落 TSDB（InfluxDB/时序库）或 MySQL 按时间分表。

### 36. Kafka 和 RabbitMQ 的核心区别？为什么产能统计选 Kafka？
- Kafka：分区、顺序写、零拷贝、高吞吐、拉模式，适合日志/流式大数据。
- RabbitMQ：路由灵活、AMQP 协议、低延迟，适合业务消息/任务分发。
- 产能统计每秒万级产线数据点，需高吞吐 + 顺序消费 → Kafka + 消费者组分区分配。

### 37. ELK 日志平台怎么搭建的？日志如何收集到 ES？
- Filebeat 采集 → Logstash 过滤/解析 → Elasticsearch 存储 → Kibana 展示。
- 或 Filebeat → Kafka → Logstash → ES（削峰缓冲）。
- 索引按天/服务切分，ILM 策略滚动；字段统一 JSON，`@timestamp` 时区处理。

### 38. Quartz 与 XXL-JOB 区别？定时任务如何保证不重复执行？
- Quartz 基于数据库锁，集群下需配置 `qrtz_locks`；XXL-JOB 中心化调度 + 执行器注册，可视化、分片广播。
- 防重复：分布式锁、唯一任务 ID、数据库乐观锁；幂等设计兜底。

### 39. Netty 的线程模型与零拷贝？
- Reactor 主从模型：BossGroup accept，WorkerGroup read/write；业务线程池隔离。
- 零拷贝：`FileRegion` sendfile、`CompositeByteBuf` 合并无需拷贝、`Unpooled.wrappedBuffer`。
- 解决粘包/拆包：LengthFieldBasedFrameDecoder / 自定义协议头长度。

### 40. 你在金会 AI 金融风控平台里 Java 部分做了什么？
- FastAPI 为主，但用户/角色/菜单/订阅管理、鉴权、订阅计费用 Java SpringCloud + MyBatis-Plus + Redis。
- RabbitMQ 做爬虫任务异步分发，Redis 做任务状态与限流。
- 风控结果落 MySQL，AOP 记录操作审计，Spring Security + JWT 鉴权。

