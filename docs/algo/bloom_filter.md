## Bloom Filter

Bloom Filter在1970年由Burton Howard Bloom提出，是一种空间效率很高的数据结构，用于判别元素是否是某集合的成员。

如果Bloom Filter判别元素在集合中，则该元素不一定在该集合中；如果Bloom Filter判别元素不在集合中，则该元素一定不在该集合中。即Bloom Filter可能产生False Positive，但一定不会产生False Negative。也就是说，Bloom Filter的返回结果有两种：

1. 可能在集合中
2. 一定不在集合中

Bloom Filter一旦被建立，它的集合可以添加元素，但不可以删除元素（Counting Bloom Filter可以删除元素）。越多的元素被添加到集合中，False Positive的可能性就越大。

### 算法原理

一个空的Bloom FIlter是一个m位的bit数组，每一位都被置“0”。此外，有k个不同的Hash函数，每一个Hash函数将一个元素映射到m位bit数组中的一位。m和k均为常数，它们的选择，由对False Positive Rate决定。

![](/techdoc/docs/algo/images/649px-Bloom_filter.svg.png)

如上图所示，在这个Bloom Filter中，`m=18`，`k=3`。

在集合中添加元素的时候，x被3个Hash函数映射成`{1, 5, 13}`，y被映射成`{4, 11, 16}`，z被映射成`{3, 5, 11}`。每在集合中添加一个元素，就先计算这个元素的Hash值（共k个），然后将对应位置“1”。

在判别元素是否属于集合时，w被3个Hash函数映射成`{4, 13, 15}`，查找bit数组中的对应位，如果都为“1”，则输出“w可能在集合中”，如果不都为“1”，则输出“w一定不在集合中”。因为，即使如果都是“1”，也存在Hash冲突的问题，命中对应位置的不一定是w。

### 引用

[1] https://en.wikipedia.org/wiki/Bloom_filter