# PostgreSQL Extensions 深入探討

> **閱讀順序：由淺入深，逐步構建 PostgreSQL 生產級技術棧**

本書按 PostgreSQL Extension 的安裝方式與應用層次編排兩大分類、共十三個 Extension：

**# 一、Non-Contrib Extensions（需額外安裝）** — 6 個，需透過 apt install、源碼編譯或第三方工具安裝：

- **pg_partman**：原生宣告式分區自動化管理，自動分區創建、retention policy 定時清理、Background Worker 驅動 partition lifecycle。
- **PgBouncer**：連接池，Transaction/Session/Statement Pooling，生產級 HAProxy 拓撲。
- **pg_repack**：四階段在線表重組，vs VACUUM FULL / pg_squeeze 三方案對比。
- **pg_cron**：PG 內建排程作業，定時 VACUUM、分區維護、物化視圖刷新。
- **pg_stat_kcache**：查詢 CPU 與實體 IO 統計，getrusage() 真實磁盤讀寫分析。
- **hypopg**：假設性索引分析，零成本試錯 EXPLAIN 計劃。

**# 二、Contrib Extensions（PG 內建，CREATE EXTENSION 即可）** — 7 個，無需額外下載：

- **pg_stat_statements**：查詢歸一化與聚合統計，Top-N 慢查詢、緩存命中率、WAL/JIT 分析。
- **auto_explain**：自動記錄執行計劃，log_triggers / log_nested_statements（PG 16）。
- **pgcrypto**：加解密完整指南，digest/hmac 校驗、crypt+gen_salt(bf) 密碼儲存、PGP 對稱/公鑰加密。
- **pg_trgm**：三元組模糊文本搜索，GIN/GiST 加速 LIKE、similarity() 相似度排序。
- **pg_prewarm**：緩存預熱，autoprewarm BGW 自動恢復，重啟冷啟動優化。
- **pg_buffercache**：緩存內容即時診斷，usagecount 時鐘演算法、pg_buffercache_summary()（PG 17）。
- **btree_gin / btree_gist**：GIN 多欄位複合索引擴展、GiST EXCLUSION CONSTRAINT。

> 更新於 2026-06-06，全面升級至 PG 16+ 生態，兩大分類共 13 個 Extension。所有範例使用 PG 16 語法與路徑。

---

# 一、Non-Contrib Extensions（需額外安裝）

> 以下 Extension 需透過 pt install、源碼編譯或第三方工具安裝，非 PG 內建 contrib。適用於需要額外功能的生產環境。

---

## 1. pg_partman — 原生分區自動化管理

### I. 解決什麼問題

#### a. 場景與痛點

電商訂單表每天寫入數百萬行，按月分區。每月要手寫 12 條 `CREATE TABLE ... PARTITION OF` DDL，哪個月忘記建—INSERT 直接失敗報錯。90 天前的舊分區沒人記得刪，磁碟吃緊才發現堆了三年資料。

pg_partman 一句話：`create_parent()` 一行配置，自動預建未來分區 + 自動清理過期分區，Background Worker 定時維護，不需要人記。

#### b. 對比圖

```mermaid
flowchart LR
    subgraph Before["❌ 手動維護"]
        B1["每月寫 12 條 DDL"] --> B2["忘了 → INSERT 報錯"]
        B3["沒人清舊分區"] --> B4["磁碟爆滿才發現"]
    end
    subgraph After["✅ pg_partman"]
        A1["create_parent() 一行"] --> A2["自動預建未來 4 個月"]
        A3["retention = '90 days'"] --> A4["自動 DROP 過期分區"]
    end
    style Before fill:#f8d7da,stroke:#dc3545
    style After fill:#d4edda,stroke:#28a745
```

### II. 核心 SQL

#### a. 一行建立分區父表

```sql
SELECT partman.create_parent(
    'public.orders',          -- 父表名
    'created_at',              -- 分區鍵（時間欄位）
    'native',                  -- 使用 PG 原生宣告式分區
    '1 month',                 -- 每月一個分區
    p_premake := 4,            -- 預建未來 4 個分區
    p_retention := '90 days',  -- 超過 90 天的舊分區自動清理
    p_retention_keep_table := false  -- false = DROP，true = DETACH 保留表
);
```

參數白話解釋：
- `p_premake`：始終預留未來 N 個空分區。例如每月分區設 4 = 即使忘記維護，接下來 4 個月 INSERT 都不會失敗
- `p_retention`：超過此時間的舊分區自動清理。`'90 days'` 表示保留最近 90 天
- `p_retention_keep_table`：`false` 直接 DROP 回收空間；`true` 只 DETACH 不刪，方便歸檔後手動處理

#### b. 查看當前分區狀態

```sql
-- 列出某父表的所有子分區
SELECT partman.show_partitions('public.orders');

-- 查看配置：分區間隔、預建數量、保留天數
SELECT parent_table, partition_interval, premake, retention, retention_keep_table
FROM partman.part_config WHERE parent_table = 'public.orders';

-- 查看每個子分區的狀態（is_managed = false 表示脫離管理，需要關注）
SELECT sub_parent, sub_partition, is_managed, last_analyzed
FROM partman.part_config_sub
WHERE sub_parent = 'public.orders'
ORDER BY sub_partition;
```

#### c. 手動觸發維護

```sql
-- 手動執行一次維護（通常由 BGW 自動跑，此為緊急補救）
SELECT partman.run_maintenance();

-- 只對特定表執行
SELECT partman.run_maintenance('public.orders');
```

### III. 分區查詢效能驗證

建了分區不等於查詢會快——必須確認 Partition Pruning 生效。

```sql
-- 查詢單一天的資料：應只掃描 1 個子分區
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2026-01-15' AND created_at < '2026-01-16';
-- 預期：Subplans Removed: N（其他分區被裁剪）
--      -> Seq Scan on orders_p2026_01_15

-- 查詢不包含分區鍵：必須掃描所有分區（分區無效場景）
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- 預期：所有子分區都出現在 Append 中，Subplans Removed: 0
```

### IV. App Dev 視角

配合 `pg_cron` 雙保險，確保分區維護絕不遺漏：

```sql
-- pg_cron 每小時觸發一次分區維護（備援 BGW）
SELECT cron.schedule('partman-maintenance', '0 * * * *',
    'SELECT partman.run_maintenance()');
```

C# Npgsql 連線字串加入 `Application Name`，方便在 `pg_stat_activity` 中識別哪些連線來自哪個服務：

```csharp
var connStr = "Host=127.0.0.1;Port=5432;Database=mydb;Username=app;" +
              "Password=xxx;Application Name=order_service;";
```

分區鍵選擇的黃金法則：選出的欄位必須是幾乎所有查詢 WHERE 條件中的常客。如果只有 30% 查詢會用到 `created_at`，分區表反而因 Planning Time 增加而整體變慢。

---

## 2. PgBouncer — 連接池

### I. 解決什麼問題

#### a. 場景與痛點

100 個 .NET 微服務實例，每實例 Npgsql 連接池開 10 個連接 = 1000 個連接到達 PostgreSQL。PG 每個 backend process 約佔 5–10 MB 記憶體——1000 個 idle 連接白白浪費 10 GB。更致命的是 `max_connections` 預設僅 100，超過後新連接直接被拒。

PgBouncer 一句話：坐落在應用與 PG 之間的輕量連接池，將 1000 個應用連接收斂為 20–50 個 PG 連接。連接只在 transaction 期間持有，事務結束立即歸還給下一個請求。

#### b. 對比圖

```mermaid
flowchart LR
    subgraph Without["❌ 無 PgBouncer"]
        A1["App ×100"] -->|"1000 connections"| PG1["PostgreSQL<br/>記憶體爆滿<br/>連接拒絕"]
    end
    subgraph With["✅ 有 PgBouncer"]
        A2["App ×100"] -->|"1000 connections"| PB["PgBouncer<br/>Transaction Pooling"] -->|"20 connections"| PG2["PostgreSQL<br/>記憶體精簡"]
    end
    style Without fill:#f8d7da,stroke:#dc3545
    style With fill:#d4edda,stroke:#28a745
```

### II. Transaction Pooling 概念

連接只在事務期間佔用 PG backend：client 執行 `BEGIN`（或 implicit transaction 開始）時才分配一個 PG 連接，`COMMIT` / `ROLLBACK` 後立即釋放回池中，供其他 client 複用。95% 的 Web App / API 服務選擇此模式。

不支援的功能（transaction 之間狀態被 `DISCARD ALL` 清除）：`LISTEN`/`NOTIFY`、跨 transaction 的 `SET`、`PREPARE`、`WITH HOLD CURSOR`、Advisory Lock。若用到這些，需為該 database entry 單獨設 `pool_mode=session`。

### III. 生產監控 SQL

#### a. SHOW POOLS — 看連接池使用率

```sql
SHOW POOLS;
-- 關鍵欄位：
-- cl_active  ：正在執行查詢的 client 數，應 < pool_size
-- cl_waiting ：等待 server 分配的 client 數，> 0 表示瓶頸
-- sv_active  ：正在執行查詢的 server 數
-- sv_idle    ：空閒的 server 數，應 > 0（有餘量）
```

`cl_waiting > 0` 時立即檢查：是否有慢查詢卡住 server？是否需要加大 `default_pool_size`？

#### b. SHOW CLIENTS — 查連接洩漏

```sql
SHOW CLIENTS;
-- 關注 state='active' 且 query_start 超過 5 分鐘的 client
-- 可能是 transaction 忘了 COMMIT，導致 server 連接一直被佔用
```

#### c. PgBouncer 連接字串

應用層連 PgBouncer 而非直連 PG，埠號為 `6432`：

```csharp
// C# Npgsql — 注意 Port=6432
var connStr = "Host=127.0.0.1;Port=6432;Database=mydb;Username=app;" +
              "Password=xxx;Application Name=order_service;";
using var conn = new NpgsqlConnection(connStr);
```

PG 側最後防線：防止 `BEGIN` 後忘記 `COMMIT` 的連接永久佔用 server：

```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();
```

### IV. App Dev 視角

兩層池的乘法效應：如果應用層 Npgsql 連接池設 20，有 5 個 App 實例 = 100 個連接到 PgBouncer。PgBouncer 的 `default_pool_size=20` 將這 100 收斂為 20 個 PG 連接——兩層池各司其職：應用層池控制單實例併發上限，PgBouncer 做跨服務的彙聚復用。

```csharp
// Npgsql 連接池設定建議
var builder = new NpgsqlConnectionStringBuilder
{
    Host = "127.0.0.1",
    Port = 6432,          // PgBouncer 埠
    Database = "mydb",
    Username = "app",
    Password = "xxx",
    ApplicationName = "order_service",
    MaxPoolSize = 20,     // 不大於 PgBouncer pool_size
    ConnectionIdleLifetime = 300,
    ConnectionPruningInterval = 10
};
```

---

## 3. pg_repack — 在線表重組

### I. 解決什麼問題

#### a. 場景與痛點

`orders` 表每天大量 UPDATE（狀態欄位變更），MVCC 不停產生死元組（dead tuple）。`autovacuum` 雖然回收空間，但回收的空間不會歸還給作業系統——實際有效資料 10 GB，表佔用磁碟已膨脹到 30 GB。Seq Scan 得遍歷空洞頁面，buffer cache 命中率持續下降。

