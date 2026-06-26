# 修复：BulkWriter.doUpdate 空 models 导致 BulkWrite 报错

## 需求描述

**现象：** 当一批 update oplog 全部是 time-series bucket 更新（含 column-store binary diff `sdata.b`），`BulkWriter.doUpdate` 会将所有 oplog 都 fallback 到 `replayUpdateViaApplyOps`，导致 `models` 切片为空。随后 `BulkWrite` 调用报错 "must provide at least one element in input slice"。

**预期行为：** 当所有 oplog 都已通过 applyOps 回放成功时，`doUpdate` 应直接返回 nil，不再调用 BulkWrite。

## 根因分析

`executor/db_writer_bulk.go` 的 `BulkWriter.doUpdate` 方法中，当处理 `$v:2` diff 格式的 update oplog 时：

1. `oplog.DiffUpdateOplogToNormal()` 尝试将 diff 转换为标准 `$set/$unset` 操作
2. 对于 time-series bucket 更新（含 `sdata.b` column-store binary diff），转换会失败
3. 代码检测到 `system.buckets.` 前缀后，fallback 到 `replayUpdateViaApplyOps` 并 `continue`
4. 如果整个 batch 中的所有 oplog 都走了 fallback 路径，`models` 切片为空
5. 空 `models` 传入 `BulkWrite` 触发 "must provide at least one element" 错误

对比同一文件中的其他方法：
- `doInsert`（line 50）已有 `if len(models) == 0 { return nil }` 保护
- `doUpdateOnInsert`（line 155）已有同样的保护
- `CommandWriter.doUpdate`（line 308）已有 `if len(updates) == 0 { return nil }` 保护

唯独 `BulkWriter.doUpdate` 缺少这个检查。

## 修复方案

### 1. 空切片检查

在 `BulkWriter.doUpdate` 的 `BulkWrite` 调用前添加空切片检查：

```go
if len(models) == 0 {
    // all oplogs were replayed via applyOps (e.g., time-series bucket updates)
    return nil
}
```

### 2. 简化 fallback 日志

三个 writer（bulk/command/single）的 fallback 日志原先打印了完整的 `oplogErr`，其中包含大量二进制数据。简化为仅打印 namespace：

```go
// Before
l.Logger.Infof("fall back to applyOps for time-series bucket update on %s.%s: %v",
    database, collection, oplogErr)

// After
l.Logger.Infof("fall back to applyOps for time-series bucket update on %s.%s",
    database, collection)
```

## 影响范围

- **文件：** `executor/db_writer_bulk.go`、`executor/db_writer_command.go`、`executor/db_writer_single.go`
- **场景：** 增量同步 time-series collection 的 bucket 更新
- **风险：** 低。仅在 models 为空时跳过 BulkWrite 调用，不影响正常写入路径
