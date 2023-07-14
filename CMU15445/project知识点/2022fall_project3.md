# C++知识

# Query Execution
## Executor
`Executor` 由 `Plan` 和 `Expression`
## PlanNode
* 通过将 `SQL` 语句转换成计划树，通过计划器获取符合表达式的数据，`PlanNode` 需要 `Expression`
## Expression
* 获取满足表达式的值
  * 基础表达式
    * 常量表达式
    * 列值表达式
  * 逻辑表达式
  * 比较表达式
  * 算法表达式
* 非基础表达式依赖于基础表达式来获取更底层的值
## Executor
* 相当于一个执行器工厂，根据不同的执行来获取 `PlanNode` 的结果
