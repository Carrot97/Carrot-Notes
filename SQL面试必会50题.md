# SQL面试必会50题

点击[这里](https://zhuanlan.zhihu.com/p/43289968)打开原网页

**1.查询课程编号为“01”的课程比“02”的课程成绩高的所有学生的学号（重点**）

```mysql
select stu.s_id
from student stu
join score sco1 on sco1.c_id=1 and sco1.s_id=stu.s_id
join score sco2 on sco2.c_id=2 and sco2.s_id=stu.s_id
where sco1.s_score > sco2.s_score
```

**2、查询平均成绩大于60分的学生的学号和平均成绩（简单，第二道重点）**

```mysql
# 注意存在未参加考试的学生
SELECT stu.s_id, AVG(IFNULL(sco.s_score, 0)) avg_s
FROM student stu
LEFT JOIN score sco on stu.s_id=sco.s_id
GROUP BY stu.s_id
HAVING avg_s < 60
```

**3、查询所有学生的学号、姓名、选课数、总成绩（不重要）**

```mysql
SELECT stu.s_id, stu.s_name, COUNT(*), SUM(IFNULL(sco.s_score, 0))
FROM student stu
left join score sco on stu.s_id= sco.s_id
GROUP BY stu.s_id
```

**4、查询姓“猴”的老师的个数（不重要）**

```mysql
SELECT COUNT(t_id)
FROM teacher
WHERE t_name LIKE "猴%"
```

**5、查询没学过“张三”老师课的学生的学号、姓名（重点）**

```mysql
SELECT s_id, s_name
FROM student
WHERE s_id not in (
    SELECT sco.s_id 
    FROM score sco
    INNER JOIN course c on sco.c_id=c.c_id
    INNER JOIN teacher t on c.t_id=t.t_id and t.t_name = "张三"
    )
```

**6、查询学过“张三”老师所教的所有课的同学的学号、姓名（重点）**

```mysql
SELECT s_id, s_name
FROM student
WHERE s_id in (
    SELECT sco.s_id 
    FROM score sco
    INNER JOIN course c on sco.c_id=c.c_id
    INNER JOIN teacher t on c.t_id=t.t_id and t.t_name = "张三"
    )
```

**7、查询学过编号为“01”的课程并且也学过编号为“02”的课程的学生的学号、姓名（重点）**

```mysql
SELECT stu.s_id, stu.s_name
FROM student stu
INNER JOIN score sco1 on stu.s_id=sco1.s_id and sco1.c_id=1
INNER JOIN score sco2 on stu.s_id=sco2.s_id and sco2.c_id=2
```

**8、查询课程编号为“02”的总成绩（不重点）**

```mysql
SELECT SUM(s_score)
FROM score
WHERE c_id = 2
```

**10、查询没有学全所有课的学生的学号、姓名(重点)**

```mysql
SELECT stu.s_id, stu.s_name
FROM student stu
LEFT JOIN score sco on sco.s_id = stu.s_id
GROUP BY stu.s_id
HAVING COUNT(sco.c_id) < (SELECT COUNT(DISTINCT c_id) FROM course)
```

**11、查询至少有一门课与学号为“01”的学生所学课程相同的学生的学号和姓名（重点）**

```mysql
SELECT DISTINCT(stu.s_id), stu.s_name
FROM student stu
JOIN score sco on stu.s_id=sco.s_id
WHERE sco.s_id != 1
and sco.c_id in (
	SELECT c_id 
	FROM score
	WHERE s_id=1
	)
```

**13、查询没学过"张三"老师讲授的任一门课程的学生姓名 和47题一样（重点，能做出来）**

```mysql
SELECT *
FROM student
WHERE s_id not in (
	SELECT s.s_id
	FROM score s
	INNER JOIN course c on c.c_id = s.c_id
	INNER JOIN teacher t on t.t_id = c.t_id and t.t_name = "张三"
)
```

**15、查询两门及其以上不及格课程的同学的学号，姓名（重点）**

```mysql
select stu.s_id, stu.s_name
FROM student stu
JOIN score sco on stu.s_id = sco.s_id and sco.s_score < 60
GROUP BY stu.s_id
HAVING COUNT(sco.c_id)
```

**16、检索"01"课程分数小于60，按分数降序排列的学生信息（和34题重复，不重点）**

```mysql
SELECT stu.*, sco.s_score
FROM student stu
JOIN score sco on stu.s_id = sco.s_id and sco.c_id = 1 and sco.s_score < 60
ORDER BY sco.s_score DESC
```

**17、按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩(重重点与35一样)**

```mysql
SELECT stu.s_id, stu.s_name, AVG(IFNULL(sco.s_score,0)) avg_s,
MAX(CASE WHEN sco.c_id = 1 THEN sco.s_score ELSE NULL END) math,
MAX(CASE WHEN sco.c_id = 2 THEN sco.s_score ELSE NULL END) chinese,
MAX(CASE WHEN sco.c_id = 3 THEN sco.s_score ELSE NULL END) english
FROM student stu
LEFT JOIN score sco on stu.s_id = sco.s_id
GROUP BY stu.s_id
ORDER BY avg_s DESC
```

**18.查询各科成绩最高分、最低分和平均分：以如下形式显示：课程ID，课程name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率**

**--及格为>=60，中等为：70-80，优良为：80-90，优秀为：>=90 (超级重点)**

```mysql
SELECT c.c_id, c.c_name, MAX(s.s_score) max, MIN(s.s_score) min, AVG(s.s_score) avg,
AVG(CASE WHEN s.s_score >= 60 THEN 1 ELSE 0 END) jige,
AVG(CASE WHEN s.s_score >= 70 THEN 1 ELSE 0 END) zhongdeng,
AVG(CASE WHEN s.s_score >= 80 THEN 1 ELSE 0 END) lianghao,
AVG(CASE WHEN s.s_score >= 90 THEN 1 ELSE 0 END) youxiu
FROM course c
JOIN score s on c.c_id = s.c_id
GROUP BY c.c_id
```

**19、按各科成绩进行排序，并显示排名(重点row_number)**

```mysql
SELECT s_id, s_score, DENSE_RANK() over(ORDER BY s_score DESC) as grade
FROM score
WHERE c_id = 1
```

**20、查询学生的总成绩并进行排名（不重点）**

```mysql
SELECT s_id, SUM(s_score) sum
FROM score
GROUP BY s_id
ORDER BY sum DESC
```

**21 、查询不同老师所教不同课程平均分从高到低显示(不重点)**

```mysql
SELECT c.t_id, AVG(s_score) avg
FROM course c
JOIN score s on c.c_id = s.c_id
GROUP BY t_id
ORDER BY avg DESC
```

**22、查询所有课程的成绩第2名到第3名的学生信息及该课程成绩（重要 25类似）**

```mysql
SELECT *
FROM (
	SELECT stu.*, sco.c_id, 
    DENSE_RANK() over(PARTITION BY sco.c_id ORDER BY sco.s_score DESC) as grade
	FROM student stu
	JOIN score sco on stu.s_id = sco.s_id
	) as mid
WHERE grade in (2, 3)
```



