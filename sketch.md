

[TOC]

# 分布式事务

分布式事务一个跨多个数据源，并且需要保证 ACID 特性的事务。其中数据源可以是数据库、消息队列、缓存、ES 等等。

![DTP model](assets/sketch/DTP model.png)
<center>X/Open 定义的分布式事务模型</center>

目前涉及到分布式事务的场景：

- 微服务架构下，保证多个服务的数据一致性
- 表因为数据量太大而分库，在单个应用中操作多个数据库

在微服务架构下，根据 CAP 定理，一致性、可用性和分区容忍性三者不可兼得。因为不能保证网络通信一定是可靠的，所以分区容忍性不能放弃。因此，分布式事务往往是在一致性和可用性之间做权衡。

为了方便说明各种分布式事务解决方案的原理，先举个购买商品的例子：购买商品时需要扣减库存和创建订单，但是这两步操作不是在同一个服务当中，我们需要保证扣减库存后，订单一定会创建，如下图所示：

![分布式事务-例子](assets/sketch/分布式事务-例子.png)

## 事务消息

为了保证一定能创建订单，最简单的方法就是重试。在扣减库存后，可以发一条消息给订单服务模块。目前消息队列基本都支持 “至少一次“ 的消息投递策略，可以保证消息最终一定会被正常消费。但是怎么保证库存扣减后，消息一定能发出去呢？可以新增一张消息表，在扣减库存后，插入一条消息记录。这两部操作需要在同一个事务里。同时会有一个后台线程，若检查到消息表有数据，则查出来并发送消息。当消息发送成功后，再删除对应的消息记录。整个流程如下图所示：

![分布式事务-事务消息](assets/sketch/分布式事务-事务消息.png)

对于这种事务提交后，要求一定能发出去的消息，称之为 ”事务消息“。事务消息有两个比较大的缺点：

- 没有事务隔离性。扣减库存后，短时间内会查询不到订单，但是业务上已经是购买成功状态。
- 可能永远都无法成功消费消息。扣减库存后，如果需要扣减用户余额，但是余额不足，这种情况就不应该用事务消息。

## XA

利用事务消息的解决方案，本质上就是重试，只满足最终一致性，跟事务所要求的 ACID 特性相差有点远。上世纪 90 年代，为了解决分布式事务的一致性问题，X/Open 组织提出一套基于 2PC 协议的规范 —— XA (eXtended Architecture).

先简单回顾下 2PC 的流程：

![分布式事务-2pc](assets/sketch/分布式事务-2pc-16283265203431.png)

XA 主要是定义了 TM 和 RM 之间的交互：

![XA定义的规范](assets/sketch/XA定义的规范.png)

其中，ax 开头的操作，是由 RM 来 调用 TM；xa 开头的操作，是由 TM 来调用 RM。

目前，MySQL 通过提供以下 SQL 来支持 XA 事务：

```sql
/* 
 * 开启全局事务，xid 由 TM 生成，要保证全局唯一
 * 执行完该语句后，紧跟着执行业务 SQL. 
 */
XA {START|BEGIN} xid [JOIN|RESUME] 

/* 
 * 执行完业务 SQL 后，需要执行该语句表明已执行完成。
 */
XA END xid [SUSPEND [FOR MIGRATE]]

/* 
 * redo log 持久化事务，进入准备阶段。
 * 若在执行该语句之前宕机，则会自动回滚；若在执行该语句之后宕机，则需要通过 XA RECOVER 来查看 PREPARE 的事务，业务方自己决定回滚还是提交。
 */
XA PREPARE xid

/* 
 * 事务提交
 */
XA COMMIT xid [ONE PHASE]

/* 
 * 回滚事务
 */
XA ROLLBACK xid

/* 
 * 查看已经 PREPARE 的事务
 */
XA RECOVER [CONVERT XID]
```

用 XA 事务来解决用户购买商品的问题，交互流程如下：

![分布式事务-例子-XA事务](assets/sketch/分布式事务-例子-XA事务.png)

其中，业务层收到所有分支事务的 `PREPARE` 响应后，需要记录事务是提交还是回滚。当宕机重启时，通过 `XA RECOVER` 发现有未决的事务，则可以根据事务状态来决定提交还是回滚。

在不同业务场景下执行 XA 事务，只是执行的业务 SQL 不一样，流程是不需要变的。也就是说 XA 事务对业务透明，不需要改变业务流程。但其也有些缺点：

