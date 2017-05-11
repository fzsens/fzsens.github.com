#### Canal
Canal Server模拟MySQL的Slave服务器，dump MySQL的binlog并解析成为事件，通过Canal Client读取，用于各种操作。可能的应用场景有：

1. 与共享缓存或者搜索引擎的集成实现数据同步
2. 事件驱动业务
3. 数据库同步

##### 实例
SpringCanalInstanceGenerator通过Spring来对CanalInstance进行初始化。
