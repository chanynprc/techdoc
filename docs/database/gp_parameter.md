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
- 含义：[true]对单个agg+distinct开启2阶段聚集，[false]关闭2阶段聚集

**gp_enable_agg_distinct_pruning**

- 默认值：true
- 含义：[true]对agg+distinct+group by和多agg+distinct使用3阶段聚集或Join，[false]不使用3阶段聚集或Join

**gp_enable_groupext_distinct_pruning**

- 默认值：true
- 含义：

**gp_enable_groupext_distinct_gather**

**gp_eager_agg_distinct_pruning**

**gp_eager_one_phase_agg**

- 默认值：false
- 含义：[true]忽略代价，而尽量使用1阶段聚集，[false]按照代价进行选择

**gp_eager_two_phase_agg**

- 默认值：false
- 含义：[true]忽略代价，而尽量使用2阶段聚集，[false]按照代价进行选择


**gp_enable_sort_limit**

**gp_enable_sort_distinct**

**gp_enable_mk_sort**

**gp_enable_motion_mk_sort**

**gp_hashagg_streambottom**

**gp_adjust_selectivity_for_outerjoins**















