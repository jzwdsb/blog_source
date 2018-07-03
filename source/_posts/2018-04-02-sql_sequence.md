---
layout: post
category: 数据库
date: 2018-04-02
title: SQL 语句中的执行顺序
---

　　sql 语句的执行步骤

1. 语法分析: 分析语句的语法是否符合规范，衡量语句中各表达式的意义
2. 语义分析: 检查语句中设计的所有数据库对象是否存在，且用户有相应的权限
3. 试图转换: 将涉及视图的查询语句转换为相应的对基表查询语句
4. 表达式转换: 将复杂的 SQL 表达式转换为较简单的等效连接表达式
5. 选择优化器: 不同的优化器一般产生不同的执行计划
6. 选择连接方式: 不同的数据库有不同的连接方式，对于多表连接可选择适当的连接方式
7. 选择连接顺序: 对多表连接选择哪一对表先连接，选择这两个表中哪个表做为源数据表
8. 选择数据的搜索路径: 根据以上条件选择合适的数据搜索路径，如是选用全表搜索还是利用索引或是其他的方式
9. 运行执行计划

　　sql 语句的执行顺序与其他语言不同，SQL 语句并不是按照书写顺序执行，一个被执行的语句是 from 子句，虽然 select 通常写在最前，但是几乎在最后执行.<br>

```sql
(8)select  (9)distinct (11)<Top Num> <select list>
(1)from [left_table]
(3)<join_type> join <right_table>
(2)     on <join_condition>
(4)where <where_condition>
(5)group by <group_by_list>
(6)with <cube | RollUP>
(7)having <having_condition>
(10)order by <order_by_list> 
```

　　SQL 语句完整的执行顺序

1. from 子句组装来自不同数据源的数据
2. where 子句基于指定的条件对记录行进行筛选
3. group by 将数据划分为多个分组
4. 使用聚集函数进行计算
5. 使用 having 子句筛选分组
6. 计算所有的表达式
7. select 的字段
8. 使用 order by 对结果集进行排序

　　每个步骤都会产生一个虚拟表，该虚拟表被用作下一个步骤的输入．这些虚拟表对调用者不可用．指示最后一步生成的表才会返回给调用者．如果没有在查询中指定某一子句，将跳过相应的步骤.

　　逻辑查询处理阶段简介:
1. from

    对 from 子句中的前两个表执行笛卡尔积(交叉联接)，生成虚拟表VT1.
2. on

    对 VT1 应用 on 筛选器，只有那些使条件为真的才插入到 VT2
3. outer(join)

    如果指定了 outer join(相对于 cross join 或者 inner join)，保留表中未找到匹配的行将作为外部行添加到 VT2，生成 VT3.如果 from 子句中包含两个以上的表，则对上一个联接生成的结果表和下一个表重复执行步骤１到步骤３，直到处理完所有的表为止
4. where

    对 VT3 应用 where 筛选器，只有使为 true 的行才插入到 VT4
5. group by

    按 group by 子句中的列列表对 VT4 中的行进行分组，生成 VT5
6. cute|rollup

    把超组插入到 VT5，生成 VT6
7. having

    对 VT6 应用 having 筛选器，只有使为 true 的插入到 VT7
8. select

    处理 select 列列表，生成 VT8
9. distinct

    将重复的行从 VT8 中删除，产生 VT9
10. order by

    将 VT9 中的行按 order by 子句中的列列表排序，生成一个游标 VC10
11. top

    从 VC10 的开始处选择指定数量或比例的行，生成表 VT11,并返回给调用者
