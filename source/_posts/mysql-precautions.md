---
title: MySQL注意事项
date: 2019-12-03 11:03:07
categories: 
- MySQL
tags:
- MySQL
description: SQL语句的注意事项
---
1. Limit偏移量大时可以先单独查询Id再关联，或者记录上次查询的最大Id。

2. 对于MAX()和MIN()函数可以考虑使用ORDER BY和LIMIT 1代替。

3. 确保GROUP BY和ORDER BY只涉及一个表中的字段，这样才可能使用索引。

4. GROUP BY会自动按照分组的字段升序排序，可直接使用DESC和ASC排序，使用ORDER BY NULL不进行排序。

5. GROUP BY * WITH ROLLP得到每个分组以及每个分组汇总级别的值。

6. 一旦使用DISTINCT和GROUP BY需要产生临时表。

7. 如果无需去重请使用UNOIN ALL，UNION如果需要排序只需要在最后一条语句上使用ORDER BY。

8. 你的SELECT语句中有一系列复杂的OR条件么？通过使用多条SELECT语句和连接它们的UNION语句，能看到极大的性能改进。

9. 如果数据检索是最重要的，则你可以通过在INSERT和INTO之间添加关键字LOW_PRIORITY，指示MySQL降低INSERT语句的优先级，如`INSERT LOW_PRIORITY INTO`。
