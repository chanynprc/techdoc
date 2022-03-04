## MySQL概览

### 限制

关于记录数、表数的限制，官方没有一个确定的值。官方表示使用的MySQL库记录数达到了5千万条记录。官方所了解到的MySQL库最大包含20万张表，50亿行记录。

### 数据文件组织

在MySQL的数据文件根目录下，用子文件夹的形式表示各个database。此外还包含一系列的文件，如日志文件、InnoDB的tablespace和日志文件、SSL/RSA证书和密钥文件、数据库运行时pid文件、配置文件等。

### 日志

MySQL的日志文件的默认存储位置直接存在数据根目录下，名字都以hostname命名。

- 错误日志名字为hostname.err
- General Query Log名字为hostname.log
- Binary Log名字为
- Slow Query Log名字为hostname-slow.log

日志的相关操作如下：

```sql
-- 启用General Query Log
SET GLOBAL general_log = 'ON';

-- 启用Slow Query Log
SET GLOBAL slow_query_log = 'ON';
```

也可以在配置文件中添加相应的配置，方法如下：

```bash
# General Query Log
general_log=1 # 1：开启日志，0：关闭日志
general_log_file=/path/to/log/file/osd.log


```

