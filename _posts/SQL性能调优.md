---
title: "SQL性能调优1-索引概述"
tags: mysql 索引
---

# 索引概述

![23索引概述](assets/23索引概述.jpg)

### 索引的种类

按功能逻辑划分：

- 普通索引

  最基础的索引，没有任何约束，主要用于提升查询效率。是非聚集索引。

  ```mysql
  CREATE INDEX indexName ON mytable(username(length)); 
  ```

- 唯一索引

  在普通索引基础上增加了数据唯一性约束，在一张数据表里可以由多个唯一索引，若是组合索引，则列值的组合必须唯一。非聚集索引。

  ```mysql
  CREATE UNIQUE INDEX indexName ON mytable(username(length)) 
  ```

- 主键索引

  在唯一索引的基础上增加了非空约束，主键等价于主键索引。是聚集索引。

  ```sql
  alter table mytable add primary key(username(length))
  ```

- 全文索引

  与普通索引不同，是目前搜索引擎使用的一种关键技术。它能够利用“分词技术“等多种算法智能分析出文本文字中关键字词的频率及重要性，然后按照一定的算法规则智能地筛选出我们想要的搜索结果。

  文章表(article)，其中有主键ID(id)、文章标题(title)、文章内容(content)三个字段。

  ```mysql
  add FULLTEXT INDEX fulltext_article (title, content)
  ```

  检索语法：

  ```sql
  SELECT * FROM article WHERE MATCH(title, content) AGAINST('查询字符串')
  ```

  MySQL自带的全文索引**只能用于数据库引擎为MyISAM的数据表**，如果是其他数据引擎，则全文索引不会生效。此外，MySQL自带的全文索引只能对英文进行全文检索，**目前无法对中文进行全文检索**。如果需要对包含中文在内的文本数据进行全文检索，需要采用Sphinx(斯芬克斯)/Coreseek技术来处理中文。

按照物理实现方式划分：

- 聚集索引

  数据库会按照主键的维度来排列存储数据，一个表只能有一个聚集索引。聚集索引下数据的查询效率非常高，但是对数据进行插入，删除，更新等操作，效率会比非聚集索引低。直接指向数据文件。

- 非聚集索引

  不影响数据表的物理存储顺序，一个表可以有多个非聚集索引，占据单独的存储空间，索引项按照顺序存储，指向的内容随机分布。索引直接指向的是数据的位置，而不是数据文件本身。

按照字段个数划分：

- 单一索引

  索引列为一列

- 联合索引

  多个列组合在一起创建的索引

  ```mysql
  CREATE INDEX nameAndId ON user_gender(user_name, user_id);
  ```

  联合索引存在最左原则：按照最左优先的方式进行索引的匹配。

  

### 实验验证

#### 什么时候用索引：

陈旸在百度网盘上提供了数据表，heros_without_index.sql 和 heros_with_index.sql，提取码为 wxho。

- 数据行数少时，索引的效率

  heros 数据表一共有 69 个英雄，数据量很少。当我们对 name 进行条件查询的时候，我们观察一下创建索引前后的效率。

  ```
  [SQL]SELECT id, name, hp_max, mp_max FROM heros_with_index WHERE name = '刘禅'
  
  受影响的行: 0
  时间: 0.001s
  ```

  ```
  [SQL]SELECT id, name, hp_max, mp_max FROM heros_without_index WHERE name = '刘禅'
  
  受影响的行: 0
  时间: 0.000s
  ```

  额，时间太短了看不出区别来。。。。。

  理论上，在数据量不大的情况下，有无索引确实区别不大。

- 性别男女字段的索引效果

  一般如果一个字段的取值很少，不需要创建索引，但是当不同值的数量很悬殊时，例如某一个值只占10万分之一，索引也能提升效率。

  女儿国的人口数据表 user_gender 见百度网盘中的 user_gender.sql。其中数据表中的 user_gender 字段取值为 0 或 1，0 代表女性，1 代表男性。

  总人口100万，10个男人，其余为女人（我想去hh）。

  筛选男性：

  ```mysql
  [SQL]SELECT * FROM user_gender WHERE user_gender = 1
  
  受影响的行: 0
  时间: 0.613s
  ```

  相比上一个测试慢了几个数量级，创建索引：

  ```mysql
  CREATE INDEX gender ON user_gender(user_gender);
  ```

  测试查询效率：

  ```mysql
  [SQL]SELECT * FROM user_gender WHERE user_gender = 1
  
  受影响的行: 0
  时间: 0.006s
  ```

  查询女性（带索引）：

  ```mysql
  [SQL]SELECT * FROM user_gender WHERE user_gender = 0
  
  受影响的行: 0
  时间: 4.622s
  ```

  查询女性（不带索引）：

  ```mysql
  [SQL]SELECT * FROM user_gender WHERE user_gender = 0
  
  受影响的行: 0
  时间: 0.666s
  ```

  这种情况下使用索引反而导致了效率大幅度下降。

- 聚集索引和非聚集索引的查询效率

  还是user_gender 表，设置user_id为主键，为其创建主键索引，试试查询效率：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE user_id = 900001
  
  受影响的行: 0
  时间: 0.001s
  ```

  同样，user_name也是数据的唯一属性，无索引时的查询效率：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE user_name = 'student_890001'
  
  受影响的行: 0
  时间: 0.656s
  ```

  为其创建普通索引，也就是非聚集索引：

  ```mysql
  CREATE INDEX name ON user_gender(user_name);
  ```

  非聚集索引查询效率：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE user_name = 'student_890001'
  
  受影响的行: 0
  时间: 0.002s
  ```

  可以看到，对where字段建立索引大大提升了查询效率，聚集索引比非聚集索引查询效率要高一些。

- 联合索引

  ```mysql
  CREATE INDEX nameAndId ON user_gender(user_id, user_name);
  ```

  看看联合主键的查询效率：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE user_id = 900001 AND user_name = 'student_890001'
  
  受影响的行: 0
  时间: 0.001s
  ```

  嗯，也很给力。看看最左原则：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE user_id = 900001 
  
  受影响的行: 0
  时间: 0.000s
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE  user_name = 'student_890001'
  
  受影响的行: 0
  时间: 0.902s
  ```

  联合主键可以调整顺序，之前创建的时候，id在前，直接用id查和2列一起查效率几乎一样，但是直接用name查效果就差了很多。这就是联合主键的最左原则。

  如果将name调整到id之前，那么name就能匹配最左原则，查询效率就会反过来：

  ```mysql
  [SQL]SELECT user_id, user_name, user_gender FROM user_gender WHERE  user_name = 'student_890001'
  
  受影响的行: 0
  时间: 0.001s
  ```

  

