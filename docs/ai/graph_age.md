## AGE图插件

### AGE中的ID生成规则

在AGE中，顶点和边都有自己的ID，这些ID的数据类型是```ag_catalog.graphid```，是一个64位整型：

```cpp
typedef int64 graphid;
```

由make_graphid函数根据label ID和entry ID生成graphid：

```cpp
#define ENTRY_ID_BITS (32 + 16)
#define ENTRY_ID_MASK INT64CONST(0x0000ffffffffffff)

graphid make_graphid(const int32 label_id, const int64 entry_id)
{
    tmp = (((uint64)label_id) << ENTRY_ID_BITS) |
          (((uint64)entry_id) & ENTRY_ID_MASK);
}
```

可见，graphid由8个字节组成，高2个字节保存lable ID，低6个字节保存entry ID。其中entry ID是从顶点或边的sequence里面拿到的。

当label ID为1，entry ID为1时，graphid的值是281474976710657。

> Tips: 在PostgreSQL中，sequence默认的cache是1，而在Greenplum中，sequence默认的cache是20。所以会发现，在Greenplum中跑AGE的回归，顶点和边的ID对不上。

### AGE的数据组织形式

#### 元数据

AGE的元数据存储在名为ag_catalog的schema下，有2个表分别存储图的元数据和label的元数据。

- ag_catalog.ag_graph
- ag_catalog.ag_label

#### 图的元数据

ag_catalog.ag_graph的表结构如下：

```sql
                                   Table "ag_catalog.ag_graph"
  Column   |     Type     | Collation | Nullable | Default | Storage | Stats target | Description
-----------+--------------+-----------+----------+---------+---------+--------------+-------------
 graphid   | oid          |           | not null |         | plain   |              |
 name      | name         |           | not null |         | plain   |              |
 namespace | regnamespace |           | not null |         | plain   |              |
Indexes:
    "ag_graph_graphid_index" UNIQUE, btree (graphid)
    "ag_graph_name_index" UNIQUE, btree (name)
    "ag_graph_namespace_index" UNIQUE, btree (namespace)
Referenced by:
    TABLE "ag_label" CONSTRAINT "fk_graph_oid" FOREIGN KEY (graph) REFERENCES ag_graph(graphid)
Access method: heap
```

其中，各个列的含义为：

- graphid：图的oid
- name：图的名字
- namespace：图数据存储的schema，与图的名字一致

```sql
db01=# select * from ag_catalog.ag_graph;
 graphid |     name     |  namespace
---------+--------------+--------------
   17796 | graph_name_1 | graph_name_1
(1 row)
```

#### label的元数据

ag_catalog.ag_label的表结构如下：

```sql
                                  Table "ag_catalog.ag_label"
  Column  |    Type    | Collation | Nullable | Default | Storage | Stats target | Description
----------+------------+-----------+----------+---------+---------+--------------+-------------
 name     | name       |           | not null |         | plain   |              |
 graph    | oid        |           | not null |         | plain   |              |
 id       | label_id   |           |          |         | plain   |              |
 kind     | label_kind |           |          |         | plain   |              |
 relation | regclass   |           | not null |         | plain   |              |
 seq_name | name       |           | not null |         | plain   |              |
Indexes:
    "ag_label_graph_oid_index" UNIQUE, btree (graph, id)
    "ag_label_name_graph_index" UNIQUE, btree (name, graph)
    "ag_label_relation_index" UNIQUE, btree (relation)
    "ag_label_seq_name_graph_index" UNIQUE, btree (seq_name, graph)
Foreign-key constraints:
    "fk_graph_oid" FOREIGN KEY (graph) REFERENCES ag_graph(graphid)
Access method: heap
```

其中，各个列的含义为：

- name：label的名字
- graph：所属图的id
- id：label的id，每个图单独编号，依赖该图数据中的sequence（\_label_id_seq）生成id编号
- kind：label的类型。顶点的label，类型为v；边的lable，类型为e
- relation：存储该label的图数据（顶点或边）的数据表
- seq_name：存储该label的图数据（顶点或边）的数据表所依赖的sequence

