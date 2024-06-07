## Greenplum集群管理

### 概述

### 代码走读

#### gpinitsystem

gpinitsystem是用shell写的，相对比较难看懂。它的-c参数的输入文件gpinitsystem_config是一个shell变量定义文件，在gpinitsystem中会source这个文件来获取定义的变量。

gpinitsystem使用的变量及含义如下：

| 变量名                           | 含义                                                         | 来源                          |
| -------------------------------- | ------------------------------------------------------------ | ----------------------------- |
| INPUT_CONFIG                     | 指代-i参数                                                   | 命令行                        |
| CLUSTER_CONFIG                   | 指代-c参数                                                   | 命令行                        |
|                                  |                                                              |                               |
| MACHINE_LIST_FILE                | 非必须，可以被-h参数替代，segment节点的host列表，不包含Coordinator | -c配置文件                    |
| SEG_PREFIX                       | 必须，-c和-I配置文件都包含                                   | -c及-I配置文件                |
| PORT_BASE                        | 必须                                                         | -c配置文件                    |
| DATA_DIRECTORY                   | 必须，数组类型，每个host上primary的数据目录                  | -c配置文件<br/>-I配置文件生成 |
| COORDINATOR_HOSTNAME             | 必须                                                         | -c配置文件                    |
| COORDINATOR_DIRECTORY            | 必须                                                         | -c配置文件                    |
| COORDINATOR_PORT                 | 必须                                                         | -c配置文件                    |
| TRUSTED_SHELL                    | 必须，-c和-I配置文件都包含                                   | -c及-I配置文件                |
| ENCODING                         | 必须，-c和-I配置文件都包含                                   | -c及-I配置文件                |
| DATABASE_NAME                    | 非必须                                                       | -c配置文件                    |
| MIRROR_PORT_BASE                 | 非必须                                                       | -c配置文件                    |
| MIRROR_DATA_DIRECTORY            | 非必须，数组类型，每个host上mirror的数据目录                 | -c配置文件<br/>-I配置文件生成 |
| PRIMARY_ARRAY                    | -I配置文件包含                                               | -I配置文件                    |
| MIRROR_ARRAY                     | -I配置文件包含                                               | -I配置文件                    |
| STANDBY_HOSTNAME                 |                                                              |                               |
|                                  |                                                              |                               |
| QD_PRIMARY_ARRAY                 | -I配置文件包含，或根据-c配置文件生成                         | -I配置文件<br/>-c配置文件生成 |
| QE_PRIMARY_ARRAY                 |                                                              | 生成                          |
| QE_MIRROR_ARRAY                  |                                                              | 生成                          |
| MIRRORING                        | 若在-c配置文件中定义了MIRROR_PORT_BASE，或在-I配置文件中定义了MIRROR_ARRAY，则此值为1 |                               |
| MACHINE_LIST                     | hostaddress列表                                              | -c或-I配置文件生成            |
| TOTAL_SEG                        | primary个数                                                  |                               |
| TOTAL_MIRRORS                    | mirror个数                                                   |                               |
| QE_PRIMARY_COUNT<br/>NUM_DATADIR | 每个host上primary的个数                                      |                               |
| NUM_MIRROR_DIRECTORY             | 每个host上mirror的个数，需要与每个host上primary个数相等      |                               |
| NUM_QES                          | host个数                                                     |                               |
| M_HOST_ARRAY                     | 类似T_HOST_ARRAY，形如“ip~hostname”的数组，数组长度为primary数 |                               |
| NUM_SEP_HOSTS                    | 按照hostname去重后的host个数                                 |                               |
|                                  |                                                              |                               |
|                                  |                                                              |                               |
|                                  |                                                              |                               |
|                                  |                                                              |                               |
|                                  |                                                              |                               |



```
# gpinitsystem

CHK_GPDB_ID #
CHK_PARAMS # 对参数正确性进行检查，在这一步会读取集群配置文件（-c参数指定或-I参数指定），并对文件中的变量进行基础校验
CHK_MULTI_HOME # 只在-c参数下执行，根据IP获取hostname，主要是处理一个hostname对应多个IP的情况
CREATE_QE_ARRAY # 只在-c参数下执行，构造Coordinator和segment的排布描述（QD_PRIMARY_ARRAY、QE_PRIMARY_ARRAY、QE_MIRROR_ARRAY），这个描述在-I参数模式中由配置文件编写
CHK_QE_ARRAY_PORT_RANGES #
CHK_QES_FROM_INPUTFILE # 只在-i参数下执行
CHK_QES # 只在-c参数下执行
DISPLAY_CONFIG
ARRAY_REORDER # 只在-c参数下执行

CREATE_QD_DB

CREATE_ARRAY_SORTED_ON_CONTENT_ID

CREATE_SEGMENT IS_PRIMARY

STOP_QD_PRODUCTION
START_QD_PRODUCTION

CREATE_GPEXTENSIONS
IMPORT_COLLATION

CREATE_DATABASE # 如果设置了DATABASE_NAME参数，则创建数据库

SET_GP_USER_PW

# 只在有MIRRORING时执行
REGISTER_MIRRORS
CREATE_SEGMENT IS_MIRROR
FORCE_FTS_PROBE

# 只在有STANDBY_HOSTNAME时执行
CREATE_STANDBY_QD


```

