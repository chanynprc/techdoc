
## Outer Join探索

### 数据模型

首先准备数据，这是一个学生在图书馆借书的模型，有3个表，学生表（student）、书籍表（book）、书籍借阅表（readlog），DDL如下。

```sql
create table student (s_id int, s_name text, s_sex char);
create table readlog (r_stuid int, r_bookid int, r_duration int);
create table book (b_id int, b_name text);

insert into student values
    (1, 's1', 'm')
    ,(2, 's2', 'f')
    ,(3, 's3', 'm')
    ,(4, 's4', 'f');
insert into readlog values
    (1, 1, 2)
    ,(1, 2, 3)
    ,(2, 1, 4)
    ,(2, 3, 5)
    ,(3, 4, 6);
insert into book values
    (1, 'b1')
    ,(2, 'b2')
    ,(3, 'b3')
    ,(4, 'b4')
    ,(5, 'b5');
```

### Inner Join和Left Outer Join

首先来看2个查询场景：

场景1：查看所有学生的读书记录，不包含没有读过书的学生

```sql
-- inner join
select s_id, r_bookid, r_duration
from student inner join readlog on s_id = r_stuid
order by s_id, r_bookid;

 s_id | r_bookid | r_duration
------+----------+------------
    1 |        1 |          2
    1 |        2 |          3
    2 |        1 |          4
    2 |        3 |          5
    3 |        4 |          6
(5 rows)
```

场景2：查看所有学生的读书记录，包含没有读过书的学生

```sql
-- left join
select s_id, r_bookid, r_duration
from student left join readlog on s_id = r_stuid
order by s_id, r_bookid;

 s_id | r_bookid | r_duration
------+----------+------------
    1 |        1 |          2
    1 |        2 |          3
    2 |        1 |          4
    2 |        3 |          5
    3 |        4 |          6
    4 |          |
(6 rows)
```

以上2个场景只在Join类型上有不同，一个是Inner Join，一个是Left Outer Join。在场景2中，由于需要包含没有读过书的学生，所以使用了Left Join，并将学生表作为Outer端。

### 过滤条件位置对语义的影响

继续看下面的场景：

场景3：查看所有学生的“图书1”的读书记录，包含没有读过“图书1”的学生

```sql
-- left join，inner端带on条件
select s_id, r_bookid, r_duration
from student left join readlog on s_id = r_stuid and r_bookid = 1
order by s_id, r_bookid;

 s_id | r_bookid | r_duration
------+----------+------------
    1 |        1 |          2
    2 |        1 |          4
    3 |          |
    4 |          |
(4 rows)
```

上述查询有2个关联条件：（1）涉及2个表的条件``s_id = r_stuid``，（2）涉及1个表的条件``r_bookid = 1``，两个条件都在Join on子句里面。

这样的关联逻辑与Outer Join一样，满足这两个条件的行将被匹配，不满足条件的Outer端将被保留，然后Inner端补空。

所以，如果条件中包含Inner端的过滤条件，在Join时，匹配的行保留，不匹配的行去除，相当于需要先将Inner端过滤一下，再处理其他关联条件，这种过滤条件可以被推入Inner端。如果条件中包含Outer端的过滤条件，在Join时，匹配的行保留，不匹配的行Inner端补空，需要在Join过程中进行过滤，这种过滤条件不能被推入Outer端。

所以，上述查询可以被改写为：

```sql
-- left join，inner端带on条件，条件下推
select s_id, r_bookid, r_duration
from student left join (select * from readlog where r_bookid = 1) as readlog on s_id = r_stuid 
order by s_id, r_bookid;
```

```sql
-- 错误，不等价
-- left join，inner端带where条件，条件不能直接下推，需将outer join转inner join后下推
select s_id, r_bookid, r_duration
from student left join readlog on s_id = r_stuid
where r_bookid = 1
order by s_id, r_bookid;
```

再看以下场景：

场景4：查看所有女生的读书记录，包含没有读过书的女生信息

```sql
-- left join，outer端带where条件
select s_id, s_sex, r_bookid, r_duration
from student left join readlog on s_id = r_stuid
where s_sex = 'f'
order by s_id, r_bookid;

 s_id | s_sex | r_bookid | r_duration
------+-------+----------+------------
    2 | f     |        1 |          4
    2 | f     |        3 |          5
    4 | f     |          |
(3 rows)
```

这里需要1个关联条件和1个过滤条件，过滤条件写在了where子句中。

where条件中的条件在逻辑上，是在Join完之后执行。

所以，如果条件中包含Outer端的过滤条件，在Join后，匹配的行保留，不匹配的行去除，相当于需要先将Outer端过滤一下，再处理其他关联条件，这种过滤条件可以推入Outer端。如果条件中包含Inner端的过滤条件，在Join后，匹配的行保留，不匹配的行去除，同时，Inner端补空的数据也会被清除，这种过滤条件不能简单地推入Inner端，但是如果有这样的过滤条件，Outer Join可以被转换为Inner Join，并按照Inner Join的逻辑进行过滤条件下推。

所以，上述查询可以被改写成：

```sql
-- left join，outer端带where条件，可将条件下推
select s_id, s_sex, r_bookid, r_duration
from (select * from student where s_sex = 'f') as student left join readlog on s_id = r_stuid
order by s_id, r_bookid;
```

```sql
-- 错误，不等价
-- left join，outer端带on条件，条件不能直接下推，而是作为join filter存在
select s_id, s_sex, r_bookid, r_duration
from student left join readlog on s_id = r_stuid and s_sex = 'f'
order by s_id, r_bookid;
```

综上所述，Outer Join的过滤条件位置对语义的影响及处理方式如下：

| 条件位置 | 条件作用表 | 处理方式 | 说明 |
|-|-|-|-|
| Join On | Outer端 | 不可下推 | 在Join时，匹配的行保留，不匹配的行Inner端补空，需要在Join过程中进行过滤 |
| Join On | Inner端 | 可推入Inner端 | 在Join时，匹配的行保留，不匹配的行去除，相当于需要先将Inner端过滤一下，再处理其他关联条件 |
| Where | Outer端 | 可推入Outer端 | 在Join后，匹配的行保留，不匹配的行去除，相当于需要先将Outer端过滤一下，再处理其他关联条件 |
| Where | Inner端 | 转Inner Join | 在Join后，匹配的行保留，不匹配的行去除，同时，Inner端补空的数据也会被清除 |





### 附录


```
-- left join，inner端带where条件，left join可转inner join
select s_id, r_bookid, r_duration
from student left join readlog on s_id = r_stuid
where r_stuid is not null
order by s_id, r_bookid;

-- 错误，不等价
-- left join，inner端带on条件
select s_id, r_bookid, r_duration
from student left join readlog on s_id = r_stuid and r_stuid is not null
order by s_id, r_bookid;
```





