# PostgreSQL Lock Wait 追蹤與剖析

> 來源：[digoal - PostgreSQL 锁等待跟踪 (2016-03-18)](https://github.com/digoal/blog/blob/master/201603/20160318_02.md)
>
> 更新於 2026-05-17，補充 PG 9.6~18 現代化 lock monitoring 技術棧

---

## 核心問題：lock wait time 被計入 total duration 但無拆分

`auto_explain` 記錄的 total duration 包含了 lock wait time，但未單獨拆分：

```
-- auto_explain 配置
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1s'
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_timing = true
```

重現步驟：

```sql
-- Session A: 持有 row lock
BEGIN;
UPDATE test2 SET info = 'a' WHERE id = 1;

-- Session B: 等待 row lock
UPDATE test2 SET info = 'b';
-- blocked, 等待 Session A commit
```

Session A `COMMIT` 後，Session B 的 auto_explain log 顯示：

```
duration: 32038.420 ms  plan:
Update on test2 (actual time=32038.418..32038.418 rows=0 loops=1)
  ->  Seq Scan on test2 (actual time=0.014..0.015 rows=1 loops=1)

duration: 32039.289 ms  statement: update test2 set info='b';
```

Seq Scan 本身只花了 0.015ms，但 total duration 高達 32 秒——全數為 lock wait 耗時。

> 補充（Senior Dev）：PG 14+ 的 `auto_explain.log_settings = on` 可記錄當時 optimizer GUC 值，但 lock wait 拆分仍需依靠以下方法。

---

## 方法一：log_lock_waits（無需重編譯）

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
```

當一個 session 等待 lock 超過 `deadlock_timeout` 時，PostgreSQL 會在 log 中輸出：

```
LOG:  process 10220 still waiting for ShareLock on transaction 574614690
      after 1000.036 ms
DETAIL:  Process holding the lock: 9725. Wait queue: 10220.
HINT:  while updating tuple (0,5) in relation "test2"
QUERY:  update test2 set info='b';
```

取得鎖之後，再輸出一條：

```
LOG:  process 10220 acquired ShareLock on transaction 574614690
      after 100194.020 ms
HINT:  while updating tuple (0,5) in relation "test2"
QUERY:  update test2 set info='b';
```

結合三條 log（auto_explain duration + log_lock_waits still waiting + acquired），可精確推算：

```
總耗時 32,039ms = 32,038ms lock wait + 1ms 實際運算
鎖等待開始時間 = log_lock_waits still waiting 時間戳
鎖等待結束時間 = log_lock_waits acquired 時間戳
```

> 補充（Senior Dev）：自 PG 13 起，`log_lock_waits` 輸出還包含 wait_event 資訊，可直接對應 `pg_stat_activity.wait_event`。若搭配 `log_line_prefix = '%m [%p] %q%a '` 中的 `%p`（PID），可以在單條 log 中完成 PID → query → wait duration 的全鏈路追蹤。

---

## 方法二：pg_stat_activity 實時監控（現代推薦，無需重編譯）

PG 9.6+ 的 `pg_stat_activity.wait_event_type` / `wait_event` + PG 14+ 的 `wait_start` 讓實時 lock monitoring 成為可能：

```sql
-- 找出所有正在等待 lock 的 session 及其等待時間
SELECT pid, usename, application_name,
       wait_event_type, wait_event,
       now() - wait_start AS wait_duration,
       pg_blocking_pids(pid) AS blocked_by,
       query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND state <> 'idle'
ORDER BY wait_start;
```

```sql
-- 結合 pg_locks 查看具體 lock 細節
SELECT l.pid, l.locktype, l.mode, l.granted,
       l.relation::regclass AS rel,
       l.transactionid,
       a.wait_event, a.wait_event_type,
       now() - a.wait_start AS wait_duration,
       a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted
ORDER BY a.wait_start;
```

---

## 方法三：LOCK_DEBUG 重編譯（原文方法，最深粒度）

### 啟用步驟

1. 修改 `src/include/pg_config_manual.h`：

```c
#define LOCK_DEBUG
```

2. 重編譯 PostgreSQL：

```bash
make clean
make distclean
./configure ...
make -j 32
make install -j 32
```

3. 配置 postgresql.conf：

```ini
trace_locks = true
```

4. 重啟：

```bash
pg_ctl restart -m fast
```

### 輸出示範（原文完整 Log）

#### Session A（PID 9725）— 持有 lock 者

```
-- 取得 RowExclusiveLock（UPDATE 本身的 row lock）
LockAcquire: lock [13241,53029] RowExclusiveLock
```

#### Session B（PID 10220）— 等待者

```
-- 嘗試取得同一 tuple 的 ExclusiveLock
LockAcquire: lock [13241,53029] ExclusiveLock

-- lock manager 檢查：no conflict（此時 tuple lock 還在 A 手上？實際上是 transaction lock 衝突）
LockAcquire: new: lock(0x7f88ead65ed8) id(13241,53029,0,3,3,1)
             grantMask(0) ... type(ExclusiveLock)
LockCheckConflicts: no conflict: ...
GrantLock: lock(0x7f88ead65ed8) id(13241,53029,0,3,3,1)
           grantMask(80) ... type(ExclusiveLock)
```

#### Session A commit — lock 釋放

```
LockReleaseAll: lockmethod=1
LockReleaseAll done
```

#### Session B — 獲得 lock 並執行

```
LockRelease: lock [13241,53029] ExclusiveLock
LockRelease: found: lock(0x7f88ead65ed8) ...
UnGrantLock: updated: lock(0x7f88ead65ed8) ... type(ExclusiveLock)
CleanUpLock: deleting: lock(0x7f88ead65ed8) ...
duration: 288750.628 ms  plan:  -- 288 秒總耗時（幾乎全為 lock wait）
```

### Lock ID 解碼

格式 `id(db_oid, rel_oid, 0, block, tuple, 1)` → `src/include/storage/lock.h` 定義了完整 lock mode bitmask：

| Lock Mode | Bitmask | 說明 |
|-----------|---------|------|
| AccessShareLock | `1<<0` | SELECT |
| RowShareLock | `1<<1` | SELECT FOR UPDATE / FOR SHARE |
| RowExclusiveLock | `1<<2` | INSERT / UPDATE / DELETE |
| ShareUpdateExclusiveLock | `1<<3` | VACUUM / ANALYZE / CREATE INDEX CONCURRENTLY |
| ShareLock | `1<<4` | CREATE INDEX (non-concurrent) |
| ShareRowExclusiveLock | `1<<5` | Triggers, some DDL |
| ExclusiveLock | `1<<6` | REFRESH MATERIALIZED VIEW CONCURRENTLY |
| AccessExclusiveLock | `1<<7` | ALTER TABLE / DROP TABLE / TRUNCATE |

---

## 三種方法的選擇矩陣

| 方法 | 需要重編譯 | 適用場景 | 粒度 |
|------|-----------|---------|------|
| `log_lock_waits` | **否** | 生產環境常駐，日常 lock monitoring | lock type + waiting PID + duration |
| `pg_stat_activity` | **否** | 即時排查，整合 wait_event + blocking chain | 實時 snapshot |
| `LOCK_DEBUG` + `trace_locks` | **是** | 極端深度除錯、lock manager 內部行為分析 | 每個 lock acquire/release/grant/conflict check |

> 補充（Senior Dev）：2016 年原文中 `LOCK_DEBUG` 是唯一能看到完整 lock 內部行為的方式。現代 PG（14+）的 `pg_stat_activity.wait_event` 已覆蓋 80% 日常需求，再加上 `log_lock_waits` 的 waiting queue 資訊和 `pg_blocking_pids()` 的 blocking tree，幾乎不需要再重編譯。僅在需要分析 **lock conflict matrix 內部狀態**（grantMask / reqMask）或 **lock manager 內部 race condition** 時，才需要 `LOCK_DEBUG`。PG 17 進一步強化了 `pg_stat_activity` 的 `wait_for` 欄位，可顯示正在等待的具體 object（relation / transaction ID / tuple）。

---

## 現代監控棧

```sql
-- 生產環境配置（無需重編譯）
-- postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
log_line_prefix = '%m [%p] %d %u '

-- 定期執行監控 query（cron / pg_cron）
SELECT pid, wait_event_type, wait_event,
       now() - wait_start AS wait_duration,
       pg_blocking_pids(pid) AS blocked_by,
       query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND now() - wait_start > '5s'::interval;

-- 若 wait_duration 持續增加且 blocked_by 不變 → 定位 blocking session
SELECT pid, state, now() - xact_start AS xact_age,
       now() - query_start AS query_age, query
FROM pg_stat_activity
WHERE pid = ANY(pg_blocking_pids(<blocked_pid>));
```

## 原始碼參考

- `src/include/storage/lock.h` — lock mode 定義與 grant mask 結構
- `src/backend/storage/lmgr/lock.c` — `LockAcquire` / `LockRelease` / `LockReleaseAll` / `LOCK_PRINT` / `PROCLOCK_PRINT` / `CleanUpLock` / `GrantLock` / `UnGrantLock`
- `src/backend/storage/lmgr/proc.c` — `ProcSleep`（waiting on lock 的 sleep/wakeup 機制）
- `src/include/pg_config_manual.h` — `LOCK_DEBUG` 開關

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `wait_event_type` / `wait_event` | PG 9.6 | pg_stat_activity 原生等待事件 |
| `pg_blocking_pids(pid)` | PG 9.6 | 一次取得完整 blocking tree |
| `log_lock_waits` 增強 | PG 13 | 附加 wait_event 資訊 |
| `wait_start` | PG 14 | 精確計算等待持續時間 |
| `pg_locks.waitstart` | PG 14 | per-lock 等待開始時間 |
| `pg_stat_activity.wait_for` | PG 17 | 顯示等待的具體 object |
