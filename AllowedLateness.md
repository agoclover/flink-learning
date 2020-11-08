# AllwoedLateness 与乱序数据

Flink 中借助 Watermark 以及 Window 和 Trigger 来处理基于 Event time 的乱序问题, 那么如何处理 Late Element 呢?

也许有人会问, out-of-order element 与 Late Element 有什么区别? 不都是一回事么? 

答案是一回事, 都是为了处理乱序问题而产生的概念. 要说区别, 可以总结如下:

- 通过 Watermark 机制来处理 out-of-order 的问题, 属于第一层防护, 属于全局性的防护, 通常说的乱序问题的解决办法, 就是指这类;
- 通过窗口上的 AllowedLateness 机制来处理 out-of-order 的问题, 属于第二层防护, 属于特定 Window operator 的防护, Late element 的问题就是指这类.

# AllowedLateness

下面我们重点介绍 AllowedLateness.

默认情况下, 当 Watermark 通过 end-of-window 之后, 再有之前的数据到达时, 这些数据会被删除. 为了避免有些迟到的数据被删除, 因此产生了 AllowedLateness 的概念. 

简单来讲, AllowedLateness 就是针对 element 的 Event time 而言, 对于 Watermark 超过 end-of-window 之后, 还允许有一段时间 (也是以 Event time 来衡量) 来等待之前的数据到达, 以便再次处理这些数据, 再超过这个时间就只能删除或者输出到侧输出流了.

默认情况下，如果不指定 AllowedLateness, 其值是 0, 即对于 Watermark 超过 end-of-window 之后, 还有此 Window 的数据到达时, 这些数据被删除掉了. 

注意: 对于 trigger 是默认的 EventTimeTrigger 的情况下, AllowedLateness 会再次触发窗口的计算, 而之前触发的数据, 会 buffer 起来, 直到 Watermark 超过 `End-of-window + allowedLateness` 的时间, 窗口的数据及元数据信息才会被删除. 再次计算就是 DataFlow 模型中的 Accumulating 的情况.

# Example

TumblingEventTime 窗口, 这里 Watermark 允许 3 秒的乱序, AllowedLateness 允许数据迟到 5 秒.

代码如下:

```scala
package com.atguigu.allowedlateness

import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.scala.function.RichWindowFunction
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.util.Collector

/**
 * <p>Title: </p>
 *
 * <p>Description: </p>
 *
 * @author Zhang Chao
 * @version java_day
 * @date 2020/9/8 6:25 下午
 */
object AllowedLatenessTest {
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val dataStream: DataStream[(String,Long)] = env
      .socketTextStream("localhost", 7777)
      .map(line => { // 数据格式为 key1,1487225040000,1
        val arr: Array[String] = line.split(",")
        (arr(0), arr(1).toLong)
      })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[(String, Long)](Time.seconds(3)) {
        override def extractTimestamp(element: (String, Long)) = element._2
      })

    val resultStream: DataStream[String] = dataStream
      .keyBy(_._1) // dummy key
      .window(TumblingEventTimeWindows.of(Time.seconds(10)))
      .allowedLateness(Time.seconds(5))
      .apply(new CountWindowFunction())

    resultStream.print()

    env.execute()
  }
}

/*
用一个全窗口函数进行测试.
它的主要功能是输出一次计算的 窗口起始和结束时间, 当前窗口计算量, 一个记录 count 的状态
 */
class CountWindowFunction() extends RichWindowFunction[(String, Long), String, String, TimeWindow] {

  lazy val count:ValueState[Long] = getRuntimeContext.getState(
    new ValueStateDescriptor[Long]("count", classOf[Long]))

  override def apply(key: String,
                     window: TimeWindow,
                     input: Iterable[(String, Long)],
                     out: Collector[String]): Unit = {
    val curCount: Long = count.value() + input.size

    count.update(curCount)

    val msg = s"s ${window.getStart} e ${window.getEnd} ws ${input.size} tc $curCount"
    out.collect(msg)
  }
}
```

通过 `nc -lk 7777` 发送数据, 将发送和收到的数据一起展示如下:

```
key1,1487225040000,1
key1,1487225046000,2
key1,1487225050000,3
key1,1487225053000,4	s 1487225040000 e 1487225050000 ws 2 tc 2
key1,1487225055000,5
key1,1487225045000,6	s 1487225040000 e 1487225050000 ws 3 tc 5
key1,1487225048000,7	s 1487225040000 e 1487225050000 ws 4 tc 9
key1,1487225058000,8
key1,1487225049000,9
key1,1487225063000,10	s 1487225050000 e 1487225060000 ws 4 tc 13
```

以每一条数据解释:

1. 收入 `s 1487225040000 e 1487225050000`, 此时 Watermark = 37, 不计算;
2. 收入 `s 1487225040000 e 1487225050000`, 此时 Watermark = 43, 不计算;
3. 收入 `s 1487225050000 e 1487225060000`, 此时 Watermark = 47, 两窗口都不计算. 注意这里是按照 Event time 收入窗口的.
4. 收入 `s 1487225050000 e 1487225060000`, 此时 Watermark = 50, 前窗口到 end of window, 触发计算, 里面有 1, 2 这两条数据. 后窗口因 Watermark 未到 63 而不触发计算.
5. 收入 `s 1487225050000 e 1487225060000`, 此时 Watermark = 52, 后窗口因 Watermark 未到 60 (即最大 Event time 不到 63) 而不触发计算.
6. 此时 Watermark = 52 > 50, Event time = 45 < 50, 而 Watermark < 50 + 5, 即算是迟到但在 AllowedLateness 之内的数据, 收入 `s 1487225040000 e 1487225050000`, 触发计算, 计时前窗口有 1, 2, 6 这 3 条数据. 但是因为重新计算之前的状态变量没有清空, 所以状态变量 count = 2 + 3 = 5;
7. 同 6, 此时 Watermark = 52 > 50, Event time = 48 < 50, 而 Watermark < 50 + 5, 即算是迟到但在 AllowedLateness 之内的数据, 收入 `s 1487225040000 e 1487225050000`, 触发计算, 计时前窗口有 1, 2, 6, 7 这 4 条数据. 但是因为重新计算之前的状态变量没有清空, 所以状态变量 count = 5 + 4 = 9;
8. 收入 `s 1487225050000 e 1487225060000`,  此时 Watermark = 55, 后窗口因 Watermark 未到 60 (即最大 Event time 不到 63) 而不触发计算.
9. 此时 Watermark = 58 - 3 = 55, 已经到延迟时间了, 所以此时进来的这个数据就不算 AllowedLateness 了, 而是真正的未到数据被丢弃或者到侧输出流了, 所以不触发计算.
10. 此时 Watermark = 63 - 3 = 60, 触发中间窗口计算, 但这条数据因为 Event time >= 60, 所以收入 `60~70`  的窗口, 所以中间窗口内一共有 3, 4, 5, 8 这四条数据, 状态 count = 9 + 4 = 13.

# Summary

对于 TumblingEventTime 窗口的累加处理, 很好的一点是及时更新了最后的结果, 但是也有一个令人无法忽视的问题, 即再次触发的窗口, 窗口内的 UDF 状态错了. 这就是我们要关注的问题, 我们要做的是去重操作. 

至于这个去重怎么做, 这个就要看 UDF 中的状态具体要算什么内容了, 例如本例中的 state 只是简单的 sum 累计值, 此时可以在 UDF 中添加一个 hashMap, map 中的 key 就设计为窗口的起始与结束时间, 如果 map 中已经存在这个 key, 则 `state.update()` 时, 就不要再加上 window.size, 而是直接加 1 即可. 

# References

1. [Flink流计算编程--Flink中allowedLateness详细介绍及思考](https://blog.csdn.net/lmalds/article/details/55259718)

 