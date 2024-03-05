## Greenplum资源管理

Greenplum的资源管理分为Resource Queue和Resource Group。

- Resource Queue：Greenplum早期的资源管理方式。主要对并发的Query构建了队列，可以在并发高的时候限制活跃的Query数量，部分Query执行，部分Query等待，间接达到资源管理和限制的目的。此外，Resource Queue还可以设置CPU的优先级。
- Resource Group：Greenplum新的资源管理方式。可以进行资源分组，每个分组内分配相应比例的CPU、内存，也可以构建队列的排队机制。

Resource Queue和Resource Group在使用上，都需要和相应的用户绑定去确定每个会话在什么Queue或Group中执行。

### 资源组Resource Group

Resource Group可以设置每个资源分组内的CPU、内存、并发数的限制。资源分组需要绑定到数据库用户或外部组件（比如PL/Container等），与用户绑定的资源分组的资源限制会应用到所有与该资源分组绑定的用户发起的会话上，与外部组件绑定的资源分组的资源限制会应用到所有涉及该外部组件的查询上。

> 绑定到外部组件的能力只在Greenplum 6的Resource Group中有，其他版本和Resource Queue不支持绑定到外部组件。

和用户绑定的Resource Group使用Linux的cgroup（control group）进行CPU的资源管理，使用vmtracker去跟踪内存的使用。和外部组件绑定的Resource Group使用cgroup去进行CPU和内存的管理，要让内存管理生效，需要让cgroup是top-level的。

#### 并发



#### Memory Auditor

`memory_auditor`属性有2种选项：

1. `vmtracker`：默认值，用于和用户绑定的资源分组
2. `cgroup`：用于和外部组件绑定的资源分组

其他参数

runaway_detector_activation_percent



