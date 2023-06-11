# C++知识
## 


## 经验


****
# Buffer Pool
缓冲池负责主存和磁盘之间来回移动物理页。<br>
需要实现 **线程安全** 的缓冲池
## 可扩展哈希表
### 可扩展哈希主要原理
参考：https://www.geeksforgeeks.org/extendible-hashing-dynamic-approach-to-dbms/ <br>
* 可扩展哈希的存储原理：使用分裂桶进行数据的存储
    * 目录：以指针的形式存储桶的地址。为每个目录分配一个 ID，每次进行扩容时，该 ID 可能会更改。其本质就等同于一个目录对应着所有 `hash` 值相同的数据，使用目录所谓索引
    * 桶：具体存储数据的容器，`hash` 值相同的数据存储到一个桶中，但是桶本身有深度（存储数据的容量），超出深度就需要扩容，然后对桶内数据进行重分配，因为目录扩容的时候会导致 hash 方案的改变
    * 深度：用于扩容和 `hash` 计算
      * 全局深度：用来对目录进行分配，`hash` 策略是数据转换成二进制后的 **后（全局深度[2]位）**
      * 局部深度：表示桶的个数，是否扩容
    * 扩容有两种主要的情况：
      * 1.桶溢出，同时局部深度（即扩容过的桶的个数）== 全局深度，就需要对目录进行扩容，目录扩容，对目录的 `hash` 策略进行更改：全局深度+1，`hash` 位数+1，然后对扩容桶的内容重新进行 hash，桶的数目也+1（即local depth），更改目录指向。
      * 2.桶溢出，同时局部深度 < 全局深度，只需要增加一个桶，更改目录指向即可，然后对数据进行重 `hash`
      * 注意事项：没有溢出的桶，即便目录进行了扩容，扩容后的目录也仍旧指向原来的桶

<div style="text-align: center"><img height="500" src="H:\Github\CPPStudy\CMU15445\src\img.png" width="500"/></div>

* 可扩展哈希操作流程：
  1. 分析数据类型
  2. 转换成而进行，通过 `global depth` 对数据进行 `hash` 操作
  3. 通过 hash 值确定目录
  4. 根据目录进入桶
  5. 分析 `global depth` 与 桶的 `local depth`,查看是否需要扩容，如果需要扩容，就判断上述两种情况是哪一种
  6. 根据溢出的问题做两种操作，最后对溢出桶进行在重 `hash` 操作
  
* 需要解决`目录`与`桶`之间对应关系如何存储
* 需要解决扩容的时候，`目录`与`桶`之间对应的关系如何更新
* 需要解决如何根据`key`获取桶的`index`

## LRU-K 替换策略
## 缓冲池管理器实例

# 调试经验
## 单测
### extendible_hash_table
* `extendible_hash_table` 在实例化的时候会设置 `bucket size`
    * 初始化实例的 bucket_size_
    * bucket 使用vector 存储，因此目录即使用下表来进行获取：`hash` 结果与 `下标` 相关
    * 初始化的时候，要实例化 bucket
* `table->find(key, result)` 会将结果写入 `result`
* `table->GetLocalDepth(0)` `bucket` 是使用 `vector` 存储的，这个方法会返回下表为 `0` 的 `bucket` 的 `local depth`
* `table->Remove(8)` `remove` `key`，返回 `true` 或者 `false`
* 