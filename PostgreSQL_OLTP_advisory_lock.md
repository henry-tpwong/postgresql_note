# PostgreSQL OLTP 高併發請求效能優化 — Advisory Lock 排隊機制

> 來源：[digoal - PostgreSQL OLTP 高並發請求性能優化 (2015-10-08)](https://github.com/digoal/blog/blob/master/201510/20151008_01.md)
>
> 相關：[PostgreSQL 秒殺場景優化](https://github.com/digoal/blog/blob/master/201509/20150914_01.md)

---

## 核心問題：並發數超過 CPU 核數後 TPS 急降

在多核系統中，TPS 一般隨 concurrent 數增加而提升，但當 concurrent 數超過 CPU 核數的 2~3 倍後，效能開始下降，concurrent 數越高，下降越嚴重。

> 補充（Senior Dev）：這是由多個因素疊加造成的：
> 1. **Context switching 開銷**：backend process 數量遠超 CPU 核數時，OS scheduler 頻繁切換導致 CPU cache miss 暴增
> 2. **Snapshot 管理成本**：每個 active backend 都需要在 transaction 開始時建立 snapshot（掃描 ProcArray），8000 個 backend 的 ProcArrayLock contention 極其嚴重
> 3. **Lock contention**：大量 backend 同時競爭 row-level lock（`UPDATE` 同一列）會造成 `LWLock` / `spinlock` 風暴
> 4. **Memory 壓力**：每個 backend 消耗 ~5-10 MB（work_mem 另計），8000 個 backend ≈ 40-80 GB memory，遠超 shared_buffers 可承載，OS 被迫 swap / page fault
>
> PostgreSQL 的 process-per-connection 模型在極高 concurrent 下的可擴展性問題，直到 PG 14（snapshot scalability 改進）和 PG 17（進一步的 ProcArray 優化）才有實質改善。即使是 PG 17，也不建議直接讓 client 連接數超過 500-1000。

### 實驗一：8000 個 active backend 全部執行 UPDATE

建立測試表（500 萬 row）：

```sql
CREATE TABLE test_8000 (id INT PRIMARY KEY, cnt INT DEFAULT 0);
INSERT INTO test_8000 SELECT generate_series(1, 5000000);
```

測試 SQL（每次隨機選一條 row 執行兩次 UPDATE）：

```sql
\setrandom id 1 5000000
UPDATE test_8000 SET cnt = cnt + 1 WHERE id = :id;
UPDATE test_8000 SET cnt = cnt + 2 WHERE id = :id;
```

分批載入 80 concurrent × 100 次 = 8000 個 active backend：

```bash
#!/bin/bash
for ((i=0; i<100; i++))
do
  sleep 1
  pgbench -M simple -n -r -f ./t.sql -c 80 -j 80 -T 100000 -U postgres &
done
```

執行 `./test.sh`，待連接數達 8000 後計算 QPS：

```sql
SELECT count(*) FROM pg_stat_activity;  -- 8002 (含查詢本身)
```

使用 `pg_stat_all_tables` 的 `n_tup_upd + n_tup_hot_upd` 差值計算：

```sql
-- 第一次取樣
SELECT now(), n_tup_upd + n_tup_hot_upd FROM pg_stat_all_tables
WHERE relname = 'test_8000';
-- now: 2015-10-08 17:01:16.574076+08
-- ?column?: 43749480

-- 第二次取樣
SELECT now(), n_tup_upd + n_tup_hot_upd FROM pg_stat_all_tables
WHERE relname = 'test_8000';
-- now: 2015-10-08 17:01:24.203089+08
-- ?column?: 43819090
```

```sql
SELECT timestamptz '2015-10-08 17:01:24.203089+08'
     - timestamptz '2015-10-08 17:01:16.574076+08';
-- 時間差: 00:00:07.629013

SELECT 43819090 - 43749480;
-- update 次數: 69610

SELECT 69610 / 07.629013;
-- TPS: 9124.38
```

**結果：8000 concurrent → TPS ≈ 9,124**，大部分時間消耗在 CPU scheduling 上。

### 實驗二：8000 idle backend + 10 active backend

先建立 8000 個 idle 連接（每個執行 `SELECT pg_sleep(100000)` 佔住連接）：

```sql
-- test.sql
SELECT pg_sleep(100000);
```

```bash
#!/bin/bash
for ((i=0; i<100; i++))
do
  sleep 1
  pgbench -M simple -n -r -f ./test.sql -c 80 -j 80 -T 100000 -U postgres &
done
```

```sql
SELECT count(*) FROM pg_stat_activity;  -- 8002 (idle)
```

然後用 10 concurrent 執行 update（`-M prepared`）：

```
pgbench -M prepared -n -r -f ./t.sql -P 1 -c 10 -j 10 -T 1000
```

```
progress:  1.0 s, 29429.2 tps, lat 0.336 ms stddev 0.109
progress:  2.0 s, 28961.1 tps, lat 0.343 ms stddev 0.114
progress:  3.0 s, 30433.8 tps, lat 0.326 ms stddev 0.103
progress:  4.0 s, 29597.1 tps, lat 0.336 ms stddev 0.114
progress:  5.0 s, 28714.1 tps, lat 0.346 ms stddev 0.117
progress:  6.0 s, 28319.0 tps, lat 0.351 ms stddev 0.121
progress:  7.0 s, 28540.0 tps, lat 0.348 ms stddev 0.118
progress:  8.0 s, 29408.9 tps, lat 0.338 ms stddev 0.111
progress:  9.0 s, 29178.1 tps, lat 0.340 ms stddev 0.119
progress: 10.0 s, 29146.9 tps, lat 0.341 ms stddev 0.118
progress: 11.0 s, 27498.5 tps, lat 0.361 ms stddev 0.123
```

**結果：10 active + 7990 idle → TPS ≈ 29,000**（idle connection 對效能幾乎無影響）。

> 補充（Senior Dev）：這驗證了關鍵結論——**只有真正執行 SQL 的 backend 才會參與 CPU 競爭**。8000 idle 連接只佔用 memory（每個約 5-10MB ≈ 40-80GB），不參與 ProcArray lock / snapshot 競爭。但這不代表 idle 連接無害：大量 idle 連接仍會讓 `pg_stat_activity` 掃描變慢、VACUUM 凍結 age 計算變重、connection storm 觸發時 OS 資源瞬間耗盡。實務中應設置 `idle_in_transaction_session_timeout` 和 `idle_session_timeout`（PG 14+）。

---

## 解法：Advisory Lock 模擬排隊（Admission Control）

德哥的思路：限制真正並行處理的 backend 數量，多出限制的連線等待。類似 pgbouncer 的 transaction pooling 或 Oracle shared server。

使用 PostgreSQL 的 advisory lock function 實現應用層排隊：

```sql
CREATE OR REPLACE FUNCTION upd(l INT, v_id INT) RETURNS void AS $$
DECLARE
BEGIN
  LOOP
    -- 嘗試獲取 advisory lock（transaction-scoped）
    IF pg_try_advisory_xact_lock(l) THEN
      -- 只有拿到 lock 才執行 update
      UPDATE test_8000 SET cnt = cnt + 1 WHERE id = v_id;
      UPDATE test_8000 SET cnt = cnt + 2 WHERE id = v_id;
      RETURN;
    ELSE
      -- 拿不到 lock 就隨機等待後重試
      PERFORM pg_sleep(30 * random());
    END IF;
  END LOOP;
END;
$$ LANGUAGE plpgsql STRICT;
```

> 補充（Senior Dev）：`pg_try_advisory_xact_lock(l)` vs `pg_try_advisory_lock(l)` 的關鍵差異：
> - `_xact_` 版本：lock 在 transaction commit/rollback 時自動釋放，不會因為 function 返回而提前釋放
> - 非 `_xact_` 版本：lock 在 session 結束時才釋放，或者手動 `pg_advisory_unlock()`
>
> 這裡用 `_xact_` 是正確的：function 內的 `UPDATE` 需要在 lock 保護下執行完成後才釋放 lock，確保同一時刻只有持有 lock 的 backend 在執行實際 update。
>
> `pg_sleep(30 * random())` 是原始實現中的簡易 backoff。在 production 中建議改為 exponential backoff（`1ms → 2ms → 4ms → ... → max 100ms`）以降低 idle CPU 消耗和 tail latency。或者使用 `pg_advisory_lock(l)` 阻塞等待（非 try），讓 OS 層面的 process scheduling 處理排隊——但這會讓等待中的 backend 依然佔用 connection slot。

### 實驗三：Advisory Lock 排隊（8000 backend，但最多 10 個同時執行）

```sql
\setrandom id 1 5000000
\setrandom l 1 10                        -- 模擬 10 個 lock slot
SELECT upd(:l, :id);
```

同樣用 80 concurrent × 100 次載入 8000 backend：

```bash
#!/bin/bash
for ((i=0; i<100; i++))
do
  sleep 1
  pgbench -M simple -n -r -f ./t.sql -c 80 -j 80 -T 100000 -U postgres &
done
```

```sql
-- 第一次取樣
SELECT now(), n_tup_upd + n_tup_hot_upd FROM pg_stat_all_tables
WHERE relname = 'test_8000';
-- now: 2015-10-08 19:06:37.951332+08
-- ?column?: 221045069

-- 第二次取樣
SELECT now(), n_tup_upd + n_tup_hot_upd FROM pg_stat_all_tables
WHERE relname = 'test_8000';
-- now: 2015-10-08 19:07:46.46325+08
-- ?column?: 222879057
```

```sql
SELECT timestamptz '2015-10-08 19:07:46.46325+08'
     - timestamptz '2015-10-08 19:06:37.951332+08';
-- 時間差: 00:01:08.511918

SELECT 222879057 - 221045069;
-- update 次數: 1,833,988

SELECT 1833988 / 68.5;
-- TPS: 26,773.55
```

**結果：Advisory Lock 排隊 → TPS ≈ 26,774，相比不排隊（9,124）提升約 2.9 倍。**

系統負載觀察（`top`）：

```
top - 19:09:37 up 119 days,  3:59,  2 users,  load average: 0.96, 0.98, 1.01
Tasks: 8872 total,   5 running, 8866 sleeping,   1 stopped,   0 zombie
Cpu(s):  5.3%us,  0.8%sy,  0.0%ni, 93.9%id,  0.0%wa
Mem:  132124976k total, 118066688k used, 14058288k free
```

> 補充（Senior Dev）：`top` 數據揭示了一個關鍵細節——即使 8000 backend 存在，**CPU idle 仍高達 93.9%**。這說明在 advisory lock 排隊機制下，大部分 backend 處於 `pg_sleep()` 狀態（CPU 不參與競爭），只有拿到 lock 的少數 backend 在執行 UPDATE。Advisory lock 本身是 O(1) hash table lookup（`LWLock` 保護），獲取成本極低，不會像 row lock 競爭那樣產生大量 spinlock / deadlock detection 開銷。

### 限制分析

這個實現並非完美——依舊有 **8000 個 backend process**（不是獨立 process pool）。德哥建議將排隊思路整合進 PostgreSQL kernel（類似 Oracle shared server），優化前仍推薦使用 pgbouncer 連接池。

> 補充（Senior Dev）：Advisory lock 排隊 vs 現代解決方案的全面對比：
>
> | 方案 | 優點 | 缺點 | 適用場景 |
> |------|------|------|----------|
> | Advisory Lock 排隊 | 應用層可控、不需外部組件、實現簡單 | 仍佔用 8000 backend process（memory）、排隊邏輯在 application / function 內 | 快速 PoC、舊版本 PG、單表熱點 |
> | pgbouncer transaction pooling | 成熟穩定、極低 overhead、C 實現 | 不支援 prepared statement / session state 跨 transaction 保留 | 通用 OLTP |
> | pgbouncer + `max_connections` 限制 | 最穩健方案 | 需部署額外組件 | 任何 production |
> | PG 14+ built-in connection management | 無需外部組件 | `idle_session_timeout` 較粗糙，不如 pgbouncer 精細 | 簡單場景 |
> | Oracle shared server | Kernel 層面排隊，徹底解決 process overhead | PostgreSQL 無內建等價物 | — |
> | `SKIP LOCKED`（PG 9.5+） | 天然支援工作佇列模式，不需要 advisory lock | 僅適用於 queue-like table design | Job queue、Task dispatcher |
>
> **現代最佳實踐**：pgbouncer transaction pooling + `max_client_conn = 10000` / `default_pool_size = 200` + PG 端 `max_connections = 200`。這能把 10,000 個 client 連線收斂到 200 個真正的 PG backend，從根源解決 process-per-connection 的可擴展性問題。
>
> 關於秒殺（flash sale）場景：德哥另一篇文章詳述了用 advisory lock 做到 **19 萬 TPS 的單條記錄更新**（1ms RT）。本質上是在 Hot-Item 上限制 concurrent degree，避免大量 backend 同時競爭同一 row lock 造成 CPU 空轉。PG 9.5+ 的 `SKIP LOCKED` 可作為替代方案（取出未被 lock 的 row 處理，不需要 application-level lock）。
>
> `random()` 在 PostgreSQL 中的 cost：`pg_sleep(30 * random())` 中的 `random()` 每次呼叫會獲取 `PG_RANDOM_STATE` 的 spinlock（在 high concurrent 下可能成為新的瓶頸點）。如果將 random wait 改為 exponential backoff（用 `clock_timestamp()` 取 microsecond 尾數），可避免 `random()` 的 spinlock contention。

---

## 參考

1. [PostgreSQL 秒殺場景優化](https://github.com/digoal/blog/blob/master/201509/20150914_01.md)
