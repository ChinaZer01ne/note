## P0面试优先级（2026）
- `P0-1` 执行主线：MapperProxy -> Executor -> StatementHandler。
- `P0-2` 参数与 SQL：`#{}` 预编译防注入、`${}` 文本拼接风险。
- `P0-3` 缓存机制：一级缓存作用域、二级缓存失效策略。
- `P0-4` 性能优化：批量写、N+1 查询规避、分页与索引配合。
- `P0-5` 可维护性：动态 SQL 复杂度控制、SQL 审核与日志。

## P0常见误区修正（本页已修订）
- `${}` 不是“更灵活就更好”，它会带来 SQL 注入风险。
- 二级缓存不是默认收益，更新频繁场景可能适得其反。
- ORM 问题本质仍是 SQL 问题，最终要回到执行计划分析。


## 重要组件
* SqlSessionFactory
* SqlSession
* Executor
* ParameterHandler
* StatementHandler
* ResultSetHandler

![](mybatis执行流程图.jpg)

## MyBatis面试补漏
- 执行流程：MapperProxy -> SqlSession -> Executor -> StatementHandler -> JDBC。
- `#{}` 使用预编译参数，占位安全；`${}` 是字符串拼接，需白名单约束。
- 一级缓存作用域是 SqlSession；二级缓存作用域是 Mapper，更新语句会触发失效。
- 常见性能问题：N+1 查询、批量写未使用 BATCH、分页未命中索引。