在新建一个新的图时，会为顶点和边生成2个默认的label，名字分别叫_ag_label_vertex和_ag_label_edge：

```sql
db01=# select * from ag_catalog.ag_label;
       name       | graph | id | kind |           relation            |        seq_name
------------------+-------+----+------+-------------------------------+-------------------------
 _ag_label_vertex | 17796 |  1 | v    | graph_name_1._ag_label_vertex | _ag_label_vertex_id_seq
 _ag_label_edge   | 17796 |  2 | e    | graph_name_1._ag_label_edge   | _ag_label_edge_id_seq
(2 rows)
```

当为该图添加带label的顶点或边时，新的label会被自动创建出来：

- 元数据部分：会在ag_catalog.ag_label表中新增一行，记录该label的元数据
- 数据部分：在该图的schema下添加一个表和一个sequence，用来存储该label的顶点或边的数据，sequence用来支撑顶点或边的id编号

```sql
db01=# select * from ag_catalog.ag_label;
       name       | graph | id | kind |     relation     |        seq_name
------------------+-------+----+------+------------------+-------------------------
 _ag_label_vertex | 17796 |  1 | v    | _ag_label_vertex | _ag_label_vertex_id_seq
 _ag_label_edge   | 17796 |  2 | e    | _ag_label_edge   | _ag_label_edge_id_seq
 Person           | 17796 |  3 | v    | "Person"         | Person_id_seq
 FRIENDS_WITH     | 17796 |  4 | e    | "FRIENDS_WITH"   | FRIENDS_WITH_id_seq
(4 rows)
```

#### 数据的组织

创建一个新的图之后，初始状态下在图的schema下会创建2个表，3个sequence：

```sql
                                  List of relations
    Schema    |          Name           |   Type   | Owner |    Size    | Description
--------------+-------------------------+----------+-------+------------+-------------
 graph_name_1 | _ag_label_edge          | table    | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_edge_id_seq   | sequence | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_vertex        | table    | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_vertex_id_seq | sequence | pg12  | 8192 bytes |
 graph_name_1 | _label_id_seq           | sequence | pg12  | 8192 bytes |
(5 rows)
```

其中：

- \_label_id_seq：用于支撑label的id编号，每新增一个label，则会从此sequence中取一个数作为id编号
- \_ag_label_vertex：存储顶点数据
- \_ag_label_vertex_id_seq：支撑顶点的id编号
- \_ag_label_edge：存储边数据
- \_ag_label_edge_id_seq：支撑边的id编号

如果顶点或边有label，则还会在专门带label的顶点或边的数据表中存储带label的顶点和边的数据，带label的顶点或边的数据也会被存储于_ag_label_vertex或_ag_label_edge中，即_ag_label_vertex或_ag_label_edge中存储了全量的数据，label的数据表中存储了该label的数据。

```sql
                                  List of relations
    Schema    |          Name           |   Type   | Owner |    Size    | Description
--------------+-------------------------+----------+-------+------------+-------------
 graph_name_1 | FRIENDS_WITH            | table    | pg12  | 16 kB      |
 graph_name_1 | FRIENDS_WITH_id_seq     | sequence | pg12  | 8192 bytes |
 graph_name_1 | Person                  | table    | pg12  | 16 kB      |
 graph_name_1 | Person_id_seq           | sequence | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_edge          | table    | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_edge_id_seq   | sequence | pg12  | 8192 bytes |
 graph_name_1 | _ag_label_vertex        | table    | pg12  | 16 kB      |
 graph_name_1 | _ag_label_vertex_id_seq | sequence | pg12  | 8192 bytes |
 graph_name_1 | _label_id_seq           | sequence | pg12  | 8192 bytes |
(9 rows)
```

#### 顶点的数据

