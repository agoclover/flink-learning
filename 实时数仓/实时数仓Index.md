# 架构

架构图 → [实时项目架构.drawio](实时项目架构.drawio)

# 实时架构理念

在实时数仓中, kafka 的一层的意思是临时存储数据的中间环节, 不是最后的结果, 就像离线数仓中的 Hive.  

lamda: 实时和离线共存, 历史数据取离线的一天以前的数据, 当天的数据取实时的. 

kappa: 所有数据都来源于实时, 历史数据是曾经实时数据出来的旧的数据, 放弃批处理. 离线数据取某时之前的实时数据. 但这存在两个问题: 

- 开发效率: 离线远远高于实时的. 实时要写代码, 写 SQL 并不能保证所有语句都能成功表达. 没有哪个平台可以用 SQL 去写实时, 比如 Flink 的 join 就不行.  
- 数据准确性: 实时准确性赶不上离线, 由于涉及的组件比较多, 延迟, 丢失, 重复都可能会有. 虽然会控制, 但仍然很难. 

所以我们一般是: 一部分需求用来做实时, 一部分需求是离线做, 然后实时需求如果可以离线做, 实时白天去算, 然后离线凌晨算完了之后把实时需求结果替换掉, 最后离线的报表以离线的为准. 但实时的报表最重要的是能看出当天的趋势. 

但离线的数据栈和实时的数据栈是完全不一样的, 所以如果想维持这两份需求: 实时监控和离线校正, 对工作量是翻倍的, 服务器也需要两套服务器. 

而理论上讲, 实时在功能上是覆盖离线的, 如果实时非常靠谱, 是可以放弃离线的, 是可以解决成本的. 所以如果离线和实时需求重复率不高, 那么 lamda 架构就可以, 如果重复需求比较多, 那么可以考虑 kappa 纯实时架构. 但目前一般都是 100:2 的比例. 

大厂自研平台, 进行实时业务 SQL 化. Flink 能够支持一部分 SQL, 但不能够支持全 SQL. Github 上开源的一些架构要么说明少, 要么有 bug, 但实时 SQL 话是一种趋势. 华为, OPPO, 阿里, 美团(没有放弃离线), 他们都是在 Flink 基础上对 Flink 支持不太好的部分进行了优化, 产生图形化的调度一体的平台, 比如在网页里填一个 SQL 就可以了, 但也没有开源. 

实时业务 SQL 话是一个趋势, 能 SQL 就 SQL, 实在不行再写底层. 

阿里有一个 Blink, 官方 Flink 从 1.09 要把 Blink 加入, 但只融入了一部分. Blink 目前已经开源了, 但只开源了内核部分, 真正厉害的是其可视化的管理平台, `join`, 数据源 `HBase` 都可以直接拖过来, 组成一个流程图, 也可以自定义 SQL. 

阿里现在有三种数仓架构:

1. EMR: 相当于阿里的 CDH, 快速安装, 但里面的服务比如 Hive 等还是原来的东西, 只不过给一个管理平台. 

2. MaxCompute: 一套可视化的 Hive. 网页里进行写 SQL, 进行定时等, 还有它自己的调度工具. 

   Realtime Compute: 就是它的 Blink;

3. Data Dolphine: 连 SQL 都没有, 纯图形化, 面向数据开发师. 只需要有数仓的理念, 所有东西都有可视化的组件, 有很多自动化方案. 没有 SQL. 



# 典型需求

## 日活需求

### 代码思路

- 消费 Kafka 中的数据; 
- 利用 Redis 过滤当日已经计入的日活设备; 
- 把每次新增的当日日活信息保存到 ES 中; 
- 从 ES 中查询出数据, 通过 Kibana 快速发布,  或发布成数据接口, 通过可视化工程调用. 

### 具体分析 

```
kafka 启动 topic → 去重 → 明细数据 (有一定数据量) 保存到 OLAP ES, ClickHouse, Druid, Kylin, 由其进行聚合, 最终对接可视化框架进行展示.
```