`VACUUM FULL` 能歸還空間，但需 `AccessExclusiveLock`——全程鎖表，生產環境等於停機。

pg_repack 一句話：建立新表 → 複製資料 → 追蹤增量 → 毫秒級原子交換 filenode，過程不阻塞讀寫。

#### b. VACUUM FULL vs pg_repack

```mermaid
flowchart LR
    subgraph VF["❌ VACUUM FULL"]
        V1["請求 AccessExclusiveLock"] --> V2["全程鎖表<br/>分鐘到小時"]
        V2 --> V3["所有 SELECT/INSERT<br/>被阻塞"]
    end
    subgraph PR["✅ pg_repack"]
        P1["階段 1-3：AccessShareLock<br/>不阻塞讀寫"] --> P2["階段 4：AccessExclusiveLock<br/>僅數毫秒交換 filenode"]
        P2 --> P3["業務無感"]
    end
    style VF fill:#f8d7da,stroke:#dc3545
    style PR fill:#d4edda,stroke:#28a745
```

### II. 生產操作

#### a. 執行 pg_repack

```bash
# 基本：重組單一表
pg_repack -d mydb -t public.orders

# 帶排序：按 created_at 排序儲存（CLUSTER 效果，範圍查詢更快）
pg_repack -d mydb -t public.orders --order-by=created_at

# 僅重組索引：表無膨脹但索引碎片化
pg_repack -d mydb -t public.orders --only-indexes

# 重組特定索引
pg_repack -d mydb -i idx_orders_status
```

#### b. 查看表大小 before/after

```sql
-- 執行前記錄
SELECT pg_size_pretty(pg_total_relation_size('public.orders')) AS size_before;
-- 結果：30 GB

-- 執行 pg_repack 後再查
SELECT pg_size_pretty(pg_total_relation_size('public.orders')) AS size_after;
-- 結果：11 GB（有效資料 10 GB + 少量餘量）
```

#### c. 監控 repack 進度

```sql
SELECT pid, state, wait_event_type, wait_event,
       query, now() - query_start AS duration
FROM pg_stat_activity
WHERE query LIKE '%repack%' AND state != 'idle';
```

### III. 注意事項

| 限制 | 說明 |
|------|------|
| 需要主鍵或 UNIQUE NOT NULL | 用於追蹤增量變更，無主鍵的表無法 repack |
| 需要約 2 倍磁碟空間 | 期間同時存在新舊兩張表 |
| WAL 產生量極大 | 等於全表重寫，streaming replica 可能延遲 |
| 不支援分區父表 | 需對每個子分區單獨執行 |

### IV. App Dev 視角

配合 `pg_cron` 定時執行，離峰時段每月或每季一次：

```sql
-- 每週日凌晨 3 點重組 orders 表（透過外部 bash 腳本包裝）
-- pg_cron 無法直接呼叫外部命令，需用 Shell + crontab 組合

-- 替代方案：監控表膨脹率，超標時觸發告警手動執行
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size,
       n_dead_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 1) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
-- dead_ratio > 20% 建議排入 repack
```

---

## 4. pg_cron — PG 內建排程作業

### I. 解決什麼問題

#### a. 場景與痛點

每天凌晨需清理 30 天前的日誌、每週凌晨 VACUUM 熱表、每小時觸發 pg_partman 維護——這些例行維護若依賴外部 crontab，存在三個結構性風險：

- **網路中斷**：`crontab → psql` 經過 TCP，網路抖動直接導致失敗
- **事務不確定**：`psql -c` 執行後連接中斷，無法確認 SQL 是否完成
- **多一個運維組件**：須管理 crontab 語法、權限、日誌、監控

pg_cron 一句話：在 PG 內部的 Background Worker 中執行定時 SQL，無外部依賴、事務由 PG 自身保證、執行日誌記錄在 `cron.job_run_details`。

### II. 核心 SQL 操作

#### a. cron.schedule() 各粒度範例

```sql
-- 每分鐘：清理過期 session
SELECT cron.schedule('clean-sessions', '* * * * *',
    'DELETE FROM sessions WHERE expires_at < now()');

-- 每天凌晨 3 點：VACUUM 熱表
SELECT cron.schedule('vacuum-orders', '0 3 * * *',
    'VACUUM ANALYZE orders');

-- 每小時：分區維護
SELECT cron.schedule('partman-hourly', '0 * * * *',
    'SELECT partman.run_maintenance()');

-- 每週日凌晨 1 點：重組 Materialized View
SELECT cron.schedule('refresh-mv', '0 1 * * 0',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_report');

-- 每月 1 號凌晨 4 點：歸檔舊資料
SELECT cron.schedule('archive-monthly', '0 4 1 * *',
    $$INSERT INTO audit_archive SELECT * FROM audit
      WHERE created_at < now() - '90 days'::interval$$);
```

Cron 表達式格式為 `分 時 日 月 週`（支援 6 段式秒級精度 `秒 分 時 日 月 週`）。

#### b. 查執行日誌

```sql
-- 最近 20 條執行記錄
SELECT jobid, status, return_message, start_time, end_time,
       EXTRACT(epoch FROM (end_time - start_time)) AS duration_sec
FROM cron.job_run_details
ORDER BY start_time DESC LIMIT 20;
```

`status = 'succeeded'` 正常，`status = 'failed'` 檢查 `return_message` 的錯誤內容。

#### c. 查失敗 job

```sql
-- 過去 24 小時內失敗的任務
SELECT jobid, status, return_message, start_time
FROM cron.job_run_details
WHERE status = 'failed'
  AND start_time > now() - '24 hours'::interval;

-- 過去 7 天各任務成功率
SELECT jobid,
       COUNT(*) FILTER (WHERE status = 'succeeded') AS succeeded,
       COUNT(*) FILTER (WHERE status = 'failed') AS failed,
       ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'succeeded') / COUNT(*), 1) AS success_pct
FROM cron.job_run_details
WHERE start_time > now() - '7 days'::interval
GROUP BY jobid ORDER BY success_pct;
```

#### d. 管理任務

```sql
-- 查看所有任務
SELECT jobid, jobname, schedule, command, active FROM cron.job ORDER BY jobid;

-- 修改排程
SELECT cron.alter_job(1, '0 4 * * *', NULL, NULL, NULL, NULL);

-- 停用不刪除（維護窗口用）
UPDATE cron.job SET active = false;
-- 恢復
UPDATE cron.job SET active = true WHERE jobname LIKE 'vacuum-%';

-- 刪除
SELECT cron.unschedule(1);
```

### III. 為什麼不用外部 cron

```mermaid
flowchart LR
    subgraph Ext["❌ 外部 crontab"]
        E1["crontab → psql"] --> E2["網路中斷 → 任務失敗"]
        E3["連接斷開"] --> E4["不確定 SQL 是否完成"]
    end
    subgraph Int["✅ pg_cron"]
        I1["BG Worker 直連 PG"] --> I2["零網路故障風險"]
        I3["PG 內部執行"] --> I4["事務安全 + 日誌完整"]
    end
    style Ext fill:#f8d7da,stroke:#dc3545
    style Int fill:#d4edda,stroke:#28a745
```

### IV. App Dev 視角

確保 job 冪等性（Idempotency）——重複執行不產生副作用：

```sql
-- ❌ 非冪等：重複執行會重複插入
INSERT INTO daily_summary (report_date, total_sales)
SELECT CURRENT_DATE, SUM(amount) FROM orders WHERE created_at::date = CURRENT_DATE;

-- ✅ 冪等：INSERT ON CONFLICT 安全重跑
INSERT INTO daily_summary (report_date, total_sales)
SELECT CURRENT_DATE, SUM(amount) FROM orders WHERE created_at::date = CURRENT_DATE
ON CONFLICT (report_date) DO UPDATE SET total_sales = EXCLUDED.total_sales;
```

還原備份後立即停用所有 job：

```sql
UPDATE cron.job SET active = false;
-- 確認環境無誤後再逐批啟用
UPDATE cron.job SET active = true WHERE jobname = 'vacuum-orders';
```

---

## 5. pg_stat_kcache — 查詢 CPU 與實體 IO 統計

### I. 解決什麼問題

#### a. 場景與痛點

`pg_stat_statements` 顯示某查詢 `shared_blks_read = 10,000`，看起來磁碟讀很多——但 `iostat` 顯示磁碟 IO 很低。原因：`shared_blks_read` 統計的是 PG **向 OS 請求**的 page 數，如果 OS 的 page cache 已快取了該頁，不會發生真實磁碟讀。buffer 層統計無法反映真實物理 IO。

pg_stat_kcache 一句話：透過 `getrusage()` 系統呼叫直接讀取 kernel 累計的 block IO 統計，補足 `pg_stat_statements` 看不到的 OS 層實體磁碟讀寫與 CPU 耗時。

#### b. 三層對比

```mermaid
flowchart LR
    Q["SQL 查詢"] --> B["pg_stat_statements<br/>Buffer 層（邏輯 IO）<br/>shared_blks_hit / read"]
    Q --> C["pg_stat_kcache<br/>OS 層（實體 IO + CPU）<br/>getrusage() 統計"]
    Q --> D["iostat / iotop<br/>硬體層（磁碟延遲）"]
    B --> B2["shared_blks_read 高<br/>≠ 磁碟 IO 高<br/>（OS Page Cache 攔截）"]
    C --> C2["reads / reads_bytes<br/>= 真實磁碟讀取量"]
    style C fill:#fff3cd,stroke:#ffc107
```

### II. 生產 SQL 操作

#### a. 真實物理讀 Top-10

```sql
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    kc.reads_bytes AS physical_reads_kb,
    s.shared_blks_read * current_setting('block_size')::bigint / 1024 AS logical_reads_kb,
    ROUND(kc.reads_bytes::numeric /
        NULLIF(s.shared_blks_read * current_setting('block_size')::bigint / 1024, 0), 2) AS cache_miss_ratio
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE kc.reads_bytes > 0
ORDER BY kc.reads_bytes DESC LIMIT 10;
```

`cache_miss_ratio` 解讀：
- **≈ 1.0**：每次 buffer 讀請求都觸發了實體磁碟 IO，buffer cache 完全沒命中，磁碟瓶頸
- **≈ 0.1**：buffer 讀請求中只有 10% 穿透到磁碟，OS page cache 有效緩解
- **> 1.0**：OS 層出現了讀放大（如 read-ahead 預讀）

#### b. 寫放大檢測

```sql
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    s.wal_bytes / 1024 AS wal_kb,
    kc.writes_bytes AS os_write_kb,
    ROUND(kc.writes_bytes::numeric /
        NULLIF(s.wal_bytes / 1024, 0), 2) AS write_amplification,
    kc.fsyncs
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE s.wal_bytes > 0
ORDER BY s.wal_bytes DESC LIMIT 10;
```

`write_amplification` 解讀：
- **≈ 1.0**：正常，WAL 量 ≈ OS 寫入量
- **> 2.0**：OS 寫入放大，可能是頻繁 hint-bit 更新、full page write 或 temp file 溢出
- **< 0.5**：WAL 量大於 OS 寫入，async write（background writer）尚未刷盤

#### c. CPU 耗時 Top-10

