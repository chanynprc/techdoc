## Architecture of a Database System

### Relational Query Processor

关系查询处理器的输入是一个SQL查询语句，在其内部对SQL语句进行验证、解析和优化，生成执行计划，然后由执行器获取（fatch，或拉取，pull）结果元组。

关系查询处理器可以被看做是一个单用户、单线程的任务。重点在于对数据操作语言（DML，如SELECT、INSERT、UPDATE、DELETE）的处理上，而数据定义语言（DDL）通常不需要查询优化器的参与。

#### Query Parsing and Authorization

查询解析器的主要任务有：（1）检查查询语句语法是否正确，（2）明确名字和引用，（3）将查询语句转换成优化器可用的内部形式，（4）进行权限检查。

查询解析器要将FROM子句中的表都规范化为"server.database.schema.table"的形式。对于不能跨服务器或只连接了一个database的数据库系统而言，"server"和"database"可以省略。

规范化表名之后，查询处理器需调用系统表管理器，检查表是否被注册到系统表中，还可能将表的元数据（metadata）存储到解析树中。与此同时，还利用系统表来确认属性引用的正确性，以及常量表达式、运算符的数据类型和种类。例如表达式`EMP.salary * 1.15 < 75000`中，需要根据`EMP.salary`的数据类型来确定`1.15`、`75000`的类型，以及比较运算符`<`的种类。此外，一些标准的SQL语法检查也被应用其中，比如检查集合操作各分支的兼容性、聚集操作中聚集列的使用、子查询的嵌套等。

如果语句被成功解析，则需要对执行权限进行检查。此处更多地是检查一些更宏观的权限，对于行级的安全检查，可能需要被延后到执行阶段。

在编译解析阶段，还可能进行简单语义层面的约束检查。例如在UPDATE语句中有`SET EMP.salary = -1`的子句，而如果有正值的约束条件，此语句应该要报错。

如果一个查询语句被成功解析并通过权限验证，那么这个查询的内部数据结构将被传递给查询重写模块进行进一步处理。

#### Query Rewrite

#### Query Optimizer

#### Query Executor


### 引用

[1] Joseph M. Hellerstein, Michael Stonebraker, and James Hamilton. 2007. Architecture of a Database System. Found. Trends databases 1, 2 (February 2007), 141-259. DOI=http://dx.doi.org/10.1561/1900000002
