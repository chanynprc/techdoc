## Greenplum资源管理

Greenplum的资源管理分为Resource Queue和Resource Group。

- Resource Queue：Greenplum早期的资源管理方式。主要对并发的Query构建了队列，可以在并发高的时候限制活跃的Query数量，部分Query执行，部分Query等待，间接达到资源管理和限制的目的。此外，Resource Queue还可以设置CPU的优先级。
- Resource Group：Greenplum新的资源管理方式。可以进行资源分组，每个分组内分配相应比例的CPU、内存，也可以构建队列的排队机制。

Resource Queue和Resource Group在使用上，都需要和相应的用户绑定去确定每个会话在什么Queue或Group中执行。

### 资源组Resource Group

Resource Group可以设置每个资源分组内的CPU、内存、并发数的限制。资源分组需要绑定到数据库用户或外部组件（比如PL/Container等），与用户绑定的资源分组的资源限制会应用到所有与该资源分组绑定的用户发起的会话上，与外部组件绑定的资源分组的资源限制会应用到所有涉及该外部组件的查询上。

> 绑定到外部组件的能力只在Greenplum 6的Resource Group中有，其他版本和Resource Queue不支持绑定到外部组件。

和用户绑定的Resource Group使用Linux的cgroup（control group）进行CPU的资源管理，使用vmtracker去跟踪内存的使用。和外部组件绑定的Resource Group使用cgroup去进行CPU和内存的管理，要让内存管理生效，需要让cgroup是top-level的。

#### 并发管理

可以通过`concurrency`属性去配置并发限制，用于限制一个资源分组内可以同时执行的并发Query数，默认值20。在每个资源分组内，资源被分成了`concurrency`个slot，基于这些slot去分配并发的Query以及内存。

Resource Group采用先进先出的队列对资源分组内的并发Query进行管理，当资源分组内的并发Query数超过其`concurrency`时，新开始的Query将被加入等待队列，当一个Query执行结束后，最早进入队列中的Query会开始执行。

可以通过参数`gp_resource_group_bypass`去bypass并发限制，让Query不进入队列而直接执行。此参数可以session级设置。

可以通过参数`gp_resource_group_queuing_timeout`去设置队列中Query的等待时长，超过一定时间后自动取消查询。此参数默认值为0，将无限等待。

#### CPU管理



#### Memory Auditor设置

可以通过`memory_auditor`属性去配置内存管理方式，有2种选项：

1. `vmtracker`：默认值，使用vmtracker去监测管理内存，用于和用户绑定的资源分组。使用这种方式，内存总量和内存的使用在Greenplum控制之下，当内存使用超过限制时，可以通过报错去停止Query的执行，不会造成OOM kill。
2. `cgroup`：使用cgroup去监测管理内存，用于和外部组件绑定的资源分组。使用这种方式，将内存管理主动权交给了操作系统，当内存使用超过限制时，会有系统级的OOM kill。

#### 内存管理



### 附录

其他参数

runaway_detector_activation_percent



