## PostgreSQL FDW (Foreign Data Wrapper)

### FDW在各个PG版本的演进

- PG 9.1
从9.1版本开始，PostgreSQL引入了FDW。在此版本中，FDW只支持对Foreign Table的Scan操作，Callback Routines也寥寥无几。
- PG 9.2
增加了获取统计信息的接口。

| | 9.1 | 9.2 | 9.3 | 9.4 | 9.5 | 9.6 | 10 | 11 | 12 |
| - | - | - | - | - | - | - | - | - | - |
|GetForeignRelSize |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|PlanForeignScan | ✓ |  |  |  |  |  |  |  |  |
|GetForeignPaths |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|GetForeignPlan |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|BeginForeignScan | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|IterateForeignScan | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ReScanForeignScan | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|EndForeignScan | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|GetForeignJoinPaths |  |  |  |  | ✓ | ✓ | ✓ | ✓ | ✓ |
|GetForeignUpperPaths |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|AddForeignUpdateTargets |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|PlanForeignModify |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|BeginForeignModify |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ExecForeignInsert |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ExecForeignUpdate |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ExecForeignDelete |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|EndForeignModify |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|BeginForeignInsert |  |  |  |  |  |  |  | ✓ | ✓ |
|EndForeignInsert |  |  |  |  |  |  |  | ✓ | ✓ |
|IsForeignRelUpdatable |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|PlanDirectModify |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|BeginDirectModify |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|IterateDirectModify |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|EndDirectModify |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|GetForeignRowMarkType |  |  |  |  | ✓ | ✓ | ✓ | ✓ | ✓ |
|RefetchForeignRow |  |  |  |  | ✓ | ✓ | ✓ | ✓ | ✓ |
|RecheckForeignScan |  |  |  |  | ✓ | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|ExplainForeignScan | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ExplainForeignModify |  |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|ExplainDirectModify |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|AnalyzeForeignTable |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
|AcquireSampleRowsFunc |  | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|ImportForeignSchema |  |  |  |  | ✓ | ✓ | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|IsForeignScanParallelSafe |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|EstimateDSMForeignScan |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|InitializeDSMForeignScan |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|ReInitializeDSMForeignScan |  |  |  |  |  |  | ✓ | ✓ | ✓ |
|InitializeWorkerForeignScan |  |  |  |  |  | ✓ | ✓ | ✓ | ✓ |
|ShutdownForeignScan |  |  |  |  |  |  | ✓ | ✓ | ✓ |
| |  |  |  |  |  |  |  |  |  |
|ReparameterizeForeignPathByChild |  |  |  |  |  |  |  | ✓ | ✓ |

### 引用

[1] https://www.postgresql.org/docs/current/fdw-callbacks.html
