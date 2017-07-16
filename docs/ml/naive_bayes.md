## Naive Bayes分类器

在机器学习中，Naive Bayes分类器是一种假定各特征之间有强独立性情况下，运用Bayes定理的概率分类器。在NLP中，常被应用于文本类别的预测。它简单、快速、准确、可靠。

### 例子

我们的目标是判断一个句子是不是和Sports相关，训练集有5个句子：

Text | Category
---|---
A great game | Sports
The election was over | Not
Very clean match | Sports
A clean but forgettable game | Sports
It was a close election | Not

a formula

$$x=\frac{-b\pm\sqrt{b^2-4ac}}{2a}$$

end formula

要使用Naive Bayes分类器对“A very close game”进行分类，其本质就是计算这句话主题为Sports的概率。即

```math
P(Sports | A~very~close~game)
```

### 引用

[1] https://yq.aliyun.com/articles/113512 （本文示例等内容主要来自此文章，仅为学习和总结目的，如有侵权，请与笔者联系）

[2] https://en.wikipedia.org/wiki/Naive_Bayes_classifier
