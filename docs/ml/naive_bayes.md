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
\left\{
\begin{aligned}
P(Sports | A~very~close~game) \\
P(Not Sports | A~very~close~game)
\end{aligned}
\right.
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

### 模型求解

应用Bayes定理，可以将$P(Sports \| A~very~close~game)$作如下的转换：

$$
P(Sports | A~very~close~game) = \frac{P(Sports) * P(A~very~close~game | Sports)}{P(A~very~close~game)}
$$

其中，$P(Sports)$可以方便地利用训练集计算得出，$P(A~very~close~game)$在

### 引用

[1] https://yq.aliyun.com/articles/113512 （本文示例等内容主要来自此文章，仅为学习和总结目的，如有侵权，请与笔者联系）

[2] https://en.wikipedia.org/wiki/Naive_Bayes_classifier
