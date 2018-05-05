## 优化器代价模型

本文以PostgreSQL为例，介绍了优化器的代价模型，并进行了部分扩展。

在PostgreSQL中，有3种代价：

- start up cost：输出第1条元组的代价
- run cost：获取所有元组的执行代价
- total cost：总代价，是start up cost与run cost之和

### 扫描算子

#### Sequential Scan

顺序扫描的start up cost可以认为是0，但其实也应包含磁盘寻道、寻找扇区、读取第1个扇区、CPU处理等的时间。

顺序扫描的run cost由2个部分组成，IO cost和CPU cost。磁盘IO要将包含对象表的所有Page读取到内存，CPU需要对所有Tuple进行处理。这里认为所有数据Page都在磁盘上，没有考虑数据Page已经在内存中的情况。

$$
\begin{align}
  runcost 
  &= iocost + cpucost \\
  &= seq\_page\_cost \times N_{page} + (cpu\_tuple\_cost + cpu\_operator\_cost) \times N_{tuple}
\end{align}
$$

这里的$$cpu\_tuple\_cost$$可能根据表存储方式的不同而不同，可以进行识别和适配。$$cpucost$$中还需要加上对输出列处理的代价，此代价只需考虑输出的行数，而不是总行数。

#### Index Scan

索引扫描的start up cost（TBA）

索引扫描的run cost由4个部分组成，索引IO cost、索引CPU cost、数据IO cost和数据CPU cost。

### 连接算子

### 引用

[1] http://www.interdb.jp/pg/pgsql03.html