OLAP 的特点是能够容纳大量的数据, 放入 OLAP 进行聚合的好处: 

1. 是开发成本小, 一般就写一个 SQL, 或者像 ES 配一个图, 稍微麻烦点写一些代码. 
2. 可以针对明细数据的各种维度进行聚合. 
3. OLAP 是解决数据一公里的问题. 长项是数据的聚合, 部分适合搜索 ES. 但是 OLAP 的 Join 一般做的都不太好. 但我们这里的日志数据一般不需要 join, 业务数据有数据库数据的一般要 join. 

如果清洗了, 比如又做了一层的聚合:

```
kafka 启动 topic → 去重 → 明细数据 → 聚合 → MySQL, Redis
```

这样需要对每一个需求都提前写一个实时计算. 

updateStateByKey 的问题: 

很不好管理, 数据不透明, 不像 Redis 是一整套的东西. 这些  state 数据存在 checkpoint 里, 日积月累会变得非常大, 很多小文件存在 hdfs 又不合适, 不好清理. 其实 Spark 对这些 checkpoint 有一些参数, 比如多长时间不访问可以淘汰掉. 但没有自己维护数据好, 比如多长时间淘汰. 

## 其他需求

见 draw.io。



# Kafka2SparkStreaming2DB 精准一次性

## 精确一次消费定义

精确一次消费 (Exactly-once)  是指消息一定会被处理且只会被处理一次. 不多不少就一次处理. 
如果达不到精确一次消费, 可能会达到另外两种情况: 

- 至少一次消费 (at least once) , 主要是保证数据不会丢失, 但有可能存在数据重复问题. 
- 最多一次消费  (at most once) , 主要是保证数据不会重复, 但有可能存在数据丢失问题. 

如果同时解决了数据丢失和数据重复的问题, 那么就实现了精确一次消费的语义了. 

注意: 

1. Kafka Producer 幂等性发送和消费幂等性不是一个概念; 
2. Kafka 不支持幂等性 (这里是说进程挂掉了, 然后再起造成重复数据, 其实这里也可以通过事务, 手动添加 pid 来保证幂等和精准一次行行行) , 但支持 Kafka Producer 幂等性发送. 



## 问题如何产生

数据何时会丢失: 比如实时计算任务进行计算, 到数据结果存盘之前, 进程崩溃, 假设在进程崩溃前kafka调整了偏移量, 那么kafka就会认为数据已经被处理过, 即使进程重启, kafka也会从新的偏移量开始, 所以之前没有保存的数据就被丢失掉了. 

数据何时会重复: 如果数据计算结果已经存盘了, 在kafka调整偏移量之前, 进程崩溃, 那么kafka会认为数据没有被消费, 进程重启, 会重新从旧的偏移量开始, 那么数据就会被2次消费, 又会被存盘, 数据就被存了2遍, 造成数据重复. 



## 如何解决精准一次性问题

### 策略1 利用关系型数据库的事务进行处理

#### 综述

出现丢失或者重复的问题, 核心就是偏移量的提交与数据的保存, 不是原子性的. 如果能做成要么数据保存和偏移量都成功, 要么两个失败. 那么就不会出现丢失或者重复了. 这样的话可以把存数据和偏移量放到一个事务里. 这样就做到前面的成功, 如果后面做失败了, 就回滚前面那么就达成了原子性.  

#### 问题与限制

但是这种方式有限制: 

1. 数据必须都要放在某一个关系型数据库中, 无法使用其他功能强大的 NoSQL 数据库; 
2. 事务本身性能不好; 
3. 如果保存的数据量较大一个数据库节点不够, 多个节点的话, 还要考虑分布式事务的问题. 

分布式事务会带来管理的复杂型, 一般企业不选择使用; 可以考虑把分布式事务变成本地事务, 把 Executor 的数据提取到 Driver (rdd.collect()), 由 Driver 统一写入数据库, 但会变成本地事务的单线程操作, 降低了写入吞吐量. 

