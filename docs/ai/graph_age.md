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

>>> Tips: 在PostgreSQL中，sequence默认的cache是1，而在Greenplum中，sequence默认的cache是20。所以会发现，在Greenplum中跑AGE的回归，顶点和边的ID对不上。

### AGE的数据组织形式

**元数据**

AGE的元数据存储在名为ag_catalog的schema下，有2个表分别存储图的元数据和label的元数据。

- ag_catalog.ag_graph
- ag_catalog.ag_label

**图的元数据**

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
db01=# select * from ag_graph;
 graphid |     name     |  namespace
---------+--------------+--------------
   17796 | graph_name_1 | graph_name_1
(1 row)
```

**label的元数据**

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
db01=# select * from ag_label;
       name       | graph | id | kind |           relation            |        seq_name
------------------+-------+----+------+-------------------------------+-------------------------
 _ag_label_vertex | 17796 |  1 | v    | graph_name_1._ag_label_vertex | _ag_label_vertex_id_seq
 _ag_label_edge   | 17796 |  2 | e    | graph_name_1._ag_label_edge   | _ag_label_edge_id_seq
(2 rows)
```

当为该图添加带label的顶点或边时，新的label会被创建出来

**数据的组织**

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

**顶点数据**

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

**边数据**

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

### AGE的基础使用示例

**创建插件**

```sql
CREATE EXTENSION age;
```

**加载插件**

```sql
LOAD 'age';
```

也可以通过设置```shared_preload_libraries='age'```来加载插件。

**使用示例**

```sql
SET search_path = ag_catalog, "$user", public;

-- 创建图
SELECT create_graph('graph_name_1');

-- 创建节点（不带label和属性）
SELECT * 
FROM cypher('graph_name_1', $$
    CREATE (n)
$$) as (v agtype);

-- 创建节点（带label和属性）
SELECT * 
FROM cypher('graph_name_1', $$
CREATE (n:Person {name: 'Alice', age: 30})
$$) as (v agtype);

-- 创建边

```










