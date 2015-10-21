####Trident简介
Trident是在storm基础上，一个以实时计算为目标的高度抽象。 它在提供处理大吞吐量数据能力（每秒百万次消息）的同时，也提供了低延时分布式查询和有状态流式处理的能力。 如果你对Pig和Cascading这种高级批处理工具很了解的话，那么应该很容易理解Trident，因为他们之间很多的概念和思想都是类似的。Tident提供了 joins, aggregations, grouping, functions, 以及 filters等能力。除此之外，Trident 还提供了一些专门的原语，从而在基于数据库或者其他存储的前提下来应付有状态的递增式处理。Trident也提供一致性（consistent）、有且仅有一次（exactly-once）等语义，这使得我们在使用trident toplogy时变得容易。

 把Bolt的运行状态仅仅保存在内存中是不可靠的，如果一个node挂掉，那么这个node的任务就会被重新分配，但是之前的状态是无法恢复的，因此，比较聪明的方式就是把storm的计算状态信息持久化到database中，因此，trident就尤为重要。因为在处理大数据时，在与database打交道时通常采用批处理的方式来避免给它带来压力，而trident就是以batch group的形式处理数据的，并提供了一些聚合功能的API。

####Stream
"Stream"是Trident中的核心数据模型，它被当作一系列的batch来处理，在Storm集群的节点之间，一个stream被划分成很多partition（分区），对流的操作是在每个partition上并行进行的。

 1. "Stream"是Trident中的核心数据模型：有些地方也称为TridentTuple。
 2. 一个stream被划分成很多partition：partition是stream的一个子集，里面可能有很多个batch，一个batch可能位于不同的partition上。
 
**Trident 的五种操作：**

 1. Partition-local operations，对每个partition的局部操作，不产生网络传输
 2. Repartitioning operations：对数据流的重新划分（仅仅是划分，但不改变内容），产生网络传输
 3. Aggregation operations：聚合操作
 4. Operations on grouped streams：作用在分组流上的操作
 5. Merge、Join操作

 ---
####**Partition-local operations**
对每个partition的局部操作包括：function、filter、partitionAggregate、stateQuery、partitionPersist、project等。

#####**Functions**
一个function收到一个输入tuple后可以输出0或多个tuple，输出tuple的字段被追加到接收到的输入tuple后面。如果对某个tuple执行function后没有输出tuple，则该tuple被过滤（filter），否则，就会为每个输出tuple复制一份输入tuple的副本。
#####**Filters**
fileter收到一个输入tuple后可以决定是否留着这个tuple。
#####**partitionAggregate**
partitionAggregate对每个partition执行一个function操作（实际上是聚合操作），但它又不同于上面的functions操作，partitionAggregate的输出tuple将会取代收到的输入tuple。

定义一个聚合器有三种不同的接口：CombinerAggregator、ReducerAggregator 和 Aggregator。

**CombinerAggregator接口：**

```
public interface CombinerAggregator extends Serializable {
	T init(TridentTuple tuple);
    T combine(T val1, T val2);
	T zero();
}
```

一个CombinerAggregator仅输出一个tuple（该tuple也只有一个字段）。每收到一个输入tuple，CombinerAggregator就会执行init()方法（该方法返回一个初始值），并且用combine()方法汇总这些值，直到剩下一个值为止（聚合值）。如果partition中没有tuple，CombinerAggregator会发送zero()的返回值。

**ReducerAggregator接口：**

```
public interface ReducerAggregator extends Serializable {
    T init();
    T reduce(T curr, TridentTuple tuple);
}
```
ReducerAggregator使用init()方法产生一个初始值，对于每个输入tuple，依次迭代这个初始值，最终产生一个单值输出tuple。

**Aggregator接口：**

```
public interface Aggregator extends Operation {
    T init(Object batchId, TridentCollector collector);
    void aggregate(T state, TridentTuple tuple, TridentCollector collector);
    void complete(T state, TridentCollector collector);
}
```
Aggregator可以输出任意数量的tuple，且这些tuple的字段也可以有多个。执行过程中的任何时候都可以输出tuple（三个方法的参数中都有collector）。Aggregator的执行方式如下：

 1. 处理每个batch之前调用一次init()方法，该方法的返回值是一个对象，代表aggregation的状态，并且会传递给下面的aggregate()和complete()方法。
 
 2. 每个收到一个该batch中的输入tuple就会调用一次aggregate，该方法中可以更新状态（第一点中init()方法的返回值）。
 3. 当该batch partition中的所有tuple都被aggregate()方法处理完之后调用complete方法。

---
####**Repartitioning operations**
Repartition操作可以改变tuple在各个task之上的划分。Repartition也可以改变Partition的数量。Repartition需要网络传输。下面都是repartition操作：

 1. shuffle：随机将tuple均匀地分发到目标partition里。
 2. broadcast：每个tuple被复制到所有的目标partition里，在DRPC中有用 — 你可以在每个partition上使用stateQuery。
 3. partitionBy：对每个tuple选择partition的方法是：(该tuple指定字段的hash值) mod (目标partition的个数)，该方法确保指定字段相同的tuple能够被发送到同一个partition。（但同一个partition里可能有字段不同的tuple）
 4. global：所有的tuple都被发送到同一个partition。
 5. batchGlobal：确保同一个batch中的tuple被发送到相同的partition中。
 6. patition：该方法接受一个自定义分区的function

---
####**Aggregation operations**
Trident中有aggregate()和persistentAggregate()方法对流进行聚合操作。aggregate()在每个batch上独立的执行，persistemAggregate() 对所有batch中的所有tuple进行聚合，并将结果存入state源中。

aggregate()对流做全局聚合，当使用ReduceAggregator或者Aggregator聚合器时，流先被重新划分成一个大分区(仅有一个partition)，然后对这个partition做聚合操作；另外，当使用CombinerAggregator时，Trident首先对每个partition局部聚合，然后将所有这些partition重新划分到一个partition中，完成全局聚合。相比而言，CombinerAggregator更高效，推荐使用。

---
####**Operations on grouped streams**
groupBy操作先对流中的指定字段做partitionBy操作，让指定字段相同的tuple能被发送到同一个partition里。然后在每个partition里根据指定字段值对该分区里的tuple进行分组。

如果你在一个grouped stream上做聚合操作，聚合操作将会在每个分组(group)内进行，而不是整个batch上。GroupStream类中也有persistentAggregate方法，该方法聚合的结果将会存储在一个key值为分组字段(即groupBy中指定的字段)的MapState中。

---
####**Merges(合并) and joins(连接)**
最后一部分内容是关于将几个stream汇总到一起，最简单的汇总方法是将他们合并成一个stream，这个可以通过TridentTopology中德merge方法完成。

另一种汇总方法是使用join（连接，类似于sql中的连接操作）。

 

