## 优化器代价模型

本文以PostgreSQL为例，介绍了优化器的代价模型，并进行了部分扩展。

在PostgreSQL中，有3种代价：

- start up cost：输出第1条元组的代价
- run cost：获取所有元组的执行代价
- total cost：总代价，是start up cost与run cost之和

### 扫描算子

#### Sequential Scan

start up cost = 0

run cost = IO cost + CPU cost

$$
\begin{align}
  run cost 
  &= IO cost + CPU cost \\
  &= (cpu_tuple_cost + cpu_operator_cost) \times N_{tuple} + seq_page_cost \times N_{page}
\end{align}
$$

### 连接算子

### 引用

[1] http://www.interdb.jp/pg/pgsql03.html
