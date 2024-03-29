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
[in] select * from t1, t2 where t1.a = t2.a and t1.a = 1;
```

可被重写为:

```sql
[out] select * from t1, t2 where t1.a = t2.a and t1.a = 1 and t2.a = 1;

[out] select * from t1, t2 where t1.a = 1 and t2.a = 1;
```

这样重写就可以将join条件转换为filter条件，可被下推到基表。此外，新生成的条件更方便优化器进行行数估算，可以减小行数估算误差。比如，在上述例子中，如果不添加```t2.a = 1```条件，则t2表可能按照a列各distinct值的平均行数来估计其行数。

### 恒假条件（Impossible Predicates and Unneeded Table Accesses）

当查询条件中出现恒假条件时，可以跳过对表的扫描等操作。

#### 显而易见的恒假条件

```sql
[in] select * from t where 1 = 0;

[in] select * from t where NULL = NULL;
```

因为条件恒假，可对上述语句进行重写，直接跳过表扫描的执行。（注意：此处```NULL = NULL```表达式的值为NULL，被认为是一个为假的条件）

#### 需推理的恒假条件：is null + not null约束

当一个is null条件被应用于一个有not null约束的列上时，条件恒假，将产生空集。

```sql
[in] select * from t where a is null;
```

假设上述语句中，t.a列有not null约束，那么条件```a is null```恒假，查询的结果集为空，所以上述语句执行时可直接跳过表扫描等操作。

#### 需推理的恒假条件：predicates + CHECK约束

假设t.a上有一个CHECK约束：```CHECK(a in (1, 2, 3))```，那么如果有如下查询：

```sql
[in] select * from t where a = 4;
```

条件```a = 4```可被证明是恒假条件，上述语句应被重写并跳过表扫描等操作。

> 在PostgreSQL中，把constraint_exclusion参数设置为on，可使用此重写规则。

### 无用条件去除（Removing “Silly” Predicates）

与恒假条件相反，这个重写规则主要处理“恒真”条件。

#### 恒真常量条件

```sql
[in] select * from t where 1 = 1;
```

这种条件一看就为真，可以直接消除。

#### 恒真变量条件

再看下面的例子：

```sql
[in] select * from t where a = a;
```

这个条件我们不能直接删除，如果a列的值不为NULL时，```a = a```表达式为真，但是如果a列的值为NULL时，该表达式的值为NULL，会被认为是一个为假的条件。所以上述语句可被重写为：

```sql
[out] select * from t where a is not NULL;
```

#### 恒真变量条件 + not NULL

上述例子中，如果在a列上有一个not NULL约束，那么条件可以被直接删除，语句可以被改写为：

```sql
[out] select * from t;
```

### 条件合并（Predicate Merging）

#### IN条件、范围条件的合并

这里的条件合并主要是指两类条件的合并：

1. 多个IN条件的合并
2. 多个范围条件的合并

例如：

```sql
[in]
select * from t where a in (1, 2, 3) and a in (2, 3, 4);

[out]
select * from t where a in (2, 3);
```

```sql
[in]
select * from t where a between 1 and 100 and a between 99 and 200;

