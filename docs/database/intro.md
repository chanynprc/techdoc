## DBMS系统概述

### 数据库的定义

目前，数据库有两个定义，分别是：

1. 信息的集合
2. 由数据库管理系统管理的数据的集合

### 数据库管理系统满足的条件

- 允许用户使用专门的**数据定义语言**（DDL）来创建新的数据库并指定其**模式**（数据库的逻辑结构）
- 给予用户使用适当的语言来查询数据和修改数据的能力，这种语言通常被称为**查询语言**（query language）或**数据操纵语言**（data-manipulation language, DML）
- 支持对非常大量的数据长期地进行存储，允许高效地存取数据以进行查询和数据库修改
- 使数据具有持久性（durability），即能够从故障、多种类型的错误或者故意滥用中进行恢复
- 控制多个用户同时对数据进行访问，不允许用户间有不恰当的相互影响（称作孤立性（isolation）），并且不会发生在数据上进行了部分的而不是完整的操作的情况（称作原子性（atomicity））

### OLTP vs OLAP

**OLTP**：联机事务处理（Online Transaction Processing）。数据量小，DML频繁

**OLAP**：联机分析处理（Online Analytical Processing）。数据量大，DML少
 
### 说明及引用

（本章内容为《数据库系统实现》（第二版）《DBMS系统概述》一章的读书笔记，仅为学习目的，如有侵权，请与笔者联系）

[1] 《数据库系统实现》第二版
 