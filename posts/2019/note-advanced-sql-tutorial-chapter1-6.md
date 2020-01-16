---
layout: default
---

## 1-6　用关联子查询比较行与行

### 增长、减少、维持现状
```sql
  -- 求与上一年营业额一样的年份(1)：使用关联子查询
  SELECT
  	year,
  	sale  
  FROM Sales S1
  WHERE sale=(SELECT sale                 
  	FROM Sales S2
  	WHERE S2.year=S1.year - 1)
  ORDER BY year;
```

### 用列表展示与上一年的比较结果
```sql
	-- 求出是增长了还是减少了，抑或是维持现状(1)：使用关联子查询
	SELECT
		S1.year,
		S1.sale,       
		CASE WHEN sale = (SELECT sale                
			FROM Sales S2
			WHERE S2.year=S1.year - 1)THEN '→' -- 持平            
		WHEN sale > (SELECT sale                
			FROM Sales S2  
			WHERE S2.year=S1.year - 1)THEN '↑' -- 增长            
		WHEN sale < (SELECT sale                
			FROM Sales S2  
			WHERE S2.year=S1.year - 1)THEN '↓' -- 减少       
		ELSE '—' END AS var  
	FROM Sales S1
	ORDER BY year;
  --
	-- 求出是增长了还是减少了，抑或是维持现状(2)：使用自连接查询（最早的年份不会出现在结果里）**
	SELECT
		S1.year,
		S1.sale,       
		CASE WHEN S1.sale=S2.sale THEN '→'            
		WHEN S1.sale > S2.sale THEN '↑'
		WHEN S1.sale < S2.sale THEN '↓'
		ELSE ' —' END AS var  
	FROM Sales S1, Sales S2
	WHERE S2.year=S1.year - 1
	ORDER BY year;
```

~可以当个参考。

### 时间轴有间断时：和过去最临近的时间进行比较**
```sql
	-- 查询与过去最临近的年份营业额相同的年份：同时使用自连接
	SELECT
		S1.year AS year,       
		S1.year AS year  
	FROM Sales2 S1, Sales2 S2
	WHERE S1.sale=S2.sale   
	AND S2.year=(SELECT MAX(year)                    
		FROM Sales2 S3                   
		WHERE S1.year > S3.year)
	ORDER BY year;

	-- 求每一年与过去最临近的年份之间的营业额之差(1)：结果里不包含最早的年份
	SELECT
		S2.year AS pre_year,       
		S1.year AS now_year,       
		S2.sale AS pre_sale,       
		S1.sale AS now_sale,       
		S1.sale - S2.saleAS diff  
	FROM Sales2 S1, Sales2 S2
	WHERE S2.year=(SELECT MAX(year)                    
		FROM Sales2 S3                   
		WHERE S1.year > S3.year)
	ORDER BY now_year;

	-- 求每一年与过去最临近的年份之间的营业额之差(2)：使用自外连接。结果里包含最早的年份**
	SELECT
		S2.year AS pre_year,
		S1.year AS now_year,       
		S2.sale AS pre_sale,
		S1.sale AS now_sale,       
		S1.sale - S2.sale AS diff  
	FROM Sales2 S1
	LEFT OUTER JOIN Sales2 S2    
		ON S2.year=(SELECT MAX(year)                    
			FROM Sales2 S3                   
			WHERE S1.year > S3.year)
	ORDER BY now_year;
```

~理解起来容易，要自己写出来就有点难度了。

### 移动累计值和移动平均值**
```sql
	-- Accounts（处理日期#prc_date，处理金额#prc_amt）
	-- 求累计值：使用冯· 诺依曼型递归集合
		SELECT
			prc_date,
			A1.prc_amt,     
			(SELECT SUM(prc_amt)         
				FROM Accounts A2        
				WHERE A1.prc_date >=A2.prc_date )AS onhand_amt  
		FROM Accounts A1
		ORDER BY prc_date;
```

```sql
	-- 求移动累计值(2)：不满3行的时间区间也输出
	SELECT
		prc_date,
		A1.prc_amt,     
		(SELECT
				SUM(prc_amt)         
			FROM Accounts A2        
			WHERE A1.prc_date >=A2.prc_date          
			AND(SELECT COUNT(*)                 
				FROM Accounts A3                
				WHERE A3.prc_date                  
				BETWEEN A2.prc_date AND A1.prc_date )<=3 )               
			AS mvg_sum  
	FROM Accounts A1
	ORDER BY prc_date;
  --
	-- 移动累计值(3)：不满3行的区间按无效处理
	SELECT
		prc_date,
		A1.prc_amt,
		(SELECT
				SUM(prc_amt)    
			FROM Accounts A2   
			WHERE A1.prc_date >=A2.prc_date     
			AND(SELECT COUNT(*)            
				FROM Accounts A3
				WHERE A3.prc_date             
				BETWEEN A2.prc_date AND A1.prc_date )<=3   
			HAVING  COUNT(*)=3)AS mvg_sum  -- 不满3行数据的不显示  
	FROM Accounts A1
	ORDER BY prc_date;
--
	-- 去掉聚合并输出
	SELECT
		A1.prc_date AS A1_date,       
		A2.prc_date AS A2_date,       
		A2.prc_amt AS amt  
	FROM Accounts A1,Accounts A2
	WHERE A1.prc_date >=A2.prc_date   
	AND(SELECT COUNT(*)          
		FROM Accounts A3
		WHERE A3.prc_date BETWEEN A2.prc_date AND A1.prc_date )<=3
	ORDER BY A1_date, A2_date;
```

```sql
	-- Reservations(入住客人#reserver，入住日期#start_date，离店日期#end_date)
	-- 求重叠的住宿期间
	SELECT
		reserver,
		start_date,
		end_date  
	FROM Reservations R1 WHERE EXISTS      
		(SELECT *         
			FROM Reservations R2        
			WHERE R1.reserver <> R2.reserver  -- 与自己以外的客人进行比较          
			AND(
				R1.start_date BETWEEN R2.start_date AND R2.end_date -- 条件(1)：自己的入住日期在他人的住宿期间内            
				OR
				R1.end_date  BETWEEN R2.start_date AND R2.end_date));  -- 条件(2)：自己的离店日期在他人的住宿期间内

	-- 升级版：把完全包含别人的住宿期间的情况也输出
	SELECT
		reserver,
		start_date,
		end_date
	FROM Reservations R1
	WHERE EXISTS      
		(SELECT *          
			FROM Reservations R2         
			WHERE R1.reserver <> R2.reserver           
			AND( (     
				R1.start_date BETWEEN R2.start_date AND R2.end_date                     
				OR
				R1.end_date BETWEEN R2.start_date AND R2.end_date)                
				OR(    
					R2.start_date
						BETWEEN R1.start_date AND R1.end_date                    
					AND
					R2.end_date
						BETWEEN R1.start_date AND R1.end_date)));
```
