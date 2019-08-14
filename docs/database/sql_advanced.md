## 一些SQL的高级用法

### Window Function

Window Function对一组Tuple中每个Tuple进行计算，它并不像Group by那样计算出一行结果，而是保留每一行。

语法形如：

```sql
select ..., FUNC_NAME(...) OVER (...) from ...;
```

其中

- FUNC_NAME可以是Agg函数，也可以是一些特殊的函数（如row_number、rank等，详见 https://www.postgresql.org/docs/current/functions-window.html ）
- OVER里面表示了数据如何分组或如何排序。OVER是必须的，用来和普通Group by区分

例1：按照depname分组，计算salary的平均值（与普通Group by不同的是每行都输出）

```sql
SELECT depname, empno, salary,
       avg(salary) OVER (PARTITION BY depname)
FROM empsalary;

  depname  | empno | salary |          avg          
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
```

例2：按照depname分组，且按照salary倒序排列，计算序号值

```sql
SELECT depname, empno, salary,
       rank() OVER (PARTITION BY depname ORDER BY salary DESC)
FROM empsalary;

  depname  | empno | salary | rank 
-----------+-------+--------+------
 develop   |     8 |   6000 |    1
 develop   |    10 |   5200 |    2
 develop   |    11 |   5200 |    2
 develop   |     9 |   4500 |    4
 develop   |     7 |   4200 |    5
 personnel |     2 |   3900 |    1
 personnel |     5 |   3500 |    2
 sales     |     1 |   5000 |    1
 sales     |     4 |   4800 |    2
 sales     |     3 |   4800 |    2
(10 rows)
```

例3：当OVER子句中无partition by但有order by时，Agg函数会计算从第1条记录到当前记录（包括当前记录所有相同值）的Agg值

```sql
SELECT salary, sum(salary)
       OVER (ORDER BY salary)
FROM empsalary;

 salary |  sum  
--------+-------
   3500 |  3500
   3900 |  7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
(10 rows)
```

### Grouping Sets

Grouping sets是对Group by功能的扩展，相当于可以从多个维度进行Group by。

下面两个查询等价：

```sql
select a, b, avg(c) from t group by grouping sets ((a), (b), ());

select a, null, avg(c) from t group by a
union all
select null, b, avg(c) from t group by b
union all
select null, null, avg(c) from t;
```

结果：

```sql
select * from t order by c;
 a | b  | c | d
---+----+---+---
 1 | 11 | 1 | 1
 1 | 22 | 2 | 1
 2 | 11 | 3 | 1
 2 | 22 | 4 | 1
(4 rows)

select a, b, avg(c) from t group by grouping sets ((a), (b), ());
 a | b  |        avg
---+----+--------------------
   |    | 2.5000000000000000
   | 11 | 2.0000000000000000
   | 22 | 3.0000000000000000
 1 |    | 1.5000000000000000
 2 |    | 3.5000000000000000
(5 rows)
```

可以在Target List中使用grouping函数，来查看某列是否为当前结果行的grouping set成员（0表示是当前grouping set成员）：

```sql
select grouping(a) as a_flag, grouping(b) as b_flag, a, b, avg(c) from t group by grouping sets ((a), (b), ());

 a_flag | b_flag | a | b  |        avg
--------+--------+---+----+--------------------
      1 |      1 |   |    | 2.5000000000000000
      1 |      0 |   | 11 | 2.0000000000000000
      1 |      0 |   | 22 | 3.0000000000000000
      0 |      1 | 1 |    | 1.5000000000000000
      0 |      1 | 2 |    | 3.5000000000000000
(5 rows)
```

此外，还有rollup和cube，它们和grouping sets之间有如下等价关系：

```sql
ROLLUP ( e1, e2, e3, ... )

-- 等价于

GROUPING SETS (
    ( e1, e2, e3, ... ),
    ...
    ( e1, e2 ),
    ( e1 ),
    ( )
)
```

```sql
CUBE ( a, b, c )

-- 等价于

GROUPING SETS (
    ( a, b, c ),
    ( a, b    ),
    ( a,    c ),
    ( a       ),
    (    b, c ),
    (    b    ),
    (       c ),
    (         )
)
```

其他等价例子如下：

```sql
ROLLUP ( a, (b, c), d )

-- 等价于

GROUPING SETS (
    ( a, b, c, d ),
    ( a, b, c    ),
    ( a          ),
    (            )
)
```

```sql
CUBE ( (a, b), (c, d) )

-- 等价于

GROUPING SETS (
    ( a, b, c, d ),
    ( a, b       ),
    (       c, d ),
    (            )
)
```

```sql
GROUP BY a, CUBE (b, c), GROUPING SETS ((d), (e))

-- 等价于

GROUP BY GROUPING SETS (
    (a, b, c, d), (a, b, c, e),
    (a, b,    d), (a, b,    e),
    (a,    c, d), (a,    c, e),
    (a,       d), (a,       e)
)
```

### Recursive CTE

### Select for Update

### 引用

[1] https://www.postgresql.org/docs/current/tutorial-window.html

[2] https://www.postgresql.org/docs/11/queries-table-expressions.html#QUERIES-GROUPING-SETS

[3] http://www.postgresqltutorial.com/postgresql-grouping-sets/
