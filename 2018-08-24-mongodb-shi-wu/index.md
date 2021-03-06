![MongoDB事务](./header.jpg)

## 前言

相信使用过主流的关系型数据库的朋友对“事务（Transactions）”不会太陌生，它可以让我们把对多张表的多次数据库操作整合为一次原子操作，这在高并发场景下可以保证多个数据操作之间的互不干扰；并且一旦在这些操作过程任一环节中出现了错误，事务会中止并且让数据回滚，这使得同时在多张表中修改数据的时候保证了数据的一致性。

以前 MongoDB 是不支持事务的，因此开发者在需要用到事务的时候，不得不借用其他工具，在业务代码层面去弥补数据库的不足。随着 4.0 版本的发布，MongoDB 也为我们带来了原生的事务操作，下面就让我们一起来认识它，并通过简单的例子了解如何去使用。

## 介绍

### 事务和副本集（Replica Sets）

副本集是 MongoDB 的一种主副节点架构，它使数据得到最大的可用性，避免单点故障引起的整个服务不能访问的情况的发生。目前 MongoDB 的多表事务操作仅支持在副本集上运行，想要在本地环境安装运行副本集可以借助一个工具包——[run-rs](https://github.com/vkarpov15/run-rs)，以下的文章中有详细的使用说明：

> https://thecodebarbarian.com/introducing-run-rs-zero-config-mongodb-runner.html

### 事务和会话（Sessions）

事务和会话（Sessions）关联，一个会话同一时刻只能开启一个事务操作，当一个会话断开，这个会话中的事务也会结束。

### 事务中的函数

* #### Session.startTransaction()

在当前会话中开始一次事务，事务开启后就可以开始进行数据操作。在事务中执行的数据操作是对外隔离的，也就是说事务中的操作是原子性的。

* #### Session.commitTransaction()

提交事务，将事务中对数据的修改进行保存，然后结束当前事务，一次事务在提交之前的数据操作对外都是不可见的。

* #### Session.abortTransaction()

中止当前的事务，并将事务中执行过的数据修改回滚。

### 重试

当事务运行中报错，catch 到的错误对象中会包含一个属性名为 **errorLabels** 的数组，当这个数组中包含以下2个元素的时候，代表我们可以重新发起相应的事务操作。

* **TransientTransactionError**：出现在事务开启以及随后的数据操作阶段
* **UnknownTransactionCommitResult**：出现在提交事务阶段

## 示例

经过上面的铺垫，你是不是已经迫不及待想知道究竟应该怎么写代码去完成一次完整的事务操作？下面我们就简单写一个例子：

**场景描述：**假设一个交易系统中有2张表——记录商品的名称、库存数量等信息的表 commodities，和记录订单的表 orders。当用户下单的时候，首先要找到 commodities 表中对应的商品，判断库存数量是否满足该笔订单的需求，是的话则减去相应的值，然后在 orders 表中插入一条订单数据。在高并发场景下，可能在查询库存数量和减少库存的过程中，又收到了一次新的创建订单请求，这个时候可能就会出问题，因为新的请求在查询库存的时候，上一次操作还未完成减少库存的操作，这个时候查询到的库存数量可能是充足的，于是开始执行后续的操作，实际上可能上一次操作减少了库存后，库存的数量就已经不足了，于是新的下单请求可能就会导致实际创建的订单数量超过库存数量。

以往要解决这个问题，我们可以用给商品数据“加锁”的方式，比如基于 Redis 的[各种锁](https://redis.io/topics/distlock)，同一时刻只允许一个订单操作一个商品数据，这种方案能解决问题，缺点就是代码更复杂了，并且性能会比较低。如果用数据库事务的方式就可以简洁很多：

commodities 表数据（stock 为库存）：

```json
{ "_id" : ObjectId("5af0776263426f87dd69319a"), "name" : "灭霸原味手套", "stock" : 5 }
{ "_id" : ObjectId("5af0776263426f87dd693198"), "name" : "雷神专用铁锤", "stock" : 2 }
```

orders 表数据：

```json
{ "_id" : ObjectId("5af07daa051d92f02462644c"), "commodity": ObjectId("5af0776263426f87dd69319a"), "amount": 2 }
{ "_id" : ObjectId("5af07daa051d92f02462644b"), "commodity": ObjectId("5af0776263426f87dd693198"), "amount": 3 }
```

通过一次事务完成创建订单操作（mongo Shell）：

```javascript
// 执行 txnFunc 并且在遇到 TransientTransactionError 的时候重试
function runTransactionWithRetry(txnFunc, session) {
  while (true) {
    try {
      txnFunc(session); // 执行事务
      break;
    } catch (error) {
      if (
        error.hasOwnProperty('errorLabels') &&
        error.errorLabels.includes('TransientTransactionError')
      ) {
        print('TransientTransactionError, retrying transaction ...');
        continue;
      } else {
        throw error;
      }
    }
  }
}

// 提交事务并且在遇到 UnknownTransactionCommitResult 的时候重试
function commitWithRetry(session) {
  while (true) {
    try {
      session.commitTransaction();
      print('Transaction committed.');
      break;
    } catch (error) {
      if (
        error.hasOwnProperty('errorLabels') &&
        error.errorLabels.includes('UnknownTransactionCommitResult')
      ) {
        print('UnknownTransactionCommitResult, retrying commit operation ...');
        continue;
      } else {
        print('Error during commit ...');
        throw error;
      }
    }
  }
}

// 在一次事务中完成创建订单操作
function createOrder(session) {
  var commoditiesCollection = session.getDatabase('mall').commodities;
  var ordersCollection = session.getDatabase('mall').orders;
  // 假设该笔订单中商品的数量
  var orderAmount = 3;
  // 假设商品的ID
  var commodityID = ObjectId('5af0776263426f87dd69319a');

  session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
  });

  try {
    var { stock } = commoditiesCollection.findOne({ _id: commodityID });
    if (stock < orderAmount) {
      print('Stock is not enough');
      session.abortTransaction();
      throw new Error('Stock is not enough');
    }
    commoditiesCollection.updateOne(
      { _id: commodityID },
      { $inc: { stock: -orderAmount } }
    );
    ordersCollection.insertOne({
      commodity: commodityID,
      amount: orderAmount,
    });
  } catch (error) {
    print('Caught exception during transaction, aborting.');
    session.abortTransaction();
    throw error;
  }

  commitWithRetry(session);
}

// 发起一次会话
var session = db.getMongo().startSession({ readPreference: { mode: 'primary' } });

try {
  runTransactionWithRetry(createOrder, session);
} catch (error) {
  // 错误处理
} finally {
  session.endSession();
}
```

上面的代码看着感觉很多，其实 runTransactionWithRetry 和 commitWithRetry 这两个函数都是可以抽离出来成为公共函数的，不需要每次操作都重复书写。用上了事务之后，因为事务中的数据操作都是一次原子操作，所以我们就不需要考虑分布并发导致的数据一致性的问题，是不是感觉简单了许多？

你可能注意到了，代码中在执行 startTransaction 的时候设置了两个参数——readConcern 和 writeConcern，这是 MongoDB 读写操作的确认级别，在这里用于在副本集中平衡数据读写操作的可靠性和性能，如果在这里展开就太多了，所以感兴趣的朋友建议去阅读官方文档了解一下：

**readConcern：**

https://docs.mongodb.com/master/reference/read-concern/

**writeConcern：**

https://docs.mongodb.com/master/reference/write-concern/

