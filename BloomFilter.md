# 布隆过滤器

## 内存占用

生产环境下, 一条数据 1k, 如果是 1 万的 uv, 那是  10m, 可以撑住. 1 亿个, 100G. 

如果使用全窗口函数, 即数据全部收集齐再去重, 这样如果点击量在亿级别的话, 状态要占用 100G, 这是不可能的. 

一种优化的思路是增量聚合函数 +  WindowFunction, 这样状态只需要保存 UV. 但是如果 UV 在 1 亿级别, 比如一条 100 byte, 也就是说这个状态还会占用 10 亿. 这仍然是不能接受的. 所以在这种特殊情况下, 可以考虑使用 概率型的 BloomFilter. 

使用 Trigger + ProcessWindowFunction, 加 Redis 外部存储介质来实现. 

如果用 bitmap 连续存储空间的数组, 1 亿 * 1bit = 10^7 byte = 10^4 kb = 10 m, 大大节省了空间. 

一般 10^8 = 2^7 * 2*20 = 2^27, 比如我们想 4 位占 1 个, 即 2^29, 大约 40m 左右, 即 `2 << 29`. 

如何优化: 

- 优化 hashcode, 
- bitmap 扩容. 

## 包

可以使用 Google Guava18 提供的布隆过滤器, 默认 0.03D 的碰撞概率. 

Guava18 还提供了 Murmur3_128HashFunction 哈希函数, mur 乘, r rotate 反转, 即不停地乘, 不停地反转. 

## 代码实现

思路是增量聚合, 即来一条处理一条, 但是 aggregate 本身支持的操作并不多. 

所以可以考虑直接**使用底层的 ProcessWindowFunction + 自定义的 Trigger 来实现来一条处理一条的布隆过滤器.** 

### Trigger

onElement : fire_and_purge

即来一条数据触发一次计算, 并且清空数据和状态, 因此这样定义之后, 就不能再在 ProcessWindowFunction 中定义状态了. 

### 布隆过滤器自定义实现

位图大小最好是**字节的整倍数**. 

Redis 中不需要提前定义, 

实现一个 Hash 函数. 

```scala
class MyBloomFilter(size: Long) extends Serializable {
  // 指定布隆过滤器的位图大小由外部参数指定（2的整次幂），位图存在redis中
  private val cap = size

  // 实现一个hash函数
  def hash(value: String, seed: Int): Long = {
    var result = 0L
    for( i <- 0 until value.length ){
      result = result * seed + value.charAt(i)
    }
    // 返回cap范围内的hash值
    (cap - 1) & result
  }
}
```