- 在单机事务中，数据库可以自动检测是否发生死锁，并选择回滚事务。但是在分布式事务中，MySQL 做不到。
- 若 MySQL 在执行 `XA PREPARE, XA COMMIT, XA ROLLBACK, XA COMMIT ... ONE PHASE` 时宕机，重启后不能保证 binlog 的一致性。
- 为了防止脏读，所有 SQL 必须要在串行化隔离级别执行，因此性能最差。
- 如果某个事务 `PREPARE` 后，一直没有 `COMMIT` 或 `ROLLBACK`，数据库不会自动释放锁，从而阻塞其它事务的执行，降低可用性（一致性和可用性不可兼得）。

## TCC 

```mermaid
graph LR
A[方形] -->B(圆角)
    B --> C{条件a}
    C -->|a=1| D[结果1]
    C -->|a=2| E[结果2]
    F[横向流程图]
```

定义、原理、怎么用、例子、优缺点

## SAGA 





定义、原理、怎么用、例子、优缺点

## Seata 框架

### XA 模式


由上面的交互时序图可以看出，使用 XA 事务时，应用程序担任了协调者的角色，2PC 和业务逻辑混合在一起，不方便使用。阿里开源的分布式事务框架 Seata (Simple Extensible Autonomous Transaction Architecture)，把协调者地角色抽离出来，使得可以像使用本地事务那样使用 XA 事务。

#### 使用示例

```java
/**
 * 替换数据源
 **/
@Bean("dataSourceProxy")
public DataSource dataSource(DruidDataSource druidDataSource) {
	return new DataSourceProxyXA(druidDataSource);
}
```

```java
/**
 * 替换事务注解
 **/
@GlobalTransactional(timeoutMills = 1000)
public void purchase(String userId, String commodityCode, int orderCount, boolean rollback) {
    String xid = RootContext.getXID();
    LOGGER.info("New Transaction Begins: " + xid);

    String result = storageFeignClient.deduct(commodityCode, orderCount);

    if (!SUCCESS.equals(result)) {
        throw new RuntimeException("库存服务调用失败,事务回滚!");
    }

    result = orderFeignClient.create(userId, commodityCode, orderCount);

    if (!SUCCESS.equals(result)) {
        throw new RuntimeException("订单服务调用失败,事务回滚!");
    }

    if (rollback) {
        throw new RuntimeException("Force rollback ... ");
    }
}
```

#### 整体机制

![seata-xa整体机制](assets/sketch/seata-xa整体机制.png)

TM：定义全局事务的范围——开始全局事务、提交或回滚全局事务。

TC：全称 Transaction Coordinator，是 Seata 服务端（应用依赖的 Seata SDK 是客户端），维护全局和分支事务的状态，驱动全局事务提交或回滚。

RM：与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

## AT 

为了解决 XA 事务下数据库长时间持有锁的问题，Seata 提供了 AT (Automatic Transaction) 模式。在此模式下，事务不依赖数据库的锁来保证串行化，而是使用 Seata 自己可控的全局锁。同时，Seata 也得负责数据的回滚操作。

### 使用示例

```java
/**
 * 替换数据源
 **/
@Bean("dataSourceProxy")
public DataSource dataSource(DruidDataSource druidDataSource) {
	return new DataSourceProxy(druidDataSource);
}
```

```java
/**
 * 替换事务注解，跟 XA 模式一样
 **/
@GlobalTransactional(timeoutMills = 1000)
public void purchase(String userId, String commodityCode, int orderCount, boolean rollback) {
    ......
}
```

### 实现原理

整体流程也是基于 2PC 协议，跟 XA 模式不同的是提交分支事务、回滚分支事务和提交全局事务这三个操作。

#### 提交分支事务

![seata at模式-分支事务提交](assets/sketch/seata at模式-分支事务提交.png)

全局锁：锁住某个表中被修改的数据，根据表名、唯一键等字段生成

#### 提交全局事务

![分布式事务-at模式-二阶段提交](assets/sketch/分布式事务-at模式-二阶段提交.png)

#### 回滚分支事务

![分布式事务-at模式-二阶段回滚](assets/sketch/分布式事务-at模式-二阶段回滚.png)

#### 隔离性

读数据：默认是读未提交。如果要读已提交，则需要使用 `select ... for update` 语句。Seata 检测到该语句，会自动请求全局锁。

写数据：如果两个事务都是分布式事务，则本身就已经通过全局锁来串行化执行；如果一个是分布式事务，另一个是单机事务，则单机事务需要通过手动加全局锁，防止分布式事务回滚异常。

#### 与 XA 事务相比

## 参考文献


