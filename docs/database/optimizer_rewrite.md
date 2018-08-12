## 查询重写

数据库优化器查询重写属于逻辑优化，通过对关系代数表达式进行等价变换，生成更优的关系代数表达式。

### 传递闭包（Transitive Closure）

某些操作符具有数学上的可传递性，传递闭包是利用这种可传递性的一种重写方式。最简单的传递闭包是：

```
A = B and B = C
=>
A = C
```

例如：

```sql
select * from t1, t2 where t1.a = t2.a and t1.a = 1;
```

可被重写为:

```sql
select * from t1, t2 where t1.a = t2.a and t1.a = 1 and t2.a = 1;

select * from t1, t2 where t1.a = 1 and t2.a = 1;
```

这样重写就可以将join条件转换为filter条件，可被下推到基表。此外，新生成的条件更方便优化器进行行数估算，可以减小行数估算误差。比如，在上述例子中，如果不添加```t2.a = 1```条件，则t2表可能按照a列各distinct值的平均行数来估计其行数。

### 恒假条件（Impossible Predicates and Unneeded Table Accesses）

当查询条件中出现恒假条件时，可以跳过对表的扫描等操作。

例如：

```sql
select * from t where 1 = 0;

select * from t where NULL = NULL;
```

因为条件恒假，可对上述语句进行重写，直接跳过表扫描的执行。

### 连接消除（JOIN Elimination）



### 提升子查询



### 提升子链接：IN

```sql
select t1.a 
from t1
where t1.b in (select t2.c from t2 where t2.d = 1);
```

提升子链接，因为是IN操作符，需要对子查询的t2.c列加group by操作：

```sql
select t1.a
from t1, (select t2.c from t2 where t2.d = 1 group by t2.c) as sub
where t1.b = sub.c;
```

提升子查询：

```sql
select t1.a
from t1, t2
where t1.b = t2.c and t2.d = 1;
```

### 提升子链接：NOT IN

### 提升子链接：EXISTS
