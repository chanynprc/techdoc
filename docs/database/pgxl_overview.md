## PGXL概览

### PGXC shuffle逻辑

PGXC会首先判断整个语句是否能够被下推到DN执行。如果可以，向每个DN直接发送语句，在CN收取结果。如果发现需要shuffle，则对语句进行拆分，拆分成可以不需要shuffle的语句，发送到DN执行，CN收取结果后，进行后续操作。

可整条语句下推的例子：

假设有一个表t，包含a、b两列，其分布列为a。

```sql
select 1 from t t1, t t2 where t1.a = t2.a;
```

不可整条语句下推的例子：

```sql
select 1 from t t1, t t2 where t1.b = t2.b;
```

在这种情况下，上述语句会被拆分成两条语句，发送到DN执行：

```sql
select b from t t1;

select b from t t2;
```

获取上述两条语句的执行结果后，会在CN执行Join操作。

所以，在PGXC的架构中，对于非整条语句下推的查询，查询效率一般较慢，其更适用于TP场景。

### PGXL的改进

PGXL实现了真正意义的MPP。在PG和PGXC的基础上，实现了DN间的数据通信。在DN之间形成生产者-消费者模式，每个DN有一个Shared Queue，生产者往里面写数据，消费者从中读取数据。

下面的例子中，Join条件在分布列上，在开启FQS的情况下，整条语句被下推：

假设有一个表t，包含a、b两列，其分布列为a。

```sql
test=# explain (verbose on, costs off) select 1 from t t1, t t2 where t1.a = t2.a;
                          QUERY PLAN
--------------------------------------------------------------
 Remote Fast Query Execution
   Output: 1
   Node/s: datanode_1, datanode_2
   Remote query: SELECT 1 FROM t t1, t t2 WHERE (t1.a = t2.a)
   ->  Merge Join
         Output: 1
         Merge Cond: (t1.a = t2.a)
         ->  Sort
               Output: t1.a
               Sort Key: t1.a
               ->  Seq Scan on public.t t1
                     Output: t1.a
         ->  Sort
               Output: t2.a
               Sort Key: t2.a
               ->  Seq Scan on public.t t2
                     Output: t2.a
(17 rows)
```

当关闭FQS时，下推计划：

```sql
test=# explain (verbose on, costs off) select 1 from t t1, t t2 where t1.a = t2.a;
                     QUERY PLAN
-----------------------------------------------------
 Remote Subquery Scan on all (datanode_1,datanode_2)
   Output: 1
   ->  Merge Join
         Output: 1
         Merge Cond: (t1.a = t2.a)
         ->  Sort
               Output: t1.a
               Sort Key: t1.a
               ->  Seq Scan on public.t t1
                     Output: t1.a
         ->  Sort
               Output: t2.a
               Sort Key: t2.a
               ->  Seq Scan on public.t t2
                     Output: t2.a
(15 rows)
```

当Join条件不在分布键上时，非分布键的那个表做了shuffle。

```sql
test=# explain (verbose on, costs off) select 1 from t t1, t t2 where t1.a = t2.b;
                           QUERY PLAN
-----------------------------------------------------------------
 Remote Subquery Scan on all (datanode_1,datanode_2)
   Output: 1
   ->  Merge Join
         Output: 1
         Merge Cond: (t2.b = t1.a)
         ->  Remote Subquery Scan on all (datanode_1,datanode_2) -- 此算子shuffle数据
               Output: t2.b
               Distribute results by H: b
               Sort Key: t2.b
               ->  Sort
                     Output: t2.b
                     Sort Key: t2.b
                     ->  Seq Scan on public.t t2
                           Output: t2.b
         ->  Sort
               Output: t1.a
               Sort Key: t1.a
               ->  Seq Scan on public.t t1
                     Output: t1.a
(19 rows)
```

另外一个例子，Join两边都做了shuffle。

```sql
test=# explain (verbose on, costs off) select 1 from t t1, t t2 where t1.b = t2.b;
                              QUERY PLAN
-----------------------------------------------------------------------
 Remote Subquery Scan on all (datanode_1,datanode_2)
   Output: 1
   ->  Merge Join
         Output: 1
         Merge Cond: (t1.b = t2.b)
         ->  Remote Subquery Scan on all (datanode_1,datanode_2) -- 此算子shuffle数据
               Output: t1.b
               Distribute results by H: b
               Sort Key: t1.b
               ->  Sort
                     Output: t1.b
                     Sort Key: t1.b
                     ->  Seq Scan on public.t t1
                           Output: t1.b
         ->  Materialize
               Output: t2.b
               ->  Remote Subquery Scan on all (datanode_1,datanode_2) -- 此算子shuffle数据
                     Output: t2.b
                     Distribute results by H: b
                     Sort Key: t2.b
                     ->  Sort
                           Output: t2.b
                           Sort Key: t2.b
                           ->  Seq Scan on public.t t2
                                 Output: t2.b
(25 rows)
```




