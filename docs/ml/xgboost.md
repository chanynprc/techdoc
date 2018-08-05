## XGBoost使用入门

### 导入库

```python
import xgboost as xgb
```

### 导入数据

XGBoost支持多种数据格式，我们这里使用的是LibSVM的数据格式。每行1个样本，第1列是标签列，表示目标分类，后续的每列格式为i:x形式，其中i表示第i个特征，x表示该样本的第i个特征值为x。

```python
dtrain = xgb.DMatrix('train.txt')
dtest = xgb.DMatrix('test.txt')
```

### 指定训练参数

```python
param = {'max_depth':2, 'eta':1, 'silent':1, 'objective':'binary:logistic'}
```

参数说明：

- max_depth：决策树的最大深度，默认值为6，取值范围为[1, ∞)
- eta：为了防止过拟合，且正确收敛，通过此参数使计算变得保守，取值范围为[0, 1]
- silent：控制运行过程是否缄默
- objective：定义学习任务和学习方法

### 训练

```python
num_round = 2
bst = xgb.train(param, dtrain, num_round)
```

其中，num_round指定训练过程的迭代次数。

### 预测

```python
preds = bst.predict(dtest)
print preds
```

preds的输出是概率，如果想输出标签，还需要进一步手动处理一下。

### 观察点

如果想查看每一轮的模型迭代情况，可以添加watch list。

```python
watchlist=[(dtrain,'train'), (dtest,'test')]
bst = xgb.train(param, dtrain, num_round, watchlist)
```

### 实例

至此，一个最基本的使用就完成了。可以使用如下的数据测试一下。

```
//train.txt
0 1:1 2:1
0 1:1 2:2
0 1:2 2:1
0 1:2 2:2
1 1:101 2:101
1 1:101 2:102
1 1:102 2:101
1 1:102 2:102

//test.txt
0 1:2 2:1
0 1:1 2:1
1 1:102 2:101
1 1:101 2:102
```

### 引用

[1] http://xgboost.readthedocs.io/en/latest/get_started.html

[2] https://blog.csdn.net/Oscar6280868/article/details/81117567
