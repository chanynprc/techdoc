## Greenplum 参数

### 优化器调优

**enable_groupagg**



**gp_enable_hashjoin_size_heuristic**

- 默认值：false
- 含义：[true]则在hash join中，使用小表作为内表（建hash表）。[false]则使用代价进行判断

**gp_enable_direct_dispatch**

- 默认值：true
- 含义：[true]则会进行分片裁剪。[false]则不会进行分片裁剪

**gp_enable_minmax_optimization**



**gp_enable_multiphase_agg**

- 默认值：true
- 含义：[true]表示允许多阶段的Agg，[false]表示不允许多阶段的Agg，使用一阶段Agg

**gp_enable_preunique**

**gp_eager_preunique**

**gp_enable_agg_distinct**

- 默认值：true
- 含义：


