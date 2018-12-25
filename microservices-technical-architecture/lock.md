# 分布式锁使用

锁是我们经常接触的用于保护资源避免并发竞争的工具，在JVM的发展过程中锁的演化更是历久弥新，当下我们可以用公平锁实现顺序排队用非公平锁实现优先级管理，用可重入锁减少死锁的几率，用读写锁提升读取性能，JVM更是实现了从偏向锁到轻量级锁再到重量级锁的逐渐膨胀优化，更多的情况下我们还可以基于CAS 的原子操作包（java.util.concurrent.atomic）实现无锁（乐观锁）编程。

下面是一个非常简单的扣款示例，先判断余额是否大于要扣款的金额，如果是则执行扣款：

```scala
var balance = db.account.getBalance(id)
if(balance < amount){
  return error("余额少于扣款金额")
}
db.account.updateBalance(id,-amount)
```

假定账户X余额100元，有两个并发请求同时扣款，在没有锁的情况可能会导致余额为负数：

|时间 |并发请求A：扣款60 |并发请求B：扣款50 |
|---|------|------|
|1	|查询余额为100，> 60，可以扣款 | |
|2	| |查询余额为100，>50，可以扣款 |
|3	|数据库原子操作，update balance = balance-60，更新后余额为40 | |
|4	| |数据库原子操作，update balance = balance-50，更新后余额为-10 |

最简单做法是用synchronized加锁，如：

```scala
synchronized {
    var balance = db.account.getBalance(id)
    if(balance < amount){
      return error("余额少于扣款金额")
    }
    db.account.updateBalance(id,-amount)
}
```

当然上面说的都是单机环境同一JVM中的锁，在分布式系统中要锁跨JVM作用于多个节点，实现的难度及成本都要远高于前者。

在谈分布式锁之前我们必须明确：如非必要切勿用锁。一方面锁会将并行逻辑转成串行严重影响性能，另一方面还要考虑锁的容错，处理不当可能导致死锁。如果可能笔者更推荐使用如下方案：

* Set化后的MQ替代分布式锁，比如上面的例子，我们可以按用户ID做Set（用户ID % Set数）进而分成多个组，为不同的组创建不同的MQ队列，这样一个用户同一时间只在一个队列中，一个队列的处理是串行化的，实现了锁的功能，同时又有多个Set来完成并行化，在性能上会好于分布式锁，并且代码上没有太多改动
* 使用乐观锁，再如上面的例子，为account创建一个更新版本字段（update_version）,每次更新时版本加1，更新的条件是版本号要等于传入版本号：
    ```scala
    var (balance,currentVersion) = db.account.getBalanceAndVersion(id)
    if(balance < amount){
      return error("余额少于扣款金额")
    }
    // 此操作对应的SQL: UPDATE account SET balance = balance  - <amount> , update_verison = update_verison + 1 WHERE id = <id> AND update_version = <currentVersion>
    if(db.account.updateBalance(id,-amount, currentVersion) == 0){
    return error("扣款失败") // 或递归执行此代码进行重试
    }
    ```

但在一些情况下我还是会不得不用锁，比如如果要同时加锁诸如用户、订单、商品、SKU等多个对象时用锁反而可能是更好的选择（当然前提要反思这样的业务操作是否合理、设计架构是否有缺陷，对此本节不展开讨论），再如在并发量很高的情况下很可能用悲观锁会比乐观锁效率更高。如需要使用分布式锁，我们必要注意这些问题：

* **锁释放与超时** 正常情况下，只要我们在使用完锁后在finally中加上锁释放的代码就可以了，比如下面的代码：
    ```scala
    val lock=new Lock()
    if(lock.tryLock()){
        try{
            // 业务处理
        }catch(Exception ex){
            // 业务异常处理
        }finally{
            //释放锁
            lock.unlock();
        }
    }
    ```
    这是单机版锁上我们常用的逻辑，unlock在finally确保每次加锁都会解锁，即使在执行业务处理或业务异常处理时OOM了也不会死锁（因为此时锁也没了），但在分布式环境下网络是不可靠的，节点是可能宕机的，这种情况下就很可能出现锁无法释放，比如上述情况下的OOM就可能导致死锁，再比如正常执行了unlock代码，但网络传输时丢失了unlock信号也可能导致死锁，所以我们必须要加上超时时间，如（tryLock(<等待锁的时长>,<锁占用的最大时长>)），超时时间设置过长会导致服务异常后无法及时获取新的锁，过短又有可能在业务没有执行完锁提前释放了。更优雅但偏复杂的方法是使用心跳超时设置，即与占有锁的服务保持心跳，在心跳超时后再释放锁
* **性能及高可用** 出于性能考虑，一般分布式锁都是非公平锁，如果要保证加锁顺序而选用公平锁时要注意对性能的影响，加解锁操作本身要保证性能及可用性，避免单点，锁信息要持久化，慎用自旋避免CPU浪费
* **数据一致性** 分布式锁要合理设置锁标记以区分是哪个实例、哪个线程的操作，可重入锁要做好计数及相应的unlock次数，同时必须保证数据的一致，这要求我们只能选择CP特性（见下文）的服务作为分布式锁的中间件

目前主流的分布式锁实现有以下几种：

* **关系型数据库** 由关系型数据库的某些特性来实现，比如使用主键唯一性约束及数据一致来确保同一时间只有一个请求能获得锁，这一方案实现简单，但对高并发场景或可重入时存在比较大的性能瓶颈
* **Redis** 可使用Redis单线程、原子化操作（setnx）来实现，这一方案也很简单，但因为没有原子化的值比较方法，无法原子化确认占用锁的是否是当前实例的当前线程导致比较难实现重入锁，另外Redis单节点有高可用问题，多节点引入RedLock也存在比较大的[争议](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)。当然在绝大多数情况下大家还是可以放心使用的
* **Zookeeper** 可使用Zookeeper的持久节点（PERSISTENT）、临时节点（EPHEMERAL），时序节点（SEQUENTIAL ）的特性组合及watcher接口实现，这一方案可保证最为严格的数据一致性、在性能及高可用也有着比较好的表现，推荐对一致性高要求极高、并发量大的场景使用

最后还要强调的是如无必要切勿用锁。