---
title: TiDB 日志备份与 PITR 使用指南
summary: 了解 TiDB 的日志备份与 PITR 功能使用。
aliases: ['/zh/tidb/dev/pitr-usage/']
---

# TiDB 日志备份与 PITR 使用指南

全量备份包含集群某个时间点的全量数据，但是不包含其他时间点的更新数据，而 TiDB 日志备份能够将业务写入 TiDB 的数据记录及时备份到指定存储中。如果你需要灵活的选择恢复的时间点，即实现 Point-in-time recovery (PITR)，可以开启[日志备份](#开启日志备份)，并[定期执行快照（全量）备份](#定期执行全量备份)。

使用 BR 备份与恢复数据前，请先[安装 BR](/br/br-use-overview.md#部署和使用-br) 。

## 对集群进行备份

### 开启日志备份

执行 `br log start` 命令启动日志备份任务，一个集群只能启动一个日志备份任务。该命令输出结果如下：

```shell
tiup br log start --task-name=pitr --pd "${PD_IP}:2379" \
--storage 's3://backup-101/logbackup?access_key=${access_key}&secret_access_key=${secret_access_key}"'
```

日志备份任务启动后，会在 TiDB 集群后台持续地运行，直到你手动将其暂停。在这过程中，TiDB 变更数据将以小批量的形式定期备份到指定存储中。如果你需要查询日志备份任务当前状态，执行如下命令：

```shell
tiup br log status --task-name=pitr --pd "${PD_IP}:2379"
```

日志备份任务状态输出如下：

```
● Total 1 Tasks.
> #1 <
    name: pitr
    status: ● NORMAL
    start: 2022-05-13 11:09:40.7 +0800
      end: 2035-01-01 00:00:00 +0800
    storage: s3://backup-101/log-backup
    speed(est.): 0.00 ops/s
checkpoint[global]: 2022-05-13 11:31:47.2 +0800; gap=4m53s
```

### 定期执行全量备份

快照备份功能可作为全量备份的方法，运行 `br backup full` 命令可以按照固定的周期（比如 2 天）进行全量备份。

```shell
tiup br backup full --pd "${PD_IP}:2379" \
--storage 's3://backup-101/snapshot-${date}?access_key=${access_key}&secret_access_key=${secret_access_key}"'
```

## 进行 PITR

如果你想恢复到备份保留期内的任意时间点，可以使用 `br restore point` 命令。执行该命令时，你需要指定**要恢复的时间点**、**恢复时间点之前最近的快照备份**以及**日志备份数据**。BR 会自动判断和读取恢复需要的数据，然后将这些数据依次恢复到指定的集群。

```shell
br restore point --pd "${PD_IP}:2379" \
--storage='s3://backup-101/logbackup?access_key=${access_key}&secret_access_key=${secret_access_key}"' \
--full-backup-storage='s3://backup-101/snapshot-${date}?access_key=${access_key}&secret_access_key=${secret_access_key}"' \
--restored-ts '2022-05-15 18:00:00+0800'
```

恢复期间，可通过终端中的进度条查看进度，如下。恢复分为两个阶段：全量恢复 (Full Restore) 和日志恢复（Restore Meta Files 和 Restore KV Files）。每个阶段完成恢复后，BR 都会输出恢复耗时和恢复数据大小等信息。

```shell
Full Restore <--------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
*** ["Full Restore success summary"] ****** [total-take=xxx.xxxs] [restore-data-size(after-compressed)=xxx.xxx] [Size=xxxx] [BackupTS={TS}] [total-kv=xxx] [total-kv-size=xxx] [average-speed=xxx]
Restore Meta Files <--------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Restore KV Files <----------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
*** ["restore log success summary"] [total-take=xxx.xx] [restore-from={TS}] [restore-to={TS}] [total-kv-count=xxx] [total-size=xxx]
```

## 清理过期的日志备份数据

如[使用指南概览](/br/br-use-overview.md)所述：

* 进行 PITR 不仅需要恢复时间点之前的全量备份，还需要全量备份和恢复时间点之间的日志备份，因此，对于超过备份保留期的日志备份，应执行 `br log truncate` 命令删除指定时间点之前的备份。**建议只清理全量快照之前的日志备份**。

你可以按照以下步骤清理超过**备份保留期**的备份数据：

1. 查找备份保留期之外的**最近一次全量备份**。
2. 使用 `validate` 指令获取该备份对应的时间点。假如需要清理 2022/09/01 之前的备份数据，则应查找该日期之前的最近一次全量备份，且保证它不会被清理。

    ```shell
    FULL_BACKUP_TS=`tiup br validate decode --field="end-version" --storage "s3://backup-101/snapshot-${date}?access_key=${access_key}&secret_access_key=${secret_access_key}"| tail -n1`
    ```

3. 清理该快照备份 `FULL_BACKUP_TS` 之前的日志备份数据。

    ```shell
    tiup br log truncate --until=${FULL_BACKUP_TS} --storage='s3://backup-101/logbackup?access_key=${access_key}&secret_access_key=${secret_access_key}"'
    ```

4. 清理该快照备份 `FULL_BACKUP_TS` 之前的快照备份数据。

    ```shell
    rm -rf s3://backup-101/snapshot-${date}
    ```

## 性能与影响

### PITR 的性能与影响

### 能力指标

- PITR 恢复速度，平均到单台 TiKV 节点：全量恢复为 280 GB/h ，日志恢复为 30 GB/h
- 使用 BR 清理过期的日志备份数据速度为 600 GB/h

> **注意：**
>
> 以上功能指标是根据下述两个场景测试得出的结论，如有出入，建议以实际测试结果为准：
>
> - 全量恢复速度 = 全量恢复集群数据量 /（时间 * TiKV 数量）
> - 日志恢复速度 = 日志恢复总量 /（时间 * TiKV 数量）

测试场景 1（[TiDB Cloud](https://tidbcloud.com) 上部署）

- TiKV 节点（8 core，16 GB 内存）数量：21
- Region 数量：183,000
- 集群新增日志数据：10 GB/h
- 写入 (INSERT/UPDATE/DELETE) QPS：10,000

测试场景 2（本地部署）

- TiKV 节点（8 core，64 GB 内存）数量：6
- Region 数量：50,000
- 集群新增日志数据：10 GB/h
- 写入 (INSERT/UPDATE/DELETE) QPS：10,000

## 探索更多

* [TiDB 集群备份与恢复实践示例](/br/backup-and-restore-use-cases.md)
* [br 命令行手册](/br/use-br-command-line-tool.md)
* [日志备份与 PITR 架构设计](/br/br-log-architecture.md)
