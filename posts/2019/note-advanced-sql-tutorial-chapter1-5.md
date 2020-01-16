---
layout: default
---


## 1-5　外连接的用法
### 用外连接进行行列转换(1)（行→列）：制作交叉表
```sql
-- 课程表（员工姓名，课程）
-- 水平展开(2)：使用标量子查询
SELECT
	C0.name,      
	(SELECT '○'
		FROM Courses C1        
		WHERE course='SQL入门'          
		AND C1.name=C0.name)AS "SQL入门",      
	(SELECT '○'         
		FROM Courses C2        
		WHERE course='UNIX基础'          
		AND C2.name=C0.name)AS "UNIX基础",     
	(SELECT '○'         
		FROM Courses C3        
		WHERE course='Java中级'          
	AND C3.name=C0.name)AS "Java中级"
FROM(SELECT DISTINCT name FROM Courses)C0;

-- 水平展开(3)：嵌套使用CASE表达式
SELECT
	name,  
	CASE WHEN SUM(
			CASE WHEN course='SQL入门'
			THEN 1 ELSE NULL END)=1       
		THEN '○'
		ELSE NULL
		END AS "SQL入门",  
	CASE WHEN SUM(
			CASE WHEN course='UNIX基础'
			THEN 1 ELSE NULL END)=1       
		THEN '○'
		ELSE NULL
		END AS "UNIX基础",  
	CASE WHEN SUM(
			CASE WHEN course='Java中级'
			THEN 1 ELSE NULL END)=1       
		THEN '○'
		ELSE NULL
		END AS "Java中级"  
FROM Courses GROUP BY name;
```

~第二种方式更容易理解。

### 用外连接进行行列转换(2)（列→行）：汇总重复项于一列
```sql
	-- 员工子女表（员工，子女1，子女2，子女3）
	-- 获取员工子女列表的SQL语句（没有孩子的员工也要输出）
	CREATE VIEW Children(child) AS
	SELECT child_1 FROM Personnel   
	UNION   
	SELECT child_2 FROM Personnel   
	UNION   
	SELECT child_3 FROM Personnel;

	SELECT
		EMP.employee,
		CHILDREN.child  
	FROM Personnel EMP       
	LEFT OUTER JOIN Children         
	ON CHILDREN.child IN(EMP.child_1, EMP.child_2, EMP.child_3);
```
>这里对子女主表和员工表执行了外连接操作，重点在于连接条件是通过IN谓词指定的。
>这样一来，当表Personnel里“孩子1～孩子3”列的名字存在于Children视图里时，返回该名字，否则返回NULL。

### 在交叉表里制作嵌套式表侧栏
```sql
	-- 年龄层级主表#TblAge（年龄层级#age_class，年龄#age_range）
	-- 性别主表#TblSex（性别编号#sex_cd，性别#sex）
	-- 人口分布表#TblPop（县名#pref_name，年龄层级#age_class，性别编号#sex_cd，人口#population）
	-- 使用外连接生成嵌套式表侧栏：正确的SQL语句
	SELECT
		MASTER.age_class AS age_class,       
		MASTER.sex_cd    AS sex_cd,       
		DATA.pop_tohoku  AS pop_tohoku,       
		DATA.pop_kanto   AS pop_kanto
	FROM(SELECT age_class, sex_cd        
		FROM TblAge CROSS JOIN TblSex )MASTER
	-- 使用交叉连接生成两张主表的笛卡儿积     
	LEFT OUTER JOIN (SELECT
		age_class,
		sex_cd,             
		SUM(CASE WHEN pref_name IN('青森', '秋田')                      
			THEN population ELSE NULL END)AS pop_tohoku,             
		SUM(CASE WHEN pref_name IN('东京', '千叶')                      
			THEN population ELSE NULL END)AS pop_kanto         
		FROM TblPop        
		GROUP BY age_class, sex_cd)DATA           
	ON  MASTER.age_class=DATA.age_class          
	AND  MASTER.sex_cd   =DATA.sex_cd;
	-- 技巧是对表TblAge和表TblSex进行交叉连接运算，生成下面这样的笛卡儿积。行数是3×2=6。
```

~仔细看看，还是比较容易理解。首先使用交叉连接生成‘年龄层级主表’与‘性别主表’的笛卡尔积，然后左连接于‘人口分布表’（此表已按‘年龄层级’，‘性别编号’分组）。

### 全外连接
>全外连接是能够从这样两张内容不一致的表里，没有遗漏地获取全部信息的方法，
>将全外连接（FULL OUTER JOIN）理解成“把两张表都当作主表来使用”的连接。
```sql
	-- Class_A（编号#id，名字#name）
	-- Class_B（编号#id，名字#name）
	-- 全外连接保留全部信息
	SELECT
		COALESCE(A.id, B.id)AS id,       
		A.name AS A_name,       
		B.name AS B_name  
	FROM Class_A  A  
	FULL OUTER JOIN Class_B  B    
	ON A.id=B.id;
```
### 用全外连接求异或集（除去共有元素的集合）
```sql
	SELECT
		COALESCE(A.id, B.id)AS id,       
		COALESCE(A.name , B.name )AS name  
	FROM Class_A  A
	FULL OUTER JOIN Class_B B ON A.id=B.id
	WHERE A.name IS NULL
	OR B.name IS NULL;
```

~需要注意的是，mysql并不支持全外连接。
