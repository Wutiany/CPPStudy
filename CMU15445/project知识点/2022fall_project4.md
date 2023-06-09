# C++知识
## 有向图
* 有向图使用 `unordered_map<point, points>` 来记录不同点对应边的集合
## 条件变量
* 用来标记互斥锁，共享锁操作
# concurrency control
## Lock Manager
锁管理器管理所有的锁，同时操作某个事务的锁，需要 `transaction_manager` 的知识，`transaction_manager` 中记录了事务的状态、锁等信息
### transction_manager
* 事务的隔离等级 -- 用来判断所加的锁
* 事务的状态 -- 标志事务当前处于什么状态（2PL 为 扩张、收缩、提交和回滚的状态）
* 各种锁的集合 -- 用来记录改事务锁定的内容
### Lock Manager
* 根据隔离级别，查看是否能授予锁，同时需要结合事务自身的状态。
  * 同时需要根据事务隔离级别判断上锁的模式能否可以被授予
* 加锁的操作为 `request`，每个表维护一个 `request` 的序列，表示这个表被上了什么锁。
  * 同时需要根据这个表的所有加锁状态来判断后续能不能加新的锁，如加了排他锁，后续其他事务就不能加其他的锁了
* `request` 表示一个事务对表或者行的加锁操作，同时表示这个加锁操作是否被授予
* `request_queue` 为了一个 `request` 序列，用来标记当前哪个事务在修改这个表的锁
* 为了检测死锁，要将事务之间等待关系表示成等待图的形式，用成环来判断是否有死锁存在，如果存在要回滚事务
  * 检测死锁使用过 `dfs` 来进行的
## Deadlock Detection
* 获取所有的边，通过回溯以及深度优先遍历的方式，将一条边的节点遍历完，通过集合记录，如果该集合中同时出现了两个相同点，则成换，回溯去将原本没成环的边的点清除
## Concurrent Query Execution
* 修改上一个 `project` 的执行器的并发任务