```sql
                                                                                                       Table "graph_name_1._ag_label_vertex"
   Column   |        Type        | Collation | Nullable |                                                                     Default                                                                      | Storage  | Stats target | Description
------------+--------------------+-----------+----------+--------------------------------------------------------------------------------------------------------------------------------------------------+----------+--------------+-------------
 id         | ag_catalog.graphid |           | not null | ag_catalog._graphid(ag_catalog._label_id('graph_name_1'::name, '_ag_label_vertex'::name)::integer, nextval('_ag_label_vertex_id_seq'::regclass)) | plain    |              |
 properties | ag_catalog.agtype  |           | not null | ag_catalog.agtype_build_map()                                                                                                                    | extended |              |
Indexes:
    "_ag_label_vertex_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

顶点数据默认保存在该图schema下的_ag_label_vertex表中，包含2列数据：

- id：顶点的id，依赖_ag_label_vertex_id_seq生成id编号
- properties：顶点的属性

```sql
db01=# select * from graph_name_1._ag_label_vertex;
       id        |           properties
-----------------+--------------------------------
 281474976710657 | {}
 844424930131969 | {"age": 30, "name": "Alice"}
 844424930131970 | {"age": 28, "name": "Bob"}
 844424930131971 | {"age": 32, "name": "Charles"}
(4 rows)
```

#### 边的数据

```sql
                                                                                                      Table "graph_name_1._ag_label_edge"
   Column   |        Type        | Collation | Nullable |                                                                   Default                                                                    | Storage  | Stats target | Description
------------+--------------------+-----------+----------+----------------------------------------------------------------------------------------------------------------------------------------------+----------+--------------+-------------
 id         | ag_catalog.graphid |           | not null | ag_catalog._graphid(ag_catalog._label_id('graph_name_1'::name, '_ag_label_edge'::name)::integer, nextval('_ag_label_edge_id_seq'::regclass)) | plain    |              |
 start_id   | ag_catalog.graphid |           | not null |                                                                                                                                              | plain    |              |
 end_id     | ag_catalog.graphid |           | not null |                                                                                                                                              | plain    |              |
 properties | ag_catalog.agtype  |           | not null | ag_catalog.agtype_build_map()                                                                                                                | extended |              |
Indexes:
    "_ag_label_edge_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

边的数据默认保存在该图schema下的_ag_label_edge表中，包含4列数据：

- id：边的id，依赖_ag_label_edge_id_seq生成id编号
- start_id：边的起点顶点id
- end_id：边的终点顶点id
- properties：边的属性

```sql
db01=# select * from graph_name_1._ag_label_edge;
        id        |    start_id     |     end_id      | properties
------------------+-----------------+-----------------+------------
 1125899906842625 | 844424930131969 | 844424930131970 | {}
(1 row)
```

### Cypher语言中的要点

#### 独立的模式匹配：多个MATCH子句组合查询

当需要从数据库中匹配多个相互独立的模式时，可以使用多个MATCH子句。每个MATCH子句会匹配其指定的模式，然后结果会被组合在一起。

注意：多个MATCH子句的结果是进行笛卡尔积组合的，除非在查询中使用了其他限制条件（如WHERE子句）来过滤结果。因此，在构建查询时要小心，以避免意外的结果集膨胀。

```sql
SELECT * FROM cypher('graph_name_1', $$
  MATCH (a:Person {name: 'Alice'})
  MATCH (b:Person {name: 'Bob'})
  RETURN a, b
$$) as (a agtype, b agtype);
```

#### 相关联的模式匹配：多个MATCH子句级联查询

当需要在一个查询中匹配多个相关联的模式时，多个MATCH子句可以用于逐步构建查询逻辑。上一级的MATCH子句会匹配出相应的顶点或边，在下一级的MATCH子句会根据上一级匹配出的顶点或边再去图中匹配这一级的关系。

注意：在级联查询方式下，查询结果也是进行笛卡尔积的，上一级MATCH子句中相同的顶点或边在下一级中会分别进行匹配并返回结果。

