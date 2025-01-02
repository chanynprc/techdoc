## 数据写入

### Starrocks的数据写入

| 导入方式        | 主要场景                                                     | 同步、异步 | 执行主体                           | 技术原理                                                     |
| --------------- | ------------------------------------------------------------ | ---------- | ---------------------------------- | ------------------------------------------------------------ |
| INSERT+FILES()  | 从云存储或HDFS中导入文件                                     | 同步       | FE，串行                           |                                                              |
| Broker Load     | 大规模数据导入<br>外部存储系统文件导入，文件较多、较大场景   | 异步       | BE，并行                           | 接口：LOAD LABEL xxx WITH BROKER xxx<br/>各个BE从数据源（HDFS、S3、OSS、NAS等）拉取数据并把数据导入到StarRocks中 |
| Stream Load     | 实时/近实时导入<br>Flink Connector导入<br>本地文件导入，文件较少较小场景 | 同步       | 一个BE作为Coordinator BE导入，单点 | 接口：CURL<BR/>基于HTTP PUT（非SQL），作业请求提交到FE，轮询选定BE执行作业 |
| Routine Load    | 流式导入，实时数据导入，例行持续导入                         | 持续       | 并行                               | CREATE ROUTINE LOAD xxx  FROM KAFKA xxx<br/>从Kafka中消费数据，FE将任务拆分成多个Task，每个Task负责一部分数据导入，并使用Stream Load导入数据，FE不间断产生Task，并对失败的Task进行重试 |
| Spark Load      | 初次迁移，大数据导入（TB）                                   |            |                                    | CREATE EXTERNAL RESOURCE<br/>LOAD LABEL xxx WITH RESOURCE xxx<br/>通过外部Spark资源实现导入数据的预处理，提高导入性能，接上StarRocks资源 |
| Pipe            | 大规模导入、不间断导入场景                                   | 持续异步   | FE，串行                           | 持续的INSERT+FILES() 大量文件分批导入，在多个事务中持续导入，并可以监听文件变化，导入新文件，中间结果用户可见 |
| Flink Connector |                                                              |            |                                    |                                                              |
| Kafka Connector | 消费Kafka                                                    |            |                                    |                                                              |
| Spark Connector |                                                              |            |                                    |                                                              |

### Snowflake数据写入

#### Snowflake数据写入概述

Snowflake的数据写入，主要是基于数据文件的，数据文件可以存放于2种不同的位置，Snowflake把他们称为stage：

- 外部存储（external stage）：可以从AWS S3、Google Cloud Storage、Microsoft Azure中加载数据，但不能加载存储在归档存储中的数据（比如Amazon S3 Glacier Flexible Retrieval、Glacier Deep Archive storage class、Microsoft Azure Archive Storage），此存储位置可以使用CREATE STAGE去创建。
- 内部存储（internal stage）：在Snowflake内部维护了几个存储数据的位置，包括3种方式：用户（分别分配给了每个用户，用于单个用户的文件暂存和管理，不支持更新和删除）、表（分别分配给了每个表，可多个用户共用，不支持更新和删除）、命名（Named，可以被多个用户暂存和管理加到到多个表的文件，此方式可以使用CREATE STAGE去创建）。

> External stage需要包含访问外部云存储的URL（一般会在URL中包含数据目录地址）、外部云存储的登录信息（用户名、密码），这些信息可以直接使用`CREDENTIALS`属性去显式制定，或者通过`STORAGE_INTEGRATION`属性指定。

Snowflake提供批量导入和连续加载的导入方式：

- 批量导入（COPY）：可以使用COPY命令进行批量导入。COPY的数据来源可以来自外部存储和内部存储中的文件。在COPY的过程中，可以对数据进行基本的转换，比如列的重新排序、列省略、类型转换、截断超出目标列长度的文本字符串，数据文件无需与目标表有相同数量和顺序的列。
- 连续加载（Snowpipe）：使用Snowpipe进行连续加载，旨在加载微批数据。在数据被添加到stage并提交到ingestion后的几分钟内，Snowpipe会加载数据。在内部的实现也是用COPY，在数据转换能力方面和批量导入一致。
- 连续加载（Snowpipe Streaming）：Snowpipe Streaming可以让数据直接将数据写入到Snowflake的数据表中，无需落到stage中，可以支持更低的延迟，使Snowflake具备处理近乎实时的数据的能力。此外，Snowpipe Streaming可以和Snowflake Connector for Kafka结合。

Snowflake还支持联邦查询去直接查询外部数据，无需将数据导入到Snowflake中：

- 外表：数据可以存储在外部云存储中，可以在这些数据的子集上创建物化视图，以提高查询性能。

#### Snowpipe

当数据文件在stage中被准备好的时候，Snowpipe会尽快将数据文件加载到Snowflake中。这使得Snowpipe可以在分钟级以微批的形式加载数据，而不需要手动调度COPY命令进行批量导入。使用Snowpipe导入时，计算资源方面使用的是Snowflake提供的计算资源，而非客户的virtual warehouse资源，这些资源会自动进行扩缩容和升降配，这些资源的使用是需要客户按量付费的。

数据的导入有2种机制：

- 基于消息的自动导入：利用云存储的新文件添加的消息通知机制，Snowpipe会去捕获这些事件通知，然后将这些文件加入导入队列，导入到相应的表中。
- 调用REST接口：调用REST接口提供pipe名和文件名列表，在调用接口时如果发现有匹配文件名列表的文件，则将他们加入导入队列。这种方式不是完全自动的导入，需要用户业务调用REST接口。

