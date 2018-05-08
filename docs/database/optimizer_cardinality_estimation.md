## 优化器行数估算

### 统计信息

#### Page数

数据表所占Page数。

#### Tuple数

数据表的Tuple数。

#### Distinct值

数据中不同的值的个数。

#### 空值率

数据中空值的比例。

#### 平均宽度

数据的平均宽度。

#### MCV (most common values)

MCV表示出现最多的一些数据，它们的比例是MCF (most common frequency)。

在收集统计信息时，MCV值不会太多，需要满足一定的条件。比如PostgreSQL中的条件是：

- 数量比平均值多25%以上（mincount = avgcount * 1.25）
- 至少有2个值（mincount = MAX(mincount, 2)）
- 为了避免直方图中一个值出现跨多个快的情况，数量比直方图1个块多的值需要是MCV值（mincount = MIN(mincount, rows / bins)）

#### 等高直方图

对数据进行划分，分成等概率的一些块。直方图块与块之间可能不是均匀分布的。在计算选择率时，认为直方图块内部是均匀分布的。

在PostgreSQL的实现中，等高直方图的值不包含MCV的值。

#### （逻辑排序与物理排序的）相关性

描述数据的逻辑排序与物理排序的相关情况，取值范围为[-1, 1]。相关性在估算索引扫描的代价时需要用到。

比如，如果数据在物理存储时按照{1, 2, 3, 4, 5, 6, 7, 8, 9}存储，则其相关性为1，如果数据存储为{9, 8, 7, 6, 5, 4, 3, 2, 1}，则其相关性为-1，如果顺序杂乱，则相关性在(-1, 1)区间。

### 行数估算方法

#### 条件 a = 1

“a = 1”条件分两种情况，

1. 值在MCV中，直接用MCV中的频率作为Selectivity
2. 值不在MCV中，认为非MCV值是均匀分布的，则其Selectivity为$$(1 - sum(MCF)) / (distinct - num\_mcv)$$

#### 条件 a > 1

“a > 1”条件的选择率由2部分组成，（1）MCV中满足该条件的值，（2）直方图中满足条件的值。

所以计算步骤如下：

1. 遍历MCV值，找到满足条件的值，对它们的频率求和
2. 扫描直方图，计算直方图中满足条件的比例，乘以直方图总体比例
3. 对MCV比例和直方图比例进行求和，计算总的Selectivity
4. 根据总行数和Selectivity估算输出行数

#### 多条件AND

多条件的行数估算，在没有多列统计信息的情况下，假设多个条件之间是相互独立的，将他们的Selectivity相乘，作为总的Selectivity，并计算相应行数。

#### Join

如果Join条件列没有MCV值，则Join的Selectivity为：

$$
selectivity = (1 - null\_frac1) \times (1 - null\_frac2) \times min(1/distinct1, 1/distinct2)
$$

如果Join条件列有MCV值，则需要先考虑MCV值上Join的Selectivity，然后按照上述方式处理非MCV值的Selectivity。

### 引用

[1] https://www.postgresql.org/docs/10/static/index.html
