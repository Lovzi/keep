---
title: On和Where的执行语序问题
categories: Hive
tags:
  - Hive
abbrlink: 2157d4fb
date: 2021-03-11 00:00:00
---



下面两条语句，由于On和Where放置位置不一样导致结果不一样

1. where放在外面

```
select be.*, bw.product_cnt as bw_cnt    --最终结果300条数据
from (
  select * from zhidou_bi.czw_beer_city_product_cnt where ds=20200429
) as be 
left join (
    select * from zhidou_bi.czw_baiwei_beer_city_product_cnt where ds=20200429
) as bw
on be.province_name=bw.province_name and be.city_name=bw.city_name
```

1. where放在里面

```
select be.*, bw.product_cnt as bw_cnt   --  最终结果244条数据 
from zhidou_bi.czw_beer_city_product_cnt as be 
left join zhidou_bi.czw_baiwei_beer_city_product_cnt as bw
on be.province_name=bw.province_name and be.city_name=bw.city_name
where bw.ds=20200429 and be.ds=20200429
```

#### 为什么两条sql逻辑相同，结果却不同？

这是由于on 是优先于 where 执行的

##### 第二条sql的执行逻辑

1. be 和 bw 表格做笛卡尔积，此时没有过滤条件（两个表格都做全表扫描，两个表格都有20200428、20200429两个分区），完全按照交叉连接，生成了一个中间表T1
2. on 关键字对中间表T1进行筛选， on表达式成立的行保留，不成立的丢弃,生成了中间表T2
3. where条件继续针对T2进行过滤，Where条件成立的行保留，不成立的丢弃，生成了中间表T3。

这里我模拟第二条Sql的逻辑来执行步骤1、2，即没有where条件生成了一个文件，也就是T2.（MaxCompute无法做笛卡尔积，所以无法生成中间表T1） ,sql如下

```
select *
from 
(select * from zhidou_bi.czw_beer_city_product_cnt where ds <= 20200429) as be 
left join 
(select * from zhidou_bi.czw_baiwei_beer_city_product_cnt where ds<=20200429) as bw
on be.province_name=bw.province_name and be.city_name=bw.city_name
```

[📎测试数据-中间表T2.xlsx](https://www.yuque.com/attachments/yuque/0/2020/xlsx/1320486/1588257113014-a641c44b-bc0d-44f7-9958-52ad98982e7c.xlsx)

之后，我们在excel进行筛选，选出ds都为20200429的分区，恰好等于244条记录，这就说明了，on 先于 where 执行，在on执行之前， 两个表格进行全表扫描之后，做了笛卡尔积。

##### 第一条sql的执行过程

1. 根据where条件从be表取出20200429分区的数据，从bw表取出20200429的数据。
2. 两个分区的数据做笛卡尔积
3. 根据on中字段的关联表达式进行筛选，表达式为true的保留，false的丢弃
4. 得出结果

结果在这[📎先where后On.xlsx](https://www.yuque.com/attachments/yuque/0/2020/xlsx/1320486/1588257793230-3df27c71-c4ee-4950-8aea-56250e27b014.xlsx)j

摘自MaxCompute用户文档 https://www.alibabacloud.com/help/zh/doc-detail/89993.htm

对于如下包含JOIN和WHERE条件的语句

```
(SELECT * FROM A WHERE {subquery_where_condition} A) A
JOIN
(SELECT * FROM B WHERE {subquery_where_condition} B) B
ON {on_condition}
WHERE {where_condition}
```

计算顺序为：

1. 子查询中的`{subquery_where_condition}`。
2. JOIN的`{on_condition}`的条件。
3. JOIN结果集合`{where_condition}`的计算。

对于不同的JOIN类型，过滤语句放在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`中，有时结果是一致的，有时候结果又是不一致的。下面分情况进行讨论

**Left Join**

结论：过滤条件在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`不一定等价。

对于左表的过滤条件，放在`{subquery_where_condition}`和`{where_condition}`是等价的。

对于右表的过滤条件，放在`{subquery_where_condition}`和`{on_condition}`中是等价的。

Left Join的处理逻辑是将左右表进行笛卡尔乘积，然后对于满足ON表达式的行进行输出，对于左表中不满足ON表达式的行，输出左表，右表补NULL。