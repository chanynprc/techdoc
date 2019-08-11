## 一些SQL的高级用法

### Window Function

Window Function对一组Tuple中每个Tuple进行计算，它并不像Group by那样计算出一行结果，而是保留每一行。

语法形如：

```sql
select ..., FUNC_NAME(...) OVER (...) from ...;
```

其中FUNC_NAME可以是Agg函数，也可以是一些特殊的函数（如row_number、rank），OVER里面表示了数据如何分组或如何排序。OVER是必须的，用来和普通Group by区分。

例1：按照depname，计算salary的平均值（与普通Group by不同的是每行都输出）

```sql
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;

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



### Grouping Sets

### Recursive CTE

### Select for Update

### 引用

[1] https://www.postgresql.org/docs/current/tutorial-window.html

