---
layout: default
---

## 1-2　自连接的用法

### 可重排列、排列、组合
```sql
-- 商品表（名称，价格）
-- 用于获取组合的SQL语句：扩展成3列
SELECT
	P1.name AS name_1,
	P2.name AS name_2,
	P3.name AS name_3  
FROM Products P1, Products P2, Products P3
WHERE P1.name > P2.name   
AND P2.name > P3.name;
```

~将大小，小于用于连表的条件，这是我过去没有想到的。

### 删除重复行*
```sql
-- 用于删除重复行的SQL语句(1)：使用极值函数
DELETE FROM Products P1
WHERE rowid <(
	SELECT MAX(P2.rowid)                   
	FROM Products P2                  
	WHERE P1.name=P2. name                    
	AND P1.price=P2.price
);
-- 用于删除重复行的SQL语句(2)：使用非等值连接
DELETE FROM Products P1
WHERE EXISTS(
	SELECT *                  
	FROM Products P2                 
	WHERE P1.name=P2.name                   
	AND P1.price=P2.price                   
	AND P1.rowid < P2.rowid
);
```

~记录在这里，可能在以后的项目当中会用到。

### 排序*
>下面这段代码的排序方法看起来很普通，但很容易扩展。例如去掉标量子查询后边的+1，就可以从0开始给商品排序，
>
>而且如果修改成COUNT(DISTINCT P2.price)，那么存在相同位次的记录时，就可以不跳过之后的位次，而是连续输出（相当于DENSE_RANK函数）。
```sql
-- 商品表（名称，价格）
-- 排序从1开始。如果已出现相同位次，则跳过之后的位次
SELECT
	P1.name,       
	P1.price,      
	(SELECT COUNT(P2.price)
		FROM Products P2
		WHERE P2.price > P1.price)+ 1 AS rank_1  
FROM Products P1  
ORDER BY rank_1;
```
```sql
-- 排序：使用自连接
SELECT
	P1.name,       
	MAX(P1.price)AS price,       
	COUNT(P2.name)+1 AS rank_1  
FROM Products P1
LEFT OUTER JOIN Products P2    ON P1.price < P2.price
GROUP BY P1.name
ORDER BY rank_1;
```

~我的理解：使用非等值连接时，如果P2有三条记录的价格大于P1某一记录的价格，则表连接后得到三条记录。