```sql
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    ROUND(kc.user_time::numeric, 2) AS user_cpu_sec,
    ROUND(kc.system_time::numeric, 2) AS sys_cpu_sec,
    ROUND((kc.user_time + kc.system_time)::numeric / NULLIF(s.calls, 0) * 1000, 2) AS avg_cpu_ms_per_call
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
ORDER BY kc.user_time DESC LIMIT 10;
```

解讀：
- `user_time` 遠大於 `system_time`：瓶頸在計算（HashAgg、Sort、Nested Loop）
- `system_time` 佔比高：kernel 層操作多（IO 排程、fsync、context switch）

### III. 快速診斷決策

```mermaid
flowchart TD
    A["發現查詢變慢"] --> B{"kcache.reads_bytes<br/>很高？"}
    B -->|"是"| C["磁碟 IO 瓶頸<br/>→ 加大 shared_buffers<br/>→ 檢查 working set > RAM<br/>→ 考慮 SSD"]
    B -->|"否"| D{"kcache.user_time<br/>很高？"}
    D -->|"是"| E["CPU 瓶頸<br/>→ 查 EXPLAIN plan<br/>→ 有無 Nested Loop 爆炸<br/>→ 考慮物化視圖"]
    D -->|"否"| F{"kcache.writes + fsyncs<br/>很高？"}
    F -->|"是"| G["寫入瓶頸<br/>→ 檢查 checkpoint 頻率<br/>→ 增大 max_wal_size<br/>→ 考慮批次寫入"]
    F -->|"否"| H["其他：Lock / LWLock / 網路"]
    style C fill:#fff3cd,stroke:#ffc107
    style E fill:#ffd3b6,stroke:#fd7e14
    style G fill:#f8d7da,stroke:#dc3545
```

### IV. App Dev 視角

驗證新索引是否真正減少了實體 IO（不只改善 buffer 層統計）：

```sql
-- Step 1：建索引前記錄基準值
SELECT queryid, reads_bytes, user_time FROM pg_stat_kcache
WHERE queryid = (SELECT queryid FROM pg_stat_statements WHERE query LIKE '%target_table%' LIMIT 1);

-- Step 2：建索引
CREATE INDEX CONCURRENTLY idx_xxx ON target_table(col);

-- Step 3：重置統計，等待業務運行一段時間
SELECT pg_stat_statements_reset();
SELECT pg_stat_kcache_reset();

-- Step 4：對比建索引後的 reads_bytes
-- 若 reads_bytes 未明顯下降，索引未被有效利用
```

注意：`pg_stat_kcache` 對每條查詢有約 8–10% 的 CPU 開銷（來自 `getrusage()` 系統呼叫），生產環境建議設定 `pg_stat_kcache.track = 'top'` 以降低影響。

---

## 6. hypopg — 假設性索引分析

### I. 解決什麼問題

#### a. 場景與痛點

想給 `users(email)` 建索引，但不確定 Planner 會不會用——貿然在生產環境 `CREATE INDEX CONCURRENTLY`，結果 Planner 不走、空間浪費（表大小的 10–30%）、且每次 INSERT/UPDATE/DELETE 都要額外維護索引。更糟的是有時建了才發現在 500 GB 的表上執行需要數小時。

hypopg 一句話：在純記憶體中「假裝」建立索引，用 `EXPLAIN` 觀察 Planner 會否選用、成本降低多少——零磁碟、零 WAL、session 結束自動銷毀。

#### b. 傳統 vs hypopg 流程對比

```mermaid
flowchart LR
    subgraph Old["❌ 傳統試錯"]
        O1["猜索引"] --> O2["CREATE INDEX<br/>花時間/空間"] --> O3["EXPLAIN"] --> O4{"Planner 用？"}
        O4 -->|"否"| O5["DROP INDEX<br/>浪費"]
        O4 -->|"是"| O6["保留"]
    end
    subgraph New["✅ hypopg"]
        N1["猜索引"] --> N2["hypopg_create_index<br/>瞬間"] --> N3["EXPLAIN"] --> N4{"Planner 用？"}
        N4 -->|"否"| N5["hypopg_reset<br/>零成本"]
        N4 -->|"是"| N6["CREATE INDEX<br/>一次成功"]
    end
    style Old fill:#f8d7da,stroke:#dc3545
    style New fill:#d4edda,stroke:#28a745
```

### II. 生產 SQL 操作

#### a. 建立假設索引 + EXPLAIN 驗證

```sql
-- 建立假設索引（語法與 CREATE INDEX 完全相同）
SELECT hypopg_create_index('CREATE INDEX ON users(email)');

-- 觀察 Planner 是否選用
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
-- 若輸出 Index Scan using <hypopg_btree_users_email> → Planner 會用這個索引
-- 若輸出 Seq Scan → Planner 不用，此索引無意義
```

絕不能用 `EXPLAIN ANALYZE`——索引不存在，executor 會報錯 `relation does not exist`。

#### b. 對比三種候選索引，選最優

```sql
-- 場景：SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20

-- 策略 A：單欄索引
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_a ON orders(user_id)');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 total cost: ____

-- 策略 B：複合覆蓋索引
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_b ON orders(user_id, status, created_at DESC)');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 total cost: ____

-- 策略 C：部分索引（更輕量）
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_c ON orders(user_id, created_at DESC) WHERE status = ''active''');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 total cost: ____
```

比較各策略的 `cost` 值與 Scan 類型（Index Scan / Index Only Scan / Seq Scan），選成本最低且空間最小的方案。

#### c. 查看與估算大小

```sql
-- 列出所有假設索引
SELECT * FROM hypopg_list_indexes();

-- 估算真實建立的磁碟大小
SELECT indexname, relname,
       pg_size_pretty(hypopg_relation_size(indexrelid)) AS estimated_size
FROM hypopg_list_indexes();

-- 清除所有假設索引
SELECT hypopg_reset();
```

#### d. 支援的索引類型

B-tree、Hash、BRIN、GiST、GIN、Expression Index（`lower(email)`）、Covering Index（`INCLUDE`）、Partial Index（`WHERE`）全部支援。

### III. App Dev 視角

分層策略——永遠不在 Production 使用 hypopg：

```mermaid
flowchart TB
    S1["開發環境<br/>hypopg 測試 3-5 種索引方案"] --> S2["Staging<br/>建真實索引 + EXPLAIN ANALYZE 驗證"]
    S2 -->|"效能回歸通過"| S3["Production<br/>CREATE INDEX CONCURRENTLY<br/>離峰時段部署"]
    S3 --> S4["監控 pg_stat_user_indexes<br/>確認 idx_scan > 0"]
    style S1 fill:#4a90d9,color:#fff
    style S3 fill:#5cb85c,color:#fff
```

注意：hypopg 的 planner 估算與真實索引存在 ±15% 的偏差——缺乏真實 `pg_stats` 直方圖和 B-tree 深度資訊。因此 hypopg 用來篩選方案（淘汰明顯無用的索引），最終確認仍需在 Staging 環境建真實索引並 `EXPLAIN ANALYZE`。

配合 `pg_stat_statements` 的工作流：

```sql
-- 1. 從 pg_stat_statements 找出 Top-5 慢查詢
SELECT queryid, LEFT(query, 150), calls, mean_exec_time
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;

-- 2. 對每條慢查詢用 hypopg 測試索引方案
-- 3. 選出成本最低的索引 → Staging 驗證 → Production 部署
```



---


# 二、Contrib Extensions（PG 內建，CREATE EXTENSION 即可）


> 以下 Extension 為 PostgreSQL 內建 contrib 模組，無需額外下載，直接 CREATE EXTENSION 即可啟用。



## 1. pg_stat_statements — 查詢統計

pg_stat_statements 做一件事：把每條執行過的 SQL 歸一化（常數替換為 `$N`），聚合所有呼叫的 calls、total_time、shared_blks_*、wal_bytes 等指標到一個視圖。它是 PG 可觀測性體系中的基石——回答「哪些 SQL 最慢 / 最頻繁 / IO 最重」。

### I. 場景與定位

應用上線半年，API 開始在尖峰時段 timeout。`top` 看到 CPU 85%，`pg_stat_activity` 裡幾十個 `SELECT` 都在跑，看不出誰是瓶頸。這是 pg_stat_statements 的經典登場時刻：把所有 SQL 按執行總時間排序，3 秒鐘就能鎖定頭號嫌疑犯。

核心工作流程只有兩步：

1. **歸一化 (Normalization)** — 收到 SQL 後，掃描語法樹，把 literal 常數（`123`, `'alice'`, `2026-01-01`）替換成 `$1`, `$2` ...。`WHERE id = 1` 和 `WHERE id = 999` 變成同一條 query template。
2. **聚合 (Aggregation)** — 在同一 `queryid` 下累計 `calls`、`total_exec_time`、`shared_blks_read/hit/dirtied`、`wal_bytes`、`rows` 等。

結果存在 shared memory 中的固定大小 hash table，預設 5000 個 entry。滿了就淘汰 `calls` 最少的 entry（簡單 LRU 近似）。

```mermaid
flowchart LR
    A["App 發出 SQL<br/>SELECT * FROM orders<br/>WHERE id = 123"] --> B["Parser / Analyzer<br/>產生 Query Tree"]
    B --> C["歸一化引擎<br/>常數 → $1, $2, ..."]
    C --> D["計算 queryid<br/>hash(normalized query)"]
    D --> E{"queryid 已存在？"}
    E -->|"是"| F["疊加至現有 entry<br/>calls++<br/>total_time += 本次時間<br/>shared_blks_read += 本次 IO"]
    E -->|"否"| G["建立新 entry<br/>若 hash table 滿<br/>淘汰 calls 最少的"]
    F --> H[("pg_stat_statements 視圖<br/>SELECT 即可讀取")]
    G --> H
```

### II. 生產查詢

以下查詢全部可以直接複製到 psql 執行。假設你只有唯讀權限，不需 superuser。

#### a. Top-10 總時間最長的 SQL

```sql
-- 找出執行總時間最長的 10 條 SQL —— 最拖垮整體效能的查詢
SELECT queryid,
       left(query, 120) AS query_snippet,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       round(min_exec_time::numeric, 2) AS min_ms,
       round(max_exec_time::numeric, 2) AS max_ms,
       round((total_exec_time / sum(total_exec_time) OVER ()) * 100, 1) AS pct
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

每欄解讀：
- `pct`：這條 SQL 佔了**所有已歸一化查詢總時間**的百分比。生產環境常見 3~5 條 SQL 合計吃掉 70~85% 時間。
- `stddev_ms`：標準差。若 `avg_ms = 500ms` 但 `stddev_ms = 1500ms`，表示這條 SQL 有時跑 5ms（cache 命中）、有時跑 5000ms（cache miss），效能不穩定。
- `min_ms / max_ms`：最極端的兩次執行時間。`max_ms` 遠大於 `avg_ms` 表示存在間歇性效能尖峰（可能被 lock 阻塞或剛好在 VACUUM）。
- `calls * avg_ms ≈ total_ms`，交叉驗證資料一致性。

#### b. Top-10 被呼叫最頻繁的 SQL

```sql
-- 找出執行次數最多的 10 條 SQL —— 高頻率查詢的累積成本不可忽視
SELECT left(query, 120) AS query_snippet,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 4) AS avg_ms,
       rows,
       round((rows::numeric / NULLIF(calls, 0)), 1) AS rows_per_call
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