[out]
select * from t where a between 99 and 100;
```

当然，如果AND连接的IN条件或者范围条件没有公共部分，它们应该被转换为一个恒假的条件。如果OR连接的IN条件或者范围条件构成了值域上的全集，则应该被去除，并适当添加is not null的条件。

#### 语句条件和CHECK条件的合并与化简

假设t.a上有一个CHECK约束：```CHECK(a in (1, 2, 3))```，那么如果有如下查询：

```sql
[in] select * from t where a not in (2, 3);
```

如果被重写为：

```sql
[out] select * from t where a = 1;
```

将大大简化条件执行的复杂度，并且，如果t.a列上有索引，上述语句进行索引扫描比顺序扫描可能更有优势。如果语句的输出列不是```*```，而是```count(*)```，重写后的语句甚至只需要扫描索引，不需要扫描数据页面。

### 使用CHECK约束（CHECK Constraints）

#### 条件与CHECK约束冲突

在前文讨论条件为假的语句的重写时，有提到条件和CHECK约束冲突从而导致条件为假的情况，可根据假条件语句的情况，对语句进行重写。

#### 条件与CHECK约束可merge

在前文讨论条件合并时，提到语句中的条件与CHECK约束进行合并的例子。其实CHECK约束也是条件的一种，可考虑将CHECH约束与语句中的条件综合在一起考虑进行优化。

### 条件下推（Predicate Pushdown）

条件下推是指将外层的条件尽可能地下推到基表执行，以便在基表层面过滤掉尽可能多的行数。并且，如果基表有索引，可以考虑使用索引扫描。

```sql
[in]
select *
from (select a, b from t) s
where s.a = 1;
```

上述语句中的条件可被推向基表t：

```sql
[out]
select *
from (select a, b from t where t.a = 1);
```

进一步地，外部查询也可被消除，最终被重写为：

```sql
[out] select a, b from t where t.a = 1;
```

同理，对于含有union的子查询：

```sql
[in]
select * from
(
      select a, b from t1
      union all
      select a, b from t2
) s
where s.a = 1;
```

可被重写为：

```sql
[out]
select * from
(
      select a, b from t1 where t1.a = 1
      union all
      select a, b from t2 where t2.a = 1
) s;

[out]
select a, b from t1 where t1.a = 1
union all
select a, b from t2 where t2.a = 1;
```

### Exists子查询输出列投影（Projections in EXISTS Subqueries）

在研究这个重写规则前，我们先研究一下Exists子查询的Target List。

```sql
select 1 / 0;
ERROR:  division by zero

select exists (select 1 / 0);
 exists 
--------
 t
(1 row)
```

可以看出，在第1条语句中，报了除0的错误，但是在第2条语句中，并没有报错，说明第2条语句的```1 / 0```没有真正执行。

```
explain verbose select exists (select 1 / 0);
                    QUERY PLAN                    
--------------------------------------------------
 Result  (cost=0.01..0.02 rows=1 width=1)
   Output: $0
   InitPlan 1 (returns $0)
     ->  Result  (cost=0.00..0.01 rows=1 width=0)
(4 rows)
```

在第2条语句的执行计划中，子查询仍然存在，但是没有提及Target List。所以说，在这种情况下，Exists子查询的Target List可以做投影消除。

来看个例子：

```sql
[in]
select * from t1
where exists
(select * from t2 where t1.a = t2.a);
```

它的查询计划是：

```
                             QUERY PLAN                              
---------------------------------------------------------------------
 Hash Semi Join  (cost=1.07..2.21 rows=6 width=16)
   Output: t1.a, t1.b, t1.c, t1.d
   Hash Cond: (t1.a = t2.a)
   ->  Seq Scan on public.t1  (cost=0.00..1.06 rows=6 width=16)
         Output: t1.a, t1.b, t1.c, t1.d
   ->  Hash  (cost=1.03..1.03 rows=3 width=4)
         Output: t2.a
         ->  Seq Scan on public.t2  (cost=0.00..1.03 rows=3 width=4)
               Output: t2.a
