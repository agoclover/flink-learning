批流一体: 

实时销量数据, 展示数据, 风险把控, 离线更复杂的计算, 比如 BI. 离线需要给实时更友好的支持. 

分布式存储和数据分层的架构

架构, 数据流转, 开发

![image-20201031160514697](https://strawberryamoszc.oss-cn-shanghai.aliyuncs.com/projects/rt/image-20201031160514697.png)

1. 无法一边写入一边读取. 

2. 离线 t+1, 全国上千数据库凌晨抽取到大数据机房, 很可能得不到保障. 稳定性是离线计算的前提

3. hive 不支持 update delete

4. 离线和实时两条链路, 业务逻辑双重, 可能不同的框架, 可能人为原因可能离线和实时对不上, 需要逐级排查. 

   ETL 分析师也会困扰. 

![image-20201031161149229](/Users/amos/Library/Application%20Support/typora-user-images/image-20201031161149229.png)

![image-20201031161359415](https://strawberryamoszc.oss-cn-shanghai.aliyuncs.com/projects/rt/image-20201031161359415.png)

关心点: 批流一体华

- acid
- etl 代码逻辑, 脏数据, 回滚一般都是伏写

hudi: merge on read, 组建索引设计

iceberg: 代码抽象程度高

sql 查询, 减少不相关数据的查询. data skip 静态分区裁剪. Spark 3.0 动态分区, 列裁剪.

dfp 脏数据

不一定一层 delta lake 跟业务逻辑有关

一层 ods, 二层清洗, 三层关联位表构建宽表, 第四层聚合

delta解决的并不是 

数据湖接 hdfs, 是主要是流批一体. 

数据湖仅仅是数据控制层. 


