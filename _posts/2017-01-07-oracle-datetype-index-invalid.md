---
layout: post
title: Timestamp导致的Oracle索引失效
date: 2017-01-07
categories: database
tags: [database]
description: 记录JDBC传递参数Timestamp类型导致的Oracle索引失效问题
---

问题的起因是DBA通知说在一套之前上线的系统中，对日期创建索引无法生效，导致大量的日期查询效率无法通过创建索引得到提升。通过分析得出如下的结论：

在JDBC中一般通过`prepareStatement`的各种`setXxx()`来传递参数，针对时间类型默认提供了三中类型，分别传递不同的时间格式：`java.sql.Date`用于传递日期类型，不带时分秒信息；`java.sql.Time`用于传递时间类型，但是不带年月日等日期信息；`java.sql.Timestamp`用于传递时期和时间信息，并将时间精确到毫秒。在实际的系统设计中，业务层面一般需要的信息精度为`YYYY-MM-DD HH:MM:SS`，但是对这种最常见的格式，JDBC却没有提供默认的格式支持。因此为了保证精度的准确性，一般会安全地使用`Timetamp`类型进行参数的传递。

而在Oracle数据库中，却可以使用`Date`类型来保持`YYYY-MM-DD HH:MM:SS`的格式，而在我们的系统中，几乎所有的时间都使用了`Date`类型作为列的存储类型。为时间创建的索引也是`Date`类型的索引。

> Date类型的实际上就是Timestamp(0)，即最后的毫秒精度保持为0

那么JDBC使用`Timestamp`而Oracle使用`Date`类型会导致什么问题？答案是Oracle为了保证兼容性，会出发`INTERNAL_FUNCTION`默认将列中的`Date`类型转换为`Timestamp`类型，类型绑定的转换也带来索引的失效。

明白了问题的根源解决的办法也就比较好得出：

- 修改数据库的类型为`Timestamp`类型，使得参数绑定和数据库字段类型保持一致。但是这种修改需要大量调整数据库的字段类型，同时业务上也确实不需要毫秒级的精度要求，因此这个方案被排除
- 修改入参数为`Date`类型，这边的`Date`类型，并非指`java.sql.Date`，因为上面的分析已经交代，`java.sql.Date`无法进行时分秒的参数传递，而是修改为Oracle的`Date`类型，将参数转换为字符类型，并在SQL语句中使用`to_date(bind_value,"yyyy-mm-dd hh24:mi:ss")`进行绑定，这个修改方案是我最初推荐的方案，这样可以将修改的影响范围限制在局部的SQL，规避整体修改带来的不确定性，但问题也很明显，就是工作量的代价和时间上的要求。提出之后因为这些原因被否定。

后面在`ojdbc6`驱动中找到了`OraclePreparedStatement`，并发现其中允许传入参数`oracle.sql.DATE`，猜测Oracle提供的驱动中提供了和数据库类型一一对应的Java类型参数，如果通过`OraclePreparedStatement`来对数据库进行访问，传入`oracle.sql.DATE`应该就可以达到目的。

因为使用的ORM框架为Hibernate，因此在参数进行设置值的时候进行了一个简单的转换，将`java.sql.Date`转换为`oracle.sql.DATE`，并通过`PreparedStatement`接口的`setObject()`方法设置之后完成了这部分功能的调整。其他的ORM框架也可以通过找到`PreparedStatement`接口设值的方式之后，对症下药进行相应的调整。

后记：虽然问题最后是通过整体修改的方式得到解决，但是在条件允许的情况下还是建议通过规范数据库的设计和标准的SQL编写方式，来提前规避这类的问题。因为全局性的调整，带来的是最大的不确定性和风险，同时也带来测试成本的提高。



参考资料：

http://stackoverflow.com/questions/6612679/non-negligible-execution-plan-difference-with-oracle-when-using-jdbc-timestamp-o





