# PostgreSQL track_commit_timestamp — PG 17 視角

> 來源：[digoal - PostgreSQL 9.5 new feature - record transaction commit timestamp (2015-04-09)](https://github.com/digoal/blog/blob/master/201504/20150409_03.md)
>
> 本文基於原始 PG 9.5 文章，以 PG 17 (2026) 為基準更新：補足實際用途、效能影響、版本演進。

---

## 功能概述

`track_commit_timestamp` 自 PG 9.5 引入，每筆 transaction commit 時記錄其 timestamp。儲存在 PGDATA 下 `pg_commit_ts/` 目錄，結構類似 `pg_xact`（原 `pg_clog`），以 SLRU (Simple Least Recently Used) 機制管理。

### 儲存結構

- 每個 page 大小 = `BLCKSZ`（預設 8KB）
- 每筆 transaction 佔 12 bytes：`TimestampTz` (8 bytes) + `CommitTsNodeId` (4 bytes)

```c
// src/backend/access/transam/commit_ts.c
typedef struct CommitTimestampEntry
{
    TimestampTz   time;
    CommitTsNodeId nodeid;
} CommitTimestampEntry;

#define SizeOfCommitTimestampEntry \
    (offsetof(CommitTimestampEntry, nodeid) + sizeof(CommitTsNodeId))

#define COMMIT_TS_XACTS_PER_PAGE \
    (BLCKSZ / SizeOfCommitTimestampEntry)
```

PGDATA 目錄結構：

```
$ cd $PGDATA
drwx------  pg_commit_ts/
```

Transaction ID 是 32-bit 會 wraparound，因此 `pg_commit_ts` 的 page / segment 編號也會對應 wraparound。截斷邏輯見 `TruncateCommitTs` → `CommitTsPagePrecedes`。

---

## 啟用與確認

```ini
# postgresql.conf
track_commit_timestamp = on
```

**必須 restart**（此參數為 `postmaster` 級別，無法 reload）：

```bash
pg_ctl restart -m fast
```

確認狀態：

```bash
pg_controldata | grep track
# Current track_commit_timestamp setting: on
```

`pg_resetxlog`（PG 10+ 改名 `pg_resetwal`）重置 WAL 時需指定安全範圍：

```bash
# -c xid,xid 指定最舊 / 最新可查 commit timestamp 的 xid 範圍
# 可從 pg_commit_ts/ 目錄中最小的 file name (hex) 推算
pg_resetwal -c 0x00000100,0x0000FF00
```

> 補充（Senior Dev）：如果 `track_commit_timestamp = off` 時資料庫曾運行過，之後再開啟，中間缺失的 transaction commit timestamp 會是 NULL。`pg_xact_commit_timestamp(xid)` 對這些 xid 回傳 NULL。

---

## 實際用途（PG 9.5 → PG 17 演進）

### 2015 年（PG 9.5，原文時期）

當時作者推測與 snapshot too old、logical replication 有關，但尚未明確。

### 2026 年（PG 17）

**1. 查詢 commit timestamp：`pg_xact_commit_timestamp(xid)`**

```sql
SELECT pg_xact_commit_timestamp(txid_current());
-- 2026-05-17 15:30:45.123456+08

-- 查詢特定 transaction 的 commit time
SELECT xmin, pg_xact_commit_timestamp(xmin)
FROM some_table WHERE id = 1;
```

**2. Logical Replication（PG 10+）**

Logical decoding 內部依賴 commit timestamp 來決定 transaction 的 ordering 與可見性。若未開啟 `track_commit_timestamp`，logical replication 仍可運作，但某些 replication slot 行為（如 `pg_logical_slot_get_changes` 的 `upto_lsn` / `upto_nchanges`）在跨 node consistency 場景會有限制。

**3. Snapshot too old（PG 9.6+）**

`old_snapshot_threshold` 依賴 commit timestamp 來判斷哪些 row version 過期可回收。未開啟 `track_commit_timestamp` 時此功能無法使用。

**4. 審計 / CDC 場景**

可直接透過 xmin / xmax + `pg_xact_commit_timestamp()` 還原 row-level 的變更時間線，無需額外 `updated_at` column。

**5. Extension 生態**

- `pg_ivm`（Incremental View Maintenance）：依賴 commit timestamp 追蹤增量變更
- 部分 CDC tool（如 Debezium PG connector）在 snapshot 模式下查詢 commit timestamp 做 watermark

| PG 版本 | 相關演進 |
|---------|---------|
| 9.5 | 引入 `track_commit_timestamp`、`pg_xact_commit_timestamp()` |
| 9.6 | `old_snapshot_threshold` 依賴 commit timestamp |
| 10 | Logical replication 正式 GA，重命名 `pg_resetxlog` → `pg_resetwal` |
| 14 | SLRU 效能改進，`pg_commit_ts` lookup 更快 |
| 15 | `pg_stat_reset_slru()` 可監控 `pg_commit_ts` SLRU 命中率 |

---

## 效能影響與 Production 考量

> 補充（Senior Dev）：

| 面向 | 影響 |
|------|------|
| **寫入效能** | 每次 commit 多一次 SLRU write（~12 bytes），OLTP 場景 overhead 約 1-3%（視 workload，write-heavy 時更明顯） |
| **儲存空間** | 每百萬 transaction 約 12MB；需關注 SLRU truncation 是否及時（與 `vacuum_freeze_min_age` 等參數相關） |
| **讀取效能** | `pg_xact_commit_timestamp(xid)` 查詢走 SLRU buffer，hit 時幾乎零成本，miss 時觸發 page read |
| **Replication** | Logical replication 場景建議開啟，避免 corner case |

**建議：**

- **OLTP 核心庫**：若不需要審計 / logical replication / snapshot too old，關閉可節省 1-3% write overhead
- **需要 logical replication**：開啟，這是 PG 內部依賴
- **需要 row-level 時間追蹤但不想開全域**：使用 `updated_at timestamp DEFAULT now()` column，別開 `track_commit_timestamp`（更輕量、更可控）

---

## 原始碼參考（2015 年原文 + 17 對應）

| 檔案路徑 | 說明 |
|---------|------|
| `src/backend/access/transam/commit_ts.c` | Commit timestamp 核心模組（PG 17 仍同路徑） |
| `src/backend/access/rmgrdesc/committsdesc.c` | WAL record descriptor for commit_ts |
| `man pg_resetwal` | PG 10+ 改名（原 `pg_resetxlog`） |