不过只要满足: 数据足够少, 比如经过聚合后的少量数据, 且可以保存在支持事务的数据库, 就可以用事物来解决. 

#### 分布式事务的具体问题描述

Spark Streaming 是集群型资源管理, 可能四台机器消费 kafka 的四个分区, 比如不考虑任何事务问题四个分区并发写入 ES 是可以的. 多线程写入 OLAP 或 MySQL, 这是一件好事, 因为增加了吞吐量, 但同时也带来了一些问题. 

首先, 如果考虑支持分布式事务, 比如使用 MySQL 和分布式关系型数据库 Tidb. 偏移量提交本身是一件事, 但因为分区就变成了 4 件事写入. 如果偏移量提交成功了, 即 4 个分区写入成功了; 但如果偏移量批量提交失败了, 这 4 个事务都得回滚. 所以需要要把偏移量提交和 4 个数据写入变成一个事务. 

最理想的情况是在一台机器上完成 1 个数据写入, 一个偏移量提交, 即本地事务, 数据库认为这是一个 sESsion, 在一个 SESsion 中控制事务是容易的. 但是你如果多机器了多个 SESsion, 每一个链接请求对于数据库之间是一个 SESsion, 再加上偏向提交又是 1 个 SESsion, 总共 5 个 SESsion. 这时再想分布式事务的管理是非常麻烦的: 

- 首先极大影响性能; 
- 要再找一个外部的中间件来去统一协调事务. 每个 SESsion 提交完了以后去一个第三方的中间件告诉已经提交了, 这 4 个都告诉提交了是一件事, 都提交完了到预备状态了, 然后中间件再吹个哨, 这 5 个事就一起提交了. 

MySQL 并不支持这种东西, 需要自己装比如阿里刚出的的 Seata. 这方面企业一般自己想办法处理分布式事务, 而且一般都是放弃分布式事务采用其他方案, 比如因为这种概率出现这种概率比较低, 出现问题了人工客服进行解决. 因为分布式事务经常会造成数据不准确, 屏障等问题, 是一个麻烦点, 所以能躲都躲. 

如何躲开分布式事务? 并发消费 4 个分区, 可以考虑 4 个 Executor 的数据回流到一个机器上的 Driver 中, 然后 Driver 有了所有数据, 且其负责统一管理偏移量, 最后将写入和提交偏移量作为一个单会话的事物写入存储介质. 

但代价问题有两个: 

- Driver 能否扛得住这么多 Executor 中的数据? 易发生 OOM;
- 原来是分布式的, 现在是单点的了, 数据也从并发写入变成了单线程写入数据库, 降低了吞吐量. 

以上三个带价其实就是用事务带来的问题, 但事务有事务的好处, 因为事物是绝对的能够保证精确性一次消费. 其他解决方案也许性能比要好一点, 但是保证精确性一次消费并不一定能保证. 

但是如果数据不多, 是可以这么做的. 比如 5 秒几百上千条数据可以扛得住, 但如果是上万条就不好说了. 吞吐量就直接下降了, QPS 就变得很低了. 

比如某些业务, 我们做了聚合, 比如根据性别进行聚合, 那么一个批次就只有 2 条数据, 这是可以用事务的; 但比如下了用户点了个赞, 做了个评论, 做了个支付, 下了个单, 这些我们要求写入的是明细数据, 数据量一定不会小, 即使大部分时间数据量小, 但只要有可能数据量大, 就不能用事务. 

那对于这些业务怎么办呢? 

### 策略2 手动提交偏移量 + 幂等性处理

如果能够同时解决数据丢失和数据重复问题, 就等于做到了精确一次消费. 

首先解决数据丢失问题, 办法就是要等数据保存成功后再提交偏移量, 所以就必须手工来控制偏移量的提交时机. 但是如果数据保存了, 没等偏移量提交进程挂了, 数据会被重复消费. 怎么办？那就要把数据的保存做成幂等性保存. 即同一批数据反复保存多次, 数据不会翻倍, 保存一次和保存一百次的效果是一样的. 如果能做到这个, 就达到了幂等性保存, 就不用担心数据会重复了. 

