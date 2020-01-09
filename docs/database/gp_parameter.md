## Greenplum 参数

### 优化器调优

**enable_groupagg**



**gp_enable_hashjoin_size_heuristic**

- 默认值：false
- 含义：如果被设置为true，则在hash join中，使用小表作为内表（建hash表）。如果被设置为false，则使用代价进行判断。

**gp_enable_direct_dispatch**

- 默认值：true
- 含义：如果被设置为true，则会进行分片裁剪。如果被设置为false，则不会进行分片裁剪。

**gp_enable_minmax_optimization**



**gp_enable_multiphase_agg**

- 默认值：





