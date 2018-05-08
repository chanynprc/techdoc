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
tableiocost &= correlation^2 \times miniocost + (1 - correlation^2) \times maxiocost
&= maxiocost + correlation^2 \times (miniocost - maxiocost)
\end{align}
$$

其中，$$correlation$$是统计信息之一，详见《优化器行数估算》一文。$$N_{page}$$为考虑缓存因素后的Page数，使用Mackert and Lohman公式计算（见引用[1]）。

最后，索引扫描的run cost为上述4个cost之和。

### 单目算子

#### Sort

我们先研究排序数据能直接在内存中放下的情况。

排序算子的start up cost包括了对数据进行排序的操作。快速排序的时间复杂度是$$O(nlogn)$$，所以start up cost为：

$$
startupcost = childtotalcost + comparison\_cost \times N_{tuple} \times log_{2}(N_{tuple})
$$

排序算子的run cost包括了对排完序的数据进行扫描输出的操作。

$$
runcost = cpu\_operator\_cost \times N_{tuple}
$$

如果不能用内排序，而要考虑外排序的话，则需要将下盘量考虑在内。要计算下盘量，要确定需要执行多少轮下盘，每一轮需要对所有数据进行并归。

数据的宽度为$$width$$，那么数据量可以简化为：

$$
inputbytes = width \times N_{tuple}
$$

假设可用于排序的内存为$$sort\_mem\_bytes$$，需要进行排序的轮次为：

$$
nsort = inputbytes / sort\_mem\_bytes
$$

假设可用内存可供$$nmerge$$个排序子序列进行并归，则需要下盘的轮次为：

$$
ndisk = log_{nmerge}(nsort)
$$

下盘Page数为（2表示一读一写）：

$$
npage = 2 \times N_{page} \times ndisk
$$

所以，下盘带来的代价为（认为下盘时有1/4的Page为随机读写，3/4的Page为顺序读写）：

$$
sort\_disk\_cost = (seq\_page\_cost \times 0.75 + random\_page\_cost \times 0.25) \times npage
$$

### 连接算子

#### Nest Loop Join

（1）参与Nest Loop Join的两个表均为顺序扫描

Nest Loop Join不需要做什么准备工作，它的start up cost为0。

Nest Loop Join的run cost由3个部分组成，outer的扫描代价，inner的扫描代价，以及Join本身的代价。

$$
\begin{align}
runcost &= Cost_{outer\_seqscan} + Cost_{inner\_seqscan} \times N_{outer\_tuple} \\
runcost &+= (cpu\_tuple\_cost + cpu\_operator\_cost) \times N_{outer\_tuple} \times N_{inner\_tuple}
\end{align}
$$

（2）参与Nest Loop Join的外表为顺序扫描，内表为物化

start up cost依然为0。

run cost与普通Nest Loop Join的区别在于，内表扫描的是物化后的rescan，而不用再去扫内表。

$$
\begin{align}
Cost_{inner\_rescan} &= cpu\_operator\_cost \times N_{inner\_tuple} \\
runcost &= Cost_{outer\_seqscan} + Cost_{inner\_materialize} + Cost_{inner\_rescan} \times (N_{outer\_tuple} - 1) \\
runcost &+= (cpu\_tuple\_cost + cpu\_operator\_cost) \times N_{outer\_tuple} \times N_{inner\_tuple}
\end{align}
$$

（3）参与Nest Loop Join的外表为顺序扫描，内表为索引扫描

这种情况下，start up cost就不能认为是0了，要考虑索引访问的代价，认为是索引访问1条内表元素的代价。

这里直接计算total cost：

$$
\begin{align}
totalcost &= Cost_{outer\_seqscan} \\
totalcost &+= (cpu\_tuple\_cost + Cost_{inner\_parameterized}) \times N_{outer\_tuple}
\end{align}
$$

当然，外表也可以使用索引，这里就不讨论它的代价模型了。

#### Merge Join

start up cost是内表和外表进行排序的代价。

run cost的代价就是将外表内表顺序扫描一遍，其复杂度为$$O(N_{outer\_tuple} + N_{inner\_tuple})$$。

#### Hash Join

Hash Join的代价模型较为复杂，具体代价模型此处不多展开，待后续补充讨论。简单地说，Hash Join的start up cost用于建立Hash表，run cost用于进行Probe。

&&
\begin{align}
startupcost &= (cpu\_operator\_cost \times N_{hashclauses} + cpu\_tuple\_cost) \times N_{inner\_tuple} \\
runcost &= cpu\_operator\_cost \times N_{hashclauses} \times N_{outer\_tuple} \\
runcost &+= qual\_cost \times N_{bucket\_size} \times 0.5 \times N_{outer\_tuple}
\end{align}
&&

粗略地讲，理想状况下，如果所有操作能在内存中完成，start up cost的复杂度为$$N_{inner\_tuple})$$，run cost的复杂度为$$O(N_{outer\_tuple} + N_{inner\_tuple})$$。

如果考虑内存放不下Hash表时，则需每次处理1个Batch，要考虑Hash Batch下盘的情况。此外，如果有一个特别的存储MCV值的Skew Batch，则能减少下盘量，代价模型又进一步复杂起来。

### 引用

[0] Code of PostgreSQL

[1] Lothar F. Mackert and Guy M. Lohman. 1989. Index scans using a finite LRU buffer: a validated I/O model. ACM Trans. Database Syst. 14, 3 (September 1989), 401-424.

[2] http://www.interdb.jp/pg/pgsql03.html

[3] http://www.chenyineng.info/techdoc/docs/database/optimizer_cardinality_estimation
