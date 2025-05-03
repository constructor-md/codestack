---
title: SQL执行过程
draft: false
weight: 4
---

# SQL执行过程
## 查询SQL执行顺序
以如下SQL为例：
```
select distinct table1.id as card_id 
from table1
join table2 on table1.id = table2.id
where table1.id < 2
group by card_id
having max(card_id) > 10
order by card_id desc
limit 1, 1;
```

执行顺序如下
1. FROM，查询语句的开始，每个步骤为下一个步骤生成一个虚拟表，作为下一个步骤的输入
    1. 如果是表，直接操作表
    2. 如果是子查询，先执行子查询
    3. 如果要关联表，执行下述JOIN、ON
2. JOIN 关联表，生成笛卡尔乘积虚拟表
3. ON，对JOIN出来的虚拟表进行按条件筛选，并生成一个新虚拟表
4. WHERE，对虚拟表进行按条件筛选，生成一张新虚拟表
5. GROUP BY，将按指定列的值分组，得到新的虚拟表。后续的所有步骤都只能操作被分组的列。
6. AVG,SUM,MAX…，聚合函数 对分组的结果进行计算，不生成虚拟表
7. HAVING，按条件筛选，主要和GROUP BY配合使用。且是唯一一个应用到已分组数据的筛选器。生成新虚拟表
8. SELECT，选择指定列，生成新虚拟表
9. DISTINCT，去重，对上出结果进行去重，移除相同的行。产生新虚拟表。使用GROUP BY后，DISTINCT多余。
10. ORDER BY，按照对指定列升序或降序。返回游标，而不是虚拟表。
11. LIMIT，取出指定行的记录，产生虚拟表并返回结果。Limit m,n表示从第m到第n数据。