每欄解讀：
- `calls`：若 > 10 萬次但 `avg_ms` < 0.5ms，這是良性高頻查詢（pool 中的簡單 lookup）。不需要最佳化 SQL 本身，但要確保 connection pool 夠用。
- `rows_per_call`：若 > 10000，代表每次回傳大量資料——網路序列化成本高，且應用層記憶體壓力大。考慮加 LIMIT 或分頁。
- `calls` 高 + `avg_ms` 小 + `rows_per_call = 1` —— 經典 N+1 query 信號（ORM 在 loop 中逐筆查），應用層需改用 batch。

#### c. Top-10 磁碟讀取最多的 SQL（cache 命中率低）

```sql
-- 找出 shared_blks_read 最多的 SQL —— IO 瓶頸候選
SELECT left(query, 120) AS query_snippet,
       calls,
       shared_blks_read,
       shared_blks_hit,
       round((shared_blks_hit::numeric /
              NULLIF(shared_blks_hit + shared_blks_read, 0) * 100), 1) AS cache_hit_pct,
       round(shared_blks_read::numeric / NULLIF(calls, 0), 1) AS blks_read_per_call
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_hit > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

每欄解讀：
- `cache_hit_pct`：< 95% 代表資料不在 shared_buffers，每次都要讀磁碟。100% 不代表效能好——可能是 Seq Scan 全從 cache 讀，但掃了不必要的大量 block。
- `blks_read_per_call`：單次呼叫的磁碟讀取量。若 > 1000 blocks (8MB)，這條 SQL 的 working set 大於可用緩存。可能原因是缺索引導致 Seq Scan，或表真的超大裝不下 shared_buffers。
- `shared_blks_read` 大 `shared_blks_hit` 小：這條 SQL 的資料從來不進 cache（或被頻繁淘汰）。檢查是否每次都在掃不同的大範圍資料。

#### d. Top-10 WAL 寫入最多的 SQL（寫密集）

```sql
-- 找出產生 WAL 最多的 SQL —— 寫入密集型瓶頸
SELECT left(query, 120) AS query_snippet,
       calls,
       wal_bytes,
       pg_size_pretty(wal_bytes) AS wal_size,
       pg_size_pretty(wal_bytes / NULLIF(calls, 0)) AS wal_per_call,
       rows,
       round((wal_bytes::numeric / NULLIF(rows, 0)), 1) AS wal_bytes_per_row
FROM pg_stat_statements
WHERE wal_bytes > 0
ORDER BY wal_bytes DESC
LIMIT 10;
```

每欄解讀：
- `wal_per_call`：單次呼叫的 WAL 產生量。若 > 100MB，可能是 `UPDATE 全表`、`DELETE 大量行`、或大批 `INSERT` 沒分批。
- `wal_bytes_per_row`：每行的平均 WAL 開銷。正常的 UPDATE 約 100~300 bytes/row。若 > 1KB，檢查是否有大欄位（text/jsonb）頻繁更新。
- 高 WAL 不只拖慢自己——它在 streaming replication 中產生滯後，在 PITR 備份中消耗更多磁碟空間。

#### e. 重置統計與管理

```sql
-- 重置所有統計（需 superuser 或 pg_stat_statements_admin 角色）
SELECT pg_stat_statements_reset();

-- 查看目前 tracked 的 query 數量
SELECT count(*) AS tracked_queries,
       pg_size_pretty(sum(calls)) AS total_calls_pretty
FROM pg_stat_statements;

-- 查看 shared memory 使用狀況
SELECT count(*) AS entries,
       (SELECT setting::int FROM pg_settings WHERE name = 'pg_stat_statements.max')
       AS max_entries,
       round(count(*) * 100.0 /
             (SELECT setting::int FROM pg_settings WHERE name = 'pg_stat_statements.max'), 1)
       AS pct_full
FROM pg_stat_statements;
-- pct_full > 80% → 考慮調高 pg_stat_statements.max（需重啟）
```

### III. App Dev 視角

C# / Npgsql 開發者有三個關鍵實務點：

**第一，連線字串設定 Application Name**。雖然 pg_stat_statements 不記錄 `application_name`，但出事時你會同時看 `pg_stat_activity` 定位連線來源。建議每個 service 使用唯一名稱：

```csharp
var connStr = "Host=10.0.1.50;Database=mydb;Application Name=order-service-v2";
```

**第二，Dapper 參數化決定歸一化品質**。以下兩行程式碼在 `pg_stat_statements` 中的行為完全不同：

```csharp
// ✅ 正確：同一 queryid，所有呼叫聚合歸一化為 SELECT ... WHERE id = $1
conn.Query<Product>("SELECT * FROM products WHERE id = @Id", new { Id = 123 });

