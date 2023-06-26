# C++知识
## 读写锁
* 读锁直接使用`std::mutex`的`lock`方法进行锁定
* 写锁使用共享锁操作
## 存储
通过数组指针存储的数据，可以通过一下方式进行存储的操作
* std::memcpy
* std::move
* std::move_backward
* std::copy
## Transction
事务的重要作用是管理一次事务操作的锁，用来实现蟹式锁
## Iterator
可以通过数组的下标进行实现
# BPlusTree
通过对数据库表的`key`构建对应的`pair<key, page_id>`,来记录键对应磁盘的`page`<br>

**`b+tree` 需要设计的时候重点注意的点：**
* 叶子节点和中间节点记录的最大值，最小值问题
* 因为`internalNode`实现的问题，指向`leafNode`的指针改成了记录，而不是使用指针进行指向，所以`record`的数量要比`leafNode`的多一个
* 由于`internalNode`实现的问题，`MaxSize`从节点记录的最大个数，变成了指针个数（仍旧是记录，但是代表的是指针）
* 增删查记录的问题：这三种操作都去找去查找记录所在叶节点的位置，但是查找到的位置却又不同
* 增加记录的问题
  * 叶节点满了的问题
* 删除节点的问题
  * 删除的是根节点
  * 删除的节点不满足`MinSize`
* 以上两种需要面临数据重分配和数据整合的问题
* 在操作的时候需要面临蟹式锁的问题，子节点安全，释放所有祖先节点的锁

**`b+tree`主要的逻辑**
* 叶节点存储`record`（pair<name, page_id>），一个叶节点有多个`record`，按顺序排列，第`0`个`record`为中间节点的所指向的`record`，中间不使用`record+pointer`的形式，而是通过`record`来指向子节点的`0`号`record`，因此中间节点的最大记录个数就比叶子节点多一个。`BPlusTree`是组织操作叶节点和中间节点的类
* 存储逻辑每个节点都有一个 `pair<key, value>` 的列表，用来存储记录
* 节点的`page_id`是通过缓存池获取的,这个`page_id`是构建树的关键，子节点通过指向父节点的`page_id`来向上寻找
* 中间节点插入的`pair`与子节点不同，子节点的`pair`是记录的`name`以及对应的磁盘中的位置，而磁盘插入的是子节点第`0`的记录的`key`以及对应节点的`page_id`，通过这种方式向下进行寻找子节点
  * 记录的结构是 `pair<key, value>`  `key`:键值， `value`:磁盘的该记录的`page_id`
  * 叶子节点中的记录就是记录本身
  * 中间节点为了寻找叶子节点，同时并不记录记录的`value`，所以存储的记录结构是 `pair<key, child_node_page_id>`，通过`child_node_page_id`向下寻找叶子，同时因为`page_id`是通过缓存池获取的，所以操作中间节点记录的时候还需要缓存池
* 主要的搜索方法以及树结构调整的方法都在`BPlusTree`中
## BPlusTreePage
`leafPage`和`internalPage`继承的类，有`node`的基本行为函数
### 基本功能
* `parent`节点的修改和记录
* 当前节点属性的设置
* 判断当前节点类型
## BPlusTreeLeafPage
叶子节点需要根据插入和删除操作来移动`record`
### 基本功能
* 将`record`进行移动
  * 基于相邻节点，之移动一个，移动第一个或者最后一个
  * 移动全部
  * 移动一半，用于节点满了，然后创建一个新节点，将满的节点的record进行平分
  * 复制record
* 插入，不需要缓存池
## BPlusTreeInternalPage
叶子节点和中间节点有很多不同，中间节点的操作需要调用缓存池来记录page_id，因为中间节点的每个record都是一个叶节点，都是通过缓存池获取的page_id，当如到缓存池的帧中
### 基本功能
* 与叶子节点类型，不过操作record的时候需要调用缓存池
* 根节点插满了，需要扩展的情况
## Iterator
* 因为所有叶子节点是通过链表连接的（逻辑上），实际上是通过page_id进行连接的，所需可以顺序访问所有叶节点，因此需要为叶节点封装iterator
## Concurrent Index
* 在删除，查询，插入的操作中，为了提高并发性，使用蟹式锁，更细粒度的锁节点
# 调试经验
## BPlusTree可视化
* 下载可视化工具：https://graphviz.org/download/
* 使用代码生成树的`dot`文件
```shell
$ # To build the tool
$ mkdir build
$ cd build
$ make b_plus_tree_printer -j$(nproc)
$ ./bin/b_plus_tree_printer
>> ... USAGE ...
>> 5 5 // set leaf node and internal node max size to be 5
>> f input.txt // Insert into the tree with some inserts 
>> g my-tree.dot // output the tree to dot format 
>> q // Quit the test (Or use another terminal) 
```
* 使用命令将dot文件生成可视化的图片
```shell
dot -Tpng -O my-tree.dot
```
