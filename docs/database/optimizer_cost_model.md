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

$$
\begin{align}
indexiocost &= random\_page\_cost \times N_{index\_page} \times selectivity \\
indexcpucost &= (cpu\_index\_tuple\_cost + qual\_op\_cost) \times N_{index\_tuple} \times selectivity \\
tablecpucost &= (cpu\_tuple\_cost + cpu\_qual\_tuple\_cost) \times N_{tuple} \times selectivity
\end{align}
$$

其中，$$cpu\_qual\_tuple\_cost$$为处理非索引条件时每个Tuple上的代价，$$selectivity$$为算子选择率。

数据IO cost的代价模型相对复杂，需要考虑索引与数据Page的存储地址相关性。如果数据文件按照条件中的字段排序，且该字段上建有索引，则这种索引与数据的存储是相关的，否则是不相关的。相关的索引与数据存储结构中，从索引地址到数据Page的访问只需1次随机块访问，后续均为顺序访问。在非相关结构中，从索引到数据Page的访问均为随机访问。这种差别会带来索引扫描代价的不同。

$$
\begin{align}
maxiocost &= random\_page\_cost \times N_{page} \\
miniocost &= random\_page\_cost \times 1 + seq\_page\_cost \times (N_{page} \times selectivity - 1) \\
tableiocost &= maxiocost + correlation^2 \times (miniocost - maxiocost)
\end{align}
$$

其中，$$correlation$$是统计信息之一，详见《优化器行数估算》一文。

最后，索引扫描的run cost为上述4个cost之和。

### 连接算子

### 引用

[1] http://www.interdb.jp/pg/pgsql03.html

[2] http://www.chenyineng.info/techdoc/docs/database/optimizer_row_estimation
