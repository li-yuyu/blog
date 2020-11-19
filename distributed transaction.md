# 分布式事务

## 什么是分布式事务？

本地事务（单数据源事务），如MySQL 的 InnoDB 引擎通过 Undo Log 、 Redo Log来实现。

分布式事务（多数据源事务），本地事务无法保证跨多数据源全局操作可靠性。

## 二将军问题

![](D:\working\分布式事务\2.jpg)

## 分布式事务的模型

![微信图片_20201115112554](D:\working\微信图片_20201115112554.png)

分布式事务中的角色：

- 协调者（事务管理器 Transaction Manager, TM），如图中shopping-service
- 参与者（资源管理器 Resource Manager, RM），如图中repo-service和order-service

分布式事务的本质：

事务协调者协调各个事务参与者的本地事务的进度，使使所有本地事务共同提交或回滚，最终达成一种全局的 ACID 特性

## 分布式事务的实现方案

### 刚性事务（强一致）

- 2PC 

  准备-提交，会锁定资源，二阶段提交前协调者宕机导致参与者阻塞，二阶段脑裂不一致。

- 3PC 

  询问-准备-提交，仍然会发生不一致，好处就是至少不会阻塞和永远锁定资源。

一个应用操作多个数据源进行2PC或3PC，为了保证强一致，所有子事务的数据库资源会被锁住。

### 柔性事务（弱一致）

- TCC 方案

TCC 模式本质是一个应用层面上的 2PC ，每个业务应用都实现Try-Confirm/Cancel 接口。

TCC 锁定的是业务资源，无需锁数据库资源，提高了吞吐量，应用实施成本较高。

![](D:\working\分布式事务\3.png)

通过补偿来解决二阶段宕机的问题，协调者不断重试、参与者实现幂等。

- 可靠消息最终一致性方案

<img src="D:\working\分布式事务\4.jpg"  />

![](D:\working\分布式事务\5.jpg)

![<u></u>](D:\working\分布式事务\6.jpg)

应用实施成本较低，缺点是无法回滚

- 事务状态表方案

| 分布式事务 ID   | 事务内容                                                     | 事务状态                                                     |
| :-------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| global_trx_id_1 | 操作 1：调用 repo-service 扣减库存 <br />操作 2：调用 order-service 生成订单 | 状态 1：初始 <br />状态 2：操作 1 成功 <br />状态 3：操作 1、2 成功 |

启动一个后台任务，扫描这张表中事务的状态，重试一直未到最终状态的事务，接收方幂等处理。

- SAGA

核心就是补偿，一阶段就是服务的正常顺序调用（数据库事务正常提交），如果都执行成功，则第二阶段则什么都不做；但如果其中有执行发生异常，则依次调用其补偿服务（一般多逆序调用未已执行服务的反交易）来保证整个交易的一致性。*应用实施成本一般*。

- 事务消息

## 处理分布式事务的原则

业务规避 > 柔性事务 > 刚性事务