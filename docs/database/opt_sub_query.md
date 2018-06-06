## 子查询

### 概念

在SQL语句中，一个“SELECT...FROM...WHERE”为一个查询块，查询块之间可以相互嵌套。嵌套查询一般由内向外层层处理，先求解子查询，其结果用于求解父查询。

### 子查询与子链接

在PostgreSQL中，对子查询做了分类。

- 子查询为一条完整的查询语句
- 子链接为一个表达式，表达式内部包含查询语句

从直观上来讲，子查询出现在FROM子句中，子链接出现在WHERE子句或HAVING子句中。

### 相关子查询与非相关子查询

相关子查询指子查询的执行依赖于父查询的某些属性值，需要接受父查询的参数。非相关子查询与父查询完全独立。

### 关键字

- EXISTS：声明了EXISTS的子查询
- ALL：声明了ALL或NOT IN的子查询
- ANY：声明了ANY或IN的子查询
- EXPR：子查询返回一个参数给父查询，如“SELECT * FROM t WHERE a > select b from t2”
- MULTIEXPR：子查询返回多个参数给父查询，如“SELECT * FROM t WHERE (a, b) > select c, d from t2”
- ARRAY：子查询为数组

### 引用

[1] 彭智勇, 彭煜玮, PostgreSQL数据库内核分析