// ❌ 錯誤：每種 id 值都是一條獨立 entry，5000 條上限很快被灌爆
conn.Query<Product>($"SELECT * FROM products WHERE id = {id}");
```

後果：5000 條 limit 被字串拼接的 SQL 塞滿後，真正需要被追蹤的核心查詢反而被擠出 hash table，pg_stat_statements 形同虛設。如果 ORM (如 EF Core) 預設使用參數化，注意是否有 raw SQL 例外。

**第三，定期 snapshot 到監控系統**。pg_stat_statements 是累計值，只看一次 snapshot 看不出趨勢。建議每 5~10 分鐘 snapshot 一次差值：

```csharp
// 定期 job (例如每 5 分鐘執行)
var snapshot = await conn.QueryAsync<QueryStatSnapshot>(@"
    SELECT queryid, left(query, 200) AS query_text,
           calls, total_exec_time, shared_blks_read, shared_blks_hit
    FROM pg_stat_statements
    WHERE calls > 50
    ORDER BY total_exec_time DESC
    LIMIT 50");
// 將 snapshot 寫入自己的監控表或推送到 Prometheus / Grafana
await SaveToMonitoringTable(snapshot, DateTime.UtcNow);
```

**第四，記錄 backend_pid**。出問題時從 `pg_stat_activity` 找到的 PID 可以直接對應到 Npgsql connection：

```csharp
using var conn = new NpgsqlConnection(connStr);
conn.Open();
_logger.LogInformation("DB connection established, backend PID: {Pid}", conn.ProcessID);
```

### IV. 診斷決策流程

```mermaid
flowchart TD
    A["API timeout / DB CPU 高<br/>使用者回報慢"] --> B["查 total_exec_time Top-10<br/>找出佔比最高的 SQL"]
    B --> C{"1~3 條 SQL<br/>佔比 > 50%？"}
    C -->|"是"| D["鎖定瓶頸 SQL<br/>→ 記錄 queryid"]
    D --> E["→ auto_explain<br/>捕獲執行計劃"]
    C -->|"否，分散在<br/>多條 SQL"| F["查 calls Top-10"]
    F --> G{"calls > 50000<br/>avg_ms < 2ms？"}
    G -->|"是"| H["N+1 query 嫌疑<br/>ORM batch size 不足<br/>→ 檢查應用層 loop"]
    G -->|"否"| I["查 shared_blks_read Top-10"]
    I --> J{"cache_hit_pct<br/>< 95%？"}
    J -->|"是"| K["IO-bound<br/>→ 缺 index / 全表掃描<br/>→ pg_buffercache 診斷"]
    J -->|"否"| L["CPU-bound<br/>→ 複雜 JOIN / JIT<br/>→ 檢查執行計劃"]
```

---


## 2. auto_explain — 自動記錄執行計劃

auto_explain 做的事：超過指定時間閥值的 SQL，自動把 `EXPLAIN ANALYZE` 的完整輸出寫進 PG log。如果 pg_stat_statements 告訴你 **WHAT** is slow，auto_explain 告訴你 **WHY** it's slow——Seq Scan？Nested Loop？work_mem 不足寫磁碟？

### I. 與 pg_stat_statements 互補

pg_stat_statements 說「查詢 id=42，平均跑 3 秒，cache hit 40%」。但 3 秒花在哪個執行節點？是 `Seq Scan on large_table` 花了 2.9 秒？還是 `Hash Join` 的 probe 階段花了 2 秒？這些只有 EXPLAIN ANALYZE 的 actual time 能回答。auto_explain 讓你不需手動複製 SQL 去跑 EXPLAIN——慢查詢發生的當下，執行計劃已經自動寫進 log，事後 grep 即可。

```mermaid
flowchart LR
    A["pg_stat_statements<br/>WHAT：哪條 SQL 慢<br/>佔總時間 45%"] --> B["auto_explain<br/>WHY：慢在哪個節點<br/>Seq Scan 花了 2.8s"]
    B --> C["定位瓶頸根因"]
    C --> D["Rows Removed by Filter<br/>→ 缺索引 CREATE INDEX"]
    C --> E["external merge Disk<br/>→ work_mem 太小"]
    C --> F["Nested Loop<br/>rows=1000 est=10<br/>→ 統計過期 ANALYZE"]
```

### II. 觸發機制與配置

auto_explain 是 shared library（`shared_preload_libraries = 'auto_explain'`，需重啟）。載入後透過 GUC 參數控制行為。核心邏輯：每條 SQL 執行完畢後比較 exec_time 與 `log_min_duration`，符合條件就輸出 EXPLAIN。

```sql
-- postgresql.conf 中載入
-- shared_preload_libraries = 'auto_explain'

-- 核心參數：任何執行超過 200ms 的 SQL，自動記錄執行計劃
SET auto_explain.log_min_duration = '200ms';

-- 輸出實際執行統計（actual time / actual rows / loops）
-- 不開 = 只輸出 planner 估計值（cost），無實際值比對，診斷價值大減
SET auto_explain.log_analyze = on;

-- 輸出 buffer 使用（shared hit / read / dirtied / written）
-- 必開：沒有這個就無法判斷 Seq Scan 還是 Index Scan、cache 命中狀況
SET auto_explain.log_buffers = on;

-- 記錄每個節點的 timing 明細
SET auto_explain.log_timing = on;

-- 記錄 function / trigger 內執行的 SQL（PL/pgSQL 中的動態 SQL）
SET auto_explain.log_nested_statements = on;

-- 記錄 trigger function 本身的 overhead
SET auto_explain.log_triggers = on;

-- 輸出格式：text（人類可讀），json / xml / yaml（機器解析）
SET auto_explain.log_format = 'text';

-- 不超過 min_duration 的 SQL 是否也記錄（通常 off，否則 log 暴量）
SET auto_explain.log_min_duration = '200ms';
```

注意：auto_explain **不記錄被取消的查詢**（statement_timeout 觸發、Npgsql CommandTimeout cancel）。這類沒跑完的查詢只能靠 PG 原生的 `log_min_duration_statement` 或 `log_statement` 來記錄 SQL text（但無執行計劃）。

### III. 日誌輸出解讀

#### a. 範例一：缺索引

```text
2026-06-19 10:23:45 CST [postgres] LOG:  duration: 2341.567 ms  plan:
Query Text: SELECT o.id, o.customer_name, oi.product_id, oi.quantity
            FROM orders o JOIN order_items oi ON o.id = oi.order_id
            WHERE o.created_at >= '2026-01-01' AND oi.product_id = 42;
Seq Scan on order_items oi  (cost=0.00..45678.00 rows=50 width=16)
                            (actual time=12.345..2100.123 rows=52 loops=1)
  Filter: (product_id = 42)
  Rows Removed by Filter: 4987654
  Buffers: shared hit=128 read=45320
```

判讀要點：
- `Seq Scan` + `Rows Removed by Filter: 4987654` → 全表掃描 500 萬行，只為了找出 52 行符合條件的記錄。這是缺 `product_id` 索引的經典信號。
- `Buffers: read=45320` → 45,320 × 8KB ≈ 354MB 來自磁碟。建立索引後，這個數字應降為個位數。
- `actual rows=52` vs `estimated rows=50`（cost 中）→ planner 估算很準，問題不在統計，在於沒有索引可用。

#### b. 範例二：work_mem 不足

```text
Sort Method: external merge  Disk: 89032kB
  Buffers: shared hit=12034 read=0, temp read=11129 written=11129
```

判讀要點：
- `external merge Disk` → 排序溢出到磁碟。記憶體排序速度是磁碟的 10~100 倍。
- `Disk: 89032kB` → 需要 87MB 的排序空間，但 `work_mem` 預設只有 4MB。
- `temp read/written` → 臨時檔案的讀寫量，大量 temp IO 會拖垮整個 instance 的磁碟效能。
- 解法：`SET LOCAL work_mem = '128MB'` 讓這條查詢的排序在記憶體中完成。

#### c. 範例三：統計資訊過期

```text
Nested Loop  (cost=0.42..500.00 rows=10 width=32)
             (actual time=0.123..4500.789 rows=500000 loops=1)
  ->  Seq Scan on small_table  (actual time=0.012..0.123 rows=10 loops=1)
  ->  Index Scan on large_table_idx  (actual time=0.010..0.400 rows=50000 loops=10)
```

判讀要點：
- Planner 估計 `rows=10`，實際 `rows=500000`——差距 50000 倍。
- 內層 Index Scan 被執行了 `loops=10` 次，每次回傳 5 萬行（planner 原以為每次 1 行）。
- Planner 因為嚴重低估 `large_table` 的資料量而選了 Nested Loop（適合小資料集），實際卻適合 Hash Join。
- 解法：`ANALYZE large_table` 更新統計，必要時調高 `default_statistics_target`。

#### d. 範例四：JIT 編譯 overhead

```text
Planning Time: 0.500 ms
JIT:
  Functions: 8
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 2.500 ms, Inlining 150.000 ms, Optimization 300.000 ms,
          Emission 200.000 ms, Total 652.500 ms
Execution Time: 800.000 ms
```

判讀要點：
- JIT 編譯花了 652ms，但總執行才 800ms。JIT 編譯成本遠大於它節省的時間。
- 這種情況發生在短查詢觸發 JIT——PG 16+ JIT 預設只在 `cost >= 100000` 時觸發，但複雜 SQL 可能快速超過閥值。
- 解法：`SET jit = off` 或調高 `jit_above_cost`。

### IV. App Dev 視角

C# 開發者要特別關注 **CommandTimeout 與 auto_explain 的互動**：

```csharp
// auto_explain 不會記錄被 cancel 的查詢
var cmd = new NpgsqlCommand(slowSql, conn) { CommandTimeout = 30 };
try {
    await cmd.ExecuteNonQueryAsync();
} catch (PostgresException ex) when (ex.SqlState == "57014") {
    // SqlState 57014 = query_canceled
    // 這條 SQL 沒有正常完成 → auto_explain 不會輸出
    // 只能從 pg_stat_statements 看到這條 SQL 的累計時間
    // 暫時提高 CommandTimeout 讓它跑完一次，以便 auto_explain 捕獲計劃
}
```

建議的故障排查作業流程：在 log 中記錄 request ID + backend PID，出問題時直接 grep PG log：

```csharp
_logger.LogWarning("Slow query detected, requestId={ReqId}, backendPid={Pid}, sql={Sql}",
    requestId, conn.ProcessID, sql);
// 在 PG log 中搜尋：grep "\[PID\]" postgresql.log | grep "duration:"
// 即可找到同一 connection 的所有 auto_explain 輸出
```

```mermaid
flowchart TD
    A["監控告警：P99 latency 飆高"] --> B["pg_stat_statements<br/>找出 queryid 和 SQL text"]
    B --> C["搜 PG log<br/>grep SQL text 或 時間範圍"]
    C --> D{"auto_explain<br/>有輸出？"}
    D -->|"有"| E["分析執行計劃<br/>定位瓶頸節點"]
    E --> F{"瓶頸類型？"}
    F -->|"Seq Scan + Rows Removed"| G["缺索引 → CREATE INDEX"]
    F -->|"external merge Disk"| H["work_mem 太小 → 調高"]
    F -->|"est vs actual 差 1000x"| I["統計過期 → ANALYZE"]
    F -->|"JIT 佔 > 50% 時間"| J["關閉 JIT 或調高閥值"]
    D -->|"無"| K{"SQL 被 timeout？"}
    K -->|"是"| L["暫時提高 timeout<br/>讓它跑完一次以便診斷"]
    K -->|"否"| M["降低 log_min_duration<br/>例如 200ms → 50ms"]
```

---


## 3. pgcrypto — 加解密

pgcrypto 是 PG 內建的加密工具箱，涵蓋三種場景：密碼儲存（bcrypt 單向雜湊）、敏感欄位加解密（PGP 對稱可逆）、跨系統安全交換（PGP 公鑰加密）。選錯方案會直接導致安全漏洞。

### I. 三種場景與方案選擇

```mermaid
flowchart TD
    A["需要保護的資料"] --> B{"資料用途？"}
    B -->|"密碼儲存<br/>只需驗證<br/>不可解密"| C["bcrypt 單向雜湊<br/>crypt() + gen_salt('bf')<br/>⚠️ 不可逆"]
    B -->|"敏感欄位<br/>需可逆解密<br/>查詢展示"| D{"是否需要<br/>跨系統交換？"}
    B -->|"資料完整性<br/>防篡改校驗"| E["digest / hmac<br/>SHA-256 / SHA-512<br/>⚠️ 非加密，是簽章"]
    D -->|"否<br/>同一系統內"| F["PGP 對稱加密<br/>pgp_sym_encrypt()<br/>單一 key 加解密"]
    D -->|"是<br/>多系統間傳輸"| G["PGP 公鑰加密<br/>pgp_pub_encrypt()<br/>公鑰加密、私鑰解密"]
    C --> H["gen_salt('bf', iterations)<br/>iteration=8 約 0.5s<br/>=10 約 3s<br/>=12 約 10s"]
    F --> I["key 管理是核心<br/>絕不可 hardcode<br/>在 SQL 中"]
    G --> J["公鑰可公開<br/>私鑰必須保護<br/>支援 passphrase"]
```

### II. 密碼儲存（bcrypt）

使用者密碼的鐵律：**不可解密，只可比對**。pgcrypto 的 `crypt()` 使用 bcrypt（Blowfish 加密演算法），內建 salt 和迭代控制。

```sql
-- 註冊帳號：儲存 bcrypt 雜湊
INSERT INTO users (username, password_hash)
VALUES ('alice', crypt('my-secret-pw', gen_salt('bf', 10)));
-- 'bf' = Blowfish bcrypt
-- 10 = 迭代次數（2^10 = 1024 輪），值越大破解成本越高但登入驗證越慢

-- 登入驗證：比對密碼
SELECT username FROM users
WHERE username = 'alice'
  AND password_hash = crypt('my-secret-pw', password_hash);
-- crypt() 會自動從 password_hash 中提取 salt
-- 對輸入密碼做相同 salt + 相同迭代次數的雜湊
-- 回傳 1 row → 密碼正確；0 row → 密碼錯誤
```

迭代次數選擇建議：
- `gen_salt('bf', 6)`：約 0.1s，適合每秒百次登入的高流量系統
- `gen_salt('bf', 8)`：約 0.5s，一般 Web App 的平衡點
- `gen_salt('bf', 10)`：約 3s，安全性要求高的後台管理
- `gen_salt('bf', 12)`：約 10s，敏感系統（但需評估使用者體驗）

每次迭代次數加 1，驗證時間約翻倍。請在 staging 環境實測後決定。

```sql
-- 測試不同迭代次數的實際耗時
\timing on
SELECT crypt('test-password', gen_salt('bf', 8));  -- 觀察執行時間
```

### III. 敏感欄位對稱加密

身份證號、銀行卡號、病歷摘要——需要加密儲存，也需要解密查詢。用 PGP 對稱加密。

```sql
-- 寫入：加密後儲存
INSERT INTO customers (name, id_number_encrypted)
VALUES ('Bob', pgp_sym_encrypt('A123456789', 'my-aes-256-key-32bytes'));

-- 讀取並解密
SELECT name,
       pgp_sym_decrypt(id_number_encrypted, 'my-aes-256-key-32bytes') AS id_number
FROM customers
WHERE name = 'Bob';
-- 回傳: (Bob, A123456789)
-- pgp_sym_decrypt 回傳 text 型別

-- 用加密欄位做等值比對（不能用 LIKE，因為密文是 binary）
-- 解決方案：額外儲存 hash 值作為索引
ALTER TABLE customers ADD COLUMN id_number_hash bytea;
UPDATE customers SET id_number_hash = digest(id_number, 'sha256');
CREATE INDEX ON customers (id_number_hash);
-- 查詢時先 hash 再比對
SELECT * FROM customers WHERE id_number_hash = digest('A123456789', 'sha256');
```

**核心安全規則：key 絕不能出現在 SQL text 中**。pg_stat_statements 會記錄 SQL text → key 直接暴露在監控報表中。任何有權限查 `pg_stat_statements` 的人都能看到 key。

```sql
-- ❌ 致命錯誤：key 明文在 SQL 中 → pg_stat_statements 可見
SELECT pgp_sym_decrypt(col, 'hardcoded-key-abc-123') FROM customers;

-- ✅ 正確：key 從 session 變數讀取 → SQL text 中無 key 出現
-- 應用層先執行：SET SESSION myapp.enc_key = '...'
SELECT pgp_sym_decrypt(col, current_setting('myapp.enc_key')) FROM customers;
```

### IV. 公鑰加密（跨系統交換）

A 系統需要把加密資料傳給 B 系統，但雙方不能共享對稱密鑰。A 用 B 的公鑰加密，B 用自己的私鑰解密。

```sql
-- A 系統：用 B 系統的公鑰加密
UPDATE customers SET id_number_encrypted = pgp_pub_encrypt(
    pgp_sym_decrypt(id_number_encrypted, current_setting('myapp.enc_key')),
    dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v2

mQENBGabcdef...（公鑰內容）...
-----END PGP PUBLIC KEY BLOCK-----')
);

-- B 系統：用自己的私鑰 + passphrase 解密
SELECT pgp_pub_decrypt(
    id_number_encrypted,
    dearmor('-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: GnuPG v2

...（私鑰內容）...
-----END PGP PRIVATE KEY BLOCK-----'),
    'private-key-passphrase'
) AS decrypted_id_number
FROM customers;
```

公鑰加密的適用環境：跨系統、跨組織、跨安全域的資料交換。例如：訂單系統 → 物流系統傳送加密的收件人姓名電話。

### V. 資料完整性校驗（digest / hmac）

非加密場景——用來確認資料在傳輸或儲存後未被篡改：

```sql
-- digest：單向 hash（無 key，任何人可重算）
SELECT digest('hello world', 'sha256');
-- \x b94d27b9934d3e08a52e52d7da7dabfa...

-- hmac：帶 key 的 hash（只有知道 key 的人能重算和驗證）
SELECT hmac('important data', 'shared-secret-key', 'sha256');
-- 用在 API 簽章、訊息驗證等場景
```

### VI. App Dev 視角

C# 中使用 pgcrypto 的完整範例，核心是 session 變數傳遞 key：

```csharp
// 取得 key（從環境變數 / vault / key management service）
var encKey = Environment.GetEnvironmentVariable("DB_ENC_KEY")
    ?? throw new InvalidOperationException("DB_ENC_KEY not set");

using var conn = new NpgsqlConnection(connStr);
conn.Open();

// 每個 connection 開啟時設定加密 key
using var setKeyCmd = new NpgsqlCommand("SET SESSION myapp.enc_key = @key", conn);
setKeyCmd.Parameters.AddWithValue("key", encKey);
await setKeyCmd.ExecuteNonQueryAsync();

// 註冊使用者（bcrypt）
await conn.ExecuteAsync(
    "INSERT INTO users (username, password_hash) VALUES (@User, crypt(@Pass, gen_salt('bf', 10)))",
    new { User = "alice", Pass = "user-password" });

// 登入驗證
var user = await conn.QuerySingleOrDefaultAsync<string>(
    "SELECT username FROM users WHERE username = @User AND password_hash = crypt(@Pass, password_hash)",
    new { User = "alice", Pass = "user-password" });
// user != null → 登入成功

// 寫入加密敏感欄位
await conn.ExecuteAsync(
    "INSERT INTO customers (name, id_number_encrypted) " +
    "VALUES (@Name, pgp_sym_encrypt(@IdNumber, current_setting('myapp.enc_key')))",
    new { Name = "Bob", IdNumber = "A123456789" });

// 讀取並解密
var result = await conn.QuerySingleAsync<(string name, string idNumber)>(
    "SELECT name, pgp_sym_decrypt(id_number_encrypted, current_setting('myapp.enc_key')) " +
    "FROM customers WHERE name = @Name",
    new { Name = "Bob" });
```

**Connection Pool 注意事項**：`SET SESSION` 是 per-connection 的狀態。從 Npgsql pool 取出的 connection 可能殘留上一個 session 的變數值（但值理應相同，因為 key 是固定的）。如果 key 需要輪換，務必在每次 Open 後重新 SET。另一種更乾淨的方式是用 connection string 的 `Options` 參數：

```csharp
// 使用 Options 傳遞 GUC 參數（pool-safe，每個 physical connection 自動套用）
var connStr = "Host=10.0.1.50;Database=mydb;" +
              $"Options=-c myapp.enc_key={encKey}";
// 不需手動 SET SESSION，key 由連線參數帶入
```

---


## 4. pg_trgm — 模糊文本搜索

pg_trgm 解決 `LIKE '%keyword%'` 在百萬行表上走 Seq Scan 的問題。透過 GIN trigram 索引將查詢從秒級降低到毫秒級，同時提供 `similarity()` 做模糊搜尋排序。

### I. 場景

一張 500 萬行的 `products` 表，前端搜尋框輸入「无线耳机」，後端產生 `WHERE name LIKE '%无线耳机%'`。沒有 trigram index 時，Seq Scan 掃 500 萬行 / 23 秒。尖峰時段 10 QPS 搜尋 = 230 秒 cumulative，DB 直接被塞爆。

GIN trigram 索引的原理很簡單：把每個字串拆成所有連續的三字元組（前後補空格），"hello" → `{" h"," he","hel","ell","llo","lo "}`。查詢 `'%ello%'` → trigram `{" el","ell","llo","lo "}`。GIN 索引用 trigram 做鍵值快速定位候選行，再對原始資料做精確 LIKE 比對。

```mermaid
flowchart LR
    A["查詢 LIKE '%ello%'"] --> B["拆解為 trigram"]
    B --> C["GIN Index Scan<br/>搜尋 trigram 鍵值"]
    C --> D["回傳候選 CTID 列表"]
    D --> E["Bitmap Heap Scan<br/>精確比對原始資料"]
    E --> F["回傳結果"]
```

### II. 建索引前後對比

```sql
-- 建 index 前的 Seq Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE name LIKE '%无线耳机%';
-- Seq Scan on products  (actual time=12345..23456 rows=15 loops=1)
--   Filter: (name ~~ '%无线耳机%'::text)
--   Rows Removed by Filter: 5000000
--   Buffers: shared read=45000
-- Execution Time: 23456.789 ms
-- 解讀：500 萬行掃描，只找到 15 行符合，22478ms

-- 建 pg_trgm GIN 索引
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- 建 index 後的 Index Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE name LIKE '%无线耳机%';
-- Bitmap Heap Scan on products  (actual time=0.123..2.456 rows=15 loops=1)
--   Recheck Cond: (name ~~ '%无线耳机%'::text)
--   Heap Blocks: exact=12
--   ->  Bitmap Index Scan on idx_products_name_trgm
--         Index Cond: (name ~~ '%无线耳机%'::text)
--         Buffers: shared hit=15
-- Execution Time: 2.789 ms
-- 解讀：只掃了 12 個 heap block，全部從 cache 命中，2.789ms
```

從 23456ms 到 2.789ms，約 8400 倍加速。`Recheck Cond` 是 GIN 的標準行為——索引可能因 trigram 碰撞回傳 false positive，需對實際 row data 做精確比對。

### III. 相似度搜尋

除了加速 LIKE，pg_trgm 的 `similarity()` 函數提供真正的模糊搜尋：

```sql
-- similarity() 回傳 0~1，值越大越相似
SELECT name,
       similarity(name, '无线耳机') AS sim
FROM products
WHERE name % '无线耳机'
-- '%' 運算子 = WHERE similarity(name, 'x') > pg_trgm.similarity_threshold
ORDER BY sim DESC
LIMIT 10;
-- 結果範例：
-- 蓝牙无线耳机降噪版      | 0.471
-- 无线耳机 Pro 运动版     | 0.389
-- 无线入耳式耳机           | 0.312
```

精度/召回平衡控制：

```sql
-- 查看當前 threshold
SHOW pg_trgm.similarity_threshold;  -- 預設 0.3

-- 召回太少 → 降低 threshold（容許更多模糊匹配）
SET pg_trgm.similarity_threshold = 0.15;

-- 雜訊太多 → 提高 threshold（只回傳高相似度結果）
SET pg_trgm.similarity_threshold = 0.5;

-- 只看 trigram 拆解結果
SELECT show_trgm('无线耳机');
-- {" 无"," 无线","无线耳","线耳机","耳机 ","机 "}
```

`similarity()` 的計算公式：兩字串共享的 trigram 數量 / 兩字串的 trigram 總數（不重複）。"hello" 和 "hallo" 共享 `{" h","ha","ll","lo"}` 中的部分 trigram，所以分數不是 0。

### IV. 使用限制

```sql
-- GIN trigram index 可加速的 LIKE 模式：
-- ✅ LIKE '%xxx%'   —— infix search（前後都有 %）
-- ✅ LIKE '%xxx'     —— suffix search（只在前面有 %）
-- ❌ LIKE 'xxx%'     —— prefix search，B-tree index 更適合
-- ❌ 字串長度 < 3     —— trigram 拆不出來，GIN 會 fallback

-- 如需同時支援 prefix + infix，可以兩個 index 都建
CREATE INDEX idx_products_name_btree ON products (name);         -- 加速 LIKE 'xxx%'
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);  -- 加速 LIKE '%xxx%'
-- planner 會根據查詢的 LIKE pattern 自動選擇最佳 index
```

```mermaid
flowchart LR
    A["SELECT * FROM t<br/>WHERE name LIKE @Pattern"] --> B{"Pattern 類型？"}
    B -->|"前後都有 %<br/>'%abc%'"| C["GIN trigram<br/>Index Scan<br/>→ Bitmap Heap Scan"]
    B -->|"只有尾 %<br/>'abc%'"| D["B-tree<br/>Index Scan<br/>→ Index Scan"]
    B -->|"只有頭 %<br/>'%abc'"| C
    C --> E["毫秒級"]
    D --> E
```

### V. App Dev 視角

```csharp
// 模糊搜尋 + 相似度排序
const string sql = @"
    SELECT id, name, similarity(name, @Keyword) AS sim
    FROM products
    WHERE name % @Keyword
    ORDER BY sim DESC
    LIMIT 20";

var results = await conn.QueryAsync<ProductSearchResult>(sql,
    new { Keyword = "无线耳机" });

// LIKE 搜尋（受益於 GIN trigram index）
const string likeSql = @"
    SELECT id, name FROM products
    WHERE name LIKE @Pattern
    LIMIT 20";

// 前後都有 %，走 GIN trigram
var results1 = await conn.QueryAsync<Product>(likeSql,
    new { Pattern = "%无线耳机%" });

// 只有尾 %，走 B-tree
var results2 = await conn.QueryAsync<Product>(likeSql,
    new { Pattern = "无线耳机%" });
```

**效能陷阱與解決**：trigram 索引對短於 3 字的查詢無效——trigram 拆不出足夠的鍵值。如果你的產品搜尋大量使用短關鍵字（`'USB'`, `'CPU'`, `'4K'`），考慮並用全文搜尋（FTS + `tsvector` + GIN）。兩者不衝突——同一張表可以同時有 trigram GIN（for LIKE）和 tsvector GIN（for FTS）。

```sql
-- 同時支援兩種類型搜尋
ALTER TABLE products ADD COLUMN fts tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''))) STORED;
CREATE INDEX idx_products_trgm ON products USING GIN (name gin_trgm_ops);
CREATE INDEX idx_products_fts ON products USING GIN (fts);
```

---


## 5. pg_prewarm — 緩存預熱

pg_prewarm 解決 PG 重啟後前幾分鐘查詢極慢的問題——shared_buffers 全空，所有資料從磁碟讀。手動或自動把熱表載入 cache，讓重啟後的第一波查詢直接命中記憶體。

### I. 場景

PG crash recovery 或計劃內重啟後，shared_buffers 一片空白。第一波使用者查詢全部命中磁碟。NVMe 的 Seq Scan 要幾十秒，HDD 要幾分鐘。P99 latency 在重啟後的前 10~30 分鐘暴增，直到 autovacuum 和正常查詢慢慢把熱資料拉回 shared_buffers。這段冷啟動窗口對 SLA 是災難。

pg_prewarm 做的事情很簡單：讀取指定表的全部或部分 block，放入 shared_buffers 或 OS page cache。一個 SELECT `pg_prewarm('orders')` 就能把整張 orders 表從磁碟拉進記憶體。

```mermaid
flowchart LR
    subgraph "❌ 重啟後無預熱"
        A1["PG 重啟"] --> B1["shared_buffers 空"]
        B1 --> C1["用戶查詢 → 全部磁碟讀"]
        C1 --> D1["前 10~30min P99 timeout<br/>惡性循環：timeout → retry → 更多壓力"]
    end
    subgraph "✅ 重啟後有預熱"
        A2["PG 重啟"] --> B2["pg_prewarm() 或 autoprewarm"]
        B2 --> C2["熱資料已在 buffer"]
        C2 --> D2["用戶查詢 → 記憶體命中<br/>P99 立即恢復正常"]
    end
