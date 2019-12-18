---
layout: default
---


## 1-1　CASE表达式

### CASE表达式概述

>注意事项 1：一定要注意CASE表达式里各个分支返回的数据类型是否一致。某个分支返回字符型，而其他分支返回数值型的写法是不正确的。
>注意事项 2：不要忘了写END。
>注意事项 3：养成写ELSE子句的习惯。

~CASE表达式需要特别的注意项。

### 用一条SQL语句进行不同条件的统计
```sql
-- 县市人口表（县名，性别，人口数量）
-- 统计县市中的男性人口与女性人口的数量。
SELECT
	pref_name,       
	-- 男性人口       
	SUM( CASE WHEN sex='1' THEN population ELSE 0 END)AS cnt_m,       
	-- 女性人口       
	SUM( CASE WHEN sex='2' THEN population ELSE 0 END)AS cnt_f  
FROM  PopTbl2
GROUP BY pref_name;
```

~这种写法还是比较常见，记录在这里当做温习之用。


### 用CHECK约束定义多个列的条件关系
```sql
CONSTRAINT check_salary CHECK(
	CASE WHEN sex='2'                  
	THEN
		CASE WHEN salary <=200000                            
		THEN 1 ELSE 0 END                  
	ELSE 1
	END=1
)
```

~对于我来说，这是一种新的写法，或者说我从来就没有在数据库上写过约束。

### 在UPDATE语句里进行条件分支*
```sql
-- 用CASE表达式写正确的更新操作
UPDATE Salaries   
SET salary =
	CASE WHEN salary >=300000                     
	THEN salary * 0.9                     
	WHEN salary >=250000 AND salary < 280000                     
	THEN salary * 1.2                     
ELSE salary END;
-- 用CASE表达式调换主键值
UPDATE SomeTable   
SET p_key =
	CASE WHEN p_key='a'                    
	THEN 'b'                    
	WHEN p_key='b'                    
	THEN 'a'                    
	ELSE p_key END
WHERE p_key IN('a', 'b');
```

~非常实用，工作中就能够用的到。

### 表之间的数据匹配*
> CASE表达式的一大优势在于能够判断表达式。也就是说，在CASE表达式里，我们可以使用BETWEEN、LIKE和<、>等便利的谓词组合，以及能嵌套子查询的IN和EXISTS谓词。
```sql
-- 表的匹配：使用IN谓词
SELECT
	course_name,       
	CASE WHEN course_id IN                   
			(SELECT course_id FROM OpenCourses                      
				WHERE month=200706)THEN '○'            
		ELSE '×' END AS "6月",       
	CASE WHEN course_id IN                   
				(SELECT course_id FROM OpenCourses                      
				WHERE month=200707)THEN '○'            
			ELSE '×' END AS "7月",       
	CASE WHEN course_id IN                   
			(SELECT course_id FROM OpenCourses                      
			WHERE month=200708)THEN '○'            
		ELSE '×' END AS "8月"  
FROM CourseMaster;
```

```sql
-- 表的匹配：使用EXISTS谓词
SELECT
	CM.course_name,       
	CASE WHEN EXISTS                   
			(SELECT course_id FROM OpenCourses OC
				WHERE month=200706                        
				AND OC.course_id=CM.course_id)THEN '○'            
		ELSE '×' END AS "6月",       
	CASE WHEN EXISTS                   
			(SELECT course_id FROM OpenCourses OC  
				WHERE month=200707                        
				AND OC.course_id=CM.course_id)THEN '○'            
		ELSE '×' END AS "7月",       
	CASE WHEN EXISTS                   
			(SELECT course_id FROM OpenCourses OC                      
				WHERE month=200708                        
				AND OC.course_id=CM.course_id)THEN '○'            
		ELSE '×' END  AS "8月"  
FROM CourseMaster CM;
```

~我之前不熟悉的写法。

### 在CASE表达式中使用聚合函数*
```sql
-- 学生俱乐部（学号ID，社团ID，社团名，主社团标志）
-- 获取只加入了一个社团的学生的社团ID；获取加入了多个社团的学生的主社团ID。
SELECT  
	std_id,        
	CASE WHEN COUNT(*)=1 -- 只加入了一个社团的学生             
		THEN MAX(club_id)             
		ELSE MAX(
			CASE WHEN main_club_flg='Y'                           
			THEN club_id                           
			ELSE NULL END)        
		END AS main_club  
FROM StudentClub
GROUP BY std_id;
```

~我之前不熟悉的写法。