(9 rows)
```

可见，上述查询进行了Exists子查询输出列投影和子链接提升的查询重写，其重写后的语句为：

```sql
[out]
select t1.*
from t1 semi join (select a from t2) s on t1.a = s.a;
```

原Exists子查询中的Target List被进行了投影操作，只输出了join列t2.a。

### 连接消除（JOIN Elimination）

某些语句可以根据表的主-外键情况或从语义上，对其部分join的表进行消除，直接跳过join的执行。

#### to-one inner join + not null foreign key

```sql
[in] select t1.* from t1 join t2 on t1.b = t2.a;
```

如果t1.b是t1表的非空外键，t2.a是t2表的主键，t2.a和t1.b是主-外键关系，那么此语句可被重写为：

```sql
[out] select t1.* from t1;
```

因为对于每个t1.b，有且仅有1个t2.a与之对应，且语句的输出列没有t2表中的列，所以t2表没有存在的意义，可以进行消除。

#### to-one inner join + nullable foreign key

还是上述语句，如果t1.b是t1表的外键，但是可以为空，t2.a是t2表的主键，那么此语句需要被重写为：

```sql
[out] select t1.* from t1 where t1.b is not NULL;
```

此时需要给t1.b加上is not NULL，因为原语句中t1.b为NULL的行会被剔除。

#### to-one outer join

```sql
[in] select t1.* from t1 left join t2 on t1.b = t2.a;
```

如果t2.a是t2表的主键，且join类型为outer join，则进行连接消除：

```sql
[out] select t1.* from t1;
```

因为当t1.b非空时，要么对应于t2中的1行，在结果集中，要么在t2中没有对应，从而被left join补空保留。当t1.b为空时，被left join补空保留。所以查询中t2表可被消除。

#### to-many distinct outer join

```sql
[in] select distinct t1.b from t1 left join t2 on t1.a = t2.a;
```

如果t2.a不是主键，且无unique key，则可能出现多个相同值的情况，这时候，如果join为outer join，且输出列上有distinct操作，则t2表可以被消除。

```sql
[out] select distinct b from t1;
```

t1表中不能匹配的行会被left join保留，能匹配的行中，即使t1.a对应于多个t2.a，1行变了多行，也有distinct操作进行去重，从而可以对t2进行消除。

#### 多余的self-join

在查询语句较为复杂，或者使用较多视图的时候，此查询重写规则可能会派上用场。

如果出现了self-join的情况，并且连接条件在其primary key上，则可对多余的self-join进行消除。

假设t.a是表t的primary key，看如下示例：

```sql
[in] select t1.a, t1.b from t t1 join t t2 on t1.a = t2.a;

[out] select t.a, t.b from t;
```

更进一步，可以利用传递闭包原理，对过滤条件和输出列进行调整。

```sql
[in] select t1.a, t2.b from t t1 join t t2 on t1.a = t2.a;

[out] select t.a, t.b from t;
```

### 可证的空集（Provably Empty Sets）

#### 为假的条件

在前文中已讨论条件为假的语句的重写，条件为假的语句将输出空集。

#### 空集参与join

```sql
[in]
select a, b
from t1 join (select * from t2 where false) s on t1.b = t2.b;
```

上述语句中，子查询s的结果集为空，t1和s为inner join，其join的结果集也必定为空，所以上述语句可被重写为：

```sql
[out]
select NULL as a, NULL as b where false;
```

对于其他join类型的情况，可总结为：

- [empty] inner join t => [empty]
- t inner join [empty] => [empty]
- [empty] left join t => [empty]
- t right join [empty] => [empty]
- [empty] semi join t => [empty]
- t semi join [empty] => [empty]
- [empty] anti join t => [empty]
- [empty] * join [empty] => [empty]

#### null intersect not null

当一个为null的包与一个含有not null约束的包进行intersect操作时，将产生空集。

```sql
[in]
select NULL
intersect
select a from t;
```

假设上述语句中，t.a列有not null约束，那么intersect后的结果为空，所以上述语句可被重写为：

```sql
[out]
select NULL where false;
```

### 消除NestLoop

#### in list

当条件中包含形如“a in (1, 2, 3, ..., N)”，且值列表很多时，查询执行一般是取a的一个数值，然后去扫描值列表中的值，找到即返回，整个执行过程类似于一个NestLoop。可以将值列表中的数值转成一个表，再与主表进行Join，这样在执行中，可以使用Hash Join，执行效率会提升。例如：

```sql
[in] select * from t where a in (1,2,3);
```

可被重写为：

```sql
[out] select * from t where a in (values (1),(2),(3));
```

在查询优化过程中，上述in list先被转换成相关子查询，然后再进行子查询提升，与主表进行了Join。

### 子查询相关

#### 提升子链接：IN

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

#### 提升子链接：NOT IN

#### 提升子链接：EXISTS

### 引用

[1] https://blog.jooq.org/2017/09/28/10-cool-sql-optimisations-that-do-not-depend-on-the-cost-model

[2] https://blog.jooq.org/2017/09/01/join-elimination-an-essential-optimiser-feature-for-advanced-sql-usage/