```

### II. 基本操作

```sql
-- 同步載入整張表到 shared_buffers（當前 session 阻塞直到完成）
SELECT pg_prewarm('public.orders');
-- 回傳載入的 block 數，例如 64000（500MB / 8KB）

-- 異步預熱：不阻塞當前 session，bgw 後台慢慢載入 OS page cache
SELECT pg_prewarm('public.orders', 'prefetch');
-- 適合大表（> 1GB），避免同步載入卡住幾十秒

-- 只預熱索引（熱查詢通常走 index scan，不必載入全表）
SELECT pg_prewarm('idx_orders_created_at');

-- mode 參數完整說明：
-- 'prefetch' → 非同步，讀入 OS page cache（最快，不阻塞）
-- 'buffer'   → 同步，讀入 shared_buffers（對查詢效果最直接）
-- 'main'     → 同步，同時讀入 OS page cache + shared_buffers

-- 指定要預熱的 block 範圍（只預熱最近一周的分區）
SELECT pg_prewarm('public.orders', 'buffer', 'main',
    first_block := 0,
    last_block  := 10000);   -- 只載入前 10000 個 block
```

### III. 驗證預熱效果

```sql
-- 預熱前：確認 cache 為空
SELECT c.relname,
       count(*) AS buffers_in_cache,
       pg_size_pretty(count(*) * 8192) AS cache_size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname = 'orders'