难点: 话虽如此, 在实际的开发中手动提交偏移量其实不难, 难的是幂等性的保存, 这个要看你架构的选型和设计, 有的时候并不一定能保证. 所以有的时候只能优先保证的数据不丢失. 数据重复难以避免. 即只保证了至少一次消费的语义. 

什么是幂等性? 重复数据写多少次结果都一样. 一般以拒绝写入或覆盖来实现. 

可以通过 upsert 操作实现, 很多数据库, 比如 MySQL, HBase, Redis 是支持幂等性操作的, 这些一般都是有主键唯一约束的. 

[而 ES 中也可以保证幂等性](http://kane-xie.github.io/2017/09/09/2017-09-09_Elasticsearch%E5%86%99%E5%85%A5%E9%80%9F%E5%BA%A6%E4%BC%98%E5%8C%96/), 如果你在 index requESt 中指定了 id, 那么 ES 会先去检查这个 id 是否存在, 如果不存在就新增一条记录, 如果存在则覆盖, 检查 id 的动作是非常耗时的, 特别是数据量特别大的情况下. 如果 Index request 中不指定id, ES 会自动分配一个唯一的 id, 省去了 id 检查的过程, 写入速度就得到提高. 但如果你希望使用 ES 的 id 来保证幂等性的话就没别的选择了, 只能自己指定 id. 没有指定 id 是 post 操作, 指定 id 是 put 操作. 

相比上个方案, 适合处理数量较多的明细, 或要保存在不支持事务的数据. 



## 手动提交偏移量

### 流程

![手动提交偏移量流程](https://strawberryamoszc.oss-cn-shanghai.aliyuncs.com/projects/rt/image-20201025141439698.png)

### 用 Redis 保存偏移量

可以用 MySQL 也可以用 Redis 保存偏移量. 

为什么用 Redis 保存偏移量? 

本身 kafka 0.9 版本以后 Consumer 的偏移量是保存在 kafka 的 `__consumer_offsets` 主题中, 但是如果用这种方式管理偏移量, 有一个限制就是在提交偏移量时, **数据流的元素结构不能发生转变, 即提交偏移量时数据流，必须是 `InputDStream[ConsumerRecord[String, String]]` 这种结构.** 

**但是在实际计算中, 数据难免发生转变, 或聚合, 或关联, 一旦发生转变, 就无法在利用以下语句进行偏移量的提交:** 

```scala
xxDstream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
```

所以实际生产中通常会利用 Zookeeper, Redis, MySQL 等工具对偏移量进行保存. 

Zookeeper: 以前版本就是存在 ZK, 主要利用其树形结构. 性能赶不上其他两种. 

Redis: 性能好一些, 但要搭建主从高可用专门的集群. 

MySQL: 比较稳定, 但性能低. 如果 5s 存一次的话, 那么 MySQL 完全可以. 

### 偏移量管理类

如下: 

```scala
import java.util

import org.apache.kafka.common.TopicPartition
import org.apache.spark.streaming.kafka010.OffsetRange
import redis.clients.jedis.Jedis
import  scala.collection.JavaConversions._
object OffsetManager {


  /**
    * 从Redis中读取偏移量
    * @param groupId
    * @param topic
    * @return
    */
  def getOffset(groupId:String,topic:String):Map[TopicPartition,Long]={
      var offsetMap=Map[TopicPartition,Long]()

      val jedisClient: Jedis = RedisUtil.getJedisClient

      val redisOffsetMap: util.Map[String, String] = jedisClient.hgetAll("offset:"+groupId+":"+topic)
      jedisClient.close()
      if(redisOffsetMap!=null&&redisOffsetMap.isEmpty){
            null
      }else {

            val redisOffsetList: List[(String, String)] = redisOffsetMap.toList

            val kafkaOffsetList: List[(TopicPartition, Long)] = redisOffsetList.map { case ( partition, offset) =>
             (new TopicPartition(topic, partition.toInt), offset.toLong)
           }
           kafkaOffsetList.toMap
      }
   }

  /**
    * 偏移量写入到Redis中
    * @param groupId
    * @param topic
    * @param offsetArray
    */
  def saveOffset(groupId:String,topic:String ,offsetArray:Array[OffsetRange]):Unit= {
    if (offsetArray != null && offsetArray.size > 0) {
      val offsetMap: Map[String, String] = offsetArray.map { offsetRange =>
        val partition: Int = offsetRange.partition
        val untilOffset: Long = offsetRange.untilOffset
        (partition.toString, untilOffset.toString)
      }.toMap

      val jedisClient: Jedis = RedisUtil.getJedisClient

      jedisClient.hmset("offset:" + groupId + ":" + topic, offsetMap)
      jedisClient.close()
    }
  }

}
```

根据自定义偏移量加载的读取 Kafka 数据: 

```scala
def main(args: Array[String]): Unit = {
  val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("dau_app")
  val ssc = new StreamingContext(sparkConf, Seconds(5))
  val groupId = "GMALL_DAU_CONSUMER"
  val topic = "GMALL_START"

  //从redis读取偏移量
  val startupOffsets: Map[TopicPartition, Long] = OffsetManager.getOffset(groupId,topic)

  //根据偏移起始点获得数据
  val startupInputDstream: InputDStream[ConsumerRecord[String, String]] = MyKafkaUtil.getKafkaStream(topic, ssc,startupOffsets,groupId)


  //获得偏移结束点
  var startupOffsetRanges: Array[OffsetRange] = Array.empty[OffsetRange]
  val startupInputGetOffsetDstream: DStream[ConsumerRecord[String, String]] = startupInputDstream.transform { rdd =>
    startupOffsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
    rdd
  }


  val startLogInfoDStream: DStream[JSONObject] = startupInputGetOffsetDstream.map { record =>
    val startupJson: String = record.value()
    val startupJSONObj: JSONObject = JSON.parseObject(startupJson)
    val ts: lang.Long = startupJSONObj.getLong("ts")
    startupJSONObj
   
  }
  startLogInfoDStream.print(100)
…..
…..
…..
…..

dauDstream.foreachRDD{rdd=>
  rdd.foreachPartition{dauInfoItr=>

///可以观察偏移量
if(startupOffsetRanges!=null&&startupOffsetRanges.size>0){
  val offsetRange: OffsetRange = startupOffsetRanges(TaskContext.get().partitionId())
  println("from:"+offsetRange.fromOffset +" --- to:"+offsetRange.untilOffset)
}


    val dauInfoWithIdList: List[(String, DauInfo)] = dauInfoItr.toList.map(dauInfo=>(dauInfo.dt+  "_"+dauInfo.mid,dauInfo))
    val dateStr: String = new SimpleDateFormat("yyyyMMdd").format(new Date())
    MyEsUtil.bulkInsert(dauInfoWithIdList,"gmall_dau_info_"+dateStr)


  }
//在保存时最后提交偏移量

  OffsetManager.saveOffset(groupId ,topic, startupOffsetRanges)

//如果流发生了转换，无法用以下方法提交偏移量
//dauDstream.asInstanceOf[CanCommitOffsets].commitAsync(startupOffsetRanges)


}
...
...
```



# 实时数仓实践

[实时数仓实践.md](./实时数仓实践.md)



# 批流一体化

一般解决思路有两种：

- 结合数据湖，参考 [JD 的实时数仓开发案例](./批流一体化/JD基于DeltaLake批流一体数仓.md)。
- [Flink + Hive 批流一体的准实时数仓](https://mp.weixin.qq.com/s?__biz=MzI0NTIxNzE1Ng==&mid=2651220644&idx=1&sn=8fc5b77da614edb9e79960bcab263000&chksm=f2a3284fc5d4a1597b55d62d5450de2a098a385a2cc38ceb62d82f5fe6efa8c33731f0dc1afe&scene=0&xtrack=1#rd)。

