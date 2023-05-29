```sql
SET 'pipeline.name' = 'taskname';
set 'state.backend'='EmbeddedRocksDBStateBackend';
SET 'state.checkpoints.dir' = 'hdfs:///bar/foo/';
--#EXACTLY_ONCE或者AT_LEAST_ONCE
SET 'execution.checkpointing.mode' = 'EXACTLY_ONCE';
SET 'table.local-time-zone' = 'Asia/Shanghai';
SET 'execution.checkpointing.interval' = '30min';
SET 'execution.checkpointing.timeout' = '30min';
set 'execution.checkpointing.unaligned'= 'true';
set 'state.checkpoints.num-retained'='1';
set 'table.exec.state.ttl'='60min';
set 'execution.checkpointing.aligned-checkpoint-timeout' = '20min';
SET 'execution.checkpointing.min-pause' = '20min';
SET 'execution.checkpointing.max-concurrent-checkpoints' = '1';
SET 'execution.checkpointing.prefer-checkpoint-for-recovery' = 'true';
```
