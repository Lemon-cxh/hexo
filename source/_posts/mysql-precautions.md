---
title: MySQL注意事项
date: 2019-12-03 11:03:07
keywords: MySQL
categories: 
- MySQL
tags:
- MySQL
description: SQL语句的注意事项
---
#### 数据模型与索引设计

1. ##### 字符集

    一般使用`utf8`，需要使用表情时使用`utf8mb4`。排序规则一般使用`utf8_general_ci`。
    排序规则：
    - `_cs`：区分大小写
    - `_ci`：不区分大小写
    - `_bin`：以二进制数据存储，且区分大小写

2. ##### 索引

   1. ###### 最左前缀匹配原则

       > 联合索引可以用于包含索引中所有列的查询条件的语句, 或者包含索引中的前N列

   2. ###### 覆盖索引

       > 从非聚簇索引中就能查到的记录，而不需要查询聚簇索引中的记录，避免了回表的产生减少了树的搜索次数，显著提升性能。

       - 聚簇索引：按创建索引时指定的列的顺序物理地排列表内容
       - 非聚簇索引：由索引键和指定数据块的标记组成

   3. ###### 索引列设置为NOT NULL,否则会全表扫描

#### 查询数据

1. ##### 过滤与查找数据

    1. ###### 查找不匹配或缺失的记录

        > 在子查询返回结果集大的时候使用`EXISTS`通常比`IN`更快，另一种是`LEFT JOIN`结合`WHERE`。

        - `EXISTS`：数据库引擎在找到第一行会停止运行子查询
        - `IN`：数据库引擎会检索所有行

    2. ###### 按时间范围正确地过滤日期和时间

        1. 不要依赖隐式日期转换，使用显式转换函数`CONVERT()`来处理日期字符。
        2. 不要将函数应用于日期和时间列，否则查询不能使用索引。
        3. 舍入误差可能导致日期和时间值不正确。使用`>=`和`<`代替`BETWEEN`。可以使用`DATEADD()`

    3. ###### 书写可参数化搜索的查询以确保引擎使用索引

        1. 使用`=`、`>`、`<`、`>=`、`<=`、`BETWEEN`、`ISNULL`、`LIKE`(不包括前导通配符)。
        2. 不要再`WHERE`子句中的一个或多个字段上使用函数。
        3. 不要对`WHERE`子句中的字段进行算术运算。

2. ##### 聚合

    1. ###### 理解GROUP BY工作原理
    
        1. `FROM`子句生成数据集
        2. `WHERE`子句过滤由`FROM`子句生成的数据集
        3. `GROUP BY`子句聚合由`WHERE`子句过滤的数据集
        4. `HAVING`子句过滤由`GROUP BY`子句聚合的数据集
        5. `SELECT`子句转换过滤的聚合数据集
        6. `ORDER BY`子句对变换后的数据集进行排序

    2. ###### 使用OUTER JOIN时避免获取错误的COUNT()

        > 数据库版本没有`RANK()`函数时，可以使用`COUNT()`子查询来实现。

        1. 使用`COUNT(*)`来统计所有记录的总数，也包括空值的记录。
   
        2. 使用`COUNT()`仅统计列值不为NULL的记录的总数。

        3. 有时一个子查询甚至一个关联子查询也会比`GROUP BY`更有效率。

    > 关联子查询: 子查询中的部分条件依赖于外部查询中正在处理的当前记录值。

    3. ###### 避免使用GROUP BY来查找最大或最小值

        > 使用`LEFT JOIN`自身以及`WHERE`来查询，`ON`子句的列添加索引

    4. ###### GROUP_COUNT()中也可以使用ORDER BY

2. ##### 排序

   1. ###### 排序时对Null值的处理

        对查询结果进行排序，但该字段可能为Null，需要将Null值排到前面或后面。

        ```sql
        SELECT
            column1,
            CASE WHEN ISNULL(column1) THEN 0 ELSE 1 END AS is_null
        FROM
            table1
        ORDER BY
            is_null, column1
        ```

   2. ###### 依据条件逻辑动态调整排序项
    
        按照某个条件逻辑来选择不同的排序列。

        ```sql
        SELECT
            column1
        FROM
            table1
        ORDER BY
            CASE WHEN column1 = 'a' THEN column2 ELSE column3 END
        ```
#### Explain type字段

- `system`

    表只有一行，这是一个`const` type的特殊情况

- `const`

    最多只有一行匹配

- `eq_ref`

    只匹配到一行的时候。除了`system`和`const`之外，这是最好的连接类型了。当我们使用主键索引或者唯一索引的时候，且这个索引的所有组成部分都被用上，才能是该类型

- `ref`

    触发联合索引最左原则，或者这个索引不是主键，也不是唯一索引

- `fulltext`

    使用全文索引的时候才会出现

- `ref_or_null`

    这个查询类型和`ref`很像，但是 MySQL 会做一个额外的查询，来看哪些行包含了NULL

- `index_merge`

    在一个查询里面很有多索引用被用到，可能会触发index_merge的优化机制

- `unique_subquery`

    此类型将`eq_ref`替换为IN形式的某些子查询。
    unique_subquery只是一个索引查找函数，可以完全替换子查询以提高效率

- `index_subquery`

    类似于unique_subquery，但是它在子查询里使用的是非唯一索引
    
- `range`

    只有给定范围内的行才能被检索，使用索引来查询出多行。 输出行中的类决定了会使用哪个索引。 key_len列表示使用的最长的 key 部分。 这个类型的ref列是NULL

- `index`

    index类型和ALL类型一样，区别就是index类型是扫描的索引树。以下两种情况会触发：
    1. 如果索引是查询的覆盖索引，就是说索引查询的数据可以满足查询中所需的所有数据，则只扫描索引树，不需要回表查询。 在这种情况下，explain 的 Extra 列的结果是 Using index。仅索引扫描通常比ALL快，因为索引的大小通常小于表数据。
    2. 全表扫描会按索引的顺序来查找数据行。使用索引不会出现在Extra列中。

- `ALL`

    全表扫描。通常，可以通过添加索引来避免ALL。

#### 注意

1. Limit偏移量大时可以先单独查询Id再关联，或者记录上次查询的最大Id。

2. 对于MAX()和MIN()函数可以考虑使用ORDER BY和LIMIT 1代替。

3. 确保GROUP BY和ORDER BY只涉及一个表中的字段，这样才可能使用索引。

4. GROUP BY会自动按照分组的字段升序排序，可直接使用DESC和ASC排序，使用ORDER BY NULL不进行排序。

5. GROUP BY * WITH ROLLP得到每个分组以及每个分组汇总级别的值。

6. 一旦使用DISTINCT和GROUP BY需要产生临时表。

7. 复杂的`OR`条件可以使用多条`SELECT`语句和连接它们的`UNION`语句。
   
8.  如果无需去重请使用`UNOIN ALL`，`UNION`如果需要排序只需要在最后一条语句上使用`ORDER BY`。
   
9.  如果数据检索是最重要的，则你可以通过在INSERT和INTO之间添加关键字LOW_PRIORITY，指示MySQL降低INSERT语句的优先级，如`INSERT LOW_PRIORITY INTO`。

---

参考：
- Effective SQL 编写高质量SQL语句的61个有效方法
- Java工程师修炼之道