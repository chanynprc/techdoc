## Bloom Filter

Bloom Filter在1970年由Burton Howard Bloom提出，是一种空间效率很高的算法，用于判别元素是否是某集合的成员。

有很多场景需要判断元素是否为集合中的成员。比如FBI判断一个嫌疑人是不是在通缉犯列表中，网络爬虫判断某个URL是不是已经被分析，判断用户名是不是已被注册等。

### 算法原理

要判断一个元素是否在集合中，有这样几种方法：（1）扫描集合中的每个元素，与目标元素进行对比，时间复杂度高，（2）使用Hash表进行对比，需保存元素的Hash值以及指向元素本身的指针，该指针用于处理Hash冲突，因为多个元素可能拥有同样的Hash值，这时候就要对比元素本身了，这种方法空间复杂度高。

Bloom Filter是Hash表方法的一种变体，使用一个bit数组作为Hash表，每个bit的下标就是Hash值，bit的为“0”表示该Hash值未被命中，“1”表示被命中。因为存在Hash冲突的问题，Bloom Filter采用多个独立的Hash函数计算多个Hash值。具体算法如下：

一个空的Bloom FIlter是一个m位的bit数组，每一位都被置“0”。此外，有k个不同的Hash函数，每一个Hash函数将一个元素映射到m位bit数组中的一位。m和k均为常数，由用户选定。

![](/techdoc/docs/algo/images/649px-Bloom_filter.svg.png)

如上图所示，在这个Bloom Filter中，`m=18`，`k=3`。

在集合中添加元素的时候，每在集合中添加一个元素，就先计算这个元素的Hash值（共k个），然后将对应位置“1”。

在判别元素是否属于集合时，将目标元素通过k个Hash函数映射成数组中的对应位。

- 在bit数组中，如果对应位都为“1”，则输出“可能在集合中”，因为，即使如果都是“1”，也存在Hash冲突的问题，不止是目标元素会命中对应位置。
- 在bit数组中，如果对应位不都为“1”，则输出“一定不在集合中”。

在图中的例子里，添加了3个元素x、y和z到集合中。x被3个Hash函数映射成`{1, 5, 13}`，y被映射成`{4, 11, 16}`，z被映射成`{3, 5, 11}`。所以bit数组中的第1、3、4、5、11、13、16位被置“1”。目标元素w被3个Hash函数映射成`{4, 13, 15}`，第15位不为“1”，所以判定元素w一定不在集合中。

### 算法特点

如果Bloom Filter判别元素在集合中，则该元素不一定在该集合中；如果Bloom Filter判别元素不在集合中，则该元素一定不在该集合中。即Bloom Filter可能产生False Positive，但一定不会产生False Negative。也就是说，Bloom Filter的返回结果有两种：

1. 可能在集合中
2. 一定不在集合中

Bloom Filter一旦被建立，可以向它的集合中添加元素，但不可以删除元素（它的一种扩展Counting Bloom Filter可以删除元素）。越多的元素被添加到集合中，False Positive的可能性就越大。

算法参数m和k的选择，由用户期望的False Positive Rate决定。

### 引用

[1] https://en.wikipedia.org/wiki/Bloom_filter

[2] https://yq.aliyun.com/articles/84938