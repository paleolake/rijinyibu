---
layout: default
---

## 1-3　三值逻辑和NULL *
### known、第三个真值
>•AND的情况：false＞unknown＞true
>  
>•OR的情况：true＞unknown＞false
>
>优先级高的真值会决定计算结果。例如true AND unknown，因为unknown的优先级更高，所以结果是unknown。
>
>而true OR unknown的话，因为true优先级更高，所以结果是true。记住这个顺序后就能更方便地进行三值逻辑运算了。

~三值逻辑（true，unknown，false）的优先级。

### 比较谓词和NULL(2)：CASE表达式和NULL	*
```sql
-- col_1为1时返回○、为NULL时返回× 的CASE表达式？
	-- 错误
	CASE col_1  
		WHEN 1     THEN '○'  
		WHEN NULL  THEN '×'
	END
	--正确
	CASE
		WHEN col_1=1 THEN '○'     
		WHEN col_1 IS NULL THEN '×'
	END
```

~也就是说NULL不能直接比较，只能使用‘IS NULL’来判断。

### 限定谓词和NULL
> ALL谓词其实是多个以AND连接的逻辑表达式的省略写法。
```sql
--1. 执行子查询获取年龄列表
SELECT *  FROM Class_A
WHERE age < ALL( 22, 23, NULL );
--2. 将ALL谓词等价改写为AND
SELECT *  FROM Class_A
WHERE(age < 22)AND(age < 23)AND(age < NULL);
--3. 对NULL使用“<”后，结果变为unknown
SELECT *  FROM Class_A
WHERE(age < 22)AND(age < 23)AND unknown;		
```

~所以当使用谓词ALL时，需要特别注意列是否可能空，不然可能会返回意想不到的结果。
