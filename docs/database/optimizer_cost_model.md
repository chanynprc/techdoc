## 优化器代价模型

本文以PostgreSQL为例，介绍了优化器的代价模型，并进行了部分扩展。

在PostgreSQL中，有3种代价：

- start up cost：输出第1条元组的代价
- run cost：获取所有元组的执行代价
- total cost：总代价，是start up cost与run cost之和

### 扫描算子

#### Sequential Scan

顺序扫描的start up cost可以认为是0，但其实也应包含磁盘寻道、寻找扇区、读取第1个扇区、CPU处理等的时间。

顺序扫描的run cost由两部分组成，IO cost和CPU cost。磁盘IO要将包含对象表的所有Page读取到内存，CPU需要对所有Tuple进行处理。这里认为所有数据Page都在磁盘上，没有考虑数据Page已经在内存中的情况。

$$
\begin{align}
  runcost 
  &= iocost + cpucost \\
  &= seq\_page\_cost \times N_{page} + (cpu\_tuple\_cost + cpu\_operator\_cost) \times N_{tuple}
\end{align}
$$

### 连接算子

### 引用

[1] http://www.interdb.jp/pg/pgsql03.html