GROUP BY c.relname;
-- buffers_in_cache = 0 → 完全不在 shared_buffers

-- 執行預熱
SELECT pg_prewarm('public.orders');
-- 回傳 64000（表有 500MB）

-- 預熱後再查
SELECT c.relname,
       count(*) AS buffers_in_cache,
       pg_size_pretty(count(*) * 8192) AS cache_size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname = 'orders'
GROUP BY c.relname;
-- buffers_in_cache 應接近表的總 block 數
```

### IV. autoprewarm（PG 11+ 自動預熱）

autoprewarm 是一個 bgw，會在 PG 關機前把 shared_buffers 中的所有 block 列表 dump 到檔案，重啟後再逐一載入回來。

```sql
-- 手動觸發 dump（將當前 cache 內容寫入檔案）
SELECT autoprewarm_dump_now();

-- 查看 autoprewarm bgw 狀態
SELECT backend_type, state, backend_start
FROM pg_stat_activity
WHERE backend_type = 'autoprewarm leader';

-- 清空記錄（重建 cache 策略後重新學習）
SELECT autoprewarm_reset();

-- autoprewarm 的 dump 檔案位置
-- 預設在 PG data_directory 下：autoprewarm.blocks
```

**關鍵限制**：autoprewarm 用 **FIFO** 順序載入，不是按照熱門程度。如果你的 `autoprewarm.blocks` 中前面排了 5GB 的冷資料，後面才是 500MB 的核心熱表，那麼重啟後的前幾分鐘，使用者仍然可能命中尚未載入的熱表 block。**生產環境最佳實踐**：autoprewarm 做基礎恢復 + 手動 `pg_prewarm('核心表')` 搶先載入 SLA 敏感的表。

```sql
-- 重啟後的初始化腳本（優先載入核心業務表）
SELECT pg_prewarm('public.orders');
SELECT pg_prewarm('public.products');
SELECT pg_prewarm('public.users');
SELECT pg_prewarm('idx_orders_created_at');
SELECT pg_prewarm('idx_products_category');
-- 其餘交由 autoprewarm 按 FIFO 順序繼續載入
```

### V. App Dev 視角

應用啟動時的預熱腳本：

```csharp
// 應用啟動初始化（例如 Startup.cs 或 Program.cs）
using var conn = new NpgsqlConnection(connStr);
conn.Open();

// 異步預熱（prefetch mode 不阻塞）
var hotTables = new[] { "orders", "products", "users" };
foreach (var table in hotTables)
{
    await conn.ExecuteAsync(
        $"SELECT pg_prewarm('public.{table}', 'prefetch')");
}
Console.WriteLine($"Cache warmup initiated for {hotTables.Length} tables");
```

**Kubernetes / Docker 環境特別注意**：Pod 重啟後可能形成以下惡性循環：

```mermaid
flowchart TD
    A["Pod 重啟"] --> B{"有預熱步驟？"}
    B -->|"無"| C["readiness probe 立即通過"]
    C --> D["K8s 將流量導入 Pod"]
    D --> E["cold cache → 查詢 timeout"]
    E --> F["liveness probe 失敗"]
    F --> G["K8s 殺掉 Pod 重啟"]
    G --> A
    B -->|"有，postStart hook<br/>執行 pg_prewarm()"| H["cache 已熱才標記 Ready"]
    H --> I["流量導入 → 正常服務"]
