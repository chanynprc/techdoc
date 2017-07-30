## Naive Bayes分类器

在机器学习中，Naive Bayes分类器是一种假定各特征之间有强独立性情况下，运用Bayes定理的概率分类器。在NLP中，常被应用于文本类别的预测。它简单、快速、准确、可靠。

### 例子及其建模

我们的目标是判断一个句子是不是和Sports相关，训练集有5个句子：

Text | Category
---|---
A great game | Sports
The election was over | Not Sports
Very clean match | Sports
A clean but forgettable game | Sports
It was a close election | Not Sports

要使用Naive Bayes分类器对“A very close game”进行分类，其本质就是计算这句话主题为Sports和Not Sports的概率，即

$$
\begin{cases}
P(Sports | A~very~close~game) \\
P(Not Sports | A~very~close~game)
\end{cases}
$$

比较它们的大小，“A very close game”属于概率较大的那一类。

### Bayes定理

Bayes定理的公式表示为：

$$
P(A|B) = \frac{P(A)*P(B|A)}{P(B)}
$$

可以使用图形来帮助理解Bayes公式。

![](/techdoc/docs/ml/images/bayes.png)

在上图中，要表示$P(A \cap B)$，可以有两种形式：

$$
P(A \cap B) = P(A)*P(B|A) = P(B)*P(A|B)
$$

经过转换，很容易得出Bayes公式。

### 模型求解I

应用Bayes定理，可以将$P(Sports \| A~very~close~game)$作如下的转换：

$$
P(Sports | A~very~close~game) = \frac{P(Sports) * P(A~very~close~game | Sports)}{P(A~very~close~game)}
$$

其中，$P(Sports)$属于先验概率，可以方便地利用训练集计算得出，$P(A~very~close~game)$在各分类中都包含，而我们只需比较概率大小，可以不予计算。所以问题集中于计算$P(A~very~close~game \| Sports)$。

### 特征提取

在机器学习中，非常关键的一环就是特征提取，它对机器学习效果起着决定性作用。

在文本分类中，常常使用词频作为文本的特征。一篇文章或一个文本由若干个词组成，假设我们不考虑上下文关系，这些词都是一个个独立的个体。通过统计每个词出现的次数，计算它们出现的频率，作为这篇文章或文本的特征。虽然这种方法忽略了上下文、词与词之间的连接关系，但在文本分类的实践中，被证明是简单而有效的。

### 模型求解II

由于假设文本中词的出现是相互独立的，那么可对模型进行处理：

$$
P(A~very~close~game | Sports) = P(A|Sports)*P(very|Sports)*P(close|Sports)*P(game|Sports)
$$

这样，计算就方便直观多了。首先计算先验概率，训练集中Sports分类出现了3次，训练集大小为5，则$P(Sports)=3/5$，$P(Not Sports)=2/5$。在Sports分类中，单词“A”出现了2次，单词“very”出现了1次，单词“close”出现了0次，单词“game”出现了2次，Sports分类共11个词，所以：

$$
\begin{cases}
P(A|Sports)=2/11 \\
P(very|Sports)=1/11 \\
P(close|Sports)=0/11 \\
P(game|Sports)=2/11
\end{cases}
$$

这个结果相乘以后会导致$P(A~very~close~game \| Sports)$结果为0，也就是说，一旦测试集中含有训练集中未出现的单词，该条训练集可能面临计算出概率为0的情况，这对我们的计算是没有意义的。

所以，为了解决这个问题，我们引入Laplace smoothing。简单来说，就是将词典中的词语计数都加1，以消除某特征概率为0的情况。在实际的文本分类中，词典很大，文本相对而言较小，所以在应用Laplace smoothing后，不会对计算的概率造成非常大的影响。对于此例，我们可认为词典为

```
A great game The election was over Very clean match but forgettable It close
```

那么，重新计算后的概率为：

$$
\begin{cases}
P(A|Sports)=\frac{2+1}{11+14} \\
P(very|Sports)=\frac{1+1}{11+14} \\
P(close|Sports)=\frac{0+1}{11+14} \\
P(game|Sports)=\frac{2+1}{11+14}
\end{cases}
$$

所以：

$$
P(Sports) * P(A~very~close~game | Sports) = 4.61*10^{-5}
$$

同理可得：

$$
P(Not~Sports) * P(A~very~close~game | Not~Sports) = 1.43*10^{-5}
$$

结论是文本“A very close game”属于Sports分类。

### 更进一步的调优方法

还有一些其他方法，可以帮助Naive Bayes分类器提升效果。

- 删除Stop words：删除一些对分类结果没有帮助的词，比如“A”、“the”、“was”等。
- 变体还原（lemmatization）：将单词的变体进行还原，将一组词结合在一起，看成一个特征，比如“better”和“good”看成一组，“game”和“match”看成一组。
- 使用N-gram：对于一些常用而明显的词组，将他们看成一个整体，而不是直接拆成一个词一个词地分析。
- 使用TF-IDF：不仅仅计算每个词的出现概率，而且将每个词在文本中的重要程度也考虑进去进行加权。

### 引用

[1] https://yq.aliyun.com/articles/113512 （本文示例等内容主要来自此文章，仅为学习和总结目的，如有侵权，请与笔者联系）

[2] https://en.wikipedia.org/wiki/Naive_Bayes_classifier

[3] https://en.wikipedia.org/wiki/Additive_smoothing

[4] https://en.wikipedia.org/wiki/Stop_words

[5] https://en.wikipedia.org/wiki/Lemmatisation

[6] http://sebastianraschka.com/Articles/2014_naive_bayes_1.html

[7] https://en.wikipedia.org/wiki/Tf%E2%80%93idf