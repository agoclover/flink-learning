# 大数据项目 Flink 实时旅游平台

**地址**: https://www.bilibili.com/video/BV1rK4y147pE?from=search&seid=13111403999472631834

用户订单明细数据打入 hdfs 用于离线数仓分析, 格式为 parquet 格式的分桶表.

# 用户行为数据实时 ETL

1. 用户交互行为点击事件明细数据实时 ETL 后打入 ES

- 过滤出 action=05 和 eventtYPE=02 的数据; 
- 设置最大乱序数据时间为 5s.

2. 用户页面浏览数据实时 ETL

- 主要是过滤 action=07 和 08 的数据
- 将用户行为中的页面浏览数据明细 --> 打入 ES
- 将用户行为中的页面浏览数据明细 --> 打入 kafka

3. 用户浏览产品列表明细数据实时 ETL

- 过滤出 action 为 05 和 eventType 为浏览和滑动两种事件, 主要是扁平化, target 数组是多个, flatMap
- 设置最大乱序时间 5s
- 将最终结果数据打入 kafka 中

# 订单数据聚合 ETL

1. 计算订单累计订单费用和订单个数

- 不需要维度
- 实时累计 fee 和 orders
- 将实时结果数据打入 es 中

2. 实时订单聚合结果写入 redis 缓存供外部系统使用

- 数据源: travel_ods_orders
- 根据 用户所在地区 (userRegion), 出游交通方式 (traffic) 两个维度, 实时累计 计算 orders, maxFee, totalFee, members
- 并指定窗口计算上述 4 个累积值
- 聚合结果打入 redis

3. 旅游订单业务实时计算

- 需求和 ordersAggCacheHandler 一模一样, 只是将结果数据打入 es 中

4. 基于订单数量和时间出发订单数据统计任务

- 基于条件 (订单数量) 触发订单任务统计
- 根据 traffic, hourtime 维度, 来统计 orders, users, totalFee 三个指标
- 将结果打入 es

5. 自定义状态统计订单数量和订单费用

- 基于时间触发

6. 订单宽表实时统计计算

- 基于产品类型和是否境内外游维度统计
- 结果数据打入 kafka

7. 订单实时统计并排名

- 基于窗口内的订单数据进行统计后的实时排名
- 订单分组维度: 出行方式, productID
- 先根据订单个数, 再根据订单总金额升序