Snowpipe可以从external stage、table和named internal stage导入数据，不支持从user internal stage导入数据。

#### Snowpipe Streaming



### 附录

#### Snowpipe的模拟方式

一般数据库只有手动COPY方式或外表导入的方式，没有自动监测数据目录中数据文件变化并进行自动导入的功能。通过业务侧实现此功能需要考虑以下几个方面：

- 如何保存已导入的数据文件列表？需要持久化到数据库中的表
- 如何监测数据目录中有新增文件？需要扫描数据目录中的文件列表，和保存的已导入的文件列表做对比，可以先写入一个临时表，然后两个表做except
- 如何进行增量导入？在得到要导入的文件列表后，可以将这些数据文件再建一个外表，然后用这个外表导入到内表中
- 如何进行周期性的调度？比如PG系产品可以使用pg_cron进行周期调度
- 数据目录中文件被删除，执行逻辑是怎样的？无任何动作
- 数据目录中文件被删除，再创建同名文件，执行逻辑是什么样的？无任何动作，新文件的数据不会被写入
- 数据目录中文件的内容被修改，执行逻辑是怎样的？无任何动作，不跟踪文件修改

简单的demo如下：

1、创建表，存储已加载的文件列表

```sql
-- 创建测试database（管理员账户）
create database db01;
grant all on database db01 to dbuser;
-- psql -d db01 -U dbuser（用户账户）
create schema pipe_toolkit;
create table pipe_toolkit.loaded_file_list (load_time timestamp, table_name text, file_path text) distributed by (load_time);
```

2、创建FDW相关的server和user mapping（以阿里云OSS为例）

```sql
CREATE SERVER oss_server FOREIGN DATA WRAPPER oss_fdw OPTIONS (endpoint 'oss-cn-xxx.aliyuncs.com', bucket 'bucket_xxx');
CREATE USER MAPPING FOR PUBLIC SERVER oss_server OPTIONS (id 'user_xxx', key 'pwd_xxx');
```

3、创建用户数据表及外表

```sql
create table t1 (a int, b int, c int, d int) distributed by (a);
CREATE FOREIGN TABLE t1_fdw (a int, b int, c int, d int) SERVER oss_server OPTIONS (dir 't1/', format 'text', delimiter ' ');
```

4、创建函数，获取新增的数据文件列表，导入数据

```sql
-- 此为demo实现，在真实实现中，云存储的连接信息、存储位置信息等可以从内表对应的外表中获取
CREATE OR REPLACE FUNCTION load_new_files(tablename text, endpoint text, bucket text, storagepath text, username text, password text)
RETURNS text AS $$
import subprocess

try:
    # 获取云存储中指定目录下的文件列表
    command = 'ossutil64 ls -e ' + endpoint + ' -i ' + username + ' -k ' + password + ' -d oss://' + bucket + '/' + storagepath + ' | grep oss:'
    process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
    output, error = process.communicate()

    if process.returncode == 0:
        # 创建临时表，存储当前云存储中的文件列表
        query = "CREATE TEMP TABLE all_file_list (LIKE pipe_toolkit.loaded_file_list) distributed by (load_time);"
        ans = plpy.execute(query)

        # 写入临时表，内容为当前云存储中的文件列表
        list = output.decode().strip().split('\n')
        for item in list:
            query = "INSERT INTO all_file_list VALUES (current_timestamp, '" + tablename + "', '" + item + "')"
            ans = plpy.execute(query)

        # 对比已导入的数据文件列表和云存储中的文件列表，获取新增文件列表
        query = "SELECT file_path FROM all_file_list WHERE table_name = '" + tablename + "' EXCEPT SELECT file_path FROM pipe_toolkit.loaded_file_list WHERE table_name = '" + tablename + "';"
        ans = plpy.execute(query)

        # 将新增文件逐一导入到数据库中
        filecount = 0
        for item in ans:
            filepath = item["file_path"]
            filename = filepath.split(bucket)[1][1:]
            tableflag = filename.replace('/', '').replace('.', '')
            query = "CREATE FOREIGN TABLE " + tablename + "__" + tableflag + " (LIKE " + tablename + ") server oss_server OPTIONS (filepath '" + filename + "', format 'text', delimiter ' '); INSERT INTO " + tablename + " SELECT * FROM " + tablename + "__" + tableflag + "; INSERT INTO pipe_toolkit.loaded_file_list VALUES (current_timestamp, '" + tablename + "', '" + filepath + "'); DROP FOREIGN TABLE IF EXISTS " + tablename + "__" + tableflag + ";"
            ans = plpy.execute(query)
            filecount = filecount + 1

        # 清理临时表
        query = "DROP TABLE IF EXISTS all_file_list;"
        ans = plpy.execute(query)

        return 'successfully loaded ' + str(filecount) + ' files'
    else:
        return "ossutil error: " + error

except Exception as e:
    return "Exception: " + str(e)

$$ LANGUAGE plpython3u;
```

5、手动调用

```sql
select load_new_files('t1', 'oss-cn-xxx.aliyuncs.com', 'bucket_xxx', 't1/', 'user_xxx', 'pwd_xxx');
```

6、定时调度

```sql
SELECT cron.schedule('* * * * *', 'select load_new_files(''t1'', ''oss-cn-xxx.aliyuncs.com'', ''bucket_xxx'', ''t1/'', ''user_xxx'', ''pwd_xxx'');');
SELECT * FROM cron.job;
```

7、对外接口

```sql
-- 对外接口需要提供表及其外表，从这些内容中可以获取到上述函数所需的所有信息
SELECT add_streaming_pipe('t1', 't1_fdw');
```

