---
layout: default
---

## 1-4　HAVING子句的力量
### 用HAVING子句进行子查询：求众数
>众数，指的是在群体中出现次数最多的值。
>
>思路是将收入相同的毕业生汇总到一个集合里，然后从汇总后的各个集合里找出元素个数最多的集合。
```sql
-- 毕业生表（姓名，收入）
-- 求众数的SQL语句(1)：使用谓词
SELECT
	income,
	COUNT(*)AS cnt
FROM Graduates
GROUP BY income
HAVING COUNT(*) >= ALL(
	SELECT COUNT(*) FROM Graduates GROUP BY income
);
-- 求众数的SQL语句(2)：使用极值函数
SELECT
	income,
	COUNT(*)AS cnt  
FROM Graduates
GROUP BY income
HAVING COUNT(*) >= (
	SELECT MAX(cnt) FROM( SELECT COUNT(*)AS cnt FROM Graduates GROUP BY income)TMP
);
```

~也就是先分组，然后在返回分组中记录数最大的一组。看了上面的例子很快能够明白。如果自己写，却需要花费点时间。

### 用HAVING子句进行自连接：求中位数
>将集合里的元素按照大小分为上半部分和下半部分两个子集，同时让这2个子集共同拥有集合正中间的元素。这样，共同部分的元素的平均值就是中位数。
```sql
-- 毕业生表（姓名，收入）
-- 求中位数的SQL语句：在HAVING子句中使用非等值自连接
SELECT
	AVG(DISTINCT income)  
FROM(SELECT T1.income  FROM Graduates T1, Graduates T2          
	GROUP BY T1.income           
	--S1的条件    
	HAVING SUM(CASE WHEN T2.income >= T1.income THEN 1 ELSE 0 END) >= COUNT(*)/ 2           
	--S2的条件       
	AND SUM(CASE WHEN T2.income <= T1.income THEN 1 ELSE 0 END) >= COUNT(*)/ 2
)TMP;
```

~我也没有完全理解。


### 查询不包含NULL的集合*
```sql
-- 学生表Students（学号ID，学院dpt，提交日期sbmt_date）
-- 查询“提交日期”列内不包含NULL的学院(1)：使用COUNT函数，COUNT(列名)将首先排除NULL列。
SELECT
	dpt  
FROM Students
GROUP BY dpt
HAVING COUNT(*)=COUNT(sbmt_date);
-- 查询“提交日期”列内不包含NULL的学院(2)：使用CASE表达式
SELECT
	dpt  
FROM Students
GROUP BY dpt
HAVING COUNT(*) = SUM(
	CASE WHEN sbmt_date IS NOT NULL THEN 1 ELSE 0 END
);
```

~如果不包括空值的列的数目与总记录数相等，就可以知道某列是不是包含空值。

### 用关系除法运算进行购物篮分析
```sql
-- 商品表（商品）；店铺商品表（店铺，商品）
-- 查询啤酒、纸尿裤和自行车（包括所有商品表中记录商品）同时在库的店铺：
SELECT
	SI.shop  
FROM ShopItems SI, Items I
WHERE SI.item=I.item
GROUP BY SI.shop
HAVING COUNT(SI.item)=(
	SELECT COUNT(item)FROM Items
);
-- 精确关系除法运算（包括所有商品表中的商品且只包括此类商品）：使用外连接和COUNT函数
SELECT
	SI.shop  
FROM ShopItems SI
LEFT OUTER JOIN Items I ON SI.item=I.item
GROUP BY SI.shop
HAVING COUNT(SI.item)=(SELECT COUNT(item)FROM Items)  -- 条件1   
AND COUNT(I.item)=(SELECT COUNT(item)FROM Items);   -- 条件2
```

~这第二种写法，比较少见。