```

解法：
1. `initialDelaySeconds` 設大一點（60~120s）
2. `postStart` lifecycle hook 執行預熱 SQL
3. 或在 `initContainer` 中跑完預熱，main container 才啟動

---


## 6. pg_buffercache — 緩存內容診斷

pg_buffercache 做的事：把 shared_buffers 中每個 8KB block 的所屬表、usagecount 攤開給你看。當你發現 cache 命中率低但不知道是 shared_buffers 太小、還是冷表 Scan 在汙染 cache、還是熱表根本沒進去過——這是第一線診斷工具。

### I. 場景

shared_buffers 設了 8GB，`pg_stat_statements` 卻顯示大量查詢的 `shared_blks_read` 很高，cache hit < 90%。問題可能是：(a) 8GB 不夠裝工作集，(b) 有冷表（如 200GB 的 log 表）的全表掃描把熱資料踢出去，(c) 熱資料在 cache 中但 usagecount 一直很低被頻繁淘汰。

`pg_buffercache` 可以逐個 block 告訴你 buffer 裡裝的是哪張表、被用了幾次、哪些即將被淘汰。

### II. 核心查詢

#### a. Top-10 表在 shared_buffers 中的佔用

```sql
SELECT c.relname,
       count(*) AS buffers_in_cache,
       pg_size_pretty(count(*) * 8192) AS cache_size,
       round(count(*) * 100.0 /
             (SELECT count(*) FROM pg_buffercache), 2) AS pct_of_total_cache,
       round(count(*) * 8192.0 /
             pg_relation_size(c.oid) * 100, 1) AS pct_of_table_cached
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname, c.oid
ORDER BY 2 DESC
LIMIT 10;
```

每欄解讀：
- `pct_of_total_cache`：這張表佔了 shared_buffers 的百分比。例：`event_logs` 佔 72%（5.76GB/8GB），而你的熱查詢表 `orders` 只佔 3%。
- `pct_of_table_cached`：這張表有多大比例被快取。100% = 整張表在 cache 中，5% = 只有少量 block 在 cache 中。
- 若 `event_logs` 佔 72% 且 `pct_of_table_cached` 只有 2%（200GB 表，只有 5.76GB 在 cache），這是最壞的信號：一張冷表的大 Seq Scan 在不斷「洗」cache。

#### b. 每張表的 cache 命中率

```sql
SELECT relname,
       heap_blks_hit,
       heap_blks_read,
       heap_blks_hit + heap_blks_read AS total_blks,
       round(100.0 * heap_blks_hit /
             NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS cache_hit_pct,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_read DESC
LIMIT 15;
```

每欄解讀：
- `cache_hit_pct`：< 95% → 需要關注，< 80% → 嚴重的 cache miss。
- `heap_blks_read` 大 → 大量磁碟讀。結合 `total_size` 判斷：小表（< 1GB）大量 read → 表很熱但一直被擠出 cache。大表（> 50GB）大量 read → 表太大裝不下 shared_buffers，需要 index / 分區 / pg_prewarm。
- 使用 `pg_statio_user_tables` 而非 `pg_buffercache`，因為前者是累計值，後者是瞬時 snapshot。

#### c. usagecount 分布分析

```sql
SELECT usagecount,
       count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS total_size,
       round(count(*) * 100.0 / (SELECT count(*) FROM pg_buffercache), 1) AS pct
FROM pg_buffercache
GROUP BY usagecount
ORDER BY usagecount DESC;
-- usagecount = 5 → 極熱，Clock Sweep 繞一圈也不會淘汰
-- usagecount = 0 → 即將被淘汰，下次需要空間時優先清除
-- usagecount = 1~2 → 一般熱度
```

每欄解讀：
- `usagecount = 5` 佔比大 → 熱資料穩定在 cache 中，Clock Sweep 壓力小。健康。
- `usagecount = 0` 佔比 > 30% → cache churn 高。新資料不斷讀入，舊資料快速淘汰。如果此時你的核心查詢 cache hit 低，說明 shared_buffers 太小。
- `usagecount = 0` 的 block 如果大部分來自你的熱表 → 熱表被冷表 Scan 汙染。

#### d. 找出 usagecount=0 最多的表（即將被淘汰的 cache）

```sql
SELECT c.relname,
       count(*) AS buffers_about_to_evict,
       pg_size_pretty(count(*) * 8192) AS size_about_to_evict
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.usagecount = 0
GROUP BY c.relname
ORDER BY 2 DESC
LIMIT 10;
-- 如果 orders 在這榜上 → 熱資料即將被淘汰，需要保護
```

### III. 診斷決策流程

```mermaid
flowchart TD
    A["觀察到 cache hit < 90%"] --> B["查 Top-10 緩存佔用表<br/>（pg_buffercache）"]
    B --> C{"有冷表<br/>佔用 > 30% total cache<br/>且 pct_of_table_cached < 10%？"}
    C -->|"是，例如 event_logs<br/>200GB 表佔了 6GB cache"| D["冷表 Seq Scan 汙染 cache<br/>→ 對該表查詢加 index<br/>→ 或移冷表到獨立 tablespace<br/>→ 或把報表改查 replica"]
    C -->|"否<br/>cache 分配合理"| E["查 usagecount 分布"]
    E --> F{"usagecount = 0<br/>佔 > 30%？"}
    F -->|"是"| G["buffer churn 過高<br/>→ 加大 shared_buffers<br/>→ 或 pg_prewarm 熱表鎖定"]
    F -->|"否，分布健康"| H["查 per-table cache_hit_pct<br/>（pg_statio_user_tables）"]
    H --> I{"特定表 < 80%？"}
    I -->|"是"| J["該表缺索引 → 全表掃描<br/>或表 > shared_buffers 裝不下<br/>→ 加 index / 分區 / pg_prewarm"]
    I -->|"否，cache 正常"| K["瓶頸不在 cache<br/>→ 檢查 CPU / lock / network"]
```

### IV. App Dev 視角

應用開發者不需要直接查 `pg_buffercache`（這是 DBA 的診斷工具），但必須理解一個行為後果：**你的每個不經意的 Seq Scan 都可能把別人的熱資料從 cache 踢出去**。

```csharp
// ❌ 有毒：批次處理全表掃描，汙染 shared_buffers
var allData = await conn.QueryAsync<Order>("SELECT * FROM orders");
foreach (var row in allData) { Process(row); }

// ✅ 健康：限制掃描範圍，最小化 cache 衝擊
var recent = await conn.QueryAsync<Order>(
    "SELECT * FROM orders WHERE created_at >= @Since",
    new { Since = DateTime.UtcNow.AddDays(-1) });
```

常見 cache 汙染模式與對策：
1. **後台報表 job** 每隔 30 分鐘跑 `SELECT * FROM huge_log_table` → 把整個 `huge_log_table` 掃進 shared_buffers，擠掉 `orders` 的 cache。對策：改用 replica 跑報表，或加 WHERE 只掃當天資料。
2. **資料遷移 / ETL job** 半夜跑 `INSERT INTO ... SELECT * FROM source` → source 表全進 cache，target 表大量 WAL + cache dirty。對策：排程在離峰，用小 batch 分批處理。
3. **管理介面的「匯出全部」按鈕** → 觸發千萬行 export。對策：限制匯出行數上限，或走 replica。

---


## 7. btree_gin / btree_gist — 複合索引

btree_gin 和 btree_gist 是兩個輕量 contrib extension，作用是在 GIN / GiST 的 operator class 目錄中註冊 B-tree 操作符（`=`, `<`, `<=`, `>`, `>=`）。核心價值：一個索引同時覆蓋多種查詢模式（FTS + 分類 + 價格），省掉 BitmapAnd 合併多個 bitmap 的開銷。

### I. 電商搜尋場景

一張 200 萬行 `products` 表，搜尋頁面需要同時支援三個條件：
1. 全文搜尋：`WHERE fts @@ to_tsquery('english', 'wireless')`
2. 分類過濾：`WHERE category = 'Electronics'`
3. 價格區間：`WHERE price <= 200`

標準做法建 3 個索引：1 個 GIN(fts)、1 個 B-tree(category)、1 個 B-tree(price)。Planner 分別對每個條件做 Bitmap Index Scan，再用 `BitmapAnd` 合併三個 bitmap 取交集。BitmapAnd 產生額外 CPU 開銷，且 3 個索引的維護成本（INSERT/UPDATE/DELETE）是 3 倍。

btree_gin 的解法：一個 GIN 複合索引同時包含三個欄位，單次 Bitmap Index Scan 直接定位，無需 BitmapAnd 合併。

```mermaid
flowchart LR
    subgraph "方案 A：3 個獨立索引"
        A1["SELECT<br/>WHERE fts @@ ...<br/>AND cat = ...<br/>AND price <= ..."] --> A2["Bitmap Index Scan on<br/>idx_fts (GIN)"]
        A1 --> A3["Bitmap Index Scan on<br/>idx_cat (B-tree)"]
        A1 --> A4["Bitmap Index Scan on<br/>idx_price (B-tree)"]
        A2 --> A5["BitmapAnd<br/>合併 3 個 bitmap"]
        A3 --> A5
        A4 --> A5
        A5 --> A6["Bitmap Heap Scan"]
    end
    subgraph "方案 B：1 個 GIN 複合索引"
        B1["SELECT<br/>WHERE fts @@ ...<br/>AND cat = ...<br/>AND price <= ..."] --> B2["Bitmap Index Scan on<br/>idx_composite (GIN)<br/>單次定位三條件"]
        B2 --> B3["Bitmap Heap Scan"]
    end
```

### II. 執行計劃對比

```sql
-- 前置
CREATE EXTENSION btree_gin;

-- 方案 A：3 個獨立索引
CREATE INDEX idx_a_fts ON products USING GIN (fts);
CREATE INDEX idx_a_cat ON products (category);
CREATE INDEX idx_a_price ON products (price);

-- 方案 B：1 個 GIN 複合索引
CREATE INDEX idx_b_composite ON products
    USING GIN (fts, category, price);

-- EXPLAIN 方案 A
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products
WHERE fts @@ to_tsquery('english', 'wireless')
  AND category = 'Electronics'
  AND price <= 200;
-- 預期看到：
--   ->  BitmapAnd
--         ->  Bitmap Index Scan on idx_a_fts
--         ->  Bitmap Index Scan on idx_a_cat
--         ->  Bitmap Index Scan on idx_a_price
--   ->  Bitmap Heap Scan

-- EXPLAIN 方案 B
-- 預期看到：
--   ->  Bitmap Index Scan on idx_b_composite
--         Index Cond: (fts @@ ... AND category = ... AND price <= ...)
--   ->  Bitmap Heap Scan
```

方案 B 通常快 10~30%（省去 BitmapAnd），且總索引大小更小（GIN 倒排結構對多欄位更緊湊）。

**空間對比**（200 萬行表，欄位含 text fts + text category + numeric price）：
- 3 個獨立索引：B-tree(category) ~12MB + B-tree(price) ~11MB + GIN(fts) ~70MB = 總計 ~93MB
- 1 個 GIN 複合索引：~72MB（節省 ~23%）

**寫入效能考量**：GIN 複合索引的 INSERT/UPDATE 比單一 B-tree 慢（GIN 需要維護 posting list），但比同時更新 3 個索引快。對於讀多寫少的搜尋場景（SELECT : INSERT > 10 : 1），這是淨收益。對於寫入密集場景（每秒千筆 INSERT），需要實測。

### III. btree_gist — EXCLUSION CONSTRAINT

btree_gist 有個 btree_gin 做不到的事：**排他約束（EXCLUSION CONSTRAINT）**。經典場景：酒店訂房——同一房型在同一時段不能重複預訂。

```sql
CREATE EXTENSION btree_gist;

CREATE TABLE hotel_bookings (
    id SERIAL PRIMARY KEY,
    hotel_id INT,
    room_id INT,
    guest_name TEXT,
    during TSRANGE,
    -- GiST EXCLUSION CONSTRAINT
    -- 同一 hotel_id + room_id 的 TSRANGE 不能有重疊 (&&)
    EXCLUDE USING GIST (
        hotel_id WITH =,
        room_id WITH =,
        during WITH &&
    )
);
```

為什麼需要 btree_gist？`EXCLUDE USING GIST` 要求所有欄位的操作符都屬於同一個 index method（GiST）的 operator class。B-tree 的 `=` 操作符預設不屬於 GiST——btree_gist 就是做這個註冊，讓 `int = int` 成為合法的 GiST 操作。`tsrange && tsrange`（範圍重疊檢查）則屬於 GiST 的內建能力。

```sql
-- 正常預訂：房型 301，6/10~6/12
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES (1, 301, 'Alice', '[2026-06-10 14:00, 2026-06-12 11:00)');
-- ✅ 成功

-- 衝突預訂：同房型時段重疊
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES (1, 301, 'Bob', '[2026-06-11 12:00, 2026-06-13 10:00)');
-- ERROR: conflicting key value violates exclusion constraint
-- DETAIL: Key (hotel_id, room_id, during)=(1, 301, [...] ) conflicts

-- 不衝突：不同房型
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES (1, 302, 'Charlie', '[2026-06-11 12:00, 2026-06-13 10:00)');
-- ✅ 成功

-- 不衝突：同房型但時段不重疊
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES (1, 301, 'David', '[2026-06-12 12:00, 2026-06-14 10:00)');
-- ✅ 成功（6/12 11:00 結束 vs 6/12 12:00 開始，不重疊）
```

EXCLUSION CONSTRAINT 的實作細節：它實際上在 GiST 索引中儲存每個 row 的 (hotel_id, room_id, during)，並在 INSERT/UPDATE 時檢查是否有衝突。效能開銷類似一個 GiST 索引 + 一次 index scan per INSERT。

### IV. 適合和不適合的場景

**適合 GIN 複合索引**：
- 電商搜尋：FTS + 分類 + 價格，條件幾乎總是同時出現
- 文件管理：全文搜尋 + 標籤 + 日期範圍
- 讀多寫少：SELECT : INSERT > 10 : 1

**不適合 GIN 複合索引**：
- 後台管理的任意組合過濾（可能只查分類、或只查價格、或只查時間），獨立 B-tree 讓 planner 靈活挑選
- 寫入極端密集的場景（每秒千筆 INSERT），GIN 維護成本高
- 欄位值高基數（如 UUID、email），GIN 的倒排結構對高基數欄位效率不如 B-tree

```mermaid
flowchart TD
    A["多條件查詢<br/>FTS + 分類 + 價格"] --> B{"查詢模式<br/>是固定組合？"}
    B -->|"是，總是<br/>3 條件一起用"| C["1 個 GIN 複合索引<br/>(btree_gin)<br/>空間小 / 無 BitmapAnd"]
    B -->|"否，任意組合"| D{"各條件<br/>各自選擇性？"}
    D -->|"都很高"| E["3 個獨立 B-tree<br/>planner 靈活挑選"]
    D -->|"FTS 選擇性強<br/>分類弱"| F["GIN(fts) +<br/>B-tree(category, price)"]
    D -->|"寫入壓力大<br/>> 1000 TPS"| G["獨立 B-tree<br/>GIN 維護成本太高"]
    C --> H["適合：電商搜尋"]
    E --> I["適合：後台管理"]
    F --> J["適合：內容平台"]
```

### V. App Dev 視角

```csharp
// GIN 複合索引下的搜尋查詢（Dapper）
const string searchSql = @"
    SELECT id, name, price, category, rating
    FROM products
    WHERE fts @@ to_tsquery('english', @Query)
      AND category = @Category
      AND price <= @MaxPrice
    ORDER BY rating DESC
    LIMIT 20";

var results = await conn.QueryAsync<ProductSearchResult>(searchSql, new {
    Query = "wireless & headphones",
    Category = "Electronics",
    MaxPrice = 300m
});

// EXCLUSION CONSTRAINT 的 INSERT（含衝突處理）
const string bookingSql = @"
    INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
    VALUES (@HotelId, @RoomId, @GuestName, @During)
    ON CONFLICT ON CONSTRAINT hotel_bookings_hotel_id_room_id_during_excl
    DO NOTHING
    RETURNING id";

var bookingId = await conn.QuerySingleOrDefaultAsync<int>(bookingSql, new {
    HotelId = 1,
    RoomId = 301,
    GuestName = "Alice",
    During = new NpgsqlRange<DateTime>(
        new DateTime(2026, 6, 10, 14, 0, 0),
        new DateTime(2026, 6, 12, 11, 0, 0))
});
// bookingId == 0 → 時段衝突（ON CONFLICT DO NOTHING 沒有 INSERT）
// bookingId > 0 → 預訂成功
```

Npgsql 的 `NpgsqlRange<T>` 原生支援 PG 的 range type（`tsrange`, `daterange` 等），在 EXCLUSION CONSTRAINT 場景中直接對應。DDD 設計中可以將 `during` 包裝成 `DateRange` value object，確保領域層的時間範圍語意正確。

**btree_gin / btree_gist 的使用成本**：兩者都是純中繼資料（只在 `pg_opclass` 系統表中註冊 operator class mapping），不會載入 shared library、不啟動 bgw、不消耗額外記憶體。唯一的注意事項：用 `pg_dump` + `pg_restore` 遷移資料庫時，目標庫需要先 `CREATE EXTENSION btree_gin / btree_gist`，否則依賴其 opclass 的索引無法建立，restore 會報錯。
