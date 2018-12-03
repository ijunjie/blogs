---
title: "Sqlbasic"
date: 2018-07-15T21:40:24+08:00
draft: true
---

# SQL 拾遗

## NULL

SQL 中包含 NULL 的运算结果都是 NULL，NULL / 0 也是 NULL.

其他语言中会抛出异常的运算，如 5 / 0， 在 SQL 中结果也是 NULL.

## 不等号

标准 SQL 里只有 `<>` 是合法的不等号，`!=` 虽然也有很多 DBMS 支持，但不为标准 SQL 所承认。

## 比较运算符与 NULL

不能对NULL使用比较运算符。要使用专门的 IS NULL 和 IS NOT NULL.

## 三值逻辑

一般的编程语言都是二值逻辑，只有 SQL 是三值逻辑。三值逻辑比二值逻辑多了一个不确定。NULL 通常表示不确定。一行记录的某个字段为 NULL，用一个是否等于某个值的 Predicate 做测试，则既非真，也非假，而是不确定。

不确定和真做 AND 运算时，得不确定；不确定和假做 AND 运算时，得假。
不确定和真做 OR 运算时，得真；不确定和假做 OR 运算时，得不确定。

## COUNT(*)

COUNT函数的结果根据参数的不同而不同。 COUNT(*)会得到包含NULL的数据行数，而COUNT(<列名>)会得到NULL之外的数据行数。

从结果上说，所有的聚合函数，如果以列名为参数，那么在计算之前就已经把 NULL 排除在外了。因此，无论有多少个 NULL 都会被无视。

AVG 函数也不例外地将 NULL 排除。AVG(x) 相当于 SUM(x)/COUNT(x).

## MAX 和 MIN

MAX 和 MIN 不止能应用到数字类型，只要能排序的列即可。

## DISTINCT 用作聚合函数的参数

DISTINCT 可以用在 COUNT 的参数中。如 `SELECT COUNT(DISTINCT product_type) FROM Product;`. 先去重再统计；如果写在外面，`SELECT DISTINCT COUNT(product_type FROM Product;` 就会先计算出数据行数，然后去除完全重复的行。

不仅限于 COUNT 函数，所有的聚合函数都可以使用 DISTINCT.

## GROUP BY 和 WHERE 并用时 SELECT 语句的执行顺序

FROM → WHERE → GROUP BY → SELECT

## SELECT 在执行顺序中的位置

一定要记住 SELECT 子句的执行顺序在 GROUP BY 子句之后， ORDER BY 子句之前。因此，在执
行 GROUP BY 子句时， SELECT 语句中定义的别名无法被识别 A。对于在 SELECT 子句之后执行的 ORDER BY 子句来说，就没有这样的问题了。

## 事务开始语句

实际上，在标准 SQL 中并没有定义事务的开始语句，而是由各个 DBMS 自己来定义的

## 自动提交

每条SQL语句就是一个事务（自动提交模式）。默认使用自动提交模式的 DBMS 有 SQL Server、 PostgreSQL 和 MySQL 等。

## 视图注意事项

尽量避免多重视图。

定义视图时，不要使用 ORDER BY 子句。

## 子查询其实就是一次性视图

## 标量子查询

scalar subquery, 标量子查询就是返回**单一值**的子查询。

## 关联子查询

在细分的组内进行比较时，需要使用关联子查询。

```sql
select sale_price
from Product as a
where sale_price > (SELECT avg(sale_price)
FROM Product as x 	
where x.product_type=a.product_type
group by product_type)
```

## 字符串拼接

在 SQL 中，可以通过由两条并列的竖线变换而成的“||”函数来实现字符串拼接。MySQL 中使用 CONCAT 函数。

## COALESCE 函数

该函数会返回可变参数 A 中左侧开始第 1个不是 NULL 的值。参数个数是可变的，因此可以根据需要无限增加。COALESCE(NULL, 'test', NULL) 返回 test.

## EXISTS 函数

作为EXISTS 参数的子查询中经常会使用SELECT *。