## AGE图插件

### AGE中的ID生成规则

在AGE中，顶点和边都有自己的ID，这些ID的数据类型是```ag_catalog.graphid```，是一个64位整型：

```cpp
typedef int64 graphid;
```

由make_graphid函数根据label ID和entry ID生成graphid：

```cpp
#define ENTRY_ID_BITS (32 + 16)
#define ENTRY_ID_MASK INT64CONST(0x0000ffffffffffff)

graphid make_graphid(const int32 label_id, const int64 entry_id)
{
    tmp = (((uint64)label_id) << ENTRY_ID_BITS) |
          (((uint64)entry_id) & ENTRY_ID_MASK);
}
```

可见，graphid由8个字节组成，高2个字节保存lable ID，低6个字节保存entry ID。其中entry ID是从顶点或边的sequence里面拿到的。

当label ID为1，entry ID为1时，graphid的值是281474976710657。

>>> Tips: 在PostgreSQL中，sequence默认的cache是1，而在Greenplum中，sequence默认的cache是20。所以会发现，在Greenplum中跑AGE的回归，顶点和边的ID对不上。