```sql
-- 此例子中，第一个MATCH子句会匹配出来满足条件的a顶点，第二个MATCH子句会根据这个a顶点再去图中匹配所有以它为顶点的关系
-- 在此文档构造的例子中，第一个MATCH子句匹配出2个关系，a顶点是2个Alice，第二个MATCH会查找以Alice为起点的关系，每个Alice有3个关系，2个Alice共6个关系，此语句最后返回6行数据
SELECT * FROM cypher('graph_name_1', $$
  MATCH p1=()<-[]-(a:Person)-[]->(b {name: 'Bob'})
  MATCH p2=(a)-[]->()
  RETURN p2
$$) as (p agtype);
```

### AGE的基础使用示例

#### 创建插件

```sql
CREATE EXTENSION age;
```

#### 加载插件

```sql
LOAD 'age';
```

也可以通过设置```shared_preload_libraries='age'```来加载插件。

#### 使用示例

```sql
SET search_path = ag_catalog, "$user", public;

-- 创建图
SELECT create_graph('graph_name_1');

-- 创建节点（不带label和属性）
SELECT * FROM cypher('graph_name_1', $$
  CREATE (n)
$$) as (n agtype);

-- 创建节点（带label和属性）
SELECT * FROM cypher('graph_name_1', $$
  CREATE (n:Person {name: 'Alice', age: 30})
$$) as (n agtype);
SELECT * FROM cypher('graph_name_1', $$
  CREATE (n:Person {name: 'Bob', age: 28})
$$) as (n agtype);
SELECT * FROM cypher('graph_name_1', $$
  CREATE (n:Person {name: 'Charles', age: 32})
$$) as (n agtype);
SELECT * FROM cypher('graph_name_1', $$
  CREATE (n:Person {name: 'David', age: 28})
$$) as (n agtype);

-- 创建边
SELECT * FROM cypher('graph_name_1', $$
  MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
  CREATE (a)-[e:FRIENDS_WITH]->(b)
$$) as (e agtype);
SELECT * FROM cypher('graph_name_1', $$
  MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Charles'})
  CREATE (a)-[e:FRIENDS_WITH]->(b)
$$) as (e agtype);
SELECT * FROM cypher('graph_name_1', $$
  MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'David'})
  CREATE (a)-[e:FRIENDS_WITH]->(b)
$$) as (e agtype);

-- 查询顶点
SELECT * FROM cypher('graph_name_1', $$
  MATCH (n:Person)
  RETURN n.name, n.age
$$) as (name agtype, age agtype);

-- 使用WHERE子句进行过滤
SELECT * FROM cypher('graph_name_1', $$
  MATCH (n:Person)
  WHERE n.age > 30
  RETURN n.name, n.age
$$) as (name agtype, age agtype);

-- 查询具有特定关系的顶点
SELECT * FROM cypher('graph_name_1', $$
  MATCH (a:Person)-[:FRIENDS_WITH]->(b:Person)
  RETURN a.name, b.name
$$) as (v1_name agtype, v2_name agtype);

SELECT * FROM cypher('graph_name_1', $$
  MATCH (b:Person)<-[:FRIENDS_WITH]-(a:Person)-[:FRIENDS_WITH]->(c:Person)
  RETURN a.name, b.name, c.name
$$) as (v1_name agtype, v2_name agtype, v3_name agtype);

-- 更新顶点的属性
SELECT * FROM cypher('graph_name_1', $$
   MATCH (n:Person {name: 'Alice'})
   SET n.age = 31
   RETURN n
$$) as (n agtype);

-- 删除顶点和边（顶点和其关联的边都会被删除）
SELECT * FROM cypher('graph_name_1', $$
  MATCH (n:Person {name: 'David'})
  DETACH DELETE n
$$) as (n agtype);

-- MERGE子句（尝试查找或创建一个名称为David的节点。如果节点不存在，则创建该节点并设置age为25；如果节点已经存在，则将age属性加1）
-- 验证失败
SELECT * FROM cypher('graph_name_1', $$
  MERGE (n:Person {name: 'David'})
  ON CREATE SET n.age = 25
  ON MATCH SET n.age = n.age + 1
$$) as (n agtype);
```










