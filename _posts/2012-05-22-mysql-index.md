---
layout: post
title: "关于 mysql 索引优化的建议"
date: 2018-06-06
excerpt: "关于 mysql 索引优化的建议。"
tags: [技术文档]
comments: false
feature: http://i.imgur.com/Ds6S7lJ.png
---
## 什么时候需要建立索引

- 检索数据过多
- 经常检索

## 建立索引需要注意什么

- 字段重复值少
- 字段不建议长字符串
- 字段设置 not null，给默认值
- 遵循最左策略


- 删除该表其他冗余索引
- 表更新/删除不频繁
- 字段尽量使用 varchar 代替 char，占用空间小
- 每条 sql 只使用一个最佳索引
- 一个表的索引不要超过 6 个，当索引的维护成本过高，不建议使用索引
- 组合索引要考虑字段的优先顺序

## sql 编写主要注意什么

- 以下情况索引会失效：
    - <> / != / or / null / in / not in
    - sql 运算
    - 索引列的值大量重复，不会去使用该索引
    - like '%%' 无效，like 'test%'有效，最左策略
    - 不使用 or 条件，尽量使用 union all
    - not in 和 in，尽量使用 between
    - 使用表达式
- 其他优化建议
    - 不使用 JOIN，用子查询替代
    - 避免使用 *

## 定期检测

- 定期分析表和检查表 analyze, check, optimize
    - 分析表

    ~~~
    sql：
    analyze table  表名 1 [, 表名 2…]
    作用：
    对表的定期分析可以改善性能，且应该成为常规维护工作的一部分。
    因为通过更新表的索引信息对表进行分析，可改善数据库性能。
    ~~~

    - 检查表

    ~~~
    sql：
    check table  表名 1 [, 表名 2…] [option]
    其中，option 有 5 个参数，分别是 QUICK、FAST、CHANGED、MEDIUM 和 EXTENDED。这 5 个参数的执行效率依次降低。 option 选项只对 MyISAM 类型的表有效，对 InnoDB 类型的表无效。
    作用：
    能够检查 InnoDB 和 MyISAM 类型的表是否存在错误。
    而且，该语句还可以检查视图是否存在错误。
    ~~~

    - 优化表

    ~~~
    sql：
    optimize table  表名 1 [, 表名 2…]
    作用：
    可以消除 delete 和 update 造成的磁盘碎片，从而减少空间的浪费。
    ~~~

- explain 分析查询语句

    ~~~
    explain [sql 语句]
    ~~~

- 越小的列越快，使用 procedure analyse() 分析字段类型建议，数据越多越精

    ~~~
    select * from table_name procedure analyse();
    ~~~
