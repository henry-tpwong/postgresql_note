# PostgreSQL Extensions 深入探討

> **閱讀順序：由淺入深，逐步構建 PostgreSQL 生產級技術棧**

本書按 PostgreSQL Extension 的應用層次編排十大章節：

- **第一章（IMPORT FOREIGN SCHEMA）**：跨庫查詢基礎。從單一 remote server 的 foreign table 批量建立開始，掌握 FDW 架構、schema 導入、type mapping，以及 PG 14-17 的 parallel foreign scan / async append / MERGE pushdown 等演進。
- **第二章（pg_partman）**：PG 10-17 原生宣告式分區的自動化管理。使用 pg_partman 實現自動分區創建、retention policy 定時清理、以及 Background Worker 驅動的 partition lifecycle。
- **第三章（Citus 12）**：分散式 SQL 引擎。hash/range sharding、co-located join、schema-based sharding、非阻塞 rebalancing 等分散式數據庫核心主題。
- **第四章（PgBouncer）**：連接池。Process-per-Connection 模型的瓶頸與 Transaction Pooling 的解決方案，含生產級 HAProxy + PgBouncer 拓撲。
- **第五章（pg_stat_statements）**：查詢歸一化與聚合統計。Top-N 慢查詢、緩存命中率、WAL 產生量、JIT 開銷追蹤。
- **第六章（auto_explain）**：自動記錄執行計劃。與 pg_stat_statements 互補——WHAT is slow vs WHY it's slow。
- **第七章（pg_repack）**：在線表重組。不鎖表回收膨脹空間，與 VACUUM FULL / pg_squeeze 三方案對比。
- **第八章（pg_cron）**：PG 內建排程作業。定時 VACUUM、分區維護、物化視圖刷新。
- **第九章（pg_stat_kcache）**：查詢 CPU 與實體 IO 統計。getrusage() 獲取真實磁盤讀寫、fsync 次數、CPU 時間。
- **第十章（hypopg）**：假設性索引分析。不實際建索引即可預覽 EXPLAIN 計劃，零成本試錯。

> 更新於 2026-05-30，全面升級至 PG 16+ 生態。所有範例使用 PG 16 語法與路徑。

---

# 一、Non-Contrib Extensions（需額外安裝）

> 以下 Extension 需透過 pt install、源碼編譯或第三方工具安裝，非 PG 內建 contrib。適用於需要額外功能的生產環境。

---

## 一、IMPORT FOREIGN SCHEMA — 一鍵創建 Foreign Table

### 1. 核心語法與背景

在 PostgreSQL 的生態中，Foreign Data Wrapper（FDW）讓你可以像操作本地資料表一樣查詢遠端資料庫的內容。然而，早期的 FDW 使用上有一個不小的痛點：你必須手動寫 `CREATE FOREIGN TABLE` 逐一對應遠端的每個欄位，當遠端表結構複雜或數量眾多時，這項工作既繁瑣又容易出錯。

`IMPORT FOREIGN SCHEMA` 正是為了解決這個問題而生的 —— 它是一條 SQL 語句，能自動查詢遠端資料庫的 `information_schema.columns`，並在本地生成對應的 `CREATE FOREIGN TABLE` DDL，瞬間完成大量 Foreign Table 的創建。

```sql
IMPORT FOREIGN SCHEMA remote_schema_name
    [ { LIMIT TO | EXCEPT } ( table_name [, ...] ) ]
  FROM SERVER server_name
  INTO local_schema_name
  [ OPTIONS ( option 'value' [, ... ] ) ]
```

核心運作機制如下圖所示：

```mermaid
flowchart TD
    A["IMPORT FOREIGN SCHEMA 執行"] --> B["查詢遠端 information_schema.columns"]
    B --> C["獲取 table_name, column_name, data_type, is_nullable 等資訊"]
    C --> D{"是否指定 LIMIT TO / EXCEPT?"}
    D -->|"LIMIT TO"| E["僅保留指定表的欄位清單"]
    D -->|"EXCEPT"| F["排除指定表，保留其餘"]
    D -->|"無過濾"| G["保留所有表"]
    E --> H["對每個表生成 CREATE FOREIGN TABLE DDL"]
    F --> H
    G --> H
    H --> I["在 local_schema 中執行 DDL"]
    I --> J["完成！Foreign Table 可立即查詢"]
```

> **初學者導讀**：你可以把 `IMPORT FOREIGN SCHEMA` 想像成「從遠端資料庫下載一份完整的表結構藍圖到本地」。之後你在本地對這些 Foreign Table 下 `SELECT`，PostgreSQL 會自動將查詢轉發到遠端執行並取回結果。你不需要關心網路細節，就像操作本地表一樣。

### 2. 完整操作示範（PG 16 範例）

以下示範以兩台執行 PostgreSQL 16 的伺服器為例：Remote PG（遠端）與 Local PG（本地）。

#### 2.1 遠端 PostgreSQL 準備

在遠端伺服器上，我們需要一個可供連線的 database 與 schema，以及一些測試資料。

```sql
-- 遠端：建立測試用 database 與 schema
CREATE DATABASE remote_db;
\c remote_db
CREATE SCHEMA remote_app;
CREATE ROLE fdw_user WITH LOGIN PASSWORD 'secure_pass';
GRANT USAGE ON SCHEMA remote_app TO fdw_user;
GRANT SELECT ON ALL TABLES IN SCHEMA remote_app TO fdw_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA remote_app GRANT SELECT ON TABLES TO fdw_user;

-- 遠端：建立測試表
SET search_path TO remote_app;

CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    username   VARCHAR(50) NOT NULL,
    email      VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    user_id    INT REFERENCES users(user_id),
    amount     NUMERIC(10, 2) NOT NULL,
    placed_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE audit_log (
    log_id     SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    action     VARCHAR(20),
    log_time   TIMESTAMPTZ DEFAULT now()
);

INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob',   'bob@example.com');

INSERT INTO orders (user_id, amount) VALUES
    (1, 100.50),
    (1, 200.00),
    (2, 50.75);
```

遠端 `pg_hba.conf`（路徑：`/etc/postgresql/16/main/pg_hba.conf`）需要允許本地連線：

```ini
## 允許來自本地子網的連線
host    remote_db    fdw_user    192.168.1.0/24    scram-sha-256
```

修改後重新載入設定：

```bash
sudo pg_ctlcluster 16 main reload
```

> **補充（Senior Dev）**：生產環境中建議使用 `scram-sha-256` 而非 `md5`。若遠端與本地在同一台機器上測試，可將 IP 設為 `127.0.0.1/32`。

#### 2.2 本地 PostgreSQL 操作

```sql
-- 1. 建立擴展
CREATE EXTENSION postgres_fdw;

-- 2. 建立遠端伺服器定義
CREATE SERVER remote_pg_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host '192.168.1.100',
        port '5432',
        dbname 'remote_db'
    );

-- 3. 建立使用者映射（將本地角色對應到遠端登入憑證）
CREATE USER MAPPING FOR CURRENT_USER
    SERVER remote_pg_server
    OPTIONS (
        user 'fdw_user',
        password 'secure_pass'
    );

-- 4. 建立本地 Schema（用於存放 Foreign Table）
CREATE SCHEMA fdw_tables;

-- 5. 一鍵導入遠端 Schema 全部資料表
IMPORT FOREIGN SCHEMA remote_app
    FROM SERVER remote_pg_server
    INTO fdw_tables;

-- 6. 驗證結果
\det+ fdw_tables.*
\d fdw_tables.users
```

#### 2.3 驗證：Foreign Table 結構與資料

```sql
-- 查看 Foreign Table 的欄位定義
SELECT
    table_name,
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'fdw_tables'
ORDER BY table_name, ordinal_position;
```

預期輸出：

| table_name | column_name | data_type             | is_nullable |
|------------|-------------|-----------------------|-------------|
| audit_log  | log_id      | integer               | NO          |
| audit_log  | table_name  | character varying     | YES         |
| audit_log  | action      | character varying     | YES         |
| audit_log  | log_time    | timestamp with time zone | YES      |
| orders     | order_id    | integer               | NO          |
| orders     | user_id     | integer               | YES         |
| orders     | amount      | numeric               | NO          |
| orders     | placed_at   | timestamp with time zone | YES      |
| users      | user_id     | integer               | NO          |
| users      | username    | character varying     | NO          |
| users      | email       | character varying     | YES         |
| users      | created_at  | timestamp with time zone | YES      |

驗證查詢：

```sql
-- 跨庫查詢！本地直接 JOIN 遠端資料
SELECT u.username, o.amount, o.placed_at
FROM fdw_tables.users u
JOIN fdw_tables.orders o ON u.user_id = o.user_id;
```

```
 username | amount |         placed_at
----------+--------+----------------------------
 alice    | 100.50 | 2024-01-15 10:30:00+00
 alice    | 200.00 | 2024-01-15 11:00:00+00
 bob      |  50.75 | 2024-01-15 12:00:00+00
```

#### 2.4 整體架構

```mermaid
graph TD
    subgraph "Local PG Instance"
        L1["fdw_tables.users<br/>(Foreign Table)"]
        L2["fdw_tables.orders<br/>(Foreign Table)"]
        L3["fdw_tables.audit_log<br/>(Foreign Table)"]
        S["postgres_fdw Extension"]
        SRV["remote_pg_server<br/>(Server Definition)"]
        MAP["USER MAPPING<br/>local_user → fdw_user"]
    end

    subgraph "Remote PG Instance"
        R1["remote_app.users<br/>(Physical Table)"]
        R2["remote_app.orders<br/>(Physical Table)"]
        R3["remote_app.audit_log<br/>(Physical Table)"]
        IC["information_schema.columns"]
    end

    L1 -.->|"SELECT 時即時轉發"| S
    L2 -.->|"SELECT 時即時轉發"| S
    L3 -.->|"SELECT 時即時轉發"| S
    S --> SRV
    SRV --> MAP
    MAP -->|"連線認證"| R1
    MAP -->|"連線認證"| R2
    MAP -->|"連線認證"| R3
    IC -.->|"IMPORT 時讀取結構"| L1
    IC -.->|"IMPORT 時讀取結構"| L2
    IC -.->|"IMPORT 時讀取結構"| L3
```

> **初學者導讀**：請注意，Foreign Table **不儲存任何資料**。它就是一個「指向遠端表的指標」。每次你查詢 Foreign Table，PostgreSQL 都會即時連線到遠端、執行 SQL、取回結果。這就和 DBLINK 的理念類似，但更加透明。

### 3. LIMIT TO 與 EXCEPT 過濾

當遠端 Schema 中包含大量資料表，但你只需要導入其中幾張時，可以使用過濾子句。`LIMIT TO` 和 `EXCEPT` 互斥，不可同時指定。

```sql
-- 只導入 users 和 orders 兩張表
IMPORT FOREIGN SCHEMA remote_app
    LIMIT TO (users, orders)
    FROM SERVER remote_pg_server
    INTO fdw_tables;

-- 導入除 audit_log 以外的全部表
IMPORT FOREIGN SCHEMA remote_app
    EXCEPT (audit_log)
    FROM SERVER remote_pg_server
    INTO fdw_tables;
```

```mermaid
flowchart LR
    subgraph "遠端 remote_app 所有表"
        A["users ✅"]
        B["orders ✅"]
        C["audit_log ✅"]
        D["products ✅"]
        E["inventory ✅"]
    end

    subgraph "LIMIT TO (users, orders)"
        A2["users"]
        B2["orders"]
    end

    subgraph "EXCEPT (audit_log)"
        A3["users"]
        B3["orders"]
        D3["products"]
        E3["inventory"]
    end

    A --> A2
    B --> B2
    A --> A3
    B --> B3
    C -.->|"❌ 排除"| A3
    D --> D3
    E --> E3
```

> **初學者導讀**：`LIMIT TO` = 「只拿這些」，`EXCEPT` = 「除了這些，其他都拿」。實務上，當遠端 Schema 表很多但你只關心幾張核心表時，`LIMIT TO` 最常用。當你幾乎全部都要、只想跳過幾張無關的日誌或暫存表時，用 `EXCEPT`。

### 4. 注意事項 — View、Materialized View、Foreign Table 也會被導入

`IMPORT FOREIGN SCHEMA` 在執行時，會查詢遠端 `information_schema.columns`。這個系統 View 並不區別底層是實體表（`relkind = 'r'`）、View（`relkind = 'v'`）、Materialized View（`relkind = 'm'`）還是 Foreign Table（`relkind = 'f'`） —— 只要它有欄位定義，就會被導入。

| relkind | 類型 | 會被導入？ | 注意事項 |
|---------|------|-----------|---------|
| `r` | 普通表 (Table) | ✅ 是 | 正常導入，使用最頻繁 |
| `v` | 視圖 (View) | ✅ 是 | 導入後可查詢遠端 View，邏輯透明 |
| `m` | 實體化視圖 (Materialized View) | ✅ 是 | 導入後可查詢，但無法觸發遠端 REFRESH |
| `f` | 外來表 (Foreign Table) | ✅ 是 | **串聯 FDW**：導入後本地再指向遠端的另一個 FDW |
| `p` | 分區表 (Partitioned Table) | ✅ 是 | 分區定義本身被導入，子分區各別導入 |

> **補充（Senior Dev）**：串聯 FDW（Foreign → Foreign）會形成多層轉發鏈。假設 Server A → Server B → Server C，則查詢 A 的 Foreign Table 時，最終會抵達 C。每一層都會增加延遲，且 SQL Pushdown 能力在每一層都可能受損。除非有明確的架構原因（例如跨網段金庫），否則不建議超過兩層。

#### 如何排除特定 relkind

`IMPORT FOREIGN SCHEMA` 本身無法按 relkind 過濾。如果你只想導入實體表，可以在導入後手動刪除不需要的 Foreign Table，或使用以下查詢輔助分析：

```sql
-- 在遠端查看 remote_app 中各類型的物件
SELECT
    c.relname AS name,
    c.relkind,
    CASE c.relkind
        WHEN 'r' THEN 'Table'
        WHEN 'v' THEN 'View'
        WHEN 'm' THEN 'Materialized View'
        WHEN 'f' THEN 'Foreign Table'
        WHEN 'p' THEN 'Partitioned Table'
    END AS type
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE n.nspname = 'remote_app'
  AND c.relkind IN ('r', 'v', 'm', 'f', 'p')
ORDER BY c.relkind, c.relname;
```

### 5. 限制與適用範圍（總結）

#### 會被導入的定義

| 定義 | 狀態 |
|------|------|
| 欄位名稱與型別 | ✅ 完整導入 |
| NOT NULL 約束 | ✅ 導入 |
| DEFAULT 值 | ❌ 不會導入（遠端 DEFAULT 在遠端執行時套用） |
| PRIMARY KEY | ❌ 不會導入 |
| FOREIGN KEY | ❌ 不會導入 |
| CHECK 約束 | ❌ 不會導入 |
| UNIQUE 約束 | ❌ 不會導入 |
| 索引 | ❌ 不會導入（遠端查詢使用遠端的索引） |
| 觸發器 (Trigger) | ❌ 不會導入（在遠端觸發） |

> **初學者導讀**：這是「代理查詢」模式，不是「同步複製」。約束條件在遠端維護，索引在遠端生效。本地只是一個「查詢入口」，並不擁有這些元數據。

#### 潛在風險

| 風險 | 說明 | 緩解方式 |
|------|------|---------|
| 網路延遲 | 每個查詢都需往返遠端伺服器，延遲取決於網路品質 | 減少頻繁的小查詢，合併成批次查詢；考慮 Materialized View 緩存 |
| 連線耗盡 | 高並發查詢 Foreign Table 時可能耗盡遠端連線池 | 配置 `max_connections`、使用 PgBouncer 連線池、設定 `fetch_size` |
| SQL Pushdown 限制 | 並非所有 SQL 都能下推到遠端執行；複雜查詢可能將大量資料拉回本地處理 | 使用 `EXPLAIN VERBOSE` 檢查 Remote SQL；必要時在遠端建立 View 簡化查詢 |
| 交易隔離 | 遠端交易預設為 `READ COMMITTED`；兩階段提交（2PC）需手動配置 | 對一致性要求高的場景，避免跨庫交易 |
| 憑證洩漏 | `pg_user_mapping` 中密碼以明文儲存 | 使用 `peer` / `cert` 認證，或將密碼放入 `~/.pgpass` |

### 6. postgres_fdw 版本演進（PG 14-17）

#### I. PG 14 — postgres_fdw 2.0

PG 14 是 postgres_fdw 的重大里程碑，內部版本號升至 2.0。

#### async_append — 非同步並行掃描

當你查詢多個 Foreign Table 並使用 `UNION ALL` 彙總時，傳統做法是**順序**逐一查詢每個遠端節點。`async_append` 允許 PostgreSQL 同時向多個遠端伺服器發出查詢，顯著減少總等待時間。

```mermaid
sequenceDiagram
    participant LP as Local PG
    participant R1 as Remote Server 1
    participant R2 as Remote Server 2
    participant R3 as Remote Server 3

    Note over LP: SELECT * FROM<br/>ft1 UNION ALL<br/>SELECT * FROM ft2 UNION ALL<br/>SELECT * FROM ft3;

    LP->>R1: SELECT * FROM remote_table_1
    LP->>R2: SELECT * FROM remote_table_2
    LP->>R3: SELECT * FROM remote_table_3

    R1-->>LP: result set 1
    R2-->>LP: result set 2
    R3-->>LP: result set 3

    Note over LP: Append 彙總結果（非同步等待最慢者）
```

啟用條件：
- `use_remote_estimate = true`（或至少 `analyze` 過 Foreign Table）
- Planner 判斷多個 Foreign Scan 路徑可以並行時

#### batch_size INSERT — 批次寫入

批量插入 Foreign Table 時，postgres_fdw 可將多行打包成一個批次傳送，而不是逐行 INSERT：

```sql
-- 傳統：逐行傳送（效率低）
INSERT INTO fdw_tables.users (username, email) VALUES ('a', 'a@x.com');
INSERT INTO fdw_tables.users (username, email) VALUES ('b', 'b@x.com');
INSERT INTO fdw_tables.users (username, email) VALUES ('c', 'c@x.com');

-- PG 14+ 批次傳送：打包成單次 INSERT 多行
INSERT INTO fdw_tables.users (username, email) VALUES
    ('a', 'a@x.com'),
    ('b', 'b@x.com'),
    ('c', 'c@x.com');
-- Remote SQL: INSERT INTO users(username, email) VALUES ($1,$2),($3,$4),($5,$6)
```

可在 Foreign Table 層級設定批次大小（單位：行數）：

```sql
ALTER FOREIGN TABLE fdw_tables.users OPTIONS (ADD batch_size '100');
```

#### parallel_commit — 平行提交

當一個本地交易修改了多個不同遠端伺服器的 Foreign Table 時：

```mermaid
flowchart LR
    LP["Local PG 交易 COMMIT"] --> S1["PREPARE TRANSACTION<br/>on Server A"]
    LP --> S2["PREPARE TRANSACTION<br/>on Server B"]
    LP --> S3["PREPARE TRANSACTION<br/>on Server C"]
    S1 --> C1["COMMIT PREPARED<br/>on Server A"]
    S2 --> C2["COMMIT PREPARED<br/>on Server B"]
    S3 --> C3["COMMIT PREPARED<br/>on Server C"]
    C1 -.- P["✅ 全部成功，或 ❌ 全部失敗<br/>(2PC 兩階段提交)"]
    C2 -.- P
    C3 -.- P
```

- **PG 13 及以前**：`COMMIT PREPARED` 是順序執行的（一次一個遠端）
- **PG 14 起**：`COMMIT PREPARED` 以平行方式發送，大幅縮短跨多伺服器交易的提交時間

#### import_default / import_not_null 選項

`IMPORT FOREIGN SCHEMA` 新增兩個選項：

| 選項 | 作用 |
|------|------|
| `import_default` | 是否導入欄位的預設值（`true`/`false`，預設 `false`）|
| `import_not_null` | 是否導入 NOT NULL 約束（`true`/`false`，預設 `true`）|

```sql
IMPORT FOREIGN SCHEMA remote_app
    FROM SERVER remote_pg_server
    INTO fdw_tables
    OPTIONS (import_default 'true', import_not_null 'true');
```

#### II. PG 15 — MERGE 支援

PG 15 引入了 SQL 標準的 `MERGE`（`MERGE INTO ... USING ... WHEN MATCHED ... WHEN NOT MATCHED ...`）。postgres_fdw 可以將完整的 `MERGE` 語句下推到遠端執行：

```sql
-- 本地 MERGE（會完整下推到遠端執行）
MERGE INTO fdw_tables.users u
USING (VALUES ('alice', 'alice_new@example.com')) AS v(username, email)
ON u.username = v.username
WHEN MATCHED THEN
    UPDATE SET email = v.email
WHEN NOT MATCHED THEN
    INSERT (username, email) VALUES (v.username, v.email);

-- 使用 EXPLAIN VERBOSE 可以看到 Remote SQL 包含了完整的 MERGE
EXPLAIN VERBOSE
MERGE INTO fdw_tables.users u
USING (VALUES ('charlie', 'charlie@example.com')) AS v(username, email)
ON u.username = v.username
WHEN MATCHED THEN UPDATE SET email = v.email
WHEN NOT MATCHED THEN INSERT (username, email) VALUES (v.username, v.email);

-- Remote SQL (範例):
-- MERGE INTO users u USING (VALUES(...)) v(username, email) ON ... WHEN MATCHED ... WHEN NOT MATCHED ...
```

> **補充（Senior Dev）**：MERGE 下推的條件包括遠端 PG 版本必須 ≥ 15，且 MERGE 中不能包含本地的 expression。若遠端低於 PG 15，postgres_fdw 會退化為逐行檢查、分別 UPDATE 或 INSERT，效能差距顯著。

#### III. PG 16 — JOIN pushdown 增強

PG 16 大幅擴展了 JOIN pushdown 的支援範圍：

| JOIN 類型 | PG 15 及以前 | PG 16 |
|-----------|-------------|-------|
| INNER JOIN | ✅ 支援 | ✅ 支援 |
| LEFT JOIN | ✅ 支援 | ✅ 支援 |
| RIGHT JOIN | ✅ 支援 | ✅ 支援 |
| FULL JOIN | ❌ 不支援 | ✅ 支援 |
| 5 表 JOIN | ❌ 限制 | ✅ 擴展至 5 表 |
| 帶表達式的 JOIN | 部分 | ✅ 更廣泛 |

FULL JOIN pushdown 範例：

```sql
-- PG 16：FULL JOIN 兩個 Foreign Table，可完整下推到遠端
SELECT
    COALESCE(u.username, 'N/A') AS username,
    COALESCE(o.order_id, -1)   AS order_id,
    o.amount
FROM fdw_tables.users u
FULL JOIN fdw_tables.orders o ON u.user_id = o.user_id;
```

```sql
-- EXPLAIN 驗證（PG 16）
EXPLAIN VERBOSE
SELECT u.username, o.amount
FROM fdw_tables.users u
FULL JOIN fdw_tables.orders o ON u.user_id = o.user_id;
```

```
Foreign Scan
  Output: username, amount
  Relations: (fdw_tables.users u) FULL JOIN (fdw_tables.orders o)
  Remote SQL: SELECT r1.username, r2.amount FROM remote_app.users r1
              FULL JOIN remote_app.orders r2 ON (r1.user_id = r2.user_id)
```

#### parallel_abort

對稱於 PG 14 的 `parallel_commit`，PG 16 新增了 `parallel_abort`：當本地交易中止時，可以同時對多個遠端伺服器執行 `ROLLBACK PREPARED`，而非順序執行。

#### IV. PG 17 — 持續改進

PG 17 進一步強化了 postgres_fdw 的功能：

#### 更多子查詢 Pushdown 情境

- 帶有 `EXISTS` / `NOT EXISTS` 子查詢可下推
- 部分 `CASE WHEN` 表達式可下推
- 部分標量子查詢可下推

#### ANALYZE 效能提升

對 Foreign Table 執行 `ANALYZE` 時，PG 17 優化了遠端取樣邏輯，減少了遠端傳回的樣本資料量，對大型表特別有感。

```sql
-- 對 Foreign Table 收集統計資訊（供本地 planner 使用）
ANALYZE fdw_tables.users;
```

#### EXPLAIN (MEMORY) 支援

```sql
EXPLAIN (ANALYZE, MEMORY, VERBOSE)
SELECT * FROM fdw_tables.orders WHERE amount > 100;
```

可以查看 Foreign Scan 節點消耗的記憶體。

### 7. 效能與監控

#### I. EXPLAIN VERBOSE 查看 Remote SQL

這是 FDW 效能調校最重要的工具。透過 `EXPLAIN VERBOSE`，你可以看到 postgres_fdw 實際發送給遠端伺服器的 SQL 內容：

```sql
EXPLAIN (VERBOSE, ANALYZE)
SELECT u.username, COUNT(*) AS order_count, SUM(o.amount) AS total
FROM fdw_tables.users u
JOIN fdw_tables.orders o ON u.user_id = o.user_id
WHERE o.amount > 50
GROUP BY u.username;
```

```
HashAggregate  (cost=... rows=... width=...) (actual time=...)
  Group Key: u.username
  ->  Foreign Scan  (cost=... rows=... width=...) (actual time=...)
        Output: u.username, o.amount
        Relations: (fdw_tables.users u) INNER JOIN (fdw_tables.orders o)
        Remote SQL: SELECT r1.username, r2.amount
                    FROM (remote_app.users r1
                    INNER JOIN remote_app.orders r2
                      ON (((r1.user_id = r2.user_id)) AND (r2.amount > (50)::numeric)))
```

> **初學者導讀**：`Remote SQL` 這一行就是黃金資訊！它告訴你哪些操作被下推到了遠端、哪些留在本地。理想狀況下，過濾條件和聚合都要出現在 Remote SQL 中。如果 Remote SQL 是 `SELECT * FROM table`（無任何 WHERE），那表示大量不需要的資料都被拉回了本地，效能會非常差。

#### II. fetch_size 調校

`fetch_size` 控制 postgres_fdw 每次從遠端遊標（cursor）提取的行數。這個值直接影響查詢的吞吐量與延遲。

| fetch_size | 行為 | 適用場景 |
|------------|------|---------|
| 0（預設） | 不建立遊標；一次性讀取全部結果集 | 小型查詢（< 1 萬行） |
| 100 | 每次提取 100 行，適合快速回傳首批結果 | Web UI 分頁、逐步渲染 |
| 1000 | 每次提取 1000 行，平衡吞吐量與延遲 | 中大型查詢的通用預設 |
| 10000 | 每次提取 10000 行，最大化吞吐量 | 批量 ETL、匯出作業 |
| > 100000 | 極大批次；注意遠端記憶體消耗 | 資料遷移 |

```sql
-- 在 Foreign Table 層級設定
ALTER FOREIGN TABLE fdw_tables.orders OPTIONS (ADD fetch_size '1000');

-- 在 Foreign Server 層級設定（所有該 Server 下的表都會繼承）
ALTER SERVER remote_pg_server OPTIONS (ADD fetch_size '1000');
```

```mermaid
flowchart TD
    A["SELECT * FROM fdw_tables.orders<br/>WHERE amount > 50"]
    A --> B{"fetch_size 設定?"}

    B -->|"fetch_size = 0"| C["遠端執行 SELECT<br/>一次性將所有結果傳回"]
    C --> D["✅ 最簡單<br/>❌ 大結果集可能 OOM<br/>❌ 首批結果需等全部完成"]

    B -->|"fetch_size = 1000"| E["遠端宣告 CURSOR<br/>批次 FETCH 1000 行"]
    E --> F["本地 FETCH 1000 行"]
    F --> G{"還有更多?"}
    G -->|"是"| E
    G -->|"否"| H["✅ 關閉遠端 CURSOR<br/>✅ 記憶體可控<br/>✅ 首批資料更快"]

    B -->|"fetch_size = 100"| I["遠端宣告 CURSOR<br/>批次 FETCH 100 行"]
    I --> J["本地 FETCH 100 行"]
    J --> K{"還有更多?"}
    K -->|"是"| I
    K -->|"否"| L["✅ 最快看到首批資料<br/>❌ 網路往返次數多"]
```

> **補充（Senior Dev）**：使用 `fetch_size > 0` 時，postgres_fdw 會利用遠端 CURSOR。這意味著遠端交易會持續到本地端讀完所有結果為止。如果本地端中途崩潰或長時間閒置，遠端的 CURSOR 會佔用資源。建議為遠端設定 `idle_in_transaction_session_timeout`。

#### III. use_remote_estimate

本地 PostgreSQL 的查詢規劃器（planner）需要統計資訊來估算查詢的成本（cost）。對於 Foreign Table，預設情況下 planner 使用內建的低成本假設（`rows = 1000`，`cost = 10`），這經常導致不準確的執行計畫。

`use_remote_estimate = true` 告訴 postgres_fdw：規劃查詢時，先向遠端伺服器查詢 `pg_stats` 等統計資訊，以獲得更精確的 row estimate。

```sql
-- 在 Foreign Server 層級（或 Foreign Table 層級）啟用
ALTER SERVER remote_pg_server OPTIONS (ADD use_remote_estimate 'true');

-- 驗證：查看 planner 是否使用了遠端統計
EXPLAIN SELECT * FROM fdw_tables.orders WHERE amount > 100;
```

```
Foreign Scan on orders  (cost=100.00..150.00 rows=500 width=40)
  Filter: (amount > '100'::numeric)
  -- 注意：rows=500 來自遠端 pg_stats，而非預設的 1000
```

取捨：

| 面向 | use_remote_estimate = false | use_remote_estimate = true |
|------|-----------------------------|----------------------------|
| 規劃速度 | 快（不使用遠端查詢） | 慢（每次規劃都需查詢遠端） |
| 行數估算 | 不準確（預設 1000 行） | 準確（使用遠端 pg_stats） |
| 適用場景 | 表小、查詢簡單 | 大表、複雜 JOIN |
| 遠端資源 | 不影響 | 輕微增加（每次規劃額外一兩個查詢） |

#### IV. Parallel Foreign Scan（PG 14+）

PG 14 引入 `async_append` 後，你可以在特定條件下實現並行掃描多個 Foreign Table。

啟用條件與步驟：

```sql
-- 1. 確保 Foreign Table 有統計資訊
ANALYZE fdw_tables.users;
ANALYZE fdw_tables.orders;

-- 2. 啟用遠端估算（讓 planner 有足夠資訊選擇並行路徑）
ALTER SERVER remote_pg_server OPTIONS (ADD use_remote_estimate 'true');

-- 3. 設定本地並行參數
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.01;
SET parallel_setup_cost = 100;

-- 4. 驗證：UNION ALL 查詢是否能觸發 async_append
EXPLAIN (VERBOSE)
SELECT * FROM fdw_tables.users
UNION ALL
SELECT * FROM fdw_tables.orders_archive;  -- 假設這指向不同伺服器或不同分區
```

```
Append
  ->  Async Foreign Scan on users
        Remote SQL: SELECT ...
  ->  Async Foreign Scan on orders_archive
        Remote SQL: SELECT ...
```

> **初學者導讀**：`async_append` 和 PG 內建的 `parallel query`（多 worker 掃描同一個表）是不同的概念。`async_append` 是並行掃描**多個不同的 Foreign Table**（可能分佈在不同伺服器上），而 `parallel query` 是多個 worker 並行掃描**同一張表**的不同分頁。兩者在 PG 14+ 可以組合使用。

### 8. App Dev 最佳實踐

#### I. 何時用 FDW？技術選型決策樹

```mermaid
flowchart TD
    A["需要跨資料庫存取資料"] --> B{"是否需要即時資料？"}
    B -->|"是，必須即時"| C{"兩個 DB 是否同為 PostgreSQL？"}
    B -->|"否，延遲可接受"| D["考慮 ETL / CDC<br/>(Debezium, Airbyte, pg_dump)"]

    C -->|"是"| E{"是否需要寫入遠端？"}
    C -->|"否"| F["考慮其他 FDW<br/>(mysql_fdw, tds_fdw, file_fdw)"]

    E -->|"是<br/>(讀寫)"| G{"寫入頻率與交易一致性要求？"}
    E -->|"否<br/>(唯讀)"| H["✅ postgres_fdw<br/>Foreign Table 唯讀查詢"]

    G -->|"高頻 / 強一致性"| I["⚠️ 審慎使用 FDW<br/>考慮使用 2PC<br/>或應用層 Saga 模式"]
    G -->|"中低頻 / 最終一致"| J["✅ postgres_fdw<br/>配合 batch_size 批次寫入"]

    H --> K{"查詢模式？"}
    K -->|"複雜 JOIN / 聚合"| L["⚠️ 測試 SQL Pushdown<br/>必要時在遠端建立 View"]
    K -->|"簡單 CRUD"| M["✅ 直接使用"]

    D --> N{"資料量大且持續增長？"}
    N -->|"是"| O["CDC + Kafka + 串流處理"]
    N -->|"否"| P["定時 pg_dump + pg_restore"]

    I --> Q["或考慮：<br/>- Citus 分佈式<br/>- 應用層雙寫<br/>- 微服務拆分"]
```

#### II. 跨版本相容矩陣表

| 本地 PG | 遠端 PG | postgres_fdw 行為 | 建議 |
|---------|---------|------------------|------|
| 17 | 17 | 完整功能，全 Pushdown 支援 | ✅ 最佳組合 |
| 17 | 16 | 完整功能（PG 17 特性自動降級） | ✅ 相容 |
| 17 | 15 | MERGE pushdown 可用；FULL JOIN pushdown 降級 | ✅ 相容 |
| 17 | 14 | 基本查詢可用；無 MERGE / FULL JOIN pushdown | ⚠️ 審慎 |
| 17 | 13 | 基本查詢可用；無 async_append / batch_size | ⚠️ 不建議 |
| 17 | 12 | 基本查詢可用 | ⚠️ 不建議 |
| 16 | 16 | FULL JOIN 可用 | ✅ |
| 16 | 15 | MERGE pushdown 可用；FULL JOIN 降級 | ✅ |
| 16 | 14 | 基本查詢可用 | ⚠️ 審慎 |
| 15 | 14 | 基本查詢可用 | ⚠️ 審慎 |
| 14 | 14 | async_append / batch_size / parallel_commit 可用 | ✅ |

> **補充（Senior Dev）**：postgres_fdw 保證向前相容（新本地可連舊遠端），但**不保證向後相容**（舊本地連新遠端可能遇到不認識的協議參數）。實務上，建議本地 PG 版本 **≥** 遠端 PG 版本。

#### III. 連接數管理（配合 PgBouncer）

FDW 會從本地 PG 程序向遠端伺服器發起連線。高並發場景下，遠端可能被大量連線淹沒。

#### 問題場景

```
每個本地 backend process → 可能持有 1 個遠端連線
max_connections = 100 → 最多 100 個本地連線 → 最多 100 個遠端連線
```

#### PgBouncer 配置範例

在遠端 PostgreSQL 前方放置 PgBouncer，使用 Transaction Pooling 模式：

```ini
## /etc/pgbouncer/pgbouncer.ini
[databases]
remote_db = host=127.0.0.1 port=5432 dbname=remote_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 500
default_pool_size = 25
reserve_pool_size = 10
reserve_pool_timeout = 5
```

遠端 `userlist.txt`：

```ini
"fdw_user" "secure_pass"
```

本地 Foreign Server 指向 PgBouncer 埠：

```sql
CREATE SERVER remote_pg_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host '192.168.1.100',
        port '6432',  -- PgBouncer port
        dbname 'remote_db'
    );
```

#### 遠端連線參數調校

```sql
-- 限制單一 Foreign Server 的連線數（PG 16+）
ALTER SERVER remote_pg_server OPTIONS (ADD max_connections '10');

-- 設定遠端連線閒置超時
ALTER SERVER remote_pg_server OPTIONS (ADD idle_session_timeout '300000');  -- 5 分鐘
```

#### IV. 安全最佳實踐

#### pg_user_mapping 的密碼風險

`CREATE USER MAPPING` 的 `password` 選項會將密碼以**明文**儲存在 `pg_user_mapping` 系統目錄中：

```sql
-- 任何人只要有權限查詢 pg_user_mapping，就能看到密碼
SELECT * FROM pg_user_mapping;
```

這是一個嚴重的安全隱患。以下是幾種替代方案：

```mermaid
flowchart TD
    A["如何避免明文密碼在 pg_user_mapping？"]
    A --> B["方案 1：peer 認證"]
    A --> C["方案 2：cert 憑證認證"]
    A --> D["方案 3：~/.pgpass 檔案"]
    A --> E["方案 4：password_required=false"]

    B --> B1["本地 PG 以 OS 用戶身份連線<br/>遠端 pg_hba.conf: local all all peer"]
    B1 --> B2["✅ 無密碼<br/>❌ 僅限 local socket 連線<br/>❌ 跨機器不適用"]

    C --> C1["使用 SSL 客戶端憑證<br/>遠端 pg_hba.conf: hostssl all all 0.0.0.0/0 cert"]
    C1 --> C2["✅ 最安全<br/>✅ 跨機器可用<br/>❌ 需維護 CA 與憑證"]

    D --> D1["將密碼寫入 ~/.pgpass<br/>postgres_fdw 自動讀取"]
    D1 --> D2["✅ 簡單易用<br/>⚠️ 密碼仍以明文存於檔案系統<br/>⚠️ 需確保檔案權限 0600"]

    E --> E1["CREATE USER MAPPING<br/>不指定 password<br/>依賴 pg_hba.conf 的 trust/peer/cert"]
    E1 --> E2["✅ 無密碼儲存<br/>❌ 需遠端配置特定認證方式"]
```

#### 方案實作對比

**方案 1：peer 認證**

遠端 `pg_hba.conf`：

```ini
local   remote_db   fdw_user   peer
```

本地：

```sql
CREATE USER MAPPING FOR CURRENT_USER
    SERVER remote_pg_server
    OPTIONS (user 'fdw_user');  -- 不指定 password
```

**方案 2：cert 憑證認證**

遠端 `pg_hba.conf`：

```ini
hostssl remote_db   fdw_user   192.168.1.0/24   cert
```

遠端 `postgresql.conf`：

```ini
ssl = on
ssl_ca_file = '/etc/postgresql/16/main/ca.crt'
```

本地 Foreign Server 設定：

```sql
CREATE SERVER remote_pg_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host '192.168.1.100',
        port '5432',
        dbname 'remote_db',
        sslmode 'verify-ca',
        sslrootcert '/etc/postgresql/16/main/ca.crt',
        sslcert    '/etc/postgresql/16/main/client.crt',
        sslkey     '/etc/postgresql/16/main/client.key'
    );

CREATE USER MAPPING FOR CURRENT_USER
    SERVER remote_pg_server
    OPTIONS (user 'fdw_user');  -- 不指定 password，由 cert 認證
```

**方案 3：~/.pgpass**

```bash
## 在執行 PostgreSQL 服務的 OS 用戶的 home 目錄下
## ~postgres/.pgpass
echo "192.168.1.100:5432:remote_db:fdw_user:secure_pass" > ~/.pgpass
chmod 0600 ~/.pgpass
```

```sql
-- 不指定 password，postgres_fdw 自動從 .pgpass 讀取
CREATE USER MAPPING FOR CURRENT_USER
    SERVER remote_pg_server
    OPTIONS (user 'fdw_user');
```

> **初學者導讀**：`~/.pgpass` 是最簡單的替代方案，但請務必設定檔案權限為 `0600`（僅擁有者可讀寫）。在生產環境中，強烈建議使用 cert 憑證認證，這是零密碼儲存的最安全方案。

#### 最小權限原則

遠端資料庫使用者應僅被授予必要的權限：

```sql
-- 遠端：最小權限
GRANT USAGE ON SCHEMA remote_app TO fdw_user;
GRANT SELECT ON users, orders TO fdw_user;       -- 唯讀
-- GRANT INSERT, UPDATE, DELETE ON orders TO fdw_user;  -- 若需寫入
REVOKE ALL ON DATABASE remote_db FROM PUBLIC;    -- 收緊公共權限
```
## 二、pg_partman — 原生分區自動化管理

> **章節導讀**：PostgreSQL 自 10 版引入宣告式分區（Declarative Partitioning）後，歷經多個大版本打磨，原生分區能力已十分成熟。然而原生分區僅提供 DDL 手動管理（`CREATE/ATTACH/DETACH PARTITION`），缺乏自動化的分區生命週期管理——建立未來分區、清理過期分區、狀態監控等，這些正是 pg_partman 的核心價值。

---

### 1. PostgreSQL 原生分區演進（PG 10 → 17）

PostgreSQL 的分區機制從 9.x 時代的 `table inheritance + trigger/rule` 手動路由，進化到現在宣告式分區，每一步都帶來了關鍵能力。

```mermaid
timeline
    title PostgreSQL 分區功能演進
    PG 10 : 宣告式分區誕生 : RANGE / LIST : ATTACH / DETACH
    PG 11 : HASH 分區 : DEFAULT 分區 : UPDATE 跨分區遷移 : Executor 階段 partition pruning 增強
    PG 12 : Partition Pruning 大幅提升 : RuntimeAppend 更多場景 : 外部鍵參照分區表
    PG 13 : 分區表邏輯複製 : BEFORE row-level trigger
    PG 14-15 : 持續效能改進 : 更多分區操作並發優化
    PG 16 : 更多 JOIN 類型支持分區裁剪 : 增量排序支持
    PG 17 : MERGE on partitioned tables
```

#### I. PG 10 — Declarative Partitioning 誕生

> **初學者導讀**
>
> 在 PG 10 之前，分區只能透過「繼承（Inheritance）+ trigger/rule」模擬，開發者需要手寫路由邏輯，容易出錯且 planner 難以最佳化。PG 10 的宣告式分區讓 DBA 宣告分區策略，PostgreSQL 自動處理行路由。

核心語法：

```sql
-- 宣告父表，指定分區鍵和分區策略
CREATE TABLE orders (
    id          bigserial,
    user_id     bigint,
    amount      numeric(12,2),
    created_at  timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- 建立子分區
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

`ATTACH / DETACH PARTITION` 使得線上掛載/解除分區成為可能，無需長時間鎖表：

```sql
-- 先建立普通表，再掛載為分區
CREATE TABLE orders_2026_03 (LIKE orders INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
-- 若有資料需要載入...
ALTER TABLE orders ATTACH PARTITION orders_2026_03
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

#### II. PG 11 — Hash 分區 + Default 分區 + UPDATE 跨分區

PG 11 三大關鍵強化：

**Hash 分區**：適用於無明顯範圍/列表語義的均勻分布鍵，將資料均勻分配到各分區。

```sql
CREATE TABLE events PARTITION BY HASH (user_id) PARTITIONS 8;
```

**Default 分區**：捕獲不匹配任何現有分區的行，避免插入失敗，同時提示 DBA 需擴充分區。

```sql
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

**UPDATE 跨分區遷移**：當 UPDATE 修改分區鍵值導致行應歸屬另一分區時，PG 11 會自動將行從舊分區遷移到正確分區（內部為 DELETE + INSERT）。這解決了 PG 10 中跨分區 UPDATE 報錯的問題。

#### III. PG 12 — Partition Pruning 大幅提升

PG 12 在分區裁剪方面做了重大重構，`RuntimeAppend` 在更多場景替代靜態 `Append`，執行階段根據參數值動態跳過不相關分區。同時分區表開始支持外部鍵參照（foreign key references）。

```sql
-- PG 12 起支持外部鍵指向分區表
ALTER TABLE order_items ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id);
```

#### IV. PG 13 — 分區表邏輯複製

PG 13 讓邏輯複製支持分區表，可將整個分區表（含所有子分區）以 single publication 形式發布。同時支持 `BEFORE row-level trigger` 在分區表上。

```sql
CREATE PUBLICATION orders_pub FOR TABLE orders;
-- 所有現有及未來子分區自動包含在內
```

#### V. PG 14-15 — 持續效能改進

這兩個版本聚焦於執行引擎最佳化：分區操作期間的並發效能、`ALTER TABLE ... DETACH PARTITION CONCURRENTLY` 等非阻塞操作，以及更多查詢節點對 partition pruning 的支持。

#### VI. PG 16 — 更多 JOIN 類型支持分區裁剪

PG 16 對分區裁剪的支持拓展到更多 JOIN 類型，使涉及多表的複雜查詢也能有效跳過不相關分區。增量排序（Incremental Sort）節點現在也能參與 partition pruning。

```sql
-- PG 16: JOIN 查詢中更精確的分區裁剪
EXPLAIN SELECT *
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.created_at >= '2026-03-01' AND o.created_at < '2026-03-15';
-- 預期輸出中 Sub-plans Removed 數量顯著增加
```

#### VII. PG 17 — MERGE on partitioned tables

PG 17 讓 `MERGE` 語句支持分區表，可直接對分區表執行 upsert 邏輯，內部正確路由到各子分區。

```sql
-- PG 17: MERGE 直接作用於分區表
MERGE INTO orders o
USING new_orders n ON o.id = n.id
WHEN MATCHED THEN UPDATE SET amount = n.amount
WHEN NOT MATCHED THEN INSERT (user_id, amount, created_at)
     VALUES (n.user_id, n.amount, n.created_at);
```

> **補充（Senior Dev）**
>
> PG 17 也增強了 `EXPLAIN` 對分區表的輸出：可在 `Sub-plans Removed` 中看到更精確的分區裁剪數量。結合 `pg_stat_user_tables` 可以用以下查詢監控每個分區的讀寫負載：
>
> ```sql
> SELECT schemaname, relname, n_tup_ins, n_tup_upd, seq_scan, idx_scan
> FROM pg_stat_user_tables
> WHERE relname LIKE 'orders%'
> ORDER BY relname;
> ```

---

### 2. pg_partman 核心概念

#### I. 為什麼需要 pg_partman

原生分區的 DDL 需要由「人」或「腳本」手動觸發。實際場景中：

- 每天需要為日誌表建立新的日分區——忘記就會導致插入失敗或落入 DEFAULT 分區
- 90 天前的舊資料需要清理——手動 DROP/DETACH 容易誤操作
- 團隊需要統一的 policy：預建多少分區、保留多久、清理方式是 DETACH 還是 DROP

pg_partman 將這些操作標準化、自動化，由 Background Worker 定時執行維護。

> **初學者導讀**
>
> 你可以這樣理解：原生分區給了你「汽車零件」（CREATE/ATTACH/DETACH PARTITION），而 pg_partman 是「自動駕駛系統」——你只需告訴它策略（按天分區、保留 90 天、預建 7 天），它自動幫你做。

#### II. 架構設計

```mermaid
graph TD
    A[pg_partman_bgw<br/>Background Worker] -->|定時調用| B[run_maintenance]
    B -->|讀取配置| C[(part_config<br/>父表級配置)]
    B -->|讀取 / 寫入| D[(part_config_sub<br/>子分區狀態)]
    B -->|自動建立| E[未來分區<br/>CREATE TABLE + ATTACH]
    B -->|自動清理| F[過期分區<br/>DETACH / DROP]

    G[DBA / App Dev] -->|CREATE EXTENSION| H[pg_partman 擴展]
    G -->|create_parent| C
    G -->|手動維護| B

    style A fill:#4a90d9,color:white
    style C fill:#e8d44d,color:black
    style D fill:#e8d44d,color:black
    style E fill:#5cb85c,color:white
    style F fill:#d9534f,color:white
```

核心組件：

| 組件 | 說明 |
|------|------|
| `part_config` | 父表級配置表，記錄每個分區父表的策略：分區間隔、預建數量、保留策略等 |
| `part_config_sub` | 子分區狀態表，記錄每個子分區的建立時間、是否需要維護等 |
| `pg_partman_bgw` | Background Worker 程序，定時調用 `run_maintenance()` |
| `run_maintenance()` | 核心函數，檢查所有父表，自動建立新分區、清理過期分區 |

pg_partman 支持兩種類型的分區：

- **native**：使用 PostgreSQL 原生宣告式分區（推薦，PG 10+）
- **partman**：使用 pg_partman 自有的 inheritance-based 實現（向後相容，不推薦新專案使用）

本書全部示例使用 **native** 模式。

---

### 3. 安裝與配置（PG 16）

#### 套件安裝

```bash
## Debian / Ubuntu
sudo apt install postgresql-16-partman

## RHEL / Rocky / AlmaLinux
sudo dnf install pg_partman_16
```

#### PostgreSQL 配置（postgresql.conf）

```ini
## 載入 Background Worker 庫
shared_preload_libraries = 'pg_partman_bgw'

## BGW 執行間隔（秒），建議 3600 = 每小時執行一次
partman_bgw.interval = 3600

## BGW 管理的目標資料庫
partman_bgw.dbname = 'yourdb'

## BGW 執行時使用的角色（需具備分區表管理權限）
partman_bgw.role = 'partman_owner'

## 可選：指定 BGW 維護哪些 schema，逗號分隔
## partman_bgw.schema = 'public'

## 可選：維護後自動 ANALYZE
partman_bgw.analyze = 'on'

## 可選：如需整合 pg_jobmon 監控
## partman_bgw.jobmon = 'on'
```

重啟 PostgreSQL 後，檢查 BGW 是否正常載入：

```sql
-- 確認 BGW 已載入
SELECT * FROM pg_stat_activity WHERE backend_type = 'pg_partman_bgw';
```

#### 建立擴展

```sql
-- 在目標資料庫中執行
CREATE SCHEMA partman;
CREATE EXTENSION pg_partman WITH SCHEMA partman;

-- 建立 partman_owner 角色（如尚未建立）
CREATE ROLE partman_owner WITH LOGIN;
GRANT ALL ON SCHEMA partman TO partman_owner;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO partman_owner;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA partman TO partman_owner;
```

> **補充（Senior Dev）**
>
> 推薦將 pg_partman 安裝到獨立 schema（如 `partman`），避免汙染 `public` schema。如果你使用 `pg_cron` 作為備用排程方案（而非 BGW），可省略 `shared_preload_libraries` 和 BGW 配置項，改用：
>
> ```sql
> SELECT cron.schedule('partman-maintenance', '*/15 * * * *',
>     $$SELECT partman.run_maintenance()$$);
> ```
>
> 兩種方案可以共存：BGW 做主排程，pg_cron 做備援，確保維護不會遺漏。

---

### 4. 核心操作

```mermaid
sequenceDiagram
    participant DBA as DBA / App Dev
    participant Parent as Orders 父表
    participant PM as pg_partman
    participant Child as 子分區
    participant Future as 未來分區

    DBA->>PM: create_parent(p_control, p_interval, p_premake, p_retention)
    PM->>Parent: 建立宣告式分區父表
    PM->>Future: 根據 p_premake 預建未來分區
    Note over PM: 設定完成，等待 BGW 或手動維護
    loop 定時維護 (BGW / pg_cron)
        PM->>PM: run_maintenance()
        PM->>PM: 檢查 part_config
        alt 需要新建分區
            PM->>Future: CREATE TABLE + ATTACH PARTITION
        end
        alt 有過期分區
            PM->>Child: DETACH 或 DROP（按 retention policy）
        end
    end

    DBA->>PM: undo_partition()
    PM->>Parent: 將子分區資料合併回父表
    PM->>Parent: ALTER TABLE ... DETACH 所有子分區
    Note over Parent: 還原為普通表
```

#### I. `create_parent()` 建立分區父表

`create_parent()` 是 pg_partman 最核心的函數。參數說明：

```sql
SELECT partman.create_parent(
    p_parent_table        := 'public.orders',
    p_control             := 'created_at',
    p_type                := 'native',
    p_interval            := '1 day',
    p_premake             := 7,
    p_start_partition     := '2026-01-01',
    p_retention           := '90 days',
    p_retention_keep_table := false
);
```

| 參數 | 說明 |
|------|------|
| `p_parent_table` | 父表名稱，格式 `schema.table` |
| `p_control` | 分區鍵欄位（須為時間型別或整數型別） |
| `p_type` | 分區類型：`native`（推薦）、`partman`（向後相容）、`id-{type}`（ID 範圍分區） |
| `p_interval` | 分區間隔，如 `1 day`、`1 week`、`1 month`、`1 hour` |
| `p_premake` | 預建的未來分區數量（確保始終有足夠的未來分區） |
| `p_start_partition` | 第一個分區的起始時間（選填，若無則自動推算） |
| `p_retention` | 資料保留時長，超時的舊分區會被清理 |
| `p_retention_keep_table` | 是否保留過期分區表（true = DETACH 不 DROP，false = DROP） |

`p_interval` 支援的值：

| 值 | 說明 | 分區命名後綴 |
|----|------|-------------|
| `1 hour` | 每小時一個分區 | `_p2026-01-01_00` |
| `1 day` | 每天一個分區 | `_p2026-01-01` |
| `1 week` | 每週一個分區 | `_p2026w01` |
| `1 month` | 每月一個分區 | `_p2026-01` |
| `1 year` | 每年一個分區 | `_p2026` |
| `custom`（整數） | 自訂數值間隔 | — |

**RANGE vs LIST 示例**：

```sql
-- RANGE：按時間範圍分區（pg_partman 最擅長的場景）
CREATE TABLE order_items (
    id          bigserial,
    order_id    bigint NOT NULL,
    product_id  bigint NOT NULL,
    quantity    int NOT NULL,
    price       numeric(10,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 交由 pg_partman 管理
SELECT partman.create_parent(
    p_parent_table := 'public.order_items',
    p_control      := 'created_at',
    p_type         := 'native',
    p_interval     := '1 day',
    p_premake      := 7
);

-- LIST：按區域分區（pg_partman 支援有限，主要優勢在 RANGE 時間分區）
CREATE TABLE events_by_region (
    id         bigint GENERATED ALWAYS AS IDENTITY,
    region     text NOT NULL,
    event_data jsonb,
    created_at timestamptz DEFAULT now()
) PARTITION BY LIST (region);

-- LIST 分區建議手動管理，pg_partman 專注於時間維度 RANGE 分區
```

> **補充（Senior Dev）**
>
> pg_partman 的核心價值在於 **時間基礎的 RANGE 分區**自動管理。對於 LIST / HASH 分區，pg_partman 的功能有限，因為 LIST 值集合通常是固定的、不易預測。對於需要按 category/region 等維度分區的場景，建議用原生 DDL 手動管理 LIST 分區，用 pg_partman 專注管理時間維度分區。

#### II. 自動分區擴展（run_maintenance）

`run_maintenance()` 會為每個啟用的父表執行以下邏輯：

1. 查詢 `part_config_sub`，找到最後一個子分區
2. 計算 `p_premake` 要求的未來分區數量
3. 檢查當前時間點後是否已有足夠的未來分區
4. 若不足，`CREATE TABLE ... (LIKE parent) ...` + `ALTER TABLE parent ATTACH PARTITION`

```sql
-- 手動觸發維護（通常由 BGW 自動執行）
SELECT partman.run_maintenance();

-- 只對特定父表執行維護
SELECT partman.run_maintenance('public.orders');

-- 預建未來 14 個分區（覆蓋 p_premake 設定，一次性操作）
UPDATE partman.part_config
SET premake = 14
WHERE parent_table = 'public.orders';

SELECT partman.run_maintenance('public.orders');
```

**調整 p_premake**：`p_premake` 決定始終預留多少個未來分區。例如設定為 7 且 `p_interval = '1 day'`，則確保未來 7 天都有對應分區。建議根據業務資料寫入的「提前量」來調整——如果資料可能提前一週寫入，p_premake 應至少設為 7。

#### III. Retention Policy 自動清理

pg_partman 支援兩種清理策略：

```sql
-- 策略 A：過期即 DROP（不可恢復，節省空間）
SELECT partman.create_parent(
    p_parent_table        := 'public.orders',
    p_control             := 'created_at',
    p_interval            := '1 day',
    p_retention           := '90 days',
    p_retention_keep_table := false       -- false = DROP
);

-- 策略 B：過期僅 DETACH（保留表供歷史查詢 / 備份後手動處理）
SELECT partman.create_parent(
    p_parent_table        := 'public.orders',
    p_control             := 'created_at',
    p_interval            := '1 day',
    p_retention           := '90 days',
    p_retention_keep_table := true         -- true = DETACH 但不 DROP
);

-- 在建立 parent 之後調整 retention
UPDATE partman.part_config
SET retention = '180 days',
    retention_keep_table = false
WHERE parent_table = 'public.orders';
```

`p_retention_keep_table = true` 時，過期分區被 `DETACH` 為獨立普通表，命名為 `orders_p2025-01-01` 的形式，方便做 `pg_dump` / 歸檔 / 手動審查後再刪除。

**策略選擇建議**：

| 場景 | 推薦策略 | `retention_keep_table` |
|------|---------|------------------------|
| 純日誌表，資料可丟 | DROP | `false` |
| 歷史資料需歸檔到冷儲存再刪 | DETACH | `true` |
| 合規要求保留副本後方可清理 | DETACH | `true` |
| 開發/測試環境 | DROP | `false` |

#### IV. Undo Partitioning

將分區父表還原為普通表：

```sql
-- p_keep_table: 是否保留父表（true = 父表變為普通表，資料合併回來）
SELECT partman.undo_partition(
    p_parent_table := 'public.orders',
    p_keep_table   := true
    -- true: 資料合併回來，父表變為普通表
    -- false: drop 父表全部資料，不保留
);
```

`undo_partition()` 的執行步驟：

1. 將所有子分區 DETACH
2. 如果 `p_keep_table = true`，將所有子分區資料 INSERT 回父表（觸發順序掃描所有子分區）
3. 刪除 `part_config` 和 `part_config_sub` 中的記錄

> **補充（Senior Dev）**
>
> `undo_partition()` 在資料量大時可能耗時很長（需 INSERT ALL rows back to parent）。在執行前建議：
> 1. 在維護窗口內執行
> 2. 先確認 `pg_stat_user_tables` 中各子分區的 `n_live_tup`，估算總行數
> 3. 若只為遷移到不同的分區策略（如從按天改為按月），可考慮直接 `DETACH` 舊子分區 + 建立新分區父表 + 將舊資料批次載入新結構

#### V. 手動管理 API

pg_partman 提供了一系列實用查詢和管理函數：

```sql
-- 查看父表配置
SELECT parent_table, partition_type, control, partition_interval,
       premake, retention, retention_keep_table
FROM partman.part_config;

-- 查看所有子分區狀態
SELECT sub_parent, sub_partition, sub_partition_type,
       has_default, is_managed, last_analyzed
FROM partman.part_config_sub
WHERE sub_parent = 'public.orders'
ORDER BY sub_partition;

-- 手動建立特定時間段的分區
SELECT partman.create_partition_time(
    p_parent_table    := 'public.orders',
    p_partition_times := ARRAY['2026-04-01'::timestamptz]
);

-- 檢查父表配置一致性
SELECT partman.check_parent('public.orders');

-- 顯示分區列表（含分區範圍）
SELECT partman.show_partitions('public.orders');
```

`part_config_sub` 查詢中 `is_managed = false` 表示該子分區不由 pg_partman 管理（手動建立或已 DETACH），應定期審查此類分區。

---

### 5. 完整示例：訂單表按天分區（PG 16）

#### Step 1：建立分區父表並配置 pg_partman

```sql
-- 建立 orders 主表（分區鍵：created_at）
CREATE TABLE orders (
    id          bigserial,
    user_id     bigint NOT NULL,
    amount      numeric(12,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 建立 order_items 子表（關聯到 orders）
CREATE TABLE order_items (
    id          bigserial,
    order_id    bigint NOT NULL,
    product_id  bigint NOT NULL,
    quantity    int NOT NULL,
    price       numeric(10,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 使用 pg_partman 管理 orders
SELECT partman.create_parent(
    p_parent_table   := 'public.orders',
    p_control        := 'created_at',
    p_type           := 'native',
    p_interval       := '1 day',
    p_premake        := 7,
    p_start_partition := '2026-01-01',
    p_retention      := '90 days',
    p_retention_keep_table := false
);

-- 使用 pg_partman 管理 order_items
SELECT partman.create_parent(
    p_parent_table   := 'public.order_items',
    p_control        := 'created_at',
    p_type           := 'native',
    p_interval       := '1 day',
    p_premake        := 7,
    p_start_partition := '2026-01-01',
    p_retention      := '90 days',
    p_retention_keep_table := false
);
```

#### Step 2：插入測試資料（跨多天）

```sql
-- 插入 orders 測試資料（約 2000 行，分佈在 1 月初到 3 月末）
INSERT INTO orders (user_id, amount, created_at)
SELECT
    (random() * 10000)::bigint,
    (random() * 5000)::numeric(12,2),
    '2026-01-01'::timestamptz
        + (gs || ' hours')::interval
FROM generate_series(0, 2000) AS gs;

-- 插入 order_items 測試資料
INSERT INTO order_items (order_id, product_id, quantity, price, created_at)
SELECT
    (random() * 500)::bigint,
    (random() * 100)::bigint,
    (random() * 10 + 1)::int,
    (random() * 200)::numeric(10,2),
    '2026-01-01'::timestamptz
        + (gs || ' hours')::interval
FROM generate_series(0, 2000) AS gs;
```

#### Step 3：驗證分區結構

```sql
-- 查看所有子分區
SELECT partman.show_partitions('public.orders');

-- 查看每個分區的行數量
SELECT
    schemaname || '.' || relname AS partition_name,
    n_live_tup AS row_count,
    n_dead_tup AS dead_rows,
    last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname LIKE 'orders_p%'
ORDER BY relname;

-- 查看分區配置
SELECT parent_table, partition_interval, premake, retention, retention_keep_table
FROM partman.part_config
WHERE parent_table = 'public.orders';
```

#### Step 4：EXPLAIN 展示 Partition Pruning

```sql
-- 查詢單一天 — 應只掃描 1 個分區
EXPLAIN (COSTS OFF)
SELECT * FROM orders
WHERE created_at >= '2026-01-15'
  AND created_at <  '2026-01-16';

-- 預期輸出：
-- Append
--   Subplans Removed: N
--   ->  Seq Scan on orders_p2026_01_15
--         Filter: ((created_at >= '2026-01-15 00:00:00+00'::timestamp with time zone)
--              AND (created_at < '2026-01-16 00:00:00+00'::timestamp with time zone))

-- 跨多天查詢 — 應僅掃描相關分區
EXPLAIN (COSTS OFF)
SELECT * FROM orders
WHERE created_at BETWEEN '2026-01-15' AND '2026-01-20';

-- 查詢近期資料（非分區鍵的過濾條件） — 仍需掃描所有分區
EXPLAIN (COSTS OFF)
SELECT * FROM orders
WHERE user_id = 42 AND amount > 1000;
-- 預期：掃描所有包含資料的分區，因為 user_id 不是分區鍵
```

#### Step 5：手動觸發維護並檢查結果

```sql
-- 手動觸發維護（確認 BGW 排程前的驗證）
SELECT partman.run_maintenance('public.orders');

-- 確認維護後分區狀態
SELECT sub_partition, is_managed, last_analyzed
FROM partman.part_config_sub
WHERE sub_parent = 'public.orders'
ORDER BY sub_partition DESC
LIMIT 10;
```

---

### 6. 分區查詢效能

#### I. Partition Pruning 原理

Partition Pruning（分區裁剪）是分區表效能的基石。PostgreSQL 在兩階段實現裁剪：

```mermaid
flowchart TD
    Q[查詢]
    Q --> P["Planner (規劃階段)"]
    P --> CE{"Constraint Exclusion<br/>條件可靜態確定？"}
    CE -->|是| PR1["排除不相關分區<br/>(Planner Pruning)"]
    CE -->|否| PR2["保留所有分區<br/>標記為 RuntimeAppend"]
    PR1 --> E1["Executor 階段<br/>只掃描相關分區"]
    PR2 --> E2["Executor 階段<br/>RuntimeAppend 動態裁剪"]
    E1 --> R[結果]
    E2 --> R

    style CE fill:#e8d44d,color:black
    style PR1 fill:#5cb85c,color:white
    style PR2 fill:#f0ad4e,color:white
```

**兩種 Pruning 模式**：

| 模式 | 說明 | 發生階段 | 典型場景 |
|------|------|---------|---------|
| Constraint Exclusion（Planner 階段） | Planner 根據 `WHERE` 條件中的常量，在計畫生成時就排除不相關分區 | 規劃階段 | `WHERE created_at = '2026-01-15'` |
| Runtime Pruning（Executor 階段） | 當 `WHERE` 條件依賴於執行時才能確定的值（如子查詢、參數化查詢）時，Executor 在執行時動態跳過分區 | 執行階段 | `WHERE created_at IN (SELECT ...)` 或 prepared statement with parameters |

PG 12+ 大幅擴展了 Runtime Pruning 的場景，PG 16 進一步支援更多 JOIN 類型中的分區裁剪。

#### II. EXPLAIN 解讀

```sql
-- 範例 1：單一分區查詢（最佳裁剪）
EXPLAIN (ANALYZE, COSTS, BUFFERS, SUMMARY)
SELECT COUNT(*) FROM orders
WHERE created_at >= '2026-01-15' AND created_at < '2026-01-16';

-- 預期輸出關鍵欄位：
--   ->  Seq Scan on orders_p2026_01_15  (actual time=...)
--          Filter: ((created_at >= '2026-01-15') AND (created_at < '2026-01-16'))
--  Planner Time: xx ms
--  Execution Time: xx ms
-- 說明：只有 1 個分區被掃描

-- 範例 2：跨分區查詢
EXPLAIN (ANALYZE, COSTS, BUFFERS, SUMMARY)
SELECT COUNT(*) FROM orders
WHERE created_at BETWEEN '2026-01-10' AND '2026-01-20';

-- 預期輸出：
--  Append  (actual time=...)
--    Subplans Removed: N
--    ->  Seq Scan on orders_p2026_01_10
--    ->  Seq Scan on orders_p2026_01_11
--    ->  ...
-- 說明：Subplans Removed 顯示被裁剪掉的分區數量

-- 範例 3：不使用分區鍵的查詢（全分區掃描）
EXPLAIN (ANALYZE, COSTS, BUFFERS, SUMMARY)
SELECT COUNT(*) FROM orders
WHERE user_id = 42;
-- 預期輸出：顯示所有包含資料的分區都被掃描
--          Subplans Removed: 0
--          Planning Time 較高（因需評估所有分區）
```

**EXPLAIN 輸出關鍵元素解讀**：

| 元素 | 含義 |
|------|------|
| `Append` | 合併多個子分區的掃描結果 |
| `Subplans Removed: N` | 被裁剪掉的分區數量，數值越大裁剪效果越好 |
| `Seq Scan on orders_pYYYY_MM_DD` | 實際被掃描的子分區 |
| `Planning Time` | 規劃時間，分區越多規劃時間越長 |
| `RuntimeAppend` | 執行階段動態裁剪（PG 12+） |

> **補充（Senior Dev）**
>
> 當分區數量達到數千個時，Planning Time 可能成為瓶頸（Planner 需要評估每個分區的約束條件）。緩解方法：
> 1. 使用 `p_retention` 定期清理舊分區，保持總分區數量在可控範圍
> 2. 對於極大規模（萬級分區），考慮切換為更大的 `p_interval`（如從按天改為按週）
> 3. 使用 PG 17 的 `enable_partition_pruning` 相關 GUC 參數微調裁剪行為

---

### 7. App Dev 最佳實踐

#### I. 分區鍵選擇 — 對齊查詢 WHERE/JOIN 模式

分區鍵的選擇直接決定 partition pruning 的效果。選擇原則：

```mermaid
flowchart TD
    Q1{"表中哪個欄位<br/>在 WHERE/JOIN 中<br/>出現頻率最高？"}
    Q1 -->|"時間欄位<br/>(created_at, event_date)"| Q2["該欄位是否<br/>幾乎所有查詢都用？"]
    Q2 -->|是| R1["✅ 選為分區鍵<br/>RANGE BY 此時間欄位"]
    Q2 -->|否，還有其他高頻欄位| Q3["是否有明確<br/>分組語義的欄位？"]

    Q1 -->|"業務 ID<br/>(customer_id, tenant_id)"| Q4["查詢是否總是<br/>包含此 ID？"]
    Q4 -->|是| Q5["資料量級？"]
    Q5 -->|百萬以下| R2["HASH BY tenant_id<br/>（均勻分布）"]
    Q5 -->|千萬以上| R3["RANGE BY timestamp<br/>+ 聯合查詢 tenant_id index"]

    Q1 -->|"無明顯模式"| R4["不建議分區<br/>使用 Index 即可"]

    style Q1 fill:#4a90d9,color:white
    style R1 fill:#5cb85c,color:white
    style R2 fill:#5cb85c,color:white
    style R3 fill:#f0ad4e,color:black
    style R4 fill:#d9534f,color:white
```

**黃金法則**：

- 分區鍵應是 **幾乎所有查詢 WHERE 條件中都會出現** 的欄位
- 最理想的場景是「按時間分區 + 按時間查詢」，例如日誌表、事件表、交易表
- 避免將「偶爾出現」的欄位作為分區鍵——因為不走分區鍵的查詢必須掃描所有分區

#### II. 控制分區總數上限

當分區數量過多時（數千乃至上萬），Planning Time 可能成為主要瓶頸。Planner 需要為每個分區評估約束條件和成本。

```sql
-- 監控當前分區數量
SELECT COUNT(*) AS partition_count
FROM partman.part_config_sub
WHERE sub_parent = 'public.orders';

-- 若分區數量過大，考慮：
-- 1. 增大 p_interval（從按天改為按週或按月）
-- 2. 縮短 retention period（減少保留天數）
-- 3. 手動合併舊分區
```

**建議上限**（參考值）：

| 查詢模式 | 建議最大分區數 | 原因 |
|---------|-------------|------|
| 僅單分區查詢（精準時間範圍） | 數千～萬級 | 裁剪效果最佳，Planning Time 可控 |
| 偶有跨分區查詢 | 數百～千級 | 跨分區查詢規劃成本隨分區數線性增長 |
| 頻繁跨分區 JOIN | 數百級 | JOIN 的規劃成本更高 |
| Prepared Statement + 動態時間 | 千級 | 使用 RuntimeAppend，規劃成本相對固定 |

#### III. 與 pg_cron 結合確保維護不遺漏

雖然 pg_partman BGW 足夠可靠，但在生產環境中建議雙保險：

```sql
-- 使用 pg_cron 作為備援維護排程（每 30 分鐘）
CREATE EXTENSION IF NOT EXISTS pg_cron;

SELECT cron.schedule(
    'partman-fallback',
    '*/30 * * * *',                    -- 每 30 分鐘執行
    $$SELECT partman.run_maintenance()$$
);

-- 查看排程狀態
SELECT jobid, schedule, command, lastrun, nextrun
FROM cron.job
WHERE jobname = 'partman-fallback';
```

#### IV. 監控 part_config_sub 確保維護無失敗

```sql
-- 定期審查：是否有未管理或被遺忘的子分區
SELECT sub_parent, sub_partition, is_managed, last_analyzed
FROM partman.part_config_sub
WHERE NOT is_managed                    -- 非管理中的分區
   OR last_analyzed IS NULL             -- 未分析過
   OR last_analyzed < now() - '7 days'::interval; -- 長期未分析

-- 檢查 DEFAULT 分區（提醒應擴充分區）
SELECT sub_parent, sub_partition
FROM partman.part_config_sub
WHERE has_default = true;
```

> **補充（Senior Dev）**
>
> 在生產環境中，建議建立一個簡單的監控查詢排程（配合 pg_cron 或外部監控系統），定期檢查：
> 1. `has_default = true` 的分區——若 DEFAULT 分區有資料，表示有行未能匹配到正常分區，需立即排查
> 2. `run_maintenance()` 的執行時間是否異常增長（可能暗示分區數量過多）
> 3. 最近一個未來分區的結束時間與 `now()` 的差距——若小於 `p_interval * p_premake`，表示預建分區不足

#### V. 索引策略

pg_partman 管理的分區表，索引策略應遵循以下原則：

```sql
-- ✅ 正確做法：在父表上建立索引，自動繼承到子分區
-- 建立分區父表時定義索引
CREATE TABLE orders (
    id          bigserial,
    user_id     bigint NOT NULL,
    amount      numeric(12,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 分區鍵上的索引（通常已由 PK 覆蓋）
-- 非分區鍵但常用於過濾的欄位也需索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_amount ON orders(amount)
    WHERE amount > 1000;       -- partial index 也支援

-- 交由 pg_partman 管理，未來新建的子分區會自動繼承這些索引
SELECT partman.create_parent(
    p_parent_table := 'public.orders',
    p_control      := 'created_at',
    p_interval     := '1 day',
    p_premake      := 7
);

-- ✅ 手動建立的子分區也會自動繼承父表索引
-- ❌ 錯誤做法：只在子分區上建立索引（未來分區不會自動獲得）
```

**索引策略清單**：

| 索引類型 | 建立時機 | 注意事項 |
|---------|---------|---------|
| PK（含分區鍵） | 建立父表時 | PK 必須包含分區鍵（PG 11+） |
| 分區鍵單獨索引 | 通常由 PK 覆蓋 | 若有範圍查詢可能需要獨立索引 |
| 高頻過濾欄位 | 建立父表時 | 自動繼承到所有子分區 |
| Partial Index | 建立父表時 | 每個子分區獨立評估 WHERE 條件 |
| Unique Index（不含分區鍵） | 不可建立 | PG 無法在全域唯一保證不含分區鍵的約束 |

---

> **補充（Senior Dev）**
>
> 關於 **non-partition-key unique constraint** 的限制：如果你需要對非分區鍵欄位（如 `order_id`）保證唯一性，有以下替代方案：
> 1. 將該欄位加入 PK 定義（`PRIMARY KEY (id, order_id, created_at)`）
> 2. 使用外部唯一性服務（如 Redis SETNX）在應用層保證
> 3. 建立一個單獨的非分區表作為「唯一性登錄表」，插入時先檢查
> 4. 評估是否可將該欄位作為 HASH 分區鍵
>
> 這是宣告式分區的結構性限制，與 pg_partman 無關。
## 三、Citus 12 — 分散式 SQL 引擎

Citus 12 是 PostgreSQL 生態系中最成熟的原生分散式 SQL 引擎，它以 Extension 形式嵌入 PG 16，將標準 PostgreSQL 轉變為水平可擴展的分散式資料庫集群。本章涵蓋 Citus 12 架構、安裝配置、核心操作、分散式查詢策略、Rebalancing、高可用與 App Dev 最佳實踐。

> **初學者導讀**
> 你可以把 Citus 想像成「PostgreSQL 的分身術」——它把一個巨大的資料表自動拆成許多小碎片（Shard），分散到多台伺服器上並行處理。對應用程式來說，你仍然寫標準的 `SELECT * FROM orders WHERE ...`，Citus 在背後幫你把查詢分發到多台機器、合併結果再回傳。這讓你可以在不改變 SQL 的前提下，把資料庫從單機擴展到集群。

---

### 1. Citus 架構與演進

#### I. 從 pg_shard 到 Citus 12

Citus 的前身 `pg_shard` 於 2015 年問世，是 PostgreSQL 生態中第一個真正意義上的分散式 Sharding Extension（僅支援 Hash Sharding + 基本的 `INSERT`/`SELECT`）。2016 年 Citus 公司開源了重新設計的 Citus 6，自此進入高速演進：

| 版本 | 年份 | 里程碑 |
|------|------|--------|
| pg_shard | 2015 | 首次實現 PG Hash Sharding |
| Citus 6 | 2016 | 開源：Distributed Tables、Reference Tables、Co-Location |
| Citus 11 | 2022 | Query from Any Node（任意節點皆可接受查詢）、非阻塞 Rebalancing |
| Citus 12 | 2024 | **Schema-Based Sharding**：以 Schema 為 Sharding 單位，完美適配 SaaS 多租戶場景 |

> **補充（Senior Dev）**
> pg_shard 使用 Foreign Data Wrapper（FDW）機制實現跨節點查詢，但其 `Custom Scan` API 受限於 PG 9.x 的 Executor Hook。Citus 6 重寫後使用 `Planner Hook` + `Executor Hook` 深度整合，直接介入 Query Planning 和 Execution 階段，這也是為什麼 Citus 能實現 Co-Located JOIN（零網路傳輸）和 Distributed Subquery Pushdown 的根本原因。如果你從 pg_shard 遷移，需注意 pg_shard 的 `pgs_distribution_metadata` 表結構與 Citus 的 `pg_dist_*` 家族完全不同，且 pg_shard 不支援 Reference Tables 和 Co-Location。

#### II. Citus 12 on PG 16 核心能力

Citus 12 運行在 PG 16 之上，提供四種 Table Type：

**1. Distributed Tables（分佈式表）**
最核心的表類型——將巨量資料按分佈鍵（Distribution Column）切分為多個 Shard，每個 Shard 都是一個獨立的 PostgreSQL Table，儲存在不同 Worker 節點上。

```sql
-- 建表時宣告為分佈式表
CREATE TABLE orders (
    id          bigserial,
    customer_id bigint NOT NULL,
    amount      numeric(12,2),
    created_at  timestamptz DEFAULT now(),
    PRIMARY KEY (id, customer_id)
);

-- 指定分佈鍵為 customer_id，切分為 32 個 Shard
SELECT create_distributed_table('orders', 'customer_id', shard_count := 32);
```

**2. Reference Tables（參考表）**
小型、經常與 Distributed Tables JOIN 的維度表（如 `products`、`categories`、`countries`），以 Full Replication 方式廣播到所有 Worker。查詢時每個 Worker 都有完整副本，避免跨節點資料傳輸。

```sql
CREATE TABLE products (
    product_id   bigint PRIMARY KEY,
    name         text NOT NULL,
    category_id  int NOT NULL,
    price        numeric(10,2)
);

-- 將 products 宣告為 Reference Table
SELECT create_reference_table('products');
```

**3. Local Tables（本地表）**
完全不做分散的普通 PostgreSQL 表，僅存在於 Coordinator 上。用於儲存集群級元數據或不需要分散的小型配置表。

**4. Schema-Based Sharding（Citus 12 新增）**
以整個 Schema 為 Sharding 單位——同一 Schema 內所有表自動歸屬同一 Tenant，無需逐表呼叫 `create_distributed_table()`。這是 Citus 12 對 SaaS 多租戶場景的關鍵回應。

```sql
-- 將整個 schema 宣告為分散式 schema
SELECT citus_schema_distribute('tenant_001');

-- 此後在 tenant_001 內建的任何表，自動成為 Distributed Table
SET search_path TO tenant_001;
CREATE TABLE invoices (id bigserial, amount numeric, ...);
-- 無需再呼叫 create_distributed_table()
```

**5. Query from Any Node（Citus 11+）**
從 Citus 11 開始，任意節點（Coordinator 或 Worker）都可以作為查詢入口，不再強制所有流量經過 Coordinator。這消除了 Coordinator 成為單點瓶頸的風險。

**6. MERGE 支援（PG 16）**
PG 16 原生的 `MERGE` 語句在 Citus 12 中直接可用於 Distributed Tables：

```sql
MERGE INTO orders AS t
USING order_updates AS s ON t.id = s.id AND t.customer_id = s.customer_id
WHEN MATCHED THEN UPDATE SET amount = s.amount
WHEN NOT MATCHED THEN INSERT VALUES (s.id, s.customer_id, s.amount);
```

#### III. Coordinator + Worker 架構

Citus 採用經典的 Shared-Nothing 架構：每個節點擁有獨立的 CPU、記憶體和存儲，節點間透過網路通訊。

```mermaid
graph TB
    subgraph "應用層"
        APP["應用程式<br/>libpq / PgBouncer"]
    end

    subgraph "Citus 集群"
        COORD["Coordinator<br/>PG 16 + Citus 12<br/>192.168.1.10"]
        W1["Worker 1<br/>PG 16 + Citus 12<br/>192.168.1.11"]
        W2["Worker 2<br/>PG 16 + Citus 12<br/>192.168.1.12"]
        W3["Worker 3<br/>PG 16 + Citus 12<br/>192.168.1.13"]
        W4["Worker 4<br/>PG 16 + Citus 12<br/>192.168.1.14"]
    end

    subgraph "Worker 1 HA"
        SR1["Streaming Replica<br/>192.168.1.21"]
    end
    subgraph "Worker 2 HA"
        SR2["Streaming Replica<br/>192.168.1.22"]
    end
    subgraph "Worker 3 HA"
        SR3["Streaming Replica<br/>192.168.1.23"]
    end
    subgraph "Worker 4 HA"
        SR4["Streaming Replica<br/>192.168.1.24"]
    end

    APP -->|"SQL 查詢<br/>Query from Any Node"| COORD
    APP -.->|"亦可直連 Worker"| W1
    COORD -->|"分發查詢 / 合併結果"| W1
    COORD -->|"分發查詢 / 合併結果"| W2
    COORD -->|"分發查詢 / 合併結果"| W3
    COORD -->|"分發查詢 / 合併結果"| W4
    W1 -.->|"WAL Streaming"| SR1
    W2 -.->|"WAL Streaming"| SR2
    W3 -.->|"WAL Streaming"| SR3
    W4 -.->|"WAL Streaming"| SR4

    COORD_META["pg_dist_node<br/>pg_dist_shard<br/>pg_dist_placement<br/>pg_dist_colocation"]
    COORD --- COORD_META

    style COORD fill:#4a90d9,color:white
    style W1 fill:#5cb85c,color:white
    style W2 fill:#5cb85c,color:white
    style W3 fill:#5cb85c,color:white
    style W4 fill:#5cb85c,color:white
    style SR1 fill:#f0ad4e,color:black
    style SR2 fill:#f0ad4e,color:black
    style SR3 fill:#f0ad4e,color:black
    style SR4 fill:#f0ad4e,color:black
```

Citus 內部 Metadata 表（儲存於 Coordinator）：

| Metadata 表 | 用途 |
|-------------|------|
| `pg_dist_node` | 集群節點資訊（hostname、port、nodeid、groupid） |
| `pg_dist_shard` | Shard 元數據（shardid、分佈鍵範圍、所屬表） |
| `pg_dist_placement` | Shard 具體位置（shardid 對應到哪個 Worker） |
| `pg_dist_colocation` | Co-Location Group 資訊 |
| `pg_dist_partition` | 分散式表的 Partition 資訊（分佈方式、分佈鍵） |

---

### 2. 安裝與集群配置（PG 16）

#### I. 安裝 Citus 12

```bash
## Debian / Ubuntu (使用 Citus 官方 apt repository)
curl https://install.citusdata.com/community/deb.sh | sudo bash
sudo apt install postgresql-16-citus-12

## RHEL / Rocky / AlmaLinux
sudo dnf install -y citus12_16
```

#### II. Coordinator 配置

```ini
-- postgresql.conf（Coordinator）
shared_preload_libraries = 'citus'

## 每個連接的最大記憶體（Citus Planner 使用）
citus.max_worker_nodes_tracked = 2048

## 預設 Shard 數量（新 create_distributed_table 使用）
citus.shard_count = 32

## 每個 Shard 的副本數（1 = 無副本，2 = 1 主 + 1 備）
citus.shard_replication_factor = 1

## 啟用 Schema-Based Sharding（Citus 12）
citus.enable_schema_based_sharding = on

## 每個分散式查詢可使用的最大 CPU 核心數
citus.max_cached_conns_per_worker = 2
```

重啟 PostgreSQL：

```bash
sudo systemctl restart postgresql@16-main
```

建立 Extension：

```sql
CREATE EXTENSION citus;
```

#### III. 新增 Worker 節點

```sql
-- 將 Worker 加入集群
SELECT citus_add_node('192.168.1.11', 5432);
SELECT citus_add_node('192.168.1.12', 5432);
SELECT citus_add_node('192.168.1.13', 5432);
SELECT citus_add_node('192.168.1.14', 5432);

-- 查看集群節點
SELECT nodeid, groupid, nodename, nodeport, noderole, isactive
FROM pg_dist_node;
--  nodeid | groupid |   nodename    | nodeport | noderole  | isactive
-- --------+---------+---------------+----------+-----------+----------
--       1 |       0 | 192.168.1.10  |     5432 | primary   | t
--       2 |       1 | 192.168.1.11  |     5432 | primary   | t
--       3 |       2 | 192.168.1.12  |     5432 | primary   | t
--       4 |       3 | 192.168.1.13  |     5432 | primary   | t
--       5 |       4 | 192.168.1.14  |     5432 | primary   | t
```

每個 Worker 節點也必須安裝 Citus Extension：

```sql
-- 在每個 Worker 上執行
CREATE EXTENSION citus;
```

> **初學者導讀**
> `citus_add_node()` 會自動在 Coordinator 的 `pg_dist_node` 表中記錄 Worker 資訊，同時在 Worker 上建立 Coordinator 的 Metadata 同步通道。`groupid` 是 Citus 內部的節點分組編號，同一個 Group 內的節點共享同一份 Shard 數據（透過 Streaming Replication 實現 HA）。

---

### 3. 核心操作

#### I. 建立分佈式表 (`create_distributed_table`)

Citus 支援兩種 Shard 分佈策略：

| 策略 | 語法 | 適用場景 |
|------|------|---------|
| **Hash Distribution** | `create_distributed_table('tbl', 'col')` | 數據均勻分佈、點查詢（`WHERE key = X`） |
| **Range Distribution** | `create_distributed_table('tbl', 'col', shard_count := N)` + 手動 `citus_shard_move()` 或排程 | 按時間/ID 範圍分段、範圍查詢（`WHERE key BETWEEN A AND B`） |

**Hash Distribution 示例：**

```sql
CREATE TABLE events (
    event_id   bigint PRIMARY KEY,
    user_id    bigint NOT NULL,
    event_type text NOT NULL,
    payload    jsonb,
    created_at timestamptz DEFAULT now()
);

-- 按 user_id Hash：同一使用者的所有事件落在同一 Shard
SELECT create_distributed_table('events', 'user_id', shard_count := 64);
```

**分佈鍵選擇原則：**

```mermaid
flowchart TD
    Q1{"主查詢模式<br/>是否按單一 KEY<br/>點查/小範圍查？"}
    Q1 -->|"是"| A["該 KEY 即為<br/>最佳分佈鍵"]
    Q1 -->|"否"| Q2{"是否有明確的<br/>JOIN 維度？"}
    Q2 -->|"是"| B["選擇 JOIN KEY<br/>作為分佈鍵<br/>實現 Co-Located JOIN"]
    Q2 -->|"否"| Q3{"數據量大於<br/>100M rows？"}
    Q3 -->|"是"| C["考慮複合分佈鍵<br/>或 Schema-Based<br/>Sharding（Citus 12）"]
    Q3 -->|"否"| D["選擇 Cardinality 最高<br/>且均勻分佈的欄位<br/>（如 user_id）"]
```

> **補充（Senior Dev）**
> 分佈鍵的 CARDINALITY 直接影響數據傾斜（Data Skew）。如果選擇 `country_code`（只有 ~200 個值）作為 Hash Key，數據將極度不均——部分 Shard 可能承載 90% 數據。可以使用以下查詢驗證分佈鍵的均勻性：
> ```sql
> SELECT count(DISTINCT country_code), count(*) / count(DISTINCT country_code) AS avg_per_value
> FROM orders;
> ```
> 理想情況下 `avg_per_value` 不應超過總行數的 10%。對於確實需要按低 Cardinality 欄位查詢的場景，考慮使用 **Reference Table + Filter** 而非作為分佈鍵。

#### II. Reference Tables (`create_reference_table`)

將小型維度表廣播到所有 Worker，每個 Worker 持有完整副本：

```sql
CREATE TABLE categories (
    category_id   int PRIMARY KEY,
    category_name text NOT NULL,
    parent_id     int
);

-- 宣告為 Reference Table：資料自動複製到所有 Worker
SELECT create_reference_table('categories');

-- 插入資料：Citus 自動廣播到所有 Worker
INSERT INTO categories VALUES (1, 'Electronics', NULL), (2, 'Books', NULL);
```

Reference Table 的特性：
- 每個 Worker 持有相同的完整副本
- 與 Distributed Table JOIN 時，每個 Worker 可在本地完成 JOIN（無網路傳輸）
- 適合行數 < 100K、更新頻率低的小型維度表
- 寫入操作需廣播到所有 Worker（寫入延遲 = max(所有 Worker 寫入時間)）

#### III. Co-Location (`colocate_with`)

當兩個 Distributed Table 使用**相同的分佈鍵**時，Citus 將它們放在同一 Co-Location Group，確保相同分佈鍵值的行存放在**同一個 Worker 的同一個 Shard** 中。

```sql
-- orders 和 order_items 使用相同的分佈鍵 customer_id
CREATE TABLE orders (
    order_id    bigint,
    customer_id bigint NOT NULL,
    amount      numeric(12,2),
    PRIMARY KEY (order_id, customer_id)
);

CREATE TABLE order_items (
    item_id     bigint,
    order_id    bigint NOT NULL,
    customer_id bigint NOT NULL,
    product_id  bigint NOT NULL,
    quantity    int,
    PRIMARY KEY (item_id, customer_id)
);

-- 兩個表使用相同的分佈鍵，並放入同一 Co-Location Group
SELECT create_distributed_table('orders',      'customer_id', colocate_with := 'none');
SELECT create_distributed_table('order_items', 'customer_id', colocate_with := 'orders');
```

Co-Location 的核心優勢：

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant W1 as "Worker 1<br/>Shard 1001 (orders)<br/>Shard 2001 (order_items)"
    participant W2 as "Worker 2<br/>Shard 1002 (orders)<br/>Shard 2002 (order_items)"

    Note over C: SELECT * FROM orders o<br/>JOIN order_items oi<br/>ON o.order_id = oi.order_id<br/>AND o.customer_id = oi.customer_id

    C->>W1: 查詢 customer_id ∈ [min1, max1]<br/>的 orders + order_items
    C->>W2: 查詢 customer_id ∈ [min2, max2]<br/>的 orders + order_items

    Note over W1: 本地 JOIN：零網路傳輸
    Note over W2: 本地 JOIN：零網路傳輸

    W1-->>C: 返回結果集
    W2-->>C: 返回結果集
    Note over C: 合併結果 → 回傳客戶端
```

> **初學者導讀**
> Co-Location 是 Citus 效能的「超級加速器」——想像 `orders` 和 `order_items` 的 JOIN，如果兩個表沒有 Co-Located，查詢時需要把其中一個表的全部資料在網路上搬來搬去（Repartition Join）；但 Co-Located 後，`customer_id = 42` 的訂單和明細**本來就在同一台機器上**，JOIN 完全本地完成，就像單機 PostgreSQL 一樣快。

#### IV. Schema-Based Sharding（Citus 12）

Citus 12 引入的 Schema-Based Sharding 徹底改變了 SaaS 多租戶架構的建置方式。傳統上每個租戶需要逐表呼叫 `create_distributed_table()`，而 Citus 12 允許**以整個 Schema 為 Sharding 單位**。

```sql
-- 為租戶 "tenant_001" 建立獨立 Schema 並宣告為分散式
CREATE SCHEMA tenant_001;
SELECT citus_schema_distribute('tenant_001');

-- 切換到租戶 Context（應用層通常由 Middleware 處理）
SET search_path TO tenant_001;

-- 從現在起，在 tenant_001 內建的表自動成為 Distributed Table
CREATE TABLE invoices (
    id        bigserial PRIMARY KEY,
    amount    numeric(12,2),
    issued_at timestamptz DEFAULT now()
);

CREATE TABLE payments (
    id          bigserial PRIMARY KEY,
    invoice_id  bigint REFERENCES invoices(id),
    paid_amount numeric(12,2),
    paid_at     timestamptz
);
-- invoices 和 payments 自動 Co-Located（同一個 Schema = 同一 Shard）
```

**三種 Table Type 對比：**

```mermaid
graph LR
    subgraph "Coordinator"
        LT["Local Tables<br/>(config, metadata)<br/>僅存於 Coordinator"]
    end

    subgraph "Worker 1"
        W1_DT["Distributed<br/>Shard 102001<br/>(customer_id 1-1000)"]
        W1_RT["Reference Table<br/>categories<br/>(完整副本)"]
        W1_SBS["Schema: tenant_001<br/>invoices + payments<br/>(Co-Located)"]
    end

    subgraph "Worker 2"
        W2_DT["Distributed<br/>Shard 102002<br/>(customer_id 1001-2000)"]
        W2_RT["Reference Table<br/>categories<br/>(完整副本)"]
    end

    subgraph "Worker 3"
        W3_DT["Distributed<br/>Shard 102003<br/>(customer_id 2001-3000)"]
        W3_RT["Reference Table<br/>categories<br/>(完整副本)"]
        W3_SBS["Schema: tenant_002<br/>invoices + payments<br/>(Co-Located)"]
    end

    LT -.->|"無複製"| W1_DT
    style LT fill:#e8d44d,color:black
    style W1_DT fill:#5cb85c,color:white
    style W2_DT fill:#5cb85c,color:white
    style W3_DT fill:#5cb85c,color:white
    style W1_RT fill:#4a90d9,color:white
    style W2_RT fill:#4a90d9,color:white
    style W3_RT fill:#4a90d9,color:white
    style W1_SBS fill:#d9534f,color:white
    style W3_SBS fill:#d9534f,color:white
```

> **補充（Senior Dev）**
> Schema-Based Sharding 內部實作是在 `citus_schema_distribute()` 被呼叫時，自動為該 Schema 建立一個 Co-Location Group，並將 Schema 內所有表的 `pg_dist_partition.colocationid` 指向該 Group。這意味著同一 Tenant 內的所有表自動享有 Co-Located JOIN 優勢。但要注意：跨 Schema 的 JOIN（即跨租戶查詢）**不會**受益於 Co-Location，且需要 Repartition Join（效能差）。強烈建議在應用層面徹底隔離租戶查詢，確保每次查詢 `search_path` 僅包含單一租戶 Schema。

---

### 4. 分佈式查詢與 JOIN 策略

Citus 的 Query Planner 根據涉及表的類型與分佈鍵，自動選擇三種 JOIN 策略：

```mermaid
flowchart TD
    Q["分散式查詢<br/>包含 JOIN"]
    Q --> Q1{"兩表是否<br/>Co-Located？"}
    Q1 -->|"是"| J1["Co-Located Join<br/>每個 Worker 本地 JOIN<br/>零網路傳輸"]
    Q1 -->|"否"| Q2{"其中一表是<br/>Reference Table？"}
    Q2 -->|"是"| J2["Reference Table Join<br/>將 Distributed Table 側的<br/>查詢分發到各 Worker<br/>各 Worker 本地與 Ref Table JOIN"]
    Q2 -->|"否"| J3["Repartition Join<br/>跨 Worker 重新分佈數據<br/>最昂貴的 JOIN 策略"]

    style J1 fill:#5cb85c,color:white
    style J2 fill:#4a90d9,color:white
    style J3 fill:#d9534f,color:white
```

#### I. Co-Located Join（零網路傳輸）

兩個使用相同分佈鍵的 Distributed Table 在同一 Worker 上進行本地 JOIN，完全無需網路傳輸。

```sql
-- orders 和 order_items 都按 customer_id 分佈
EXPLAIN (VERBOSE, COSTS OFF)
SELECT o.order_id, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id AND o.customer_id = oi.customer_id;
```

預期 EXPLAIN 輸出：
```
Custom Scan (Citus Adaptive)
  Task Count: 32
  Tasks Shown: One of 32 tasks
  ->  Task
        Node: host=192.168.1.11 port=5432 dbname=postgres
        ->  Hash Join  (actual rows=... loops=1)
              Hash Cond: (o.order_id = oi.order_id)
              ->  Seq Scan on orders_102001 o  (...)
              ->  Hash  (...)
                    ->  Seq Scan on order_items_202001 oi  (...)
```

> **初學者導讀**
> 當你看到 EXPLAIN 中同時出現 `orders` 和 `order_items` 的 Shard 在同一 Task 內執行時，代表這是 Co-Located Join——兩個表的數據在同一 Worker 上，JOIN 就像單機 PostgreSQL 一樣高效。

#### II. Reference Table Join（Broadcast + Local JOIN）

當 Distributed Table 與 Reference Table JOIN 時，Citus 將 Distributed Table 側的查詢分發到各 Worker，各 Worker 在本地與完整的 Reference Table 副本進行 JOIN。

```sql
-- events 是 Distributed Table（按 user_id），categories 是 Reference Table
SELECT e.event_type, c.category_name
FROM events e
JOIN categories c ON e.category_id = c.category_id
WHERE e.user_id = 42;
```

Citus 執行流程：
1. Coordinator 收到查詢
2. Planner 確定 `user_id = 42` 落在 Shard 102007（Worker 2）
3. 將查詢（包含 `JOIN categories`）發送到 Worker 2
4. Worker 2 本地執行 JOIN（`categories` 已有完整副本）
5. 結果回傳 Coordinator

#### III. Repartition Join（跨 Worker 重新分佈）

當兩個 Distributed Table 使用**不同的分佈鍵**且都不是 Reference Table 時，Citus 必須將其中一側的數據在 Worker 之間重新分佈（Repartition），這是最昂貴的 JOIN 策略。

```sql
-- users 按 user_id 分佈，orders 按 customer_id 分佈
-- 即使用戶和訂單邏輯相關，分佈鍵不一致導致 Repartition Join
EXPLAIN (COSTS OFF)
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.user_id = o.customer_id;
```

預期 EXPLAIN 輸出：
```
Custom Scan (Citus Adaptive)
  Task Count: 32
  ->  Task
        ->  Hash Join
              ->  Seq Scan on users_102001 u  (Worker 1)
              ->  Hash
                    ->  Custom Scan (Citus Adaptive)   <-- Repartition
                          Task Count: 32
                          ->  Task
                                ->  Seq Scan on orders_202005 o (Worker 3)
```

> **補充（Senior Dev）**
> Repartition Join 的嚴重性：假設 `users` 有 1M 行分佈在 32 個 Worker，`orders` 有 10M 行分佈在 32 個 Worker。Repartition Join 需要將兩側的數據都在 Worker 之間重新 Hash 分發——這意味著每個 Worker 可能需要接收來自其他 31 個 Worker 的數據。網路傳輸量接近 `SUM(shard_size) * (1 - 1/worker_count)`。這就是為什麼 Co-Location 設計在 Citus 中如此重要——一個好的分佈鍵選擇可以完全消除這種開銷。

---

### 5. EXPLAIN 解讀分佈式計畫

Citus 的 `EXPLAIN` 輸出與單機 PG 有顯著不同：它顯示分佈式查詢計畫，包括 Task Count、Shard Count 以及每個 Task 發送到 Worker 的具體 SQL。

#### I. 基本 EXPLAIN 解讀

```sql
EXPLAIN (VERBOSE, COSTS ON)
SELECT customer_id, count(*), sum(amount)
FROM orders
WHERE created_at >= '2026-01-01'
GROUP BY customer_id
ORDER BY sum(amount) DESC
LIMIT 10;
```

預期輸出（簡化）：
```
Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=0 width=0)
  Output: remote_scan.customer_id, remote_scan.count, remote_scan.sum
  Task Count: 32
  Tasks Shown: One of 32 tasks
  ->  Task
        Node: host=192.168.1.11 port=5432 dbname=postgres
        ->  Limit  (cost=...)
              ->  Sort  (cost=...)
                    Sort Key: (sum(orders.amount)) DESC
                    ->  HashAggregate
                          Group Key: orders.customer_id
                          ->  Seq Scan on orders_102001 orders
                                Filter: (created_at >= '2026-01-01 00:00:00+00'::timestamp with time zone)
```

關鍵解讀指標：

| 輸出欄位 | 含義 |
|---------|------|
| `Task Count: 32` | 該查詢總共產生 32 個 Task（對應 32 個 Shard） |
| `Tasks Shown: One of 32 tasks` | EXPLAIN 僅顯示其中一個 Task 的執行計畫（其餘結構相同） |
| `Node: host=...` | 該 Task 將在哪個 Worker 上執行 |
| `Seq Scan on orders_102001` | Shard ID = 102001 的具體查詢計畫 |

#### II. 複雜查詢的 EXPLAIN 解讀

```sql
EXPLAIN (VERBOSE, COSTS ON)
SELECT c.category_name, sum(o.amount) AS total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id AND o.customer_id = oi.customer_id
JOIN categories c ON oi.category_id = c.category_id
WHERE o.created_at >= '2026-01-01'
GROUP BY c.category_name;
```

預期輸出（跨 Worker 聚合）：
```
Custom Scan (Citus Adaptive)
  Output: intermediate_column_1, intermediate_column_2
  Task Count: 32
  ->  Task
        Node: host=192.168.1.12 port=5432 dbname=postgres
        ->  HashAggregate
              Group Key: c.category_name
              ->  Hash Join
                    ->  Hash Join
                          ->  Seq Scan on orders_102002 o
                          ->  Hash
                                ->  Seq Scan on order_items_202002 oi
                    ->  Hash
                          ->  Seq Scan on categories_99001 c  -- Reference Table 本地副本
```

觀察重點：
1. `order_items_202002` 與 `orders_102002` 在同一 Task 內 → Co-Located JOIN（Shard ID 編號尾數匹配）
2. `categories_99001` 出現在每個 Task → Reference Table 被廣播（每個 Worker 都有副本）
3. 最上層的 `Custom Scan (Citus Adaptive)` 包含 Coordinator 側的合併邏輯（`HashAggregate` on Coordinator 合併各 Worker 的 Partial Aggregates）

> **初學者導讀**
> 閱讀 Citus EXPLAIN 的訣竅：「從外到內，從上到下」——最外層 `Custom Scan (Citus Adaptive)` 告訴你 Coordinator 如何合併結果；內層 `Task` 告訴你每個 Worker 做什麼；`Node` 告訴你 Task 在哪個 Worker 執行；`Seq Scan` 上的表名（如 `orders_102001`）告訴你具體掃描了哪個 Shard。

---

### 6. Rebalancing（非阻塞 Shard 遷移）

當集群擴容（新增 Worker）或縮容（移除 Worker）時，Citus 12 提供**非阻塞**的 Shard Rebalancing，允許在不停機的情況下重新分配 Shard。

#### I. `citus_rebalance_start()` — 零停機擴容

```sql
-- 情境：從 4 個 Worker 擴展到 8 個 Worker
-- Step 1：新增 Worker 節點
SELECT citus_add_node('192.168.1.15', 5432);
SELECT citus_add_node('192.168.1.16', 5432);
SELECT citus_add_node('192.168.1.17', 5432);
SELECT citus_add_node('192.168.1.18', 5432);

-- Step 2：啟動 Rebalancing（預設使用 Non-Blocking 策略）
SELECT citus_rebalance_start();

-- Step 3（可選）：指定特定 Table 進行 Rebalancing
SELECT citus_rebalance_start(table_name := 'orders'::regclass);
```

Rebalancing 內部機制：

```mermaid
sequenceDiagram
    participant Coord as Coordinator
    participant W1 as "Worker 1<br/>(來源)"
    participant W5 as "Worker 5<br/>(目標，新節點)"

    Coord->>Coord: 計算 Rebalancing Plan<br/>（決定哪些 Shard 需要移動）

    Note over Coord: 開始移動 Shard 102005<br/>從 Worker 1 → Worker 5

    Coord->>W1: 標記 Shard 102005 為 READ-ONLY
    Coord->>W5: 建立空 Shard 102005

    loop Logical Replication
        W1->>W5: 同步現有數據<br/>（使用 COPY protocol）
        W1->>W5: 持續同步增量 WAL
    end

    Coord->>W1: Flush 所有進行中事務
    Coord->>W1: 切換 Shard 102005 為 INACTIVE
    Coord->>W5: 啟動 Shard 102005 為 ACTIVE
    Coord->>Coord: 更新 pg_dist_placement

    Note over Coord: Shard 遷移完成<br/>寫入流量重新路由到 Worker 5
```

> **補充（Senior Dev）**
> Citus 12 的 Non-Blocking Rebalancing 使用 PostgreSQL Logical Replication 協議進行數據同步：首先使用 `COPY` 做初始全量同步，然後透過 Logical Decoding 追蹤增量變更。整個過程中**來源 Shard 保持可讀可寫**，直到最後切換的瞬間（通常 < 100ms）才短暫暫停寫入。你可以使用 `citus.shard_replication_factor` 在遷移期間保留冗餘副本以進一步降低風險。

#### II. `citus_rebalance_status()` — 監控進度

```sql
-- 查看當前 Rebalancing 進度
SELECT * FROM citus_rebalance_status();

-- 預期輸出：
--  shardid | source_node | target_node | status   | bytes_moved | bytes_total
-- ---------+-------------+-------------+----------+-------------+------------
--   102001 |           2 |           5 | running  |    52428800 |   104857600
--   102005 |           1 |           6 | pending  |           0 |    209715200
--   102009 |           3 |           7 | done     |    10485760 |    10485760
```

#### III. `citus_drain_node()` — 安全移除 Worker

在移除 Worker 之前，需先將其上的所有 Shard 遷移到其他節點：

```sql
-- 將 Worker 5 (192.168.1.15:5432) 上的所有 Shard 安全遷出
SELECT citus_drain_node('192.168.1.15', 5432);

-- 遷出完成後，檢查是否還有 Shard 在該節點
SELECT count(*) FROM pg_dist_placement
WHERE groupid = (SELECT groupid FROM pg_dist_node WHERE nodename = '192.168.1.15');

-- 確認無 Shard 後，移除節點
SELECT citus_remove_node('192.168.1.15', 5432);
```

> **初學者導讀**
> `citus_drain_node()` 本質上是對該節點的所有 Shard 逐一呼叫 `citus_move_shard_placement()`。這是一個**安全**的操作——Citus 確保每個 Shard 在目標節點完全可用後，才從來源節點移除。但如果目標集群剩餘空間不足，操作會中止並報錯，絕不會遺失數據。

---

### 7. 高可用配置

Citus 集群的高可用策略分為兩個層級：Coordinator HA 和 Worker HA。

#### I. Worker 層級 HA — Streaming Replication

每個 Worker 節點配置一個（或多個）Streaming Replica：

```ini
-- Worker Primary postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = '4GB'

-- Worker Replica postgresql.conf
hot_standby = on
```

```sql
-- 將 Replica 加入 Citus Metadata（作為 Secondary）
-- pg_dist_node 中 groupid 相同的節點被視為同一 Group
SELECT citus_add_node('192.168.1.21', 5432, groupid := 1, noderole := 'secondary');
SELECT citus_add_node('192.168.1.22', 5432, groupid := 2, noderole := 'secondary');
```

#### II. Coordinator HA

Coordinator 本身也是 PG 16 節點，可配置 Streaming Replica。此外，從 Citus 11 開始，**任何節點都可以接受查詢**，這意味著：

- 當 Coordinator 故障時，您可以將應用程式指向任何 Worker 進行讀取查詢
- 寫入操作仍需 Coordinate（涉及 2PC），但可透過 **虛擬 IP 切換** 或 **Patroni/Stolon** 實現自動 Failover

#### III. `citus.ha` 配置

```ini
-- postgresql.conf
## Citus 自動檢測 Worker Replica 並在 Primary 故障時切換
citus.node_conninfo = 'sslmode=require'
citus.use_secondary_nodes = 'always'    -- 'never' | 'always' | 'read-only'

## 當 Primary 不可用時，多長時間後切換到 Secondary
citus.remote_copy_flush_threshold = '1MB'
```

```sql
-- 查看集群中 Primary 和 Secondary 的狀態
SELECT nodename, nodeport, groupid, noderole, isactive, shouldhaveshards
FROM pg_dist_node
ORDER BY groupid, noderole;
```

> **補充（Senior Dev）**
> Citus 本身**不是**一個自動 Failover 工具——它依賴 PostgreSQL 原生 Streaming Replication 提供數據冗餘，但 Failover 決策需要外部工具（如 Patroni、Stolon、Pgpool-II）。在生產環境中的推薦組合是：
> - **Worker HA**：Patroni（每 Worker 一組 etcd + Patroni + vip-manager）
> - **Coordinator HA**：同上，加上 PgBouncer 作為連接入口
> - **自動節點切換**：自訂腳本監聽 `pg_dist_node` 並在 Failover 後透過 `citus_update_node()` 更新 Metadata

---

### 8. App Dev 最佳實踐

#### I. 分佈鍵策略 — 避免數據傾斜

```sql
-- 驗證分佈鍵的均勻性
SELECT width_bucket(customer_id, min_id, max_id, 32) AS bucket,
       count(*) AS rows_per_bucket,
       pg_size_pretty(sum(pg_total_relation_size('orders'::regclass)) / 32) AS est_size
FROM orders,
     (SELECT min(customer_id) AS min_id, max(customer_id) AS max_id FROM orders) AS bounds
GROUP BY bucket
ORDER BY bucket;

-- 檢查每個 Shard 的實際大小
SELECT shardid, shard_size
FROM citus_shard_sizes()
WHERE table_name = 'orders'::regclass
ORDER BY shard_size DESC;
```

**選擇分佈鍵的黃金法則：**

1. **必須出現在所有 JOIN 的 ON 子句中**（實現 Co-Location）
2. **必須出現在 `create_distributed_table` 的 PRIMARY KEY 中**
3. **Cardinality 高於 Shard Count 的 100 倍以上**
4. **在 WHERE 查詢中頻繁使用**

#### II. INSERT..SELECT 效能考量

當 `INSERT..SELECT` 的來源和目標表使用相同的分佈鍵時，Citus 可以在各 Worker 上本地執行，無需跨節點資料傳輸：

```sql
-- 高效：兩個表按相同鍵分佈，資料在本地 Worker 間流動
INSERT INTO order_archive
SELECT * FROM orders WHERE created_at < '2025-01-01';

-- 低效：分佈鍵不同，觸發 Repartition
INSERT INTO analytics_events (user_id, event_type)
SELECT o.customer_id, 'PURCHASE'
FROM orders o
WHERE o.created_at >= '2026-01-01';
-- 若 orders 按 order_id 分佈，analytics_events 按 user_id 分佈，會觸發跨 Worker 資料搬移
```

> **初學者導讀**
> 大規模 `INSERT..SELECT` 在 Citus 中可能非常慢，原因是每個 INSERT 行都進入一個分散式事務。優化策略： (1) 確保源表和目標表 Co-Located；(2) 使用 `citus.local_table_join_policy = 'prefer_local'` 強制本地執行；(3) 分批插入（每批 1000-5000 行）。

#### III. 避免跨 Shard 事務（2PC 開銷）

```sql
-- BAD：跨 Shard 事務
BEGIN;
INSERT INTO orders VALUES (1, 42, 99.99);          -- customer_id=42 → Shard A
INSERT INTO orders VALUES (2, 9999, 199.99);       -- customer_id=9999 → Shard B
COMMIT;  -- 觸發 2PC（Two-Phase Commit），效能大幅下降

-- GOOD：同 Shard 事務（或依賴應用層確保）
-- 方案 A：應用層按 customer_id 分組，同一批只處理同一 customer_id
-- 方案 B：使用 Schema-Based Sharding（Citus 12），同一 Tenant 內所有操作為本地事務
BEGIN;
INSERT INTO tenant_001.invoices VALUES (1, 500.00);
INSERT INTO tenant_001.payments VALUES (1, 1, 500.00, now());
COMMIT;  -- 本地事務，無 2PC 開銷
```

```mermaid
flowchart TD
    TX["應用層事務"]
    TX --> Q{"是否跨 Shard？"}
    Q -->|"是"| TPC["Two-Phase Commit<br/>◆ Coordinator 發送 PREPARE<br/>◆ 所有 Worker 回應 READY<br/>◆ Coordinator 發送 COMMIT<br/>◆ 2 次網路往返，效能代價高"]
    Q -->|"否<br/>(同 Shard / 同 Tenant Schema)"| LOCAL["本地事務<br/>◆ 單次 COMMIT<br/>◆ 零額外網路開銷<br/>◆ 等同單機 PG 效能"]
    style TPC fill:#d9534f,color:white
    style LOCAL fill:#5cb85c,color:white
```

#### IV. 連接管理 — PgBouncer 雙層架構

```mermaid
graph TB
    subgraph "應用層"
        A1["App Server 1"]
        A2["App Server 2"]
        A3["App Server N"]
    end

    subgraph "Layer 1: PgBouncer (Transaction Pooling)"
        PB1["PgBouncer<br/>192.168.1.100:6432"]
    end

    subgraph "Layer 2: PgBouncer per Worker (Session Pooling)"
        COORD_PB["PgBouncer<br/>Coordinator<br/>:6432"]
        W1_PB["PgBouncer<br/>Worker 1<br/>:6432"]
        W2_PB["PgBouncer<br/>Worker 2<br/>:6432"]
        WN_PB["PgBouncer<br/>Worker N<br/>:6432"]
    end

    subgraph "PostgreSQL"
        COORD_DB["Coordinator<br/>PG 16"]
        W1_DB["Worker 1<br/>PG 16"]
        W2_DB["Worker 2<br/>PG 16"]
        WN_DB["Worker N<br/>PG 16"]
    end

    A1 --> PB1
    A2 --> PB1
    A3 --> PB1
    PB1 --> COORD_PB
    PB1 -.->|"必要時直連 Worker"| W1_PB
    COORD_PB --> COORD_DB
    W1_PB --> W1_DB
    W2_PB --> W2_DB
    WN_PB --> WN_DB
```

配置示例：

```ini
; Layer 1 PgBouncer（Transaction Pooling）
[databases]
citus_cluster = host=192.168.1.10 port=5432 dbname=postgres

[pgbouncer]
pool_mode = transaction
max_client_conn = 5000
default_pool_size = 50

; Layer 2 PgBouncer per Worker（Session Pooling）
[databases]
* = host=/var/run/postgresql port=5432

[pgbouncer]
pool_mode = session        ; Citus 內部連接需要 Session 模式
max_client_conn = 500
default_pool_size = 100
reserve_pool_size = 10
```

> **補充（Senior Dev）**
> Citus 內部節點間使用持久化的 Session 連接——Coordinator 到每個 Worker 維護一個連接池（由 `citus.max_cached_conns_per_worker` 控制，預設 1）。因此 **Layer 2 PgBouncer 必須使用 Session Pooling**，否則 Citus 的內部連接可能會被 PgBouncer 意外中斷。Layer 1（面向應用層）則應使用 Transaction Pooling，因為應用層的連接通常是短暫的、無狀態的。

#### V. 監控 — `citus_stat_statements`

Citus 提供 `citus_stat_statements` 視圖，記錄每個分散式查詢在各 Worker 上的執行統計，比 `pg_stat_statements` 更適合分散式環境。

```sql
-- 啟用 Citus 級別查詢統計
SET citus.stat_statements_track = 'all';
-- 'all' 記錄所有查詢 | 'none' 關閉 | 預設為 'top'

-- 查看最耗時的分散式查詢
SELECT queryid,
       calls,
       total_exec_time,
       mean_exec_time,
       rows,
       query
FROM citus_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 重置統計
SELECT citus_stat_statements_reset();
```

日常健康檢查 SQL 集合：

```sql
-- 1. 檢查集群節點健康狀態
SELECT nodename, nodeport, groupid, noderole, isactive, shouldhaveshards
FROM pg_dist_node
ORDER BY groupid;

-- 2. 檢查 Shard 分佈（是否有不活躍的 Placement）
SELECT shardid, shardstate, shardlength, nodename, nodeport
FROM pg_dist_shard
JOIN pg_dist_placement USING (shardid)
JOIN pg_dist_node ON pg_dist_placement.groupid = pg_dist_node.groupid
WHERE shardstate != 1;  -- 1 = ACTIVE

-- 3. 檢查各表 Shard 大小分佈
SELECT logicalrelid::regclass AS table_name,
       shard_count,
       pg_size_pretty(total_size) AS total_size
FROM citus_tables;

-- 4. 檢查 Reference Table 同步狀態
SELECT logicalrelid::regclass,
       count(DISTINCT groupid) AS replica_count
FROM pg_dist_shard
JOIN pg_dist_placement USING (shardid)
JOIN pg_dist_partition USING (logicalrelid)
WHERE partmethod = 'n'  -- 'n' = none (Reference Table)
GROUP BY logicalrelid;
```

> **初學者導讀**
> 分散式系統的日常監控重點：(1) `pg_dist_node.isactive = true` 確保所有節點在線；(2) `pg_dist_placement.shardstate = 1` 確保所有 Shard 為 ACTIVE 狀態；(3) `citus_shard_sizes()` 檢查各 Shard 大小是否均勻（差距 > 5x 代表有數據傾斜）；(4) Coordinator 的 `pg_stat_activity` 檢查是否有長時間執行的分散式查詢。
## 四、PgBouncer — 連接池

### 1. PostgreSQL 連接模型的問題

#### I. Process-per-Connection 先天瓶頸

PostgreSQL 採用 **Process-per-Connection** 模型：每個 client 連接進來，postmaster 就 `fork()` 一個 backend process 獨立服務它。這是 PG 最核心的架構決策之一，但也是它最大的資源瓶頸來源。

> **初學者導讀**
>
> 想像一個餐廳：PostgreSQL 是廚房，每個客人進來，廚房就專門為他開一個新灶台（fork 一個 process）。客人再多，灶台就越多，但廚房空間（記憶體）有限。客人吃完不走（idle connection），灶台還占著，新客人就進不來。PgBouncer 就像一個前台服務員，它把 100 個客人輪流分配到 10 個灶台上，客人感覺不到差異，但廚房負擔大幅降低。

每個 backend process 的初始開銷：

| 項目 | 典型值 | 說明 |
|------|--------|------|
| 進程記憶體 | ~5–10 MB | shared_buffers 之外每個 process 的私有記憶體 |
| 1000 個 idle connection | **5–10 GB** | 白白浪費的記憶體 |
| context switch 開銷 | 隨連線數增長 | OS scheduler 負擔加重 |
| 最大連線數限制 | `max_connections`（預設100） | 超過後新連線直接被拒 |

```sql
-- 查看當前連線數
SELECT count(*) FROM pg_stat_activity;

-- 查看 idle 連線
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

核心矛盾：**應用層需要大量短暫連接來併發處理請求，但 PG 每個連接都貴重且應長久持有**。這就是連接池必須存在的原因。

#### II. 應用層 vs PG 側連接池

| 層級 | 代表 | 適用場景 | 局限 |
|------|------|----------|------|
| 應用層 | HikariCP（Java）、r2d2（Rust） | 單一應用服務的多線程併發 | 應用重啟丟失、多應用無法共享 |
| 資料庫層 | PgBouncer、Pgpool-II | 多應用共享、微服務架構 | 需要額外部署運維 |

**PgBouncer 的核心價值在於 Transaction Pooling**：它不把一個 client connection 綁定到一個 server connection 上，而是在 transaction 級別復用。一個 client 執行 `BEGIN...COMMIT` 期間才佔用一個真實 PG connection，事務結束後立即釋放。

> **補充（Senior Dev）**
>
> 即使是 Serverless Postgres（如 AWS RDS Proxy、Supabase Pgbouncer）底層幾乎都是 PgBouncer 或其變體。理解 PgBouncer 就是理解雲端 PG 的連線管理基礎。HikariCP 和 PgBouncer 並不衝突——最佳實踐是兩層都配：應用層用 HikariCP 控制每個應用實例的連線數上限，PG 側用 PgBouncer 做跨服務的彙聚與復用。

```mermaid
graph LR
    subgraph 使用 PgBouncer 之前
        APP1[App Server 1<br/>直接 50 連接] --> PG[(PostgreSQL<br/>150 idle 連接<br/>記憶體大量浪費)]
        APP2[App Server 2<br/>直接 50 連接] --> PG
        APP3[App Server 3<br/>直接 50 連接] --> PG
    end
```

```mermaid
graph LR
    subgraph 使用 PgBouncer 之後
        APP1[App Server 1<br/>50 連接] --> PB[(PgBouncer<br/>Transaction Pool<br/>pool_size=20)]
        APP2[App Server 2<br/>50 連接] --> PB
        APP3[App Server 3<br/>50 連接] --> PB
        PB --> PG[(PostgreSQL<br/>僅 20 連接<br/>記憶體精簡)]
    end
```

### 2. PgBouncer 架構

#### I. 三種 Pooling Mode

| Mode | 綁定時機 | 釋放時機 | 適用場景 |
|------|----------|----------|----------|
| **Session Pooling** | client 連接建立時 | client 斷開時 | LISTEN/NOTIFY、需要 `PREPARE`、需要 session 級 SET |
| **Transaction Pooling** | `BEGIN`（或 implicit transaction 開始時）| `COMMIT`/`ROLLBACK` 後 | **絕大多數 Web App、API 服務**（推薦預設）|
| **Statement Pooling** | 每條 SQL 執行前 | 每條 SQL 執行後 | 極少使用（autocommit 模式下的簡單查詢）|

```mermaid
sequenceDiagram
    participant C as Client
    participant PB as PgBouncer
    participant S as PG Backend

    Note over C,S: Session Pooling
    C->>PB: connect
    PB->>S: assign server (整個 session)
    C->>PB: SELECT 1
    PB->>S: SELECT 1
    C->>PB: SELECT 2
    PB->>S: SELECT 2
    C->>PB: disconnect
    PB->>S: release server

    Note over C,S: Transaction Pooling
    C->>PB: connect
    Note over PB: 暫不分配 server
    C->>PB: BEGIN
    PB->>S: assign server
    C->>PB: SELECT 1
    PB->>S: SELECT 1
    C->>PB: COMMIT
    S-->>PB: COMMIT OK
    PB->>S: release server
    Note over PB: server 回池, 其他 client 可用

    Note over C,S: Statement Pooling
    C->>PB: connect
    C->>PB: SELECT 1
    PB->>S: assign → execute → release
    C->>PB: SELECT 2
    PB->>S: assign → execute → release
```

> **初學者導讀**
>
> - **Session Pooling**：一個客人從進門到離開，全程專屬一個服務生。
> - **Transaction Pooling**：客人只有點餐（BEGIN）到結帳（COMMIT）期間才佔用服務生，其他時候服務生可以去服務其他客人。
> - **Statement Pooling**：客人說一句話就換一個服務生（幾乎沒人這樣用）。
>
> 實際開發中，選 Transaction Pooling 就對了——除非你確定需要 LISTEN/NOTIFY 或 Prepared Statements。

> **補充（Senior Dev）**
>
> 當使用 Session Pooling + `server_reset_query` 時，可在每次 server 復用前執行清理（如 `DISCARD ALL`）。但在 Transaction Pooling 下，PgBouncer 自動在 transaction 之間執行 `DISCARD ALL`（1.19+），不需要額外配置。注意：`DISCARD ALL` 會清除 `SET` 變數和 `PREPARE` 的 statement，這是 Transaction Pooling 不支援這些功能的原因。

#### II. 內部架構

PgBouncer 的設定檔核心結構：

```ini
[databases]
## <dbname> = host=<host> port=<port> dbname=<dbname>
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

pool_mode = transaction
default_pool_size = 20
max_client_conn = 100
```

關鍵概念：

| 參數 | 說明 |
|------|------|
| `admin_users` | 可以連入 `pgbouncer` 虛擬資料庫執行管理命令 |
| `stats_users` | 只能讀取統計資訊（只讀權限）|
| `pool_mode` | 預設 pooling 模式（`session` / `transaction` / `statement`）|
| `default_pool_size` | 每個 database/user 組合最多開多少個 server connection |
| `max_client_conn` | 允許的最大 client connection 數 |
| `auth_file` | 使用者認證檔路徑（`userlist.txt`）|

**Connection 生命週期（Transaction Pooling）：**

```mermaid
graph TD
    CC[Client Connect] --> CA{client_conn > max?}
    CA -->|Yes| REJ[拒絕連接]
    CA -->|No| Q[排隊等待]
    Q --> SA[Server Assign<br/>當有空閒 server 且 pool 未滿]
    SA --> TX[BEGIN / Implicit TX]
    TX --> EX[執行 SQL]
    EX --> CO[COMMIT / ROLLBACK]
    CO --> SR[Server Release<br/>歸還到 pool]
    SR -->|有新 transaction| SA
    SR -->|無請求| IDLE[Server Idle<br/>等待下次分配]
    IDLE -->|超過 server_idle_timeout| DC[斷開 Server Connection]
```

> **補充（Senior Dev）**
>
> PgBouncer 本身是單進程、事件驅動（libevent）的 C 程式，記憶體佔用極小（通常 < 10 MB）。它是非同步 I/O 模型，一個 process 可以處理數萬個 client connection。這和 PG 的 process-per-connection 形成鮮明對比。Pgbouncer 本身只需要極少的 CPU 和記憶體，幾乎不會成為瓶頸。

### 3. 安裝與配置（PG 16）

#### 安裝

```bash
## Debian / Ubuntu
sudo apt update
sudo apt install -y pgbouncer

## 檢查版本（建議 1.19+）
pgbouncer --version
```

#### 完整配置

**`/etc/pgbouncer/pgbouncer.ini`**（含完整中文註解）：

```ini
[databases]
## 格式：<對外名稱> = host=<PG位址> port=<PG埠> dbname=<PG庫名>
## 可以為同一 PG 庫定義多個對外名稱，使用不同 pool_mode
mydb = host=127.0.0.1 port=5432 dbname=mydb
## 若需要針對特定 database 使用 session pooling：
## mydb_session = host=127.0.0.1 port=5432 dbname=mydb pool_mode=session

[pgbouncer]
## ===== 監聽設定 =====
listen_addr = 127.0.0.1          # 生產環境建議監聽內網 IP
listen_port = 6432               # PgBouncer 預設埠
unix_socket_dir = /var/run/postgresql

## ===== 認證設定 =====
auth_type = scram-sha-256        # PG 16 預設認證方式
auth_file = /etc/pgbouncer/userlist.txt

## ===== 管理使用者 =====
admin_users = pgbouncer_admin    # 可執行 SHOW/SET/RELOAD 等管理指令
stats_users = pgbouncer_stats    # 只能查看統計資訊

## ===== Pooling 設定 =====
pool_mode = transaction          # App Dev 首選
default_pool_size = 20           # 每個 database/user 的最大 server 連接數
min_pool_size = 5                # 最少保持的 server 連接數（1.19+）
reserve_pool_size = 5            # 備用池，常規池滿時啟用
reserve_pool_timeout = 3         # 備用池啟用前的等待時間（秒）

## ===== Client 連接設定 =====
max_client_conn = 100            # 最大 client 連接數
max_db_connections = 50          # 每個 database 最大 server 連接數限制
max_user_connections = 50        # 每個 user 最大 server 連接數限制

## ===== Timeout 設定 =====
server_idle_timeout = 600        # server 空閒超過此時長則關閉（秒）
server_lifetime = 3600           # server 最大存活時間，到期後關閉重建（秒）
server_connect_timeout = 15      # 連接 PG 的超時時間（秒）
client_idle_timeout = 0          # client 空閒超時（0=不限制）
query_timeout = 0                # 單條查詢超時（0=不限制）
query_wait_timeout = 120         # client 等待 server 分配的逾時（秒）

## ===== 連線生命週期 =====
server_fast_close = 0            # 1=PgBouncer 關閉時立即斷開所有 server
server_round_robin = 0           # 1=輪詢分配 server（而非 LIFO）

## ===== 日誌設定 =====
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
log_connections = 1              # 記錄 client 連接
log_disconnections = 1           # 記錄 client 斷開
log_pooler_errors = 1            # 記錄 pool 錯誤
stats_period = 60                # SHOW STATS 彙總週期（秒）

## ===== 安全性 =====
disable_pqexec = 0               # 1=禁用簡單查詢協議（僅允許 extended query）
client_tls_sslmode = disable     # PgBouncer ↔ Client 的 TLS
server_tls_sslmode = disable     # PgBouncer ↔ PG 的 TLS
```

**`/etc/pgbouncer/userlist.txt`**（PG 16 的 SCRAM-SHA-256）：

PgBouncer 的 `userlist.txt` 有兩種填寫方式：

#### 方式一：直接從 PG 導出密碼

```bash
## 在 PG 中以 postgres 身份執行
sudo -u postgres psql -c "SELECT rolname, rolpassword FROM pg_authid WHERE rolpassword IS NOT NULL;" -At
```

將輸出的 `rolpassword`（形如 `SCRAM-SHA-256$4096:...`）填入：

```
"pgbouncer_admin" "SCRAM-SHA-256$4096:AbCdEf123456..."
"myapp_user" "SCRAM-SHA-256$4096:XyZ789012345..."
```

#### 方式二：使用 userlist.txt 驗證（不依賴 PG auth）

如果 `auth_type = scram-sha-256`，PgBouncer 可以直接在 userlist.txt 中存明文密碼，它會自行進行 SCRAM 驗證：

```
"myapp_user" "mypassword"
```

> **補充（Senior Dev）**
>
> 方式二讓 PgBouncer 自行驗證密碼，完全不查詢 PG 的 `pg_authid`。這意味著 PgBouncer 無需事先建立 server connection 即可完成 client 認證——在 server 資源緊張時尤其有用。缺點是密碼需在 `userlist.txt` 中兩邊維護。可配合 `auth_query`（PgBouncer 1.18+）從 PG 動態查詢認證資訊：
>
> ```ini
> auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename = $1
> ```
>
> 這樣無需維護 `userlist.txt`，PgBouncer 會以 admin user 身份查詢 PG 進行認證。

#### 啟動與驗證

```bash
## 啟動 PgBouncer
sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

## 檢查狀態
sudo systemctl status pgbouncer

## 透過 PgBouncer 連線（埠 6432）
psql -h 127.0.0.1 -p 6432 -U myapp_user -d mydb

## 連入管理 console
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin -d pgbouncer
```

### 4. Transaction Pooling 注意事項

#### I. 不支援的功能及替代方案

由於 Transaction Pooling 在每個 transaction 之間執行 `DISCARD ALL`（清除 session 狀態），以下功能**無法**在 Transaction Pooling 下使用：

| 不支援的功能 | 原因 | 替代方案 |
|-------------|------|----------|
| **`PREPARE` / `DEALLOCATE`** | Prepared statement 隨 `DISCARD ALL` 清除 | 使用 `max_prepared_statements`（PgBouncer 1.22+）或改用 Session Pooling |
| **`LISTEN` / `NOTIFY`** | 需要 session 級持久監聽 | 為 LISTEN 的連接使用 Session Pooling（獨立的 database entry）|
| **`WITH HOLD CURSOR`** | Cursor 需要在 transaction 外存活 | 使用 temporary table + LIMIT/OFFSET 分頁 |
| **`SET` / `SET LOCAL`** | 設定在 transaction 後被清除 | 在 `BEGIN` 後使用 `SET LOCAL`（僅 transaction 內有效）或改用 `PGOPTIONS` |
| **Advisory Lock (`pg_advisory_lock`)** | Session 級鎖在 transaction 後釋放 | 使用 row-level lock（`SELECT ... FOR UPDATE`）或 Redis 分散式鎖 |
| **`SET ROLE`** | 角色設定在 transaction 後清除 | 在 `BEGIN` 後 `SET LOCAL ROLE` 或為不同角色建立獨立的 database entry |

```sql
-- 不支援的寫法（Transaction Pooling 下）
SET statement_timeout = '30s';
SELECT * FROM large_table;  -- SET 可能已被 DISCARD ALL 清除

-- 正確寫法：在 transaction 內使用 SET LOCAL
BEGIN;
SET LOCAL statement_timeout = '30s';
SELECT * FROM large_table;
COMMIT;
```

> **初學者導讀**
>
> 簡短記法：**Transaction Pooling 下，不要把任何狀態帶到下一個 transaction**。每個 transaction 都是「乾淨的」——忘記前一個 transaction 的任何設定。如果你寫了一段程式碼依賴「上一個請求設定了某個變數」，那在 Transaction Pooling 下就會出錯。

#### II. PgBouncer 1.22+ Prepared Statement 追蹤

從 PgBouncer 1.22 開始，引入了 `max_prepared_statements` 參數來追蹤 client 端的 prepared statement，在 transaction pooling 下也能使用：

```ini
## pgbouncer.ini
max_prepared_statements = 100   # 每個 client 最多追蹤 100 個 prepared statement
```

工作原理：PgBouncer 會追蹤 client 傳送的 `Parse` 訊息（prepared statement 的名稱和 SQL），當 server 被 `DISCARD ALL` 清除後，下一個 transaction 需要該 prepared statement 時，PgBouncer 會自動在 server 上重新 `Parse`。這對 client 是透明的。

限制：
- 僅追蹤 **named** prepared statement（unnamed statement 不追蹤）
- 僅在 Protocol-level prepared statement 有效（`libpq` 的 `PQprepare`）
- 不支援 `parse-ampersand` 參數的高階場景

```ini
## 完整相關配置
max_prepared_statements = 100
track_extra_parameters = 1      # 追蹤額外的連線參數（1.22+）
```

> **補充（Senior Dev）**
>
> 如果你的 ORM（如 Hibernate、Prisma）大量使用 prepared statement，PgBouncer 1.22+ 的這個特性是升級的關鍵動力。升級前務必測試 ORM 的行為是否正確。若 ORM 使用的是 simulated prepared statement（client 端模擬），則不受 `DISCARD ALL` 影響，但也享受不到 server-side prepare 的效能優勢。

### 5. 生產部署拓撲

#### I. 單實例

最簡單的部署方式：PgBouncer 與 PG 安裝在同一台機器上。

```
[Client] → [PgBouncer :6432] → [PostgreSQL :5432]
```

優點：低延遲、配置簡單。適合小型專案或單一應用。

#### II. 多實例 + HAProxy（高可用）

為避免 PgBouncer 成為單點故障（SPOF），部署多個 PgBouncer 實例並使用 HAProxy 做負載均衡：

```mermaid
graph TD
    subgraph Application Tier
        A1[App Server 1]
        A2[App Server 2]
        A3[App Server N]
    end

    subgraph Load Balancer
        HA[HAProxy<br/>TCP Mode<br/>Port 6432]
    end

    subgraph PgBouncer Tier
        PB1[PgBouncer 1<br/>pool_size=20]
        PB2[PgBouncer 2<br/>pool_size=20]
        PB3[PgBouncer 3<br/>pool_size=20]
    end

    subgraph Database Tier
        PG_P[Primary<br/>Port 5432]
    end

    A1 --> HA
    A2 --> HA
    A3 --> HA
    HA --> PB1
    HA --> PB2
    HA --> PB3
    PB1 --> PG_P
    PB2 --> PG_P
    PB3 --> PG_P
```

HAProxy 配置要點：

```ini
## haproxy.cfg
frontend pgbouncer_frontend
    bind *:6432
    default_backend pgbouncer_backend

backend pgbouncer_backend
    balance leastconn                          # 最小連接數演算法
    option tcp-check
    server pb1 10.0.1.1:6432 check inter 5s
    server pb2 10.0.1.2:6432 check inter 5s
    server pb3 10.0.1.3:6432 check inter 5s
```

**注意**：每個 PgBouncer 實例的 `pool_size` 應該是總目標 pool size 除以實例數。若 PG 總承載 60 個 server connection，有 3 個 PgBouncer 實例，每個 `default_pool_size = 20`。

> **補充（Senior Dev）**
>
> HAProxy 的 `leastconn` 演算法將新 client 分配給當前 server 連接最少的 PgBouncer，這是連接池場景的最佳選擇。如果使用 Kubernetes，可以用 Service（ClusterIP）+ `sessionAffinity: ClientIP` 代替 HAProxy，但靈活度較低。

#### III. Kubernetes Sidecar 模式

將 PgBouncer 作為 Sidecar 部署在每個 Application Pod 中：

```yaml
## deployment.yaml（簡化示例）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          env:
            - name: DATABASE_URL
              value: "postgresql://user:pass@127.0.0.1:6432/mydb"
        - name: pgbouncer
          image: edoburu/pgbouncer:latest
          env:
            - name: DB_HOST
              value: "pg-primary.svc.cluster.local"
            - name: DB_PORT
              value: "5432"
            - name: POOL_MODE
              value: "transaction"
            - name: DEFAULT_POOL_SIZE
              value: "10"
```

Sidecar 模式的優點：
- 每個 Pod 有自己的 PgBouncer，無需額外服務發現
- 透過 `127.0.0.1:6432` 本地連接，延遲最小
- 跟隨 Pod 伸縮，無需單獨管理 PgBouncer 集群

#### IV. Citus + PgBouncer 兩層配置

Citus 的分散式架構需要兩層 PgBouncer：

```mermaid
graph TD
    subgraph App Tier
        APP[Application]
    end

    subgraph "Layer 1 PgBouncer (Coordinator Side)"
        PB_C[PgBouncer<br/>pool_mode=transaction<br/>→ Coordinator :5432]
    end

    subgraph Coordinator
        COORD[Citus Coordinator<br/>pg_catalog 含分片元數據]
    end

    subgraph "Layer 2 PgBouncer (Worker Side)"
        PB_W1[PgBouncer<br/>→ Worker 1 :5432]
        PB_W2[PgBouncer<br/>→ Worker 2 :5432]
        PB_W3[PgBouncer<br/>→ Worker 3 :5432]
    end

    subgraph Workers
        W1[Worker 1<br/>Shard 1]
        W2[Worker 2<br/>Shard 2]
        W3[Worker 3<br/>Shard 3]
    end

    APP --> PB_C
    PB_C --> COORD
    COORD --> PB_W1
    COORD --> PB_W2
    COORD --> PB_W3
    PB_W1 --> W1
    PB_W2 --> W2
    PB_W3 --> W3
```

第一層（Coordinator 前）：管理應用連到 Coordinator 的連接池。

第二層（Worker 前）：管理 Coordinator 到各 Worker 的連接池。這特別關鍵——Coordinator 在分散式查詢時會向多個 Worker 並行發送查詢，產生的連接數可能很大，需要 PgBouncer 做節流。

```ini
## 第二層 Worker PgBouncer 配置示例
[databases]
worker_db = host=worker1.internal port=5432 dbname=citus

[pgbouncer]
pool_mode = transaction
default_pool_size = 50          # 較大的 pool，因為 Coordinator 併發查詢多
max_client_conn = 500
server_idle_timeout = 300
```

### 6. 監控與管理

#### I. Admin Console

連入 Admin Console：

```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin -d pgbouncer
```

核心管理命令：

```sql
-- 查看所有 pool 狀態（最常用）
SHOW POOLS;

-- 查看當前 client 連接
SHOW CLIENTS;

-- 查看 PgBouncer ↔ PG 的 server 連接
SHOW SERVERS;

-- 查看彙總統計
SHOW STATS;

-- 查看當前等待中的 client（排隊列表）
SHOW LISTS;

-- 查看配置參數
SHOW CONFIG;

-- 查看目前連線的 database 列表
SHOW DATABASES;

-- 動態重載設定檔
RELOAD;

-- 暫停/恢復新的 client 連接（用於維護）
PAUSE;
RESUME;

-- 關閉 PgBouncer
SHUTDOWN;
```

> **初學者導讀**
>
> `SHOW POOLS` 是你最常使用的命令。它告訴你每個 database 的 pool 狀況——多少 server 在使用（active）、多少空閒（idle）、多少 client 在排隊（waiting）。如果 `cl_waiting > 0`，表示連接池飽和，需要加大 `default_pool_size` 或檢查是否有 slow query。

#### II. 關鍵指標

#### SHOW POOLS 輸出解讀

```
 database  | user | cl_active | cl_waiting | cl_active_cancel_req |...
-----------+------+-----------+------------+----------------------+---
 mydb      | app  |        15 |          3 |                    0 |...
```

關鍵欄位：

| 欄位 | 說明 | 健康值 |
|------|------|--------|
| `cl_active` | 正在執行查詢的 client 數 | < `pool_size` |
| `cl_waiting` | 等待 server 分配的 client 數 | **= 0**（>0 表示瓶頸）|
| `sv_active` | 正在執行查詢的 server 數 | 接近 `cl_active` |
| `sv_idle` | 空閒的 server 數 | > 0（有餘量）|
| `sv_used` | 曾被使用過的 server 數 | 參考 |
| `maxwait` | 最長等待時間（微秒）| < 1000000（<1秒）|

```sql
-- 查看哪些 client 在等待（找出瓶頸來源）
SHOW CLIENTS;
-- 關注 state='waiting' 的 client，查看其 query_start 欄位確認等待時長
```

#### SHOW STATS 輸出

```
total_xact_count  | total_query_count | total_received | total_sent |...
------------------+-------------------+----------------+------------+---
         12345678 |          89012345 |    12345678901 | 9876543210 |...
```

核心指標：
- `total_xact_count`：處理的 transaction 總數
- `avg_query_count`：每個 transaction 平均查詢數
- `avg_recv`：平均每個 transaction 接收的資料量
- `avg_sent`：平均每個 transaction 發送的資料量

#### III. 動態重載（RELOAD）

PgBouncer 支援不中斷服務的配置變更：

```bash
## 修改 pgbouncer.ini 後
sudo systemctl reload pgbouncer

## 或透過 Admin Console
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin -d pgbouncer -c "RELOAD;"
```

**RELOAD 的限制**：
- 可以動態變更：`pool_size`、`max_client_conn`、`server_idle_timeout`、`query_timeout`、`log_connections` 等
- **不能**動態變更：`listen_addr`、`listen_port`、`auth_type`、`auth_file`、`pool_mode`（須重啟）
- 已建立的 server connection 不受影響，只在下次分配時套用新設定

```ini
## 方便動態調整的參數範例
default_pool_size = 20     # RELOAD 後生效 → 新 transaction 使用新的限制
min_pool_size = 5          # RELOAD 後逐步調整
query_timeout = 30         # RELOAD 後新查詢套用
```

### 7. App Dev 最佳實踐

#### Pool Mode 選擇決策樹

```mermaid
flowchart TD
    Q1{需要 LISTEN/NOTIFY?} -->|Yes| Session
    Q1 -->|No| Q2{需要跨 Transaction<br/>的 Prepared Statement?}
    Q2 -->|Yes, PgBouncer < 1.22| Session
    Q2 -->|Yes, PgBouncer >= 1.22| Q2b{max_prepared_statements<br/>能滿足?}
    Q2b -->|Yes| Transaction
    Q2b -->|No| Session
    Q2 -->|No| Q3{需要 SET ROLE 跨<br/>Transaction 有效?}
    Q3 -->|Yes| Session
    Q3 -->|No| Q4{需要 Advisory Lock?}
    Q4 -->|Yes| Session
    Q4 -->|No| Q5{需要 WITH HOLD CURSOR?}
    Q5 -->|Yes| Session
    Q5 -->|No| Transaction[Transaction Pooling<br/>✅ 首選]
```

**結論**：95% 的應用選擇 **Transaction Pooling**。只有在需要上述 Session 級功能時，才為特定 database entry 配置 `pool_mode=session`。

#### Connection String 配置

```python
## Python / psycopg2 或 psycopg3
## 直連 PG（開發環境）
DATABASE_URL = "postgresql://user:pass@localhost:5432/mydb"

## 透過 PgBouncer（Transaction Pooling）
DATABASE_URL = "postgresql://user:pass@localhost:6432/mydb"

## 關鍵參數
## application_name — 用於 SHOW CLIENTS 中識別
## statement_timeout — 在 transaction 內設置
conn = psycopg.connect(
    "postgresql://user:pass@localhost:6432/mydb"
    "?application_name=my_service"
)
```

```java
// Java / HikariCP + PgBouncer
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:6432/mydb");
config.setUsername("myapp_user");
config.setPassword("password");

// 配合 Transaction Pooling 的關鍵配置
config.setMaximumPoolSize(20);              // 應用層 pool 上限
config.setConnectionTimeout(30000);         // 等待連接的超時
config.setIdleTimeout(600000);              // 空閒連接回收時間
config.setMaxLifetime(1800000);             // 連接最大存活（< PgBouncer server_lifetime）
config.addDataSourceProperty("ApplicationName", "my_service");
config.addDataSourceProperty("prepareThreshold", "0"); // 避免 unnamed prepare
```

> **補充（Senior Dev）**
>
> HikariCP 的 `maximumPoolSize` 應該設置為不大於 PgBouncer 的池容量。兩層池的乘法效應：如果每個 App 實例有 20 個 HikariCP 連接，有 5 個 App 實例，直接打到 PG 就是 100 個連接。加上 PgBouncer 後，PG 側限制為 20 個，這才發揮了 PgBouncer 的價值。

#### idle_in_transaction_session_timeout 協調

```sql
-- PG 端設定（pg 側的最後防線）
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();
```

```ini
## PgBouncer 端設定（更精細的控制）
query_timeout = 30           # 單條查詢超時 30 秒
client_idle_timeout = 600    # client 空閒 10 分鐘斷開
server_idle_timeout = 600    # server 空閒 10 分鐘斷開（釋放 PG 連接）
```

> **初學者導讀**
>
> 一個常見的生產事故：某個 API handler 開了 `BEGIN` 但忘記 `COMMIT`，導致該 server connection 一直處於 `idle in transaction` 狀態。這個連接不會被 PgBouncer 釋放，因為 transaction 還沒結束。`idle_in_transaction_session_timeout` 是 PG 側的安全網——超過設定時間後 PG 會強制 `ROLLBACK` 並斷開連接，讓 PgBouncer 可以分配新的 server connection。

#### Retry Logic 防止 Pool Exhaustion

當 `cl_waiting > 0` 時，client 可能因 `query_wait_timeout` 超時而收到錯誤。應用層應實作重試機制：

```python
## Python 中的重試範例
import psycopg
import time

def execute_with_retry(conn_str, sql, max_retries=3):
    retry_count = 0
    while retry_count <= max_retries:
        try:
            with psycopg.connect(conn_str) as conn:
                with conn.cursor() as cur:
                    cur.execute(sql)
                    return cur.fetchall()
        except psycopg.OperationalError as e:
            # PgBouncer 的 pool exhaustion 通常表現為 OperationalError
            if "no server connection" in str(e).lower() or \
               "no connection to the server" in str(e).lower():
                retry_count += 1
                if retry_count > max_retries:
                    raise
                wait = 2 ** retry_count * 0.1  # Exponential backoff
                time.sleep(wait)
            else:
                raise
```

```java
// Java / Spring Boot 使用 resilience4j
@Retry(
    name = "pgbouncerRetry",
    fallbackMethod = "fallback",
    retryExceptions = { org.postgresql.util.PSQLException.class }
)
public Result queryDatabase() {
    // 當 PgBouncer 返回 pool exhaustion 錯誤時自動重試
}
```

#### Connection Leak Detection（透過 SHOW CLIENTS）

```sql
-- 查看是否有 client 長時間處於 active 狀態（可能是 connection leak）
SELECT client_addr, state, age(now(), query_start) AS query_age, query
FROM pgbouncer_clients()
WHERE state = 'active'
  AND query_start < now() - interval '5 minutes';
```

定期檢查 `SHOW CLIENTS` 輸出中 `query_start` 欄位過舊的 active client——這可能是 connection leak 的信號。

```bash
## 快速腳本：檢查是否有卡住的 client
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin -d pgbouncer \
  -c "SHOW CLIENTS;" | grep -E "active|waiting"
```

#### 綜合配置清單

```ini
## 生產環境 PgBouncer 配置參考總結
[databases]
* = host=127.0.0.1 port=5432
## 若需要 session pooling 的特定庫：
## mydb_session = host=127.0.0.1 port=5432 dbname=mydb pool_mode=session

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

max_client_conn = 200
max_db_connections = 50
max_user_connections = 50

server_idle_timeout = 600
server_lifetime = 3600
server_connect_timeout = 15
client_idle_timeout = 0
query_timeout = 0
query_wait_timeout = 120

## 1.22+ 新增
max_prepared_statements = 100
track_extra_parameters = 1

logfile = /var/log/postgresql/pgbouncer.log
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

admin_users = pgbouncer_admin
stats_users = pgbouncer_stats
```
## 七、pg_repack — 在線表重組

> **初學者導讀**  
> `pg_repack` 是一個 PostgreSQL 第三方擴展（extension），能在**不阻塞讀寫**的前提下對表進行全面重組（reorganization），回收膨脹空間（bloat）、將空間歸還給作業系統，甚至可選地對資料進行實體排序（CLUSTER 效果）。與原生的 `VACUUM FULL` 不同——後者會用 `AccessExclusiveLock` 鎖住整張表，導致業務中斷——`pg_repack` 只在最後一步進行**毫秒級**的 filenode 交換，對線上業務幾乎無影響。

---

### 1. 核心價值 — 為什麼需要 pg_repack

#### I. 什麼是表膨脹（Bloat）

PostgreSQL 採用 MVCC（Multiversion Concurrency Control，多版本並行控制）機制，`UPDATE` 和 `DELETE` 不會直接覆蓋或刪除舊版本的行（tuple），而是在表中留下**死元組（Dead Tuple）**。`autovacuum` 雖然會自動回收這些死元組佔用的空間，並將其標記為「可重用」，但回收的空間**不會歸還給作業系統**——它仍然保留在該表的資料檔案中，等待未來 `INSERT` 重複利用。

長期高頻率的 `UPDATE`/`DELETE` 會導致以下後果：

```
實際有效資料：10GB
表實際佔用磁碟：50GB（40GB 為無法還給 OS 的空洞）
```

**膨脹的影響**：

| 影響 | 說明 |
|------|------|
| 更大的 Seq Scan 範圍 | 順序掃描需遍歷空洞頁面，即便它們已無有效資料 |
| Buffer Cache 命中率下降 | 大量空洞佔用 shared_buffers，排擠真正的熱資料 |
| I/O 增加 | 讀取大量無效頁面，實體磁碟 I/O 浪費 |
| 索引膨脹 | B-Tree 葉節點因死元組而碎片化，索引掃描效率遞減 |
| 查詢規劃器誤判 | 空洞頁面仍貢獻 `relpages` 估算，導致 planner 高估成本 |

```mermaid
graph TD
    subgraph "正常表（50 頁，全部有效）"
        P1["頁1<br/>✅ 有效"] --> P2["頁2<br/>✅ 有效"] --> P3["頁3<br/>✅ 有效"]
    end
    subgraph "膨脹表（200 頁，僅 50 頁有效）"
        D1["頁1<br/>✅ 有效"] --> D2["頁2<br/>❌ 死元組"] --> D3["頁3<br/>❌ 死元組"] --> D4["頁4<br/>✅ 有效"] --> D5["頁5<br/>❌ 死元組"] --> D6["...180 頁空洞..."] --> D7["頁200<br/>✅ 有效"]
    end
    D1 -.->|"Seq Scan 必須遍歷<br/>所有 200 頁"| D7
    D2 -.->|"佔用 shared_buffers<br/>浪費快取空間"| D3
    style D2 fill:#E74C3C,color:#fff
    style D3 fill:#E74C3C,color:#fff
    style D5 fill:#E74C3C,color:#fff
    style P1 fill:#27AE60,color:#fff
    style P2 fill:#27AE60,color:#fff
    style P3 fill:#27AE60,color:#fff
```

#### II. VACUUM FULL 問題

PostgreSQL 內建的 `VACUUM FULL` 會對整張表進行重組，將有效資料壓縮到連續頁面，並將空間歸還給 OS。但它有一個**致命的缺點**：需要獲取 `AccessExclusiveLock`。

```mermaid
sequenceDiagram
    participant App as 應用程式
    participant PG as PostgreSQL
    participant VF as VACUUM FULL

    App->>PG: SELECT * FROM orders WHERE id = 123
    PG->>App: 返回結果
    Note over VF: 發起 VACUUM FULL
    VF->>PG: 請求 AccessExclusiveLock
    App->>PG: SELECT * FROM orders WHERE id = 456
    PG-->>App: ❌ 等待鎖...（blocked）
    App->>PG: INSERT INTO orders ...
    PG-->>App: ❌ 等待鎖...（blocked）
    Note over VF,PG: 鎖持有期間（數分鐘到數小時）
    VF->>PG: 釋放鎖
    PG->>App: 返回結果（延遲數分鐘）
```

- **`AccessExclusiveLock`** 是 PostgreSQL 中最嚴格的鎖級別，與**所有其他鎖衝突**，包括 `SELECT` 所需的 `AccessShareLock`
- 在生產環境中等同於**停機維護**（downtime maintenance）
- 對大表執行 `VACUUM FULL` 可能需要數十分鐘甚至數小時，且期間完全無法存取該表

#### III. pg_repack 承諾

| 特性 | 說明 |
|------|------|
| 在線重組（Online） | 整個過程不阻塞讀寫，只在最後交換階段鎖表數毫秒 |
| 歸還空間給 OS | 有效解決表膨脹問題，磁碟空間真正縮小 |
| 可選排序（CLUSTER 效果） | 支援 `--order-by` 按指定欄位實體排序，提升範圍查詢效能 |
| 僅索引重組 | `--only-indexes` 選項，當表本身無膨脹但索引碎片化時使用 |
| 表空間遷移 | `--tablespace` 選項，可在重組同時將表遷移到其他表空間（如冷儲存） |

> **補充（Senior Dev）**  
> `pg_repack` 不適合**完全沒有空閒磁碟空間**的場景——它需要約等於原表大小的額外空間來容納新表。如果磁碟已滿，你需要先手動清理無用資料、歸檔舊分區或擴容，才能執行 repack。此外，`pg_repack` 的排序選項僅對 repack 後新寫入的資料無約束力——它不是 CLUSTER 指令的替代品，而是「一次性排序」的快照。

---

### 2. pg_repack 工作原理

#### I. 四階段流程

`pg_repack` 的核心思想是：**建立新表 → 複製資料 → 追蹤增量變更 → 原子交換**。這個過程類似線上 DDL 工具（如 pt-online-schema-change），但專注於空間回收與表重組。

```mermaid
flowchart TD
    subgraph Phase1["階段 1：全量複製（Full Copy）"]
        A1["創建新表（有序寫入）"] --> A2["批量 SELECT + INSERT<br/>從舊表複製所有行"]
        A2 --> A3["複製期間<br/>舊表仍可正常讀寫"]
    end
    subgraph Phase2["階段 2：安裝觸發器"]
        B1["在舊表上安裝<br/>INSERT / UPDATE / DELETE 觸發器"] --> B2["觸發器將增量變更<br/>同步記錄到變更日誌表"]
    end
    subgraph Phase3["階段 3：應用增量變更"]
        C1["將變更日誌中的增量<br/>應用到新表"] --> C2{"有新的<br/>增量產生？"}
        C2 -->|"是"| C1
        C2 -->|"否（追上）"| C3["增量追平，準備切換"]
    end
    subgraph Phase4["階段 4：原子交換"]
        D1["獲取 AccessExclusiveLock<br/>（毫秒級）"] --> D2["交換新舊表的 relfilenode"]
        D2 --> D3["移除觸發器<br/>刪除舊表"]
    end

    Phase1 --> Phase2 --> Phase3 --> Phase4

    Phase1 -.- L1["鎖：AccessShareLock<br/>（不阻塞 DML）"]
    Phase2 -.- L2["鎖：AccessShareLock<br/>（不阻塞 DML）"]
    Phase3 -.- L3["鎖：AccessShareLock<br/>（不阻塞 DML）"]
    Phase4 -.- L4["🔒 AccessExclusiveLock<br/>⏱ 僅數毫秒"]

    style Phase1 fill:#4A90D9,color:#fff
    style Phase2 fill:#E67E22,color:#fff
    style Phase3 fill:#F39C12,color:#fff
    style Phase4 fill:#E74C3C,color:#fff
```

**各階段詳解**：

1. **階段 1 — Full Copy**：`pg_repack` 建立一個與原表結構相同的新表（臨時表），然後使用 `SELECT ... ORDER BY`（如果指定了 `--order-by`）或無序掃描將舊表的所有行批量複製到新表。新表寫入時資料緊湊，無空洞。

2. **階段 2 — Install Triggers**：在全量複製完成後，`pg_repack` 在原表上安裝三個觸發器（`INSERT`、`UPDATE`、`DELETE`），將此後的所有資料變更記錄到一張內部的日誌表（log table）。這樣可以捕捉全量複製期間和之後產生的增量。

3. **階段 3 — Apply Incremental Changes**：`pg_repack` 將日誌表中記錄的增量變更應用到新表。因為在應用期間可能又有新的增量，此階段會**循環迭代**，直到增量趨近於零（或低於閾值），確保新舊表資料一致。

4. **階段 4 — Atomic Filenode Swap**：在獲取短暫的 `AccessExclusiveLock` 後，`pg_repack` 執行原子性的 `pg_relation_filenode_swap()` 操作——交換新舊表的實體檔案節點（filenode）。交換完成後立即釋放鎖，舊表被刪除，新表繼承原表的 OID 與名稱。**鎖定時間通常僅數毫秒**（取決於當前的活躍交易），對業務透明。

#### II. 鎖定時間分析

| 階段 | 鎖類型 | 相容性 | 鎖定時間 | 業務影響 |
|------|--------|--------|---------|---------|
| 階段 1 | `AccessShareLock` | 與 `SELECT` / DML 相容 | 取決於表大小（分鐘到小時） | 無阻塞 |
| 階段 2 | `AccessShareLock` | 同上 | 毫秒級（建立觸發器） | 無阻塞 |
| 階段 3 | `AccessShareLock` | 同上 | 取決於增量追趕（秒到分鐘） | 無阻塞 |
| 階段 4 | `AccessExclusiveLock` | 與**所有其他鎖**衝突 | **毫秒級** | 瞬時阻塞 |

> **補充（Senior Dev）**  
> 階段 4 的鎖定時間雖然通常只有幾毫秒，但存在一個理論上的最壞情況：如果存在長時間執行的交易（long-running transaction），`AccessExclusiveLock` 必須**等待所有既有鎖釋放**後才能獲取。這不是 `pg_repack` 本身的問題，而是 PostgreSQL 鎖隊列（lock queue）機制使然：`AccessExclusiveLock` 必須排在所有現有 `AccessShareLock` 之後。因此，建議在執行 `pg_repack` 前，先檢查 `pg_stat_activity` 中是否有長時間未提交的交易並處理之。

---

### 3. 安裝與配置（PG 16）

#### 3.1 安裝擴展套件

```bash
## Debian / Ubuntu（PGDG APT repo）
sudo apt install postgresql-16-repack

## RHEL / Rocky Linux（PGDG YUM repo）
sudo dnf install pg_repack_16

## 從原始碼編譯
git clone https://github.com/reorg/pg_repack.git
cd pg_repack
make && sudo make install
```

#### 3.2 在資料庫中註冊擴展

```sql
-- 在目標資料庫中執行（需超級使用者權限）
CREATE EXTENSION pg_repack;
```

驗證安裝：

```sql
SELECT * FROM pg_extension WHERE extname = 'pg_repack';
-- 應返回一條記錄，確認擴展已註冊

-- 或檢查可用功能
\dx pg_repack
```

#### 3.3 注意事項

- `pg_repack` 的二進位檔（`pg_repack` 命令列工具）必須與 PostgreSQL 伺服器版本一致（16.x 對應 `postgresql-16-repack`）
- 擴展需要安裝在**目標資料庫**中，而非 `template1` 或 `postgres` 資料庫
- 執行 `pg_repack` 的系統使用者需要能夠以 PostgreSQL 角色（通常為 superuser 或擁有 `pg_repack` 權限的角色）連接資料庫

---

### 4. 核心操作

#### I. 在線重組表

```bash
## === 基本：重組單一表 ===
pg_repack -d mydb -t public.large_table

## === 帶排序（CLUSTER 效果）：按 event_time 排序儲存 ===
pg_repack -d mydb -t public.events --order-by=event_time

## === 遷移表空間：將冷資料移到低速儲存 ===
pg_repack -d mydb -t public.old_events --tablespace=cold_storage

## === 僅重組索引：表本身無膨脹但索引碎片化 ===
pg_repack -d mydb -t public.large_table --only-indexes

## === 重組單一索引 ===
pg_repack -d mydb -i idx_events_time

## === 重組整個資料庫的所有表 ===
pg_repack -d mydb

## === 重組特定 schema 的所有表 ===
pg_repack -d mydb -n public

## === 排除特定表 ===
pg_repack -d mydb --exclude-table=public.sessions --exclude-table=public.logs

## === 等待被阻塞的鎖（而非直接超時） ===
pg_repack -d mydb -t public.large_table --wait-timeout=60

## === 使用多個 Job 平行處理（pg_repack 1.5+） ===
pg_repack -d mydb -t public.large_table --jobs=4
```

**完整選項對照表**：

| 選項 | 說明 | 範例 |
|------|------|------|
| `-d, --dbname` | 目標資料庫名稱 | `-d mydb` |
| `-t, --table` | 目標表（可指定多次） | `-t public.orders` |
| `-n, --schema` | 目標 schema | `-n public` |
| `-i, --index` | 目標索引 | `-i idx_orders_date` |
| `--order-by` | 按指定欄位排序儲存 | `--order-by=created_at` |
| `--tablespace` | 遷移到指定表空間 | `--tablespace=ssd_storage` |
| `--only-indexes` | 僅重組索引，不重建表 | `--only-indexes` |
| `--exclude-table` | 排除特定表 | `--exclude-table=public.logs` |
| `--wait-timeout` | 等待鎖的秒數 | `--wait-timeout=30` |
| `--jobs` | 平行執行的工作數（1.5+） | `--jobs=4` |
| `--no-order` | 不強制排序（略快，空間回收效果較差） | `--no-order` |
| `--dry-run` | 僅顯示將要執行的操作，不實際執行 | `--dry-run` |
| `-h, --host` | 資料庫主機 | `-h 10.0.1.50` |
| `-p, --port` | 資料庫埠號 | `-p 5432` |
| `-U, --username` | 連接使用者 | `-U postgres` |

#### II. 監控進度

```sql
-- 查看 pg_repack 目前的查詢活動
SELECT pid, state, wait_event_type, wait_event,
       query,
       now() - query_start AS duration
FROM pg_stat_activity
WHERE query LIKE '%repack%'
  AND state != 'idle';

-- 查看表大小（repack 前後對比）
SELECT pg_size_pretty(pg_total_relation_size('public.large_table'))
       AS size_before;
-- 執行 pg_repack ...
-- 再次執行上方的查詢，對比 size_after
```

```bash
## 在另一個終端監控磁碟使用量（repack 期間需 2x 空間）
watch -n 5 'df -h /var/lib/postgresql/16/main'

## 監控 WAL 產生量
watch -n 5 'psql -d mydb -c "SELECT pg_current_wal_lsn(), pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), pg_last_wal_receive_lsn()))"'
```

#### III. 注意事項

| 注意事項 | 說明 |
|----------|------|
| 必須有主鍵或 UNIQUE NOT NULL | 用於觸發器追蹤增量變更；無主鍵的表無法 repack |
| 需要 2 倍磁碟空間 | 新表大小約等於原表的有效資料量，但期間會同時存在新舊兩張表 |
| 大量 WAL 產生 | 全量複製相當於一次完整的資料重寫，WAL 產生量激增，可能導致 replication slot 延遲 |
| 不支援分割區主表 | 需對每個分割區單獨執行 `pg_repack`（PG 16 原生分割區） |
| TOAST 表自動處理 | `pg_repack` 會自動重組相關聯的 TOAST 表和索引 |
| 觸發器短暫存在 | 階段 2 ~ 4 期間，原表上存在 pg_repack 的內部觸發器，不影響業務觸發器 |

```mermaid
graph TD
    subgraph "磁碟空間與 WAL 警告"
        A["pg_repack 執行中"] --> B["磁碟使用量"]
        A --> C["WAL 產生量"]
        B --> D{"可用空間 ><br/>原表大小 × 2？"}
        D -->|"否"| E["❌ 磁碟滿 → repack 失敗<br/>資料庫可能崩潰"]
        D -->|"是"| F["✅ 安全"]
        C --> G{"Replication Slot<br/>有滯後風險？"}
        G -->|"是"| H["⚠️ 考慮暫停 replica<br/>或在低峰期執行"]
        G -->|"否"| I["✅ 安全"]
    end
    style E fill:#E74C3C,color:#fff
    style H fill:#F39C12,color:#fff
    style F fill:#27AE60,color:#fff
    style I fill:#27AE60,color:#fff
```

> **補充（Senior Dev）**  
> `pg_repack` 產生的 WAL 量約等於**全表資料量的 1~2 倍**（取決於是否包含索引重建）。如果使用了 streaming replication，請提前檢查 `pg_stat_replication` 中的 `write_lag` / `flush_lag` 指標，並確保 `wal_keep_size` 或 replication slot 有足夠的保留容量。對於數百 GB 級別的表，建議先對小表測試以評估 WAL 產生速率，再決定是否需要在維護窗口內執行。

---

### 5. VACUUM FULL vs pg_repack vs pg_squeeze 對比

三種方案都是為了解決表膨脹（bloat）問題，但它們在設計理念、鎖定行為和適用場景上有本質差異。

| 維度 | VACUUM FULL | pg_repack | pg_squeeze |
|------|-------------|-----------|------------|
| **鎖類型** | `AccessExclusiveLock`（全程） | 階段 1~3：`AccessShareLock`；階段 4：`AccessExclusiveLock`（毫秒級） | 透過邏輯複製解碼 WAL，無長時間鎖 |
| **鎖定時間** | 分鐘到小時 | 毫秒級 | 接近零（短暫排他鎖） |
| **阻塞讀寫** | 完全阻塞 | 不阻塞（最後毫秒級除外） | 不阻塞 |
| **磁碟空間需求** | 約 1x 原表大小（原地重組） | 約 2x 原表大小（新舊表並存） | 約 2x 原表大小 |
| **速度** | 最快（直接原地操作） | 中等（全量複製 + 增量追趕） | 較慢（依賴 logical decoding） |
| **WAL 產生量** | 中等 | 大量（全表重寫） | 中等（僅增量） |
| **需要主鍵** | 否 | 是 | 是 |
| **成熟度** | PostgreSQL 內建（最穩定） | 社群活躍維護，生產驗證廣泛 | 較新，社群活躍度中等 |
| **離峰執行需求** | 強制（完全停機） | 建議（降低增量追趕輪次） | 可隨時執行 |
| **適用場景** | 停機維護窗口、開發環境 | 定期線上維護、大表重組 | 無法接受任何鎖等待的極端場景 |

```mermaid
graph TD
    A["發現表膨脹"] --> B{"是否可以<br/>停機維護？"}
    B -->|"是"| C["VACUUM FULL<br/>（最快，無額外需求）"]
    B -->|"否"| D{"表是否有<br/>主鍵 或 UNIQUE NOT NULL？"}
    D -->|"否"| E["⚠️ 無法使用 pg_repack<br/>考慮清理無用資料或手動遷移"]
    D -->|"是"| F{"磁碟空間是否<br/>> 表大小 × 2？"}
    F -->|"否"| G["⚠️ 先擴容磁碟<br/>或分批處理"]
    F -->|"是"| H{"是否有活躍的<br/>Streaming Replica？"}
    H -->|"是（WAL 滯後風險）"| I["pg_squeeze<br/>（WAL 產生量較低）"]
    H -->|"否"| J["pg_repack<br/>（最成熟的在線方案）"]
    H -->|"每次寫入量可控"| J

    style C fill:#E74C3C,color:#fff
    style J fill:#27AE60,color:#fff
    style I fill:#4A90D9,color:#fff
    style E fill:#E74C3C,color:#fff
    style G fill:#F39C12,color:#fff
```

> **補充（Senior Dev）**  
> `pg_squeeze` 使用 PostgreSQL 的邏輯複製（logical decoding）機制來追蹤增量變更，這意味著它**不需要在表上安裝觸發器**。這對已有大量業務觸發器的表來說是一個優勢。但邏輯複製本身對 WAL 有依賴：如果 WAL 被過快回收（例如 `wal_keep_size` 過小或 replication slot 未正確設定），pg_squeeze 會失敗。取捨：`pg_repack` 用觸發器（簡單粗暴但可靠），`pg_squeeze` 用邏輯解碼（精緻但對 WAL 配置敏感）。

---

### 6. 配合 pg_cron 自動化

將 `pg_repack` 與 `pg_cron` 結合，可實現定時自動化表維護。

#### 6.1 cron.schedule 設定

```sql
-- 每週日凌晨 3 點重組特定表
SELECT cron.schedule(
    'weekly-repack-large-table',
    '0 3 * * 0',
    'SELECT pg_repack.repack_table(''public.large_table'', ''created_at'')'
);

-- 每日凌晨 4 點重組高流量表的索引
SELECT cron.schedule(
    'daily-index-repack',
    '0 4 * * *',
    'SELECT pg_repack.repack_index(''idx_events_time'')'
);
```

> **注意**：`pg_cron` 無法直接呼叫外部命令 `pg_repack`，需要透過 PL/Proxy 函數或外部包裝指令碼實現。以下是推薦的 Shell 包裝方案。

#### 6.2 Shell 包裝指令碼（含日誌與通知）

```bash
#!/bin/bash
## /usr/local/bin/auto_repack.sh
## 定時 repack 包裝指令碼 — 透過 crontab 或 pg_cron 觸發

set -euo pipefail

DB_NAME="mydb"
TABLE_NAME="public.events"
LOG_FILE="/var/log/pg_repack/repack_$(date +%Y%m%d_%H%M%S).log"
LOCK_TIMEOUT=30
THRESHOLD_GB=5  # 僅當膨脹超過 5GB 時才 repack

## 確保日誌目錄存在
mkdir -p "$(dirname "$LOG_FILE")"

## 1. 檢查磁碟空間
DISK_AVAIL_GB=$(df -BG /var/lib/postgresql/16/main | awk 'NR==2 {print $4}' | sed 's/G//')
TABLE_SIZE_GB=$(psql -d "$DB_NAME" -Atc \
    "SELECT pg_total_relation_size('$TABLE_NAME') / 1073741824")

if [ "$DISK_AVAIL_GB" -lt $((TABLE_SIZE_GB * 2)) ]; then
    echo "[$(date)] ❌ 磁碟空間不足：可用 ${DISK_AVAIL_GB}GB，需要 ${TABLE_SIZE_GB}x2 GB" \
        | tee -a "$LOG_FILE"
    exit 1
fi

## 2. 檢查膨脹閾值（可選）
BLOAT_GB=$(psql -d "$DB_NAME" -Atc \
    "SELECT pg_size_pretty(
        pg_total_relation_size('$TABLE_NAME') -
        pg_relation_size('$TABLE_NAME', 'main')
    )")
echo "[$(date)] ℹ️  表膨脹估算：$BLOAT_GB" | tee -a "$LOG_FILE"

## 3. 記錄開始時間與原始大小
SIZE_BEFORE=$(psql -d "$DB_NAME" -Atc \
    "SELECT pg_size_pretty(pg_total_relation_size('$TABLE_NAME'))")
echo "[$(date)] 🔄 開始 repack：$TABLE_NAME（原大小：$SIZE_BEFORE）" \
    | tee -a "$LOG_FILE"

## 4. 執行 pg_repack
START_TIME=$(date +%s)

if pg_repack \
    --dbname="$DB_NAME" \
    --table="$TABLE_NAME" \
    --wait-timeout="$LOCK_TIMEOUT" \
    --order-by=created_at \
    --no-superuser-check \
    >> "$LOG_FILE" 2>&1; then

    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))
    SIZE_AFTER=$(psql -d "$DB_NAME" -Atc \
        "SELECT pg_size_pretty(pg_total_relation_size('$TABLE_NAME'))")

    echo "[$(date)] ✅ repack 完成（耗時 ${DURATION}s）" | tee -a "$LOG_FILE"
    echo "[$(date)] 📏 $TABLE_NAME：$SIZE_BEFORE → $SIZE_AFTER" | tee -a "$LOG_FILE"

    # 5. 可選：發送通知（Slack / 郵件）
    # curl -X POST -H 'Content-type: application/json' \
    #     --data "{\"text\":\"pg_repack 完成：$TABLE_NAME $SIZE_BEFORE → $SIZE_AFTER (${DURATION}s)\"}" \
    #     "$SLACK_WEBHOOK_URL"

else
    echo "[$(date)] ❌ repack 失敗，請檢查日誌：$LOG_FILE" | tee -a "$LOG_FILE"
    # 可選：發送告警
    exit 1
fi
```

```bash
## crontab 設定（每日凌晨 3:00）
## 0 3 * * * /usr/local/bin/auto_repack.sh
```

---

### 7. App Dev 最佳實踐

#### 7.1 定期 Repack 排程（離峰時段）

```ini
## 建議：每月/每週在離峰時段執行（取決於表的寫入頻率和膨脹速度）

## 高寫入頻率表（日誌、事件、交易記錄）
## → 每月 1~2 次

## 中等寫入頻率表（使用者、訂單）
## → 每季 1 次

## 低寫入頻率表（配置、字典）
## → 按需執行，透過膨脹監控自動觸發
```

#### 7.2 磁碟空間監控指令碼

```sql
-- 查詢所有表的膨脹率（需 pgstattuple 擴展）
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT schemaname || '.' || relname AS table_name,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size,
       pg_size_pretty(
           (SELECT free_space FROM pgstattuple(schemaname || '.' || relname))
       ) AS dead_tuple_space,
       round(
           (SELECT free_percent FROM pgstattuple(schemaname || '.' || relname))
       , 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE pg_total_relation_size(schemaname || '.' || relname) > 1073741824  -- > 1GB
ORDER BY (SELECT free_space FROM pgstattuple(schemaname || '.' || relname)) DESC
LIMIT 20;
```

> **注意**：`pgstattuple` 對大表會進行全表掃描，生產環境謹慎使用，建議在離峰時段或 Staging 環境執行。

#### 7.3 Replication Slot 監控

```sql
-- 檢查 Streaming Replica 的延遲情況
SELECT client_addr,
       state,
       sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sent_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```

```bash
## Shell 監控腳本 — 在 repack 執行期間持續監控
while true; do
    psql -d mydb -c "
        SELECT now(),
               pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
               replay_lag
        FROM pg_stat_replication
        WHERE application_name = 'my_replica';
    "
    sleep 10
done
```

#### 7.4 Transaction ID Wraparound 考量

`pg_repack` 在重組過程中會產生大量交易 ID（XID）消耗——全表複製的每一行都涉及一個新的 XID。如果資料庫的 XID 消耗速度本就接近上限（`autovacuum_freeze_max_age`），執行 repack 可能加速逼近 **Transaction ID Wraparound** 閾值。

```sql
-- 檢查資料庫的年齡（age）
SELECT datname,
       age(datfrozenxid) AS xid_age,
       pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**建議**：在執行大型 repack 之前，先手動執行一次 `VACUUM FREEZE` 降低 XID 年齡。

#### 7.5 Production 包裝指令碼範例

以下是一個適用於生產環境的完整 repack 指令碼，包含**前置檢查、執行期監控、失敗回滾策略**：

```bash
#!/bin/bash
## /usr/local/bin/production_repack.sh
## 生產環境 pg_repack 安全包裝指令碼

set -euo pipefail

## === 配置區 ===
DB_HOST="10.0.1.50"
DB_PORT="5432"
DB_NAME="production_db"
DB_USER="postgres"
TABLE_NAME="$1"                         # 從命令列參數取得表名
WAIT_TIMEOUT=60
MIN_DISK_FREE_GB=50                     # 最低可用磁碟空間（GB）
LOG_DIR="/var/log/pg_repack"
NOTIFY_HOOK="/usr/local/bin/notify_ops.sh"  # 可選：通知指令碼

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="$LOG_DIR/repack_${TABLE_NAME//./_}_${TIMESTAMP}.log"

mkdir -p "$LOG_DIR"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

## === 前置檢查 ===

## 0. 檢查參數
if [ -z "${TABLE_NAME:-}" ]; then
    echo "Usage: $0 <schema.table_name>"
    exit 1
fi

## 1. 檢查資料庫連線
if ! psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -c "SELECT 1" > /dev/null 2>&1; then
    log "❌ 無法連接到資料庫 $DB_NAME@$DB_HOST:$DB_PORT"
    exit 1
fi
log "✅ 資料庫連線正常"

## 2. 檢查磁碟空間
PGDATA=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc "SHOW data_directory")
DISK_AVAIL_GB=$(df -BG "$PGDATA" | awk 'NR==2 {print $4}' | sed 's/G//')
TABLE_SIZE_BYTES=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
    "SELECT pg_total_relation_size('$TABLE_NAME')")
TABLE_SIZE_GB=$((TABLE_SIZE_BYTES / 1073741824))

log "📏 表 $TABLE_NAME 大小：${TABLE_SIZE_GB}GB"
log "💾 可用磁碟空間：${DISK_AVAIL_GB}GB"

if [ "$DISK_AVAIL_GB" -lt $((TABLE_SIZE_GB * 2)) ]; then
    log "❌ 磁碟空間不足（需至少 ${TABLE_SIZE_GB}x2 = $((TABLE_SIZE_GB * 2))GB）"
    exit 1
fi
if [ "$DISK_AVAIL_GB" -lt "$MIN_DISK_FREE_GB" ]; then
    log "❌ 可用空間低於安全閾值（${MIN_DISK_FREE_GB}GB）"
    exit 1
fi
log "✅ 磁碟空間充足"

## 3. 檢查長時間執行的交易
LONG_TX_COUNT=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
    "SELECT count(*) FROM pg_stat_activity
     WHERE state = 'active'
       AND now() - xact_start > interval '5 minutes'
       AND pid != pg_backend_pid()")
if [ "$LONG_TX_COUNT" -gt 0 ]; then
    log "⚠️  警告：存在 $LONG_TX_COUNT 個超過 5 分鐘的交易（可能導致階段 4 鎖等待）"
fi

## 4. 記錄 repack 前大小
SIZE_BEFORE=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
    "SELECT pg_size_pretty(pg_total_relation_size('$TABLE_NAME'))")
INDEX_SIZES=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
    "SELECT indexrelid::regclass || ': ' || pg_size_pretty(pg_relation_size(indexrelid))
     FROM pg_index WHERE indrelid = '$TABLE_NAME'::regclass")

log "🔄 開始 repack：$TABLE_NAME（當前總大小：$SIZE_BEFORE）"

## === 執行 repack ===
START_EPOCH=$(date +%s)

## 啟動背景監控（WAL 產生量 & 複製延遲）
monitor_bg() {
    while true; do
        local lag=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
            "SELECT coalesce(max(replay_lag)::text, 'no replica') FROM pg_stat_replication")
        echo "[$(date +%H:%M:%S)] Replay Lag: $lag" >> "$LOG_FILE.monitor"
        sleep 30
    done
}
monitor_bg &
MONITOR_PID=$!

if pg_repack \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --table="$TABLE_NAME" \
    --wait-timeout="$WAIT_TIMEOUT" \
    --no-superuser-check \
    >> "$LOG_FILE" 2>&1; then

    END_EPOCH=$(date +%s)
    DURATION=$((END_EPOCH - START_EPOCH))
    SIZE_AFTER=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -Atc \
        "SELECT pg_size_pretty(pg_total_relation_size('$TABLE_NAME'))")

    log "✅ Repack 成功完成"
    log "⏱  耗時：${DURATION} 秒（$(date -u -d @${DURATION} +%H:%M:%S)）"
    log "📏 大小變化：$SIZE_BEFORE → $SIZE_AFTER"

    if [ -x "$NOTIFY_HOOK" ]; then
        "$NOTIFY_HOOK" "pg_repack 完成" \
            "表：$TABLE_NAME\n耗時：${DURATION}s\n大小：$SIZE_BEFORE → $SIZE_AFTER"
    fi

    EXIT_CODE=0
else
    log "❌ Repack 失敗，請檢查日誌：$LOG_FILE"

    if [ -x "$NOTIFY_HOOK" ]; then
        "$NOTIFY_HOOK" "pg_repack 失敗" \
            "表：$TABLE_NAME\n日誌：$LOG_FILE"
    fi

    EXIT_CODE=1
fi

## 清理背景監控
kill $MONITOR_PID 2>/dev/null || true

log "📝 完整日誌：$LOG_FILE"
exit $EXIT_CODE
```

#### 7.6 注意事項彙總

| 注意事項 | 說明 |
|----------|------|
| 主鍵必要 | 無主鍵/UNIQUE NOT NULL 的表無法使用 pg_repack |
| 2x 磁碟空間 | repack 期間需容納新舊兩張表 |
| WAL 激增 | 對 Streaming Replica 可能造成延遲，建議在維護窗口執行 |
| XID 消耗 | 大表 repack 消耗大量 XID，提前執行 VACUUM FREEZE |
| 分區表 | 需逐個分區執行，無法對父表執行 |
| 中斷處理 | 若 repack 中斷，pg_repack 會自動清理臨時表（leave_cleanup） |
| 事件觸發器 | `DDL_command_end` 事件觸發器不會被 pg_repack 的內部操作觸發 |
| 邏輯複製 | 目標表作為邏輯複製的發布端時，repack 期間增量變更仍正常解碼 |
## 八、pg_cron — PG 內建排程作業

> **初學者導讀**
> `pg_cron` 是一款 PostgreSQL extension，能讓你在資料庫**內部**執行定時任務（scheduled job）。不同於 Linux `crontab` 或 CI/CD pipeline 中的排程，pg_cron 的任務跑在 PostgreSQL 本身的 Background Worker 程序中，你和 DBA 可以用熟悉的 SQL 語法定義、管理、監控所有定期作業——從定時 VACUUM、分割區維護、Materialized View 刷新，到資料清理和 GDPR 合規任務，全部在一個地方搞定。

```mermaid
graph LR
    subgraph "❌ 傳統外部排程"
        A["crontab / systemd timer"] -->|"網路呼叫"| B["psql / REST API"]
        B --> C["PostgreSQL"]
    end
    subgraph "✅ pg_cron 內部排程"
        D["pg_cron BGW"] -->|"程序內執行"| E["PostgreSQL"]
    end
    F["網路中斷風險"] -.->|"❌"| A
    G["交易一致性保證"] -.->|"✅"| D
    H["單一管理介面"] -.->|"✅"| D
    style A fill:#E74C3C,color:#fff
    style B fill:#E67E22,color:#fff
    style D fill:#27AE60,color:#fff
    style E fill:#4a90d9,color:#fff
```

---

### 1. 核心價值

#### I. 為什麼需要資料庫內建排程

外部排程工具（Linux `crontab`、Kubernetes `CronJob`、Airflow 等）在處理資料庫維護任務時，存在三個結構性缺陷：

| 問題 | 外部排程 | pg_cron |
|------|----------|---------|
| **網路依賴** | 任務需通過 TCP 連線到 PG，網路抖動或 DNS 故障會導致任務失敗 | 程序內執行，無網路單點故障（SPOF） |
| **交易安全** | `psql -c "VACUUM ..."` 執行後若連線中斷，無法得知任務是否完成 | BGW 在 PG 內部執行，transactional safety 由 PG 自身保證 |
| **維護成本** | 多一個移動部件（moving part）：需管理 crontab 語法、權限、日誌、監控 | 全部在 `cron.job` 表中管理，一個 `SELECT` 即可查看狀態 |

> **補充（Senior Dev）**
> 在容器化環境中，外部排程的問題更為突出：Kubernetes `CronJob` 每次執行都要啟動一個新 Pod，冷啟動延遲可能高達數秒；若搭配 sidecar 模式，還要在 Pod 內安裝 `psql` 客戶端並管理連線憑證。pg_cron 把這些複雜性全部吸收進 PG 核心，只需一次安裝，終身受益。

#### II. 常見應用場景

```mermaid
mindmap
  root((pg_cron<br/>典型場景))
    資料維護
      定時 VACUUM / ANALYZE
      分割區自動管理
      Materialized View 刷新
      索引重建 REINDEX CONCURRENTLY
    資料清理
      GDPR 資料保留合規
      稽核日誌歸檔
      臨時表清理
      過期 Session 清除
    監控與警報
      定期健康檢查
      慢查詢統計彙總
      磁碟空間監控
    業務邏輯
      每日報表預計算
      資料聚合與快取
      批次通知推送
```

---

### 2. 架構

#### I. Background Worker 實作

pg_cron 基於 PostgreSQL 的 **Background Worker（BGW）** 框架實作。PostgreSQL 9.4 引入的 BGW 機制允許 extension 註冊一個長期執行的後台程序，該程序可安全地執行 SQL 查詢（透過 SPI，Server Programming Interface）。

```mermaid
graph TD
    subgraph "PostgreSQL 程序群"
        PM["Postmaster<br/>主控程序"]
        PS["psql Client<br/>用戶端連線"]
        CK["Checkpointer<br/>檢查點寫入"]
        BG["pg_cron BGW<br/>排程程序"]
    end

    subgraph "pg_cron 內部結構"
        Scheduler["排程引擎<br/>Cron Scheduler"]
        JobQueue["任務佇列<br/>Job Queue"]
        Runner["任務執行器<br/>Job Runner<br/>（多個 BGW）"]
    end

    subgraph "中繼資料表"
        M1[("cron.job<br/>任務定義")]
        M2[("cron.job_run_details<br/>執行日誌")]
    end

    PM --> BG
    BG --> Scheduler
    Scheduler -->|"每分鐘輪詢<br/>poll every minute"| M1
    Scheduler -->|"符合排程"| JobQueue
    JobQueue -->|"分派任務"| Runner
    Runner -->|"SPI 執行 SQL"| PS
    Runner -->|"記錄結果"| M2
    Runner -.->|"PG 16: 多 BGW<br/>cron.max_workers"| Runner

    style PM fill:#4a90d9,color:#fff
    style BG fill:#e8d44d,color:#333
    style Scheduler fill:#e8d44d,color:#333
    style Runner fill:#5cb85c,color:#fff
    style M1 fill:#4a90d9,color:#fff
    style M2 fill:#4a90d9,color:#fff
```

架構設計的關鍵點：

1. **輪詢機制**：排程引擎每秒（或由 `cron.use_background_workers` 控制）檢查 `cron.job` 表，找出該執行的任務。
2. **任務分派**：PG 16 起支援多個 BGW 並行執行任務（`cron.max_workers`），避免單一長任務阻塞排程。
3. **SPI 隔離**：每個任務透過 SPI 獨立建立資料庫連線，與用戶端連線完全隔離，不受 `connection_limit` 限制。

#### II. cron.job 中繼資料表

`cron.job` 是 pg_cron 的核心配置表，每一行代表一個定時任務：

```sql
-- 查看所有已註冊的任務
SELECT jobid, schedule, command, nodename, nodeport, database, username, active, jobname
FROM cron.job;
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| `jobid` | `bigint` | 自動遞增的唯一 ID，用於 `unschedule()` / `alter_job()` |
| `schedule` | `text` | 標準 cron 表達式，支持秒級精度（6 段式） |
| `command` | `text` | 要執行的 SQL 語句或函數呼叫 |
| `nodename` | `text` | 任務執行的節點名稱（預設 `localhost`，用於多節點部署） |
| `nodeport` | `int` | 任務執行的埠號（預設與當前 PG instance 相同） |
| `database` | `text` | 任務執行的目標資料庫（預設為 `cron.database_name` 設定的資料庫） |
| `username` | `text` | 任務執行的角色（預設為 `cron.host` 設定的使用者） |
| `active` | `boolean` | 是否啟用（`true` / `false`），可動態啟停，不需刪除任務 |
| `jobname` | `text` | 任務名稱（PG 17+），用於 `human-friendly` 識別，可選 |

#### III. cron.job_run_details 執行日誌

每次任務執行都會在 `cron.job_run_details` 中留下一筆詳細記錄：

```sql
-- 查看最近的任務執行日誌
SELECT jobid, runid, job_pid, database, username, command,
       status, return_message, start_time, end_time
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 20;
```

| 欄位 | 說明 |
|------|------|
| `jobid` | 關聯到 `cron.job.jobid` |
| `runid` | 單次執行的唯一 ID（自增） |
| `job_pid` | 執行該任務的 OS 程序 PID（用於 `kill` 卡住的任務） |
| `status` | 執行狀態：`starting`、`running`、`sending`、`connecting`、`succeeded`、`failed` |
| `return_message` | 錯誤訊息（成功則為 `OK` 或 SQL 的回傳結果） |
| `start_time` / `end_time` | 任務開始與結束的時間戳 |

#### IV. 任務生命週期狀態機

```mermaid
stateDiagram-v2
    [*] --> Starting: cron.job 符合排程時間
    Starting --> Connecting: 建立 SPI 連線
    Connecting --> Sending: 發送 SQL 命令
    Sending --> Running: SQL 開始執行
    Running --> Succeeded: SQL 執行成功
    Running --> Failed: SQL 執行失敗 / 逾時
    Failed --> Starting: 下一個排程週期重試
    Succeeded --> Starting: 下一個排程週期執行

    note right of Starting: 排程引擎標記任務為 starting
    note right of Failed: 錯誤資訊寫入 return_message 欄位
```

---

### 3. 安裝與配置（PG 16）

#### 3.1 套件安裝

```bash
## Debian / Ubuntu
sudo apt install postgresql-16-cron

## RHEL / Rocky / AlmaLinux
sudo dnf install pg_cron_16

## 從原始碼編譯
git clone https://github.com/citusdata/pg_cron.git
cd pg_cron
make && sudo make install
```

#### 3.2 postgresql.conf 配置

```ini
## ==========================================
## pg_cron 基本配置（必須設定）
## ==========================================

## 必須載入 pg_cron 庫（需要重啟 PG）
shared_preload_libraries = 'pg_cron'

## 設定 pg_cron 管理的目標資料庫
## 所有 cron.job 中未指定 database 的任務，預設在此資料庫執行
cron.database_name = 'your_db'

## ==========================================
## PG 16 專屬配置項
## ==========================================

## 🆕 是否使用 Background Workers 並行執行任務
## on:  每個符合排程的任務在獨立的 BGW 中執行（PG 16 推薦）
## off: 所有任務在單一 cron launcher 程序中順序執行
cron.use_background_workers = on

## 🆕 並行執行的最大任務數量（僅在 use_background_workers = on 時生效）
## 當達到上限時，新任務需排隊等待
cron.max_workers = 5

## 🆕 單一任務的最大執行時間（毫秒）
## 超過此時間的任務會被強制終止（timeout kill）
## 0 表示不限制（謹慎使用）
cron.max_job_execution_time = 0

## ==========================================
## 通用配置項
## ==========================================

## 是否將任務執行結果寫入 PostgreSQL 日誌
## on：每次執行都會記錄（類似 auto_explain 的日誌輸出）
## off：只記錄到 cron.job_run_details 表
cron.log_run = on

## 是否記錄每個任務執行的 SQL 語句內容
## on：便於除錯，但日誌量較大
cron.log_statement = on

## 同時執行中的最大任務數（不分新舊行為，全部任務共用此限制）
## 若無設定 cron.use_background_workers，此值等同於並行上限
cron.max_running_jobs = 5

## 排程引擎使用的時區（影響 cron 表達式的時間基準）
## 建議與應用層時區一致，避免夏令時間（DST）混淆
cron.timezone = 'Asia/Taipei'

## pg_cron 使用的連線主機（通常為 localhost）
cron.host = 'localhost'
```

#### 3.3 重啟並驗證

修改 `postgresql.conf` 後**必須重啟 PostgreSQL**（`pg_ctl restart` 或 `systemctl restart postgresql`），`pg_reload_conf()` **無效**——因為 `shared_preload_libraries` 只在程序啟動時載入。

```sql
-- 建立 extension（在目標資料庫中執行）
CREATE EXTENSION pg_cron;

-- 檢查 pg_cron 相關 GUC 是否生效
SELECT name, setting, unit, context
FROM pg_settings
WHERE name LIKE 'cron.%'
ORDER BY name;
```

```sql
-- 檢查 pg_cron BGW 是否正常運行
SELECT backend_type, pid, state, backend_start
FROM pg_stat_activity
WHERE backend_type = 'pg_cron launcher';
-- 預期結果：一筆 backend_type = 'pg_cron launcher' 的記錄

-- 若 use_background_workers = on，還可看到任務執行中的 worker
SELECT backend_type, pid, state, backend_start
FROM pg_stat_activity
WHERE backend_type ILIKE 'pg_cron%';
```

> **補充（Senior Dev）**
> PG 16 引入 `cron.use_background_workers = on` 後，pg_cron 的行為從「單線程順序執行」變為「多工作程序並行執行」。這意味著：
> - **好處**：一個長任務不會阻塞其他任務（如一個跑 2 分鐘的 VACUUM 不會擋住每 5 秒一次的 heartbeat check）
> - **風險**：若 `cron.max_workers` 與 `cron.max_running_jobs` 設定不當，大量並行任務可能耗盡系統資源，尤其是 I/O 密集型任務（如多個 `REFRESH MATERIALIZED VIEW CONCURRENTLY`）
> - **建議**：從 `cron.max_workers = 3` 開始，逐步上調，觀察 `pg_stat_activity` 中的 active backend 數量

---

### 4. 核心操作

```mermaid
sequenceDiagram
    participant DBA as DBA / App Dev
    participant Cron as pg_cron Scheduler
    participant Job as cron.job
    participant Detail as cron.job_run_details
    participant PG as PostgreSQL Executor

    DBA->>Cron: cron.schedule('job-name', '0 2 * * *', 'VACUUM orders')
    Cron->>Job: INSERT INTO cron.job (schedule, command, ...)
    Cron-->>DBA: 返回 jobid = 1

    Note over Cron: 每分鐘檢查一次 cron.job
    loop 每天凌晨 2:00
        Cron->>Job: SELECT * WHERE schedule matches now()
        Cron->>Detail: INSERT status = 'starting'
        Cron->>PG: 執行 VACUUM orders
        PG-->>Cron: 執行完成
        Cron->>Detail: UPDATE status = 'succeeded', return_message = 'VACUUM'
    end

    DBA->>Cron: SELECT * FROM cron.job_run_details ORDER BY start_time DESC
    Cron-->>DBA: 返回執行日誌

    DBA->>Cron: cron.unschedule(1)
    Cron->>Job: DELETE FROM cron.job WHERE jobid = 1
```

#### I. 建立定時任務

#### cron.schedule() 核心函數

```sql
-- 基本語法
SELECT cron.schedule(
    'job-name',              -- 任務名稱（可選，用於 human-friendly 識別）
    'schedule-cron-expr',     -- cron 排程表達式
    'SQL command'             -- 要執行的 SQL 語句
);

-- 最簡用法：每天凌晨 3 點執行 VACUUM
SELECT cron.schedule('vacuum-orders', '0 3 * * *', 'VACUUM ANALYZE orders');
```

#### Cron 表達式詳解

pg_cron 支援兩種 cron 語法：

| 格式 | 欄位 | 範例 |
|------|------|------|
| **標準 5 段式** | `分 時 日 月 週` | `0 3 * * *` = 每天凌晨 3:00 |
| **精密 6 段式** | `秒 分 時 日 月 週` | `30 0 3 * * *` = 每天凌晨 3:00:30 |

```sql
-- === 時間單位對照表 ===

-- 每分鐘執行
SELECT cron.schedule('every-minute', '* * * * *', 'SELECT 1');

-- 每天凌晨 3:00
SELECT cron.schedule('daily-3am', '0 3 * * *', 'VACUUM ANALYZE orders');

-- 每 15 分鐘
SELECT cron.schedule('every-15min', '*/15 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_report');

-- 每小時的第 5 分鐘
SELECT cron.schedule('hourly-at-5', '5 * * * *', 'DELETE FROM user_sessions WHERE expires_at < now()');

-- 每週日凌晨 1:00
SELECT cron.schedule('sunday-1am', '0 1 * * 0', 'SELECT partman.run_maintenance()');

-- 每月 1 號凌晨 4:00
SELECT cron.schedule('monthly-1st', '0 4 1 * *', 'REINDEX DATABASE CONCURRENTLY your_db');

-- 秒級精度（6 段式）：每 30 秒
SELECT cron.schedule('every-30sec', '*/30 * * * * *', 'SELECT pg_notify(''heartbeat'', now()::text)');
```

#### 典型任務範例

```sql
-- 1. 定時 VACUUM（避免交易 ID 回繞，見第 5 節）
SELECT cron.schedule('vacuum-off-peak',
    '0 3 * * *',
    'VACUUM (VERBOSE, ANALYZE) orders'
);

-- 2. 分割區自動維護（配合 pg_partman）
SELECT cron.schedule('partman-maintenance',
    '0 2 * * *',
    'SELECT partman.run_maintenance()'
);

-- 3. Materialized View 定時刷新
SELECT cron.schedule('refresh-daily-report',
    '0 5 * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales'
);

-- 4. GDPR 合規：刪除 90 天前的使用者資料
SELECT cron.schedule('gdpr-cleanup',
    '0 4 * * *',
    'DELETE FROM user_activity WHERE created_at < now() - INTERVAL ''90 days'''
);

-- 5. 日誌歸檔：將舊記錄移至歸檔表
SELECT cron.schedule('archive-logs',
    '0 1 * * 0',
    $$INSERT INTO audit_logs_archive
      SELECT * FROM audit_logs WHERE created_at < now() - INTERVAL '30 days';
      DELETE FROM audit_logs WHERE created_at < now() - INTERVAL '30 days';$$
);
```

> **初學者導讀**
> 建立任務時，一個常見的疑惑是：**時區到底是什麼？** 答案取決於 `cron.timezone` GUC 設定（預設為 `GMT`）。建議在 `postgresql.conf` 中明確設定為你的本地時區（如 `'Asia/Taipei'`），確保 cron 表達式中的時間是你期望的本地時間。你可以隨時用 `SHOW cron.timezone;` 確認當前設定。

#### II. 管理任務

#### 查看任務

```sql
-- 查看所有已註冊的任務
SELECT jobid, jobname, schedule, command, database, username, active
FROM cron.job
ORDER BY jobid;
```

#### 取消任務

```sql
-- 依 jobid 取消（unschedule）
SELECT cron.unschedule(1);       -- 取消 jobid = 1 的任務
SELECT cron.unschedule('vacuum-off-peak');  -- 依 jobname 取消（PG 17+）
```

#### 修改任務

```sql
-- alter_job 支援修改排程、命令、啟用狀態
-- 語法：cron.alter_job(job_id, schedule, command, database, username, active)

-- 只改排程（command 傳 NULL 表示不變）
SELECT cron.alter_job(1, '0 4 * * *', NULL, NULL, NULL, NULL);

-- 同時改排程和 SQL
SELECT cron.alter_job(1, '0 3 * * *', 'VACUUM ANALYZE orders, order_items', NULL, NULL, NULL);

-- 修改目標資料庫
SELECT cron.alter_job(1, NULL, NULL, 'analytics_db', NULL, NULL);

-- 修改執行的使用者角色
SELECT cron.alter_job(1, NULL, NULL, NULL, 'maintenance_role', NULL);
```

#### 啟用 / 停用任務（不刪除）

```sql
-- 停用（保留定義，但不執行）
SELECT cron.alter_job(1, NULL, NULL, NULL, NULL, false);

-- 重新啟用
SELECT cron.alter_job(1, NULL, NULL, NULL, NULL, true);
```

```sql
-- 批次停用所有任務（維護期間用）
UPDATE cron.job SET active = false;

-- 批次啟用特定 jobname 的任務
UPDATE cron.job SET active = true WHERE jobname LIKE 'vacuum-%';
```

> **補充（Senior Dev）**
> 停用優於刪除：在維護窗口期間，與其 `cron.unschedule()` 刪除任務後再重建，不如直接 `UPDATE cron.job SET active = false` 批次停用。維護結束後 `SET active = true` 一鍵恢復，保留所有原始配置，避免重建時的人為出錯。這是 pg_cron 相比外部排程的一個重要 DX（Developer Experience）優勢。

#### III. 查看日誌

```sql
-- 最近的執行記錄（最新在上）
SELECT jobid, runid, job_pid, database, username,
       status, return_message,
       start_time, end_time,
       EXTRACT(epoch FROM (end_time - start_time)) AS duration_seconds
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 20;
```

#### 狀態值完整說明

| 狀態 | 含義 | 下一步 |
|------|------|--------|
| `starting` | 任務已被排程引擎選定，準備執行 | 正常過渡狀態，通常瞬間跳轉 |
| `connecting` | 正在建立到目標資料庫的 SPI 連線 | 若長時間卡住，檢查 `cron.host` 和連線數限制 |
| `sending` | 正在發送 SQL 命令 | 幾乎瞬時的狀態 |
| `running` | SQL 正在執行中 | 若持續過久，檢查查詢本身是否需要調校 |
| `succeeded` | 執行成功 | — |
| `failed` | 執行失敗 | 檢查 `return_message` 欄位的錯誤資訊 |

#### 監控查詢範例

```sql
-- 檢查最近失敗的任務
SELECT jobid, status, return_message, start_time
FROM cron.job_run_details
WHERE status = 'failed'
  AND start_time > now() - INTERVAL '24 hours'
ORDER BY start_time DESC;

-- 統計每個任務的成功率
SELECT jobid,
       COUNT(*) FILTER (WHERE status = 'succeeded') AS succeeded,
       COUNT(*) FILTER (WHERE status = 'failed')    AS failed,
       ROUND(
         100.0 * COUNT(*) FILTER (WHERE status = 'succeeded') / COUNT(*), 1
       ) AS success_rate_pct
FROM cron.job_run_details
WHERE start_time > now() - INTERVAL '7 days'
GROUP BY jobid
ORDER BY success_rate_pct;

-- 找出執行時間最長的任務（可能需調校）
SELECT jobid,
       MIN(start_time) AS recent_start,
       EXTRACT(epoch FROM (end_time - start_time)) AS duration_secs,
       command
FROM cron.job_run_details
WHERE start_time > now() - INTERVAL '1 day'
  AND status = 'succeeded'
ORDER BY duration_secs DESC
LIMIT 10;
```

---

### 5. 典型應用場景

```mermaid
gantt
    title pg_cron 每日任務排程範例
    dateFormat HH:mm
    axisFormat %H:%M

    section 資料維護
    VACUUM ANALYZE orders      :00:00, 00:30
    VACUUM ANALYZE order_items :00:00, 00:30

    section 分區管理
    pg_partman run_maintenance :02:00, 02:15
    過期分區 DETACH/DROP       :02:00, 02:15

    section 快取刷新
    REFRESH MV daily_report    :05:00, 05:15
    REFRESH MV hourly_stats    :05:15, 05:30

    section 資料清理
    GDPR 保留策略清理           :04:00, 04:30
    稽核日誌歸檔               :04:30, 05:00

    section 監控
    健康檢查 heartbeat         :active, 00:00, 23:59
```

#### I. 定時 VACUUM（off-peak）

PostgreSQL 的 MVCC 機制需要定期 VACUUM 以回收死元組（dead tuples）並防止交易 ID 回繞（Transaction ID Wraparound）。在離峰時段安排 VACUUM 是最經典的 pg_cron 應用：

```sql
-- 每天晚上 3:00 對核心業務表做 VACUUM ANALYZE
SELECT cron.schedule('vacuum-core-tables', '0 3 * * *',
    $$VACUUM (VERBOSE, ANALYZE) orders;
      VACUUM (VERBOSE, ANALYZE) order_items;
      VACUUM (VERBOSE, ANALYZE) users;$$
);

-- 對超大表只做 VACUUM FREEZE（更輕量、只凍結交易 ID，不回收空間）
SELECT cron.schedule('freeze-giant-table', '0 4 * * 0',
    'VACUUM (FREEZE, VERBOSE) giant_logs'
);
```

```sql
-- 監控：檢查哪個表最久未 VACUUM
SELECT schemaname, relname,
       now() - last_vacuum AS time_since_vacuum,
       now() - last_analyze AS time_since_analyze,
       n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY time_since_vacuum DESC NULLS LAST
LIMIT 10;
```

> **補充（Senior Dev）**
> 不要對所有表用同一套 VACUUM 排程。應根據 `pg_stat_user_tables.n_dead_tup` 和 `n_mod_since_analyze` 的變動速率來分級：
> - **高寫入表**（如 `orders`、`order_items`）：每日 VACUUM + ANALYZE
> - **中寫入表**（如 `users`、`products`）：每週 VACUUM + ANALYZE
> - **低寫入表**（如 `config`、`country_codes`）：每月一次或觸發閾值時手動執行
> 過度 VACUUM 也是一種資源浪費，尤其是對幾乎無更新的靜態表。

#### II. 配合 pg_partman 自動管理分區

pg_partman 本身內建 Background Worker，但使用 pg_cron 排程可以提供更靈活的排程控制和備援機制：

```sql
-- 主排程：每小時的第 0 分鐘執行分區維護
SELECT cron.schedule('partman-maintenance', '0 * * * *',
    'SELECT partman.run_maintenance()'
);

-- 備援排程：每天早上 3:00 額外執行一次（防止主排程遺漏）
SELECT cron.schedule('partman-maintenance-fallback', '0 3 * * *',
    'SELECT partman.run_maintenance()'
);

-- 每週一：檢查分區健康狀態
SELECT cron.schedule('partman-health-check', '0 9 * * 1',
    $$INSERT INTO partition_health_log (check_time, default_partitions, unmanaged_partitions)
     SELECT now(),
            COUNT(*) FILTER (WHERE has_default = true),
            COUNT(*) FILTER (WHERE NOT is_managed)
     FROM partman.part_config_sub;$$
);
```

#### III. 定時刷新 Materialized View

Materialized View 的快照是靜止的，必須手動刷新。pg_cron 讓這個過程自動化：

```sql
-- 每天早上 5:00 刷新日報表
SELECT cron.schedule('refresh-daily-report', '0 5 * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales'
);

-- 每小時刷新一次即時統計（CONCURRENTLY 不鎖定讀取）
SELECT cron.schedule('refresh-hourly-stats', '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_hourly_active_users'
);
```

```sql
-- 監控 MV 的刷新時間與狀態
SELECT schemaname, matviewname, ispopulated,
       last_refresh_time,
       now() - last_refresh_time AS staleness
FROM pg_matviews
LEFT JOIN (
    SELECT schemaname AS m_schema, matviewname AS m_name,
           last_refresh_time
    FROM pg_stat_user_tables  -- 近似檢查（無直接的 MV stats 表）
) stats ON matviewname = m_name
WHERE ispopulated
ORDER BY staleness DESC;
```

> **補充（Senior Dev）**
> `REFRESH MATERIALIZED VIEW CONCURRENTLY` 需要 MV 上存在一個 unique index，否則會報錯。若你的 MV 不具備唯一欄位組合（例如聚合結果中 COUNT > 1 的場景），有兩個選擇：
> 1. 使用非並行刷新 `REFRESH MATERIALIZED VIEW`（會鎖定讀取，適用於 off-peak）
> 2. 手動建立一個 composite unique index 涵蓋所有欄位：`CREATE UNIQUE INDEX ON mv_name (col1, col2, ...)`，但這對大量欄位的 MV 可能極度膨脹索引空間

#### IV. 數據清理（GDPR Retention、Log Archival）

合規要求（如 GDPR 第 17 條「被遺忘權」）要求企業在特定時限內刪除過期個人資料：

```sql
-- GDPR：刪除 90 天前的最後登入記錄（含 PII）
SELECT cron.schedule('gdpr-cleanup-sessions', '0 4 * * *',
    $$DELETE FROM user_sessions
      WHERE last_active < now() - INTERVAL '90 days'$$
);

-- 審計日誌歸檔：先備份再刪除
SELECT cron.schedule('archive-audit-logs', '0 2 * * 0',
    $$BEGIN;
      INSERT INTO audit_logs_archive
      SELECT * FROM audit_logs WHERE created_at < now() - INTERVAL '180 days';
      DELETE FROM audit_logs WHERE created_at < now() - INTERVAL '180 days';
      COMMIT;$$
);

-- 清理孤兒資料：沒有訂單的購物車
SELECT cron.schedule('clean-orphan-carts', '0 6 * * *',
    $$DELETE FROM shopping_carts
      WHERE updated_at < now() - INTERVAL '7 days'
        AND id NOT IN (SELECT cart_id FROM orders WHERE created_at > now() - INTERVAL '7 days')$$
);
```

```sql
-- 批次刪除（避免長事務鎖表）
-- 對大表，建議用 LOOP + LIMIT 方式分批刪除
SELECT cron.schedule('batch-gdpr-cleanup', '0 4 * * *',
    $$DO $$
    DECLARE
        deleted_rows INT := 1;
    BEGIN
        WHILE deleted_rows > 0 LOOP
            DELETE FROM user_activity
            WHERE ctid IN (
                SELECT ctid FROM user_activity
                WHERE created_at < now() - INTERVAL '90 days'
                LIMIT 10000
            );
            GET DIAGNOSTICS deleted_rows = ROW_COUNT;
            COMMIT;
        END LOOP;
    END;
    $$;$$
);
```

---

### 6. App Dev 最佳實踐

#### I. 任務冪等性（Idempotency）

每個 pg_cron 任務都應該被設計為**可安全重複執行**，不會因重複執行而產生副作用或錯誤資料：

```sql
-- ❌ 非冪等：重複執行會重複插入
SELECT cron.schedule('bad-daily-report', '0 6 * * *',
    $$INSERT INTO daily_summary (report_date, total_sales)
      SELECT CURRENT_DATE, SUM(amount) FROM orders WHERE created_at::date = CURRENT_DATE;$$
);

-- ✅ 冪等：使用 UPSERT（INSERT ... ON CONFLICT）
SELECT cron.schedule('good-daily-report', '0 6 * * *',
    $$INSERT INTO daily_summary (report_date, total_sales)
      SELECT CURRENT_DATE, SUM(amount) FROM orders WHERE created_at::date = CURRENT_DATE
      ON CONFLICT (report_date) DO UPDATE
        SET total_sales = EXCLUDED.total_sales,
            updated_at = now();$$
);

-- ✅ 冪等：使用條件檢查
SELECT cron.schedule('good-conditional-insert', '0 6 * * *',
    $$INSERT INTO daily_summary (report_date, total_sales)
      SELECT CURRENT_DATE, SUM(amount) FROM orders WHERE created_at::date = CURRENT_DATE
      WHERE NOT EXISTS (
        SELECT 1 FROM daily_summary WHERE report_date = CURRENT_DATE
      );$$
);
```

#### II. 錯誤處理策略

pg_cron 任務失敗時不會自動重試，上一個執行失敗不影響下一個排程。開發者需要自行實作錯誤處理：

```sql
-- 寫入 PL/pgSQL 函數，內含 EXCEPTION 處理
CREATE OR REPLACE FUNCTION safe_vacuum_orders()
RETURNS text AS $$
DECLARE
    err_msg text;
BEGIN
    VACUUM (VERBOSE, ANALYZE) orders;
    RETURN 'OK';
EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS err_msg = MESSAGE_TEXT;
        -- 寫入自訂的警報表
        INSERT INTO job_alerts (job_name, alert_time, error_message)
        VALUES ('vacuum-orders', now(), err_msg);
        RETURN 'ERROR: ' || err_msg;  -- 此訊息會寫入 cron.job_run_details.return_message
END;
$$ LANGUAGE plpgsql;

SELECT cron.schedule('safe-vacuum-orders', '0 3 * * *',
    'SELECT safe_vacuum_orders()'
);
```

```sql
-- 監控：每小時檢查最近 1 小時內是否有失敗任務
SELECT cron.schedule('monitor-failed-jobs', '0 * * * *',
    $$INSERT INTO job_alert_log (alert_time, failed_count, detail)
     SELECT now(), COUNT(*), string_agg('jobid=' || jobid || ': ' || return_message, '; ')
     FROM cron.job_run_details
     WHERE status = 'failed'
       AND start_time > now() - INTERVAL '1 hour'
     HAVING COUNT(*) > 0;$$
);
```

#### III. 時區設定

```sql
-- 確認當前時區
SHOW cron.timezone;

-- 查看所有可用時區
SELECT * FROM pg_timezone_names WHERE name LIKE 'Asia/%';
```

```ini
## postgresql.conf 建議設定
cron.timezone = 'Asia/Taipei'
```

> **補充（Senior Dev）**
> 時區的陷阱在於夏令時間（DST）。以 `'America/New_York'` 為例，春季「跳過」的 2:00 AM 和秋季「重複」的 1:00 AM 會導致部分排程任務被跳過或執行兩次。如果你有精確到「每日 2:01 AM 執行」的任務，強烈建議使用 `'UTC'` 或無 DST 的時區（如 `'Asia/Taipei'`）。所有業務時間換算交由應用層處理，pg_cron 只管「絕對時間點」。

#### IV. 避免長任務堆積

```sql
-- 使用 cron.max_job_execution_time 限制單任務最長執行時間（PG 16+）
-- postgresql.conf:
-- cron.max_job_execution_time = 300000  # 5 分鐘（毫秒）
```

長任務的危害：

```mermaid
graph TD
    A["長任務（如 10 分鐘 VACUUM）"] --> B["佔用 BGW Worker"]
    B --> C{"cron.max_workers 耗盡？"}
    C -->|"是"| D["其他任務排隊等待"]
    C -->|"否"| E["仍可執行其他任務<br/>但系統資源被佔用"]
    D --> F["任務堆積（Pile-Up）"]
    F --> G["⚠️ 下一個週期到來<br/>同一任務又觸發"]
    G --> H["雪崩效應<br/>Avalanche Effect"]
    style A fill:#E67E22,color:#fff
    style F fill:#E74C3C,color:#fff
    style H fill:#E74C3C,color:#fff
```

**預防策略**：

```sql
-- 1. 將重任務拆成多個小任務，錯開執行時間
SELECT cron.schedule('vacuum-orders',    '0 3 * * *', 'VACUUM ANALYZE orders');
SELECT cron.schedule('vacuum-items',     '0 3 * * *', 'VACUUM ANALYZE order_items');
SELECT cron.schedule('vacuum-users',     '0 3 * * *', 'VACUUM ANALYZE users');
-- PG 16: use_background_workers = on 時，三者可並行執行（各自佔一個 worker）

-- 2. 在任務本身加入防護鎖（Advisory Lock），確保同時只有一個 instance 執行
SELECT cron.schedule('guarded-job', '*/5 * * * *',
    $$SELECT pg_try_advisory_lock(12345) AS acquired;
      -- 若 acquired = true，執行實際任務；否則跳過
      -- 配合函數封裝以實際檢查鎖狀態$$
);
```

#### V. 監控體系搭建

```sql
-- 建立監控視圖：任務健康總覽
CREATE OR REPLACE VIEW v_cron_job_health AS
SELECT
    j.jobid,
    j.jobname,
    j.schedule,
    j.active,
    j.command,
    COUNT(d.runid) FILTER (WHERE d.start_time > now() - INTERVAL '24 hours') AS runs_24h,
    COUNT(d.runid) FILTER (WHERE d.status = 'failed'
                              AND d.start_time > now() - INTERVAL '24 hours') AS failures_24h,
    MAX(d.start_time) AS last_run,
    MAX(d.end_time) - MAX(d.start_time) AS last_duration,
    ROUND(
        COALESCE(
            100.0 * COUNT(d.runid) FILTER (WHERE d.status = 'succeeded'
                                               AND d.start_time > now() - INTERVAL '7 days')
            / NULLIF(COUNT(d.runid) FILTER (WHERE d.start_time > now() - INTERVAL '7 days'), 0),
            100
        ), 1
    ) AS success_rate_7d_pct
FROM cron.job j
LEFT JOIN cron.job_run_details d ON j.jobid = d.jobid
GROUP BY j.jobid, j.jobname, j.schedule, j.active, j.command;

-- 日常巡檢用
SELECT * FROM v_cron_job_health ORDER BY success_rate_7d_pct, last_run DESC NULLS LAST;
```

#### VI. cron.max_job_execution_time（PG 16）Timeout 保護

PG 16 新增的 `cron.max_job_execution_time` 提供了一層緊急煞車機制：

```ini
## postgresql.conf
## 全域設定：所有任務的預設 timeout（毫秒）
cron.max_job_execution_time = 3600000  # 1 小時
```

```sql
-- 針對特定任務覆蓋 timeout（使用 alter_job 無法直接設定，需透過 command 自行管理）
-- 替代方案：利用 statement_timeout 在任務 SQL 中限制
SELECT cron.schedule('timeout-protected-job', '0 5 * * *',
    $$SET LOCAL statement_timeout = '30min';
      REFRESH MATERIALIZED VIEW CONCURRENTLY mv_huge_report;$$
);
```

#### VII. 安全與權限

```sql
-- pg_cron 任務以 cron.host 指定的角色執行（預設為啟動 PG 的 OS 使用者）
-- 確保該角色對目標物件有足夠權限

-- 授予 cron schema 的讀取權限給監控角色
GRANT USAGE ON SCHEMA cron TO monitoring_role;
GRANT SELECT ON cron.job, cron.job_run_details TO monitoring_role;

-- 限制只有特定角色能管理排程任務
-- pg_cron 本身沒有內建的 RBAC，需透過 schema 權限控制
REVOKE ALL ON SCHEMA cron FROM public;
GRANT USAGE ON SCHEMA cron TO cron_admin_role;
GRANT ALL ON ALL TABLES IN SCHEMA cron TO cron_admin_role;
```

#### VIII. 備份與還原注意事項

```sql
-- pg_dump 匯出時，cron.job 的內容會被包含在 schema-only dump 中
-- 但還原後任務預設為 active = true，若排程任務在還原環境執行可能造成問題

-- 還原後立即停用所有任務
UPDATE cron.job SET active = false;

-- 確認環境無誤後再逐批啟用
UPDATE cron.job SET active = true WHERE jobname IN ('vacuum-orders', 'health-check');
```

> **補充（Senior Dev）**
> 若你使用 CI/CD 自動化資料庫遷移（如 Flyway、Sqitch），建議將 `SELECT cron.schedule(...)` 語句納入版本控制。這樣的好處是：
> 1. 每個排程任務的變更有 Git 歷史可追溯
> 2. 新環境佈署時自動建立排程（無需手動還原）
> 3. 遷移腳本中可同時安排 `active = false` 作為安全預設值，後續再由 DBA 手動啟用
>
> ```sql
> -- V1.2.3__add_vacuum_schedule.sql (Flyway migration)
> SELECT cron.schedule('vacuum-orders', '0 3 * * *', 'VACUUM ANALYZE orders');
> UPDATE cron.job SET active = false WHERE jobname = 'vacuum-orders';
> -- 由 DBA 確認環境後手動啟用
> ```

#### IX. pg_cron 與其他排程工具的互補關係

```mermaid
graph TD
    subgraph "資料庫層"
        A["pg_cron<br/>資料庫內部定時任務"]
    end
    subgraph "系統層"
        B["systemd timer / crontab<br/>OS 層級排程"]
    end
    subgraph "調度層"
        C["Airflow / Dagster<br/>資料管線調度"]
    end
    subgraph "容器層"
        D["K8s CronJob<br/>容器化排程"]
    end

    A -->|"適合：VACUUM、MV 刷新、<br/>資料清理、分割區維護"| E["資料庫維運任務"]
    B -->|"適合：pg_dump 備份、<br/>日誌輪轉、磁碟清理"| F["系統維運任務"]
    C -->|"適合：ETL 管線、<br/>跨系統資料流"| G["複雜資料管線"]
    D -->|"適合：微服務批次、<br/>外部 API 呼叫"| H["應用層批次"]

    style A fill:#27AE60,color:#fff
    style B fill:#4a90d9,color:#fff
    style C fill:#E67E22,color:#fff
    style D fill:#8E44AD,color:#fff
```

**使用建議**：pg_cron 不應該取代所有排程工具——它的角色是處理「純資料庫維運任務」。需要呼叫外部 API、跨系統協調、複雜依賴管理的場景，仍應交由 Airflow 等專業調度工具處理。pg_cron 的優勢在於**零外部依賴、SQL 原生表達、交易一致**，請把這三個優勢場景留給它。

---

### 附錄：常用 cron 表達式速查表

| cron 表達式 | 含義 | 適用場景 |
|-------------|------|----------|
| `* * * * *` | 每分鐘 | Heartbeat 檢查、即時監控 |
| `*/5 * * * *` | 每 5 分鐘 | 高頻輕量任務 |
| `*/15 * * * *` | 每 15 分鐘 | 中頻監控 |
| `0 * * * *` | 每小時整點 | 小時級聚合刷新 |
| `0 2 * * *` | 每天凌晨 2:00 | Off-peak 維護任務 |
| `0 3 * * 0` | 每週日凌晨 3:00 | 週維護窗口 |
| `0 4 1 * *` | 每月 1 號凌晨 4:00 | 月結報表、批量歸檔 |
| `0 1 1 1 *` | 每年 1 月 1 日凌晨 1:00 | 年度任務 |
| `0 9 * * 1-5` | 工作日（週一至五）早上 9:00 | 上班時間通知 |
| `*/30 * * * * *` | 每 30 秒（6 段式） | 秒級監控 |
## 九、pg_stat_kcache — 查詢 CPU 與實體 IO 統計

### 1. 核心價值

`pg_stat_statements` 僅記錄 buffer 層級命中與讀取（logical IO），無法反映作業系統層級真實實體磁碟 IO。`pg_stat_kcache` 透過 `getrusage()` 系統呼叫，補足以下 OS 層統計：

| 層級 | 追蹤內容 | 代表意義 |
|------|----------|----------|
| `pg_stat_statements` | `shared_blks_hit`, `shared_blks_read` | **邏輯 IO**：PostgreSQL 共享緩衝區命中 / 要求 OS 讀取頁面 |
| `pg_stat_kcache` | `reads`, `reads_bytes` | **實體 IO**：OS 實際從磁碟讀取的次數與 KB 數 |
| 實際磁碟 | iostat, iotop | **硬體 IO**：儲存裝置的真實讀寫延遲與吞吐量 |

```mermaid
graph TD
    Q["🐘 PostgreSQL Query"] --> PSS["pg_stat_statements<br/>Buffer Layer"]
    Q --> PKC["pg_stat_kcache<br/>OS Layer (getrusage)"]
    Q --> DISK["Physical Disk<br/>iostat / iotop"]

    PSS --> |"shared_blks_read (pages)"| B_HIT{"OS Page Cache?"}
    B_HIT -->|"✅ 命中"| PGC["Page Cache 回應<br/>無實體 IO"]
    B_HIT -->|"❌ 未命中"| REAL["真實磁碟讀取"]

    PKC --> |"reads / reads_bytes"| OS_READ["OS 層 block IO 統計<br/>已排除 Page Cache"]
    DISK --> |"await / r/s"| HARDWARE["硬體級延遲與吞吐量"]

    style PSS fill:#d4edda,stroke:#28a745
    style PKC fill:#fff3cd,stroke:#ffc107
    style DISK fill:#f8d7da,stroke:#dc3545
```

> **初學者導讀**  
> `shared_blks_read` 代表 PostgreSQL **向 OS 發起**讀取請求的頁面數，但如果 OS 的 page cache 已經快取了該頁，就不會發生真實磁碟 IO。因此 `shared_blks_read` 高 ≠ 磁碟 IO 高。`pg_stat_kcache` 透過 `getrusage()` 直接讀取 kernel 累計的 block IO 統計，才反映物理真相。

核心統計欄位一覽：

| 欄位 | 類型 | 說明 |
|------|------|------|
| `user_time` | double | 查詢在 user mode 消耗的 CPU 秒數 |
| `system_time` | double | 查詢在 kernel mode 消耗的 CPU 秒數（含 IO 排程） |
| `reads` | bigint | OS 層讀取系統呼叫次數 |
| `writes` | bigint | OS 層寫入系統呼叫次數 |
| `fsyncs` | bigint | fsync 呼叫次數（強制刷盤） |
| `reads_bytes` | bigint | 實際從儲存裝置讀取的 KB 數 |
| `writes_bytes` | bigint | 實際寫入儲存裝置的 KB 數 |

---

### 2. 工作原理

`pg_stat_kcache` 以 hook 方式掛載在 PostgreSQL Executor 層。每當一條標準化查詢開始執行前，記錄當下 process 的 `getrusage(RUSAGE_SELF)` 快照；查詢結束後再取一次，差值即為該次執行的 OS 資源消耗，累計到對應 `queryid` 的共享記憶體槽位中。

```mermaid
sequenceDiagram
    participant Q as Query
    participant EX as Executor
    participant KC as pg_stat_kcache Hook
    participant SHM as Shared Memory
    participant PSS as pg_stat_statements

    Q->>EX: 開始執行標準化查詢
    EX->>KC: ExecutorStart hook
    KC->>OS: getrusage(RUSAGE_SELF)
    OS-->>KC: ru_utime, ru_stime, ru_inblock, ru_oublock
    KC->>KC: 儲存快照 (snapshot)

    Q->>EX: 查詢執行中（掃描、JOIN、Sort、Agg…）
    Q->>EX: 查詢結束

    EX->>KC: ExecutorEnd hook
    KC->>OS: getrusage(RUSAGE_SELF) 再取一次
    OS-->>KC: 當前累計值

    KC->>KC: 計算差值 = 本次 CPU / IO 消耗
    KC->>PSS: 取得 queryid（與 pg_stat_statements 對齊）
    KC->>SHM: queryid → 累加 user_time, reads, writes, fsyncs 等

    Note over KC,SHM: 每 500ms flush 到共用記憶體<br/>pg_stat_statements_reset() 同步重置
```

> **補充（Senior Dev）**  
> `getrusage(RUSAGE_SELF)` 回傳的是 **當前 process 所有執行緒**的總計。PostgreSQL 使用 process-per-connection 模型，因此統計值不會受其他 session 干擾。但 `RUSAGE_SELF` 不包含已結束子程序的資源（例如 `CREATE INDEX CONCURRENTLY` 可能 fork 額外 process），這些 fork 的 IO 不會納入。此外 `ru_inblock` / `ru_oublock` 在 Linux 上單位為 512-byte sectors，`pg_stat_kcache` 內部會轉換為 KB。

---

### 3. 安裝與配置（PG 16）

#### 3.1 載入順序（重要）

`pg_stat_kcache` 依賴 `pg_stat_statements`。必須在 `shared_preload_libraries` 中 **先列出** `pg_stat_statements`：

```ini
## postgresql.conf
shared_preload_libraries = 'pg_stat_statements, pg_stat_kcache'
```

> **初學者導讀**  
> `shared_preload_libraries` 決定了 extension 的初始化順序。`pg_stat_kcache` 初始化時需要 `pg_stat_statements` 的 hash table 已就緒，因此順序不可顛倒。

修改後重啟 PostgreSQL：

```bash
pg_ctl restart
```

#### 3.2 建立 Extension

```sql
-- 確認 pg_stat_statements 已啟用
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- 建立 pg_stat_kcache
CREATE EXTENSION pg_stat_kcache;
```

#### 3.3 GUC 參數

```sql
-- 查看當前設定
SELECT name, setting, context, short_desc
FROM pg_settings
WHERE name LIKE 'pg_stat_kcache%';
```

| 參數 | 預設值 | 可選值 | 說明 |
|------|--------|--------|------|
| `pg_stat_kcache.track` | `top` | `all`, `top`, `none` | 追蹤範圍：全部查詢 / 僅巢狀最上層 / 關閉 |
| `pg_stat_kcache.track_planning` | `off` | `on`, `off` | 是否追蹤 Planning 階段的 CPU/IO |

```sql
-- 生產環境建議：僅追蹤 top-level 查詢
ALTER SYSTEM SET pg_stat_kcache.track = 'top';
SELECT pg_reload_conf();
```

#### 3.4 驗證安裝

```sql
-- 檢查是否有資料進入
SELECT count(*) AS tracked_queries FROM pg_stat_kcache;

-- 檢查與 pg_stat_statements 的對齊狀況
SELECT
    k.userid,
    k.dbid,
    k.queryid,
    k.reads,
    k.reads_bytes,
    s.calls
FROM pg_stat_kcache k
JOIN pg_stat_statements s USING (userid, dbid, queryid)
LIMIT 5;
```

---

### 4. 核心視圖與查詢

#### I. `pg_stat_kcache` 列詳解

```sql
SELECT
    userid, dbid, queryid,           -- 與 pg_stat_statements 對齊的三元組
    plan_user_time, plan_system_time, -- Planning 階段 CPU（需 track_planning = on）
    plan_reads, plan_writes,          -- Planning 階段 IO（同上）
    user_time,                        -- 此查詢累計 user-mode CPU 秒數
    system_time,                      -- 此查詢累計 kernel-mode CPU 秒數
    reads,                            -- OS 層讀取呼叫次數
    writes,                           -- OS 層寫入呼叫次數
    fsyncs,                           -- fsync 呼叫次數
    reads_bytes,                      -- 讀取 KB 數（從儲存裝置）
    writes_bytes                      -- 寫入 KB 數（至儲存裝置）
FROM pg_stat_kcache
LIMIT 10;
```

> **補充（Senior Dev）**  
> `user_time + system_time` 除以 `calls` 可得 **每呼叫平均 CPU 秒數**，這是衡量查詢計算密度的關鍵指標。若 `user_time` 遠大於 `system_time`，瓶頸在計算；反之若 `system_time` 佔比高，表示 kernel 層操作（IO、網路、context switch）較多。

#### II. 實戰查詢 — 與 pg_stat_statements JOIN

#### Top 10 實體 IO 查詢

```sql
SELECT
    s.queryid,
    LEFT(s.query, 120) AS query_preview,
    s.calls,
    kc.reads                       AS physical_reads,
    kc.reads_bytes                 AS physical_reads_kb,
    s.shared_blks_read * current_setting('block_size')::bigint / 1024
                                   AS logical_reads_kb,
    ROUND(kc.reads_bytes::numeric
          / NULLIF(s.shared_blks_read * current_setting('block_size')::bigint / 1024, 0), 2)
                                   AS cache_miss_ratio
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE kc.reads_bytes > 0
ORDER BY kc.reads_bytes DESC
LIMIT 10;
```

#### Top 10 CPU 消耗查詢

```sql
SELECT
    s.queryid,
    LEFT(s.query, 120)        AS query_preview,
    s.calls,
    ROUND(kc.user_time::numeric, 2)    AS user_cpu_sec,
    ROUND(kc.system_time::numeric, 2)  AS sys_cpu_sec,
    ROUND((kc.user_time + kc.system_time)::numeric, 2) AS total_cpu_sec,
    ROUND((kc.user_time + kc.system_time)::numeric
          / NULLIF(s.calls, 0) * 1000, 2) AS avg_ms_per_call
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
ORDER BY total_cpu_sec DESC
LIMIT 10;
```

#### Write Amplification 分析

```sql
-- 比對 WAL 產生量 vs 實際 OS 寫入量
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    s.wal_bytes / 1024             AS wal_kb,
    kc.writes_bytes                AS os_write_kb,
    ROUND(kc.writes_bytes::numeric
          / NULLIF(s.wal_bytes / 1024, 0), 2) AS write_amplification,
    kc.fsyncs                      AS fsync_calls
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE s.wal_bytes > 0
ORDER BY s.wal_bytes DESC
LIMIT 10;
```

> **補充（Senior Dev）**  
> **Write Amplification** 指 OS 實際寫入量與 WAL 產生量的比值。正常情況下 `wal_bytes ≈ writes_bytes`（WAL 順序寫）。若 `writes_bytes` 遠大於 `wal_bytes`，可能是頻繁的 hint-bit 更新、full page write 或 temp file spill 導致額外 IO。若 `wal_bytes` 遠大於 `writes_bytes`，通常代表 async write（background writer / checkpointer）尚未刷盤。

#### Cache Hit Ratio 驗證

```sql
WITH cache_stats AS (
    SELECT
        s.queryid,
        s.calls,
        s.shared_blks_hit,
        s.shared_blks_read,
        kc.reads_bytes
    FROM pg_stat_kcache kc
    JOIN pg_stat_statements s USING (userid, dbid, queryid)
)
SELECT
    CASE WHEN shared_blks_hit + shared_blks_read = 0 THEN 0
         ELSE ROUND(shared_blks_hit::numeric
              / (shared_blks_hit + shared_blks_read) * 100, 1)
    END AS buffer_cache_hit_pct,
    shared_blks_hit,
    shared_blks_read,
    reads_bytes AS physical_reads_kb
FROM cache_stats
WHERE reads_bytes > 0
ORDER BY reads_bytes DESC
LIMIT 20;
```

#### III. 生產診斷案例

```mermaid
flowchart TD
    A["發現查詢變慢"] --> B{"shared_blks_read<br/>高嗎？"}
    B -->|"是"| C{"kcache reads<br/>高嗎？"}
    B -->|"否"| D{"user_time / calls<br/>高嗎？"}

    C -->|"是"| E["🟡 Case 1: 真實磁碟 IO 瓶頸<br/>• 檢查 shared_buffers 大小<br/>• 檢查有效工作集 > 記憶體<br/>• 考慮增加記憶體 / SSD<br/>• enable bitmapscan 優化"]
    C -->|"否（shared_blks_read 高<br/>但 kcache reads 低）"| F["🟢 Case 1b: OS Page Cache 緩解<br/>• shared_buffers 太小，但 OS 快取夠<br/>• 可考慮增大 shared_buffers<br/>• 減少雙重快取浪費"]

    D -->|"是"| G["🟠 Case 2: CPU-bound<br/>• 檢查查詢計劃<br/>• 是否存在 Nested Loop 爆炸<br/>• 考慮物化視圖 / 預計算<br/>• 檢查 JIT 是否有幫助"]
    D -->|"否"| H{"kcache writes<br/>高 + fsyncs 高？"}

    H -->|"是"| I["🔴 Case 3: Write-heavy<br/>• 檢查 checkpoint 頻率<br/>• 增大 max_wal_size<br/>• 考慮 wal_compression<br/>• 檢查 full_page_writes<br/>• 考慮批次寫入策略"]
    H -->|"否"| J["其他瓶頸：Lock / LWLock / 網路"]

    style E fill:#fff3cd,stroke:#ffc107
    style F fill:#d4edda,stroke:#28a745
    style G fill:#ffd3b6,stroke:#fd7e14
    style I fill:#f8d7da,stroke:#dc3545
    style J fill:#e2e3e5,stroke:#6c757d
```

**Case 1：low `shared_blks_read` 但 high `kcache reads` — OS Page Cache 未命中**

```sql
-- 發現 shared_blks_read 不高，但 kcache.reads_bytes 很高
-- 代表查詢的 page 根本沒進 shared_buffers，要求 OS 讀
-- 可能原因：大量 Seq Scan 不同區塊，每次都從磁碟讀
SELECT
    s.queryid,
    LEFT(s.query, 120) AS query_preview,
    s.calls,
    s.shared_blks_read                                   AS buf_reads,
    kc.reads_bytes                                       AS os_reads_kb,
    s.shared_blks_read * current_setting('block_size')::bigint / 1024
                                                         AS buf_reads_kb,
    -- 正常情況下 buf_reads_kb ≈ os_reads_kb
    -- 若 os_reads_kb << buf_reads_kb，表示 OS page cache 有效緩解
    -- 若 os_reads_kb ≈ buf_reads_kb，表示每次 read 都穿透到磁碟
    ROUND(kc.reads_bytes::numeric / NULLIF(
          s.shared_blks_read * current_setting('block_size')::bigint / 1024, 0), 2)
                                                         AS read_efficiency
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE s.shared_blks_read > 1000
  AND kc.reads_bytes > 0
ORDER BY os_reads_kb DESC
LIMIT 10;
```

> **初學者導讀**  
> `read_efficiency ≈ 0.0` 表示 OS page cache 幾乎攔截了所有讀取，是最佳狀態（如果你有足夠記憶體）。`read_efficiency ≈ 1.0` 表示每次要求 OS 讀取都觸發了實體 IO，是磁碟瓶頸信號。`read_efficiency > 1.0` 則可能代表 OS 層有 read-ahead 或其他 I/O 放大。

**Case 2：低 buffer IO 但 high `user_time` — CPU-bound**

```sql
-- 查詢 IO 量極少但 CPU 時間極長：典型計算密集
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    ROUND(kc.user_time::numeric, 2)       AS user_cpu_sec,
    kc.reads_bytes                        AS os_reads_kb,
    ROUND((kc.user_time + kc.system_time)::numeric
          / NULLIF(s.calls, 0) * 1000, 2) AS avg_cpu_ms_per_call,
    ROUND(kc.reads_bytes::numeric
          / NULLIF(s.calls, 0), 2)        AS avg_reads_kb_per_call
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE (kc.user_time + kc.system_time) > 10
  AND kc.reads_bytes < 1024  -- IO < 1MB
ORDER BY avg_cpu_ms_per_call DESC
LIMIT 10;
```

> **補充（Senior Dev）**  
> 若 `avg_cpu_ms_per_call > 10ms` 且幾乎無 IO，常見根因：
> - 大型 HashAgg 或 Sort 在 work_mem 內執行（純 CPU）
> - Nested Loop 掃描行數極大（plan rows vs actual rows 偏差）
> - 大量 function call（PL/pgSQL loop, regexp, JSON 操作）
> - JIT 編譯開銷大於收益（PG 12+，檢查 `jit=on` 對該查詢是否有益）

**Case 3：high `writes` + `fsyncs` — Write-heavy 且頻繁刷盤**

```sql
-- 寫入密集查詢，觀察 fsync 頻率
SELECT
    s.queryid,
    LEFT(s.query, 100) AS query_preview,
    s.calls,
    kc.writes                                             AS os_writes,
    kc.writes_bytes                                       AS os_write_kb,
    kc.fsyncs                                             AS fsync_calls,
    ROUND(kc.fsyncs::numeric / NULLIF(kc.writes, 0), 3)  AS fsync_per_write,
    ROUND(kc.writes_bytes::numeric / NULLIF(s.calls, 0), 0) AS avg_write_kb_per_call,
    s.wal_bytes / 1024                                    AS wal_kb
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE kc.writes > 0
ORDER BY kc.writes_bytes DESC
LIMIT 10;
```

```mermaid
flowchart LR
    subgraph Diagnosis["🔍 Write-heavy 診斷決策樹"]
        direction TB
        W1["kcache.fsyncs / writes<br/>> 0.5?"] -->|"是"| W2["每兩次 write 就一次 fsync<br/>→ 檢查是否缺批次寫入<br/>→ 檢查 synchronous_commit 設定<br/>→ 考慮 WAL writer tuning"]
        W1 -->|"否"| W3["kcache.writes_bytes<br/>遠大於 wal_kb？"]
        W3 -->|"是"| W4["OS 寫入放大<br/>→ 檢查 full_page_writes<br/>→ 檢查 hint-bit 寫入<br/>→ 檢查 temp file spill"]
        W3 -->|"否"| W5["正常寫入模式<br/>→ 監控 checkpoint IO spike<br/>→ 調整 max_wal_size"]
    end
```

---

### 5. 與 pg_stat_statements 整合分析

#### 5.1 完整查詢效能儀表板

```sql
CREATE OR REPLACE VIEW query_performance_dashboard AS
SELECT
    -- 識別
    s.queryid,
    s.userid,
    s.dbid,
    LEFT(s.query, 200) AS query_preview,
    s.calls,

    -- 執行時間
    ROUND(s.total_exec_time::numeric / 1000, 2)   AS total_exec_sec,
    ROUND(s.mean_exec_time::numeric, 2)            AS avg_exec_ms,

    -- Buffer IO
    s.shared_blks_hit                              AS buf_hit,
    s.shared_blks_read                             AS buf_read,
    ROUND(s.shared_blks_hit::numeric
      / NULLIF(s.shared_blks_hit + s.shared_blks_read, 0) * 100, 1)
                                                    AS buf_hit_pct,

    -- OS IO（來自 pg_stat_kcache）
    kc.reads                                       AS os_reads,
    kc.reads_bytes                                 AS os_reads_kb,
    kc.writes                                      AS os_writes,
    kc.writes_bytes                                AS os_writes_kb,
    kc.fsyncs                                      AS os_fsyncs,

    -- CPU（來自 pg_stat_kcache）
    ROUND(kc.user_time::numeric, 2)                AS user_cpu_sec,
    ROUND(kc.system_time::numeric, 2)              AS sys_cpu_sec,
    ROUND((kc.user_time + kc.system_time)::numeric, 2) AS total_cpu_sec,

    -- WAL
    CASE WHEN s.wal_bytes > 0
         THEN (s.wal_bytes / 1024)::bigint
         ELSE 0 END                                 AS wal_kb,

    -- 衍生指標
    ROUND((kc.user_time + kc.system_time)::numeric
      / NULLIF(s.calls, 0) * 1000, 2)              AS avg_cpu_ms_per_call,
    ROUND(kc.reads_bytes::numeric
      / NULLIF(s.calls, 0), 2)                     AS avg_reads_kb_per_call,
    ROUND(kc.writes_bytes::numeric
      / NULLIF(kc.writes, 0), 2)                   AS avg_write_size_kb,
    ROUND(kc.fsyncs::numeric
      / NULLIF(kc.writes, 0), 3)                   AS fsync_ratio,

    -- 計畫時間
    ROUND(s.total_plan_time::numeric / 1000, 2)    AS total_plan_sec,
    ROUND(s.mean_plan_time::numeric, 2)            AS avg_plan_ms

FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE s.calls > 0;

COMMENT ON VIEW query_performance_dashboard IS
'整合 pg_stat_statements 與 pg_stat_kcache 的查詢效能全貌視圖';
```

#### 5.2 使用儀表板快速診斷

```sql
-- 找出 IO 最重的查詢（實體讀取 + 寫入）
SELECT query_preview, calls,
       os_reads_kb, os_writes_kb,
       total_cpu_sec, avg_cpu_ms_per_call,
       fsync_ratio
FROM query_performance_dashboard
WHERE os_reads_kb + os_writes_kb > 10240  -- IO > 10MB
ORDER BY os_reads_kb + os_writes_kb DESC
LIMIT 20;

-- 找出 CPU 最重的查詢
SELECT query_preview, calls,
       total_cpu_sec, avg_cpu_ms_per_call,
       os_reads_kb, os_writes_kb
FROM query_performance_dashboard
WHERE total_cpu_sec > 10
ORDER BY total_cpu_sec DESC
LIMIT 20;

-- 找出快取命中率低的查詢（可能浪費 IO）
SELECT query_preview, calls,
       buf_hit_pct, os_reads_kb,
       avg_cpu_ms_per_call
FROM query_performance_dashboard
WHERE calls > 100 AND buf_hit_pct < 90
ORDER BY os_reads_kb DESC
LIMIT 20;
```

> **補充（Senior Dev）**  
> 儀表板中 `avg_write_size_kb` 若明顯小於 block size（預設 8KB），表示存在大量 partial page write，通常是 hint-bit updates 或 index B-tree page splits 導致。可考慮：
> - 定期 `VACUUM` 設定 hint bits
> - 檢查 index fillfactor 設定
> - 考慮 `hot_standby_feedback` 對 hint bits 的影響

---

### 6. App Dev 最佳實踐

#### 6.1 驗證索引優化是否真正減少實體 IO

```sql
-- 步驟 1：建立索引前，記錄基準
-- （此處假設已累積一段時間的查詢統計）
SELECT queryid, calls, reads_bytes, user_time
FROM pg_stat_kcache
WHERE queryid = (
    SELECT queryid FROM pg_stat_statements
    WHERE query LIKE '%target_table%target_column%'
    LIMIT 1
);

-- 步驟 2：建立索引
-- CREATE INDEX idx_xxx ON target_table(target_column);

-- 步驟 3：重置統計
SELECT pg_stat_statements_reset();
SELECT pg_stat_kcache_reset();

-- 步驟 4：等待業務運行一段時間後，再次查詢
SELECT
    s.queryid,
    LEFT(s.query, 120) AS query_preview,
    s.calls,
    kc.reads_bytes          AS physical_reads_kb,
    ROUND(kc.user_time::numeric, 2) AS user_cpu_sec,
    s.shared_blks_read      AS logical_reads,
    s.shared_blks_hit       AS buffer_hits
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE s.query LIKE '%target_table%'
ORDER BY physical_reads_kb DESC;
```

> **初學者導讀**  
> `pg_stat_statements_reset()` 只重置 `pg_stat_statements` 的統計。要同時重置 `pg_stat_kcache`，必須呼叫 `pg_stat_kcache_reset()`。兩個函數獨立運作，不等價。

#### 6.2 偵測 CPU-Heavy 但低 IO 查詢

```sql
-- 此類查詢可能受益於應用層快取（Redis / Memcached）或物化視圖
SELECT
    s.queryid,
    LEFT(s.query, 150)       AS query_preview,
    s.calls,
    ROUND((kc.user_time + kc.system_time)::numeric, 2) AS total_cpu_sec,
    kc.reads_bytes                                     AS physical_reads_kb,
    ROUND((kc.user_time + kc.system_time)::numeric
          / NULLIF(s.calls, 0) * 1000, 2)              AS avg_cpu_ms_per_call,
    -- IO per CPU-second：數值越低，代表越偏向純計算
    ROUND(kc.reads_bytes::numeric
          / NULLIF(kc.user_time + kc.system_time, 0), 2) AS io_per_cpu_sec
FROM pg_stat_kcache kc
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE (kc.user_time + kc.system_time) > 30
  AND kc.reads_bytes < 5120 -- IO < 5MB
ORDER BY avg_cpu_ms_per_call DESC
LIMIT 10;
```

#### 6.3 定期快照比較

```sql
-- 建立快照表
CREATE TABLE IF NOT EXISTS kcache_snapshot (
    snapshot_ts   timestamptz DEFAULT now(),
    queryid       bigint,
    userid        oid,
    dbid          oid,
    calls         bigint,
    user_time     double precision,
    system_time   double precision,
    reads_bytes   bigint,
    writes_bytes  bigint,
    fsyncs        bigint,
    PRIMARY KEY (snapshot_ts, queryid, userid, dbid)
);

-- 拍攝快照
INSERT INTO kcache_snapshot (queryid, userid, dbid,
                             calls, user_time, system_time,
                             reads_bytes, writes_bytes, fsyncs)
SELECT queryid, userid, dbid,
       (SELECT calls FROM pg_stat_statements ps
        WHERE ps.userid = kc.userid
          AND ps.dbid   = kc.dbid
          AND ps.queryid = kc.queryid),
       user_time, system_time,
       reads_bytes, writes_bytes, fsyncs
FROM pg_stat_kcache kc;
```

```sql
-- 比較最近兩次快照，找出 IO 漲幅最大的查詢
WITH snapshots AS (
    SELECT DISTINCT snapshot_ts FROM kcache_snapshot
    ORDER BY snapshot_ts DESC LIMIT 2
), diff AS (
    SELECT
        cur.queryid,
        cur.reads_bytes  - COALESCE(prev.reads_bytes, 0)  AS delta_reads_bytes,
        cur.writes_bytes - COALESCE(prev.writes_bytes, 0) AS delta_writes_bytes,
        cur.user_time    - COALESCE(prev.user_time, 0)    AS delta_user_time,
        cur.calls        - COALESCE(prev.calls, 0)        AS delta_calls
    FROM kcache_snapshot cur
    CROSS JOIN (SELECT MIN(snapshot_ts) AS prev_ts,
                       MAX(snapshot_ts) AS cur_ts FROM snapshots) t
    LEFT JOIN kcache_snapshot prev
        ON  prev.queryid = cur.queryid
        AND prev.userid  = cur.userid
        AND prev.dbid    = cur.dbid
        AND prev.snapshot_ts = t.prev_ts
    WHERE cur.snapshot_ts = t.cur_ts
)
SELECT
    d.queryid,
    LEFT(s.query, 150) AS query_preview,
    d.delta_calls       AS calls_in_period,
    d.delta_reads_bytes AS physical_reads_kb,
    d.delta_writes_bytes AS physical_writes_kb,
    ROUND(d.delta_user_time, 2) AS cpu_sec_in_period
FROM diff d
JOIN pg_stat_statements s USING (userid, dbid, queryid)
WHERE d.delta_reads_bytes + d.delta_writes_bytes > 0
ORDER BY d.delta_reads_bytes + d.delta_writes_bytes DESC
LIMIT 20;
```

> **補充（Senior Dev）**  
> 快照策略建議：
> - 開發 / 測試環境：每 5 分鐘快照，用於即時觀察索引變更、查詢改寫效果
> - 預發 / Staging：每小時快照，用於 load test 前後對比
> - 生產環境：每小時快照，保留 7 天。快照表建議 partition by day，並設定 retention policy

#### 6.4 效能開銷

| 環境 | `pg_stat_kcache.track` | 每查詢額外開銷 |
|------|------------------------|---------------|
| 開發 | `all` | ~15% |
| 預發 | `top` | ~10% |
| 生產 | `top` | ~8-10% |

開銷來源：`getrusage()` 是系統呼叫（syscall），每次查詢執行前後各呼叫一次。對於極短查詢（< 1ms），開銷比例較明顯；對於長查詢（> 100ms），開銷可忽略。

```sql
-- 生產環境建議設定
ALTER SYSTEM SET pg_stat_kcache.track = 'top';
ALTER SYSTEM SET pg_stat_kcache.track_planning = 'off';
SELECT pg_reload_conf();
```

#### 6.5 與其他工具對照

```mermaid
graph LR
    A["PostgreSQL Query"] --> B["pg_stat_statements<br/>Buffer IO / Timing"]
    A --> C["pg_stat_kcache<br/>OS IO / CPU"]
    A --> D["auto_explain<br/>Execution Plan"]
    A --> E["pg_stat_activity<br/>Current Activity"]

    B --> F["Logical Buffer 分析"]
    C --> G["Physical Resource 分析"]
    D --> H["Plan 分析（Node-Level）"]
    E --> I["即時 Session 狀態"]

    F --> J["Dashboard<br/>Grafana / Datadog"]
    G --> J
    H --> J
    I --> J

    style C fill:#fff3cd,stroke:#ffc107
    style J fill:#e8daef,stroke:#8e44ad
```

> **初學者導讀**  
> `pg_stat_kcache` 補足的是 OS 層「真實資源消耗」，與 `auto_explain`（補足執行計劃細節）是互補關係。三者搭配使用：`pg_stat_statements` 看「哪些查詢慢」→ `pg_stat_kcache` 看「慢在哪個資源（CPU/IO）」→ `auto_explain` 看「為什麼慢（哪個 plan node）」。

#### 6.6 限制與注意事項

| 限制 | 說明 | 應對方式 |
|------|------|----------|
| `ru_inblock` = 0（Linux 64-bit） | 某些 kernel 版本 `getrusage` 不支援 block IO 欄位，`reads`/`reads_bytes` 永遠為 0 | 升級 kernel ≥ 4.x；或改用 eBPF tool（bpftrace） |
| 不區分 Block Device | 所有檔案系統 IO 合併計算，無法分辨表空間位於哪個磁碟 | 搭配 `iotop -oP` per-process 觀察 |
| 不包含 Parallel Worker | Parallel query worker process 的 CPU/IO 不計入 leader | 自行查看 worker PID 的 /proc/[pid]/io |
| 不計入 BgWriter/Checkpointer | 背景程序的寫入不計入 | 用 `pg_stat_bgwriter` 分開查看 |
| `pg_stat_kcache_reset()` | 重置後全部歸零，無時間維度保留 | 自行建立快照機制（見 6.3） |
## 十、hypopg — 假設性索引分析

> **初學者導讀**
> 想像你是一位建築師，客戶問：「如果我們在 3 樓和 5 樓之間加裝一台電梯，每天能節省多少時間？」你不會真的立刻施工——你會先用建築模型模擬人流，估算改善效果。`hypopg` 做的事情就類似：它讓你「假裝」建立了一個索引，然後用 `EXPLAIN` 觀察 PostgreSQL 的查詢計劃器如何利用這個索引，估算能省多少 I/O 開銷。整個過程中，PostgreSQL 不會寫入任何磁碟、不會消耗任何儲存空間、不會影響任何正在執行的查詢。它是一個純記憶體中的「沙盒環境」，讓你在零成本的情況下驗證索引策略。
>
> **補充（Senior Dev）**
> hypopg 透過 PostgreSQL 的 `planner_hook` 攔截查詢計劃階段，在 planner 掃描可用的索引時，將「假設性索引」注入候選路徑中。這意味著它**只影響 planner，不影響 executor**——你觀察到的是計劃器的決策變化，而非實際執行效能的變化。因此在關鍵場合（例如 Production 資料量級的效能回歸測試），`EXPLAIN ANALYZE` 才能真正驗證 executor 層面的行為，而 hypopg 的 `EXPLAIN`（不含 `ANALYZE`）是純計劃層面的分析。

---

### 1. 核心價值

#### I. 「如果我建立這個索引，查詢會快多少？」

hypopg 回答的是所有 DBA 和 App Dev 最常面對的問題：**在真正付出索引成本之前，先驗證它能帶來多大收益**。一個真實的 B-tree 索引需要：

- **磁碟空間**：通常佔資料表大小的 10-30%
- **寫入懲罰（Write Penalty）**：每次 `INSERT` / `UPDATE` / `DELETE` 都需維護所有相關索引
- **VACUUM 開銷**：索引頁面需要定期清理死元組
- **建立時間**：在大型表上執行 `CREATE INDEX CONCURRENTLY` 可能需要數小時

hypopg 的假設性索引全部繞過了這些成本，讓你可以在**完全不影響 Production 效能**的情況下測試任意數量的索引策略。

```mermaid
graph TB
    subgraph "傳統索引評估流程"
        A1["猜測：need index on col X"] --> B1["CREATE INDEX（花費時間/空間）"]
        B1 --> C1["EXPLAIN 驗證"]
        C1 --> D1{"效果好？"}
        D1 -->|"是"| E1["保留索引 · 定期維護"]
        D1 -->|"否"| F1["DROP INDEX（浪費時間/資源）"]
    end

    subgraph "hypopg 評估流程"
        A2["猜測：need index on col X"] --> B2["hypopg_create_index（瞬間完成）"]
        B2 --> C2["EXPLAIN 驗證"]
        C2 --> D2{"效果好？"}
        D2 -->|"是"| E2["CREATE INDEX · 一次性成功"]
        D2 -->|"否"| F2["hypopg_reset（零成本放棄）"]
    end

    style A1 fill:#ffd43b,color:black
    style F1 fill:#ff6b6b,color:white
    style A2 fill:#ffd43b,color:black
    style E2 fill:#5cb85c,color:white
    style F2 fill:#ffd43b,color:black
```

#### II. 零成本特性總結

| 維度 | 真實索引 | hypopg 假設索引 |
|------|---------|---------------|
| 磁碟空間 | 表大小的 10-30% | 0（純記憶體） |
| 寫入懲罰 | 每次 DML 都需維護 | 無（不實際存在） |
| WAL 產生 | 是 | 否 |
| 建立時間（大表） | 數分鐘～數小時 | 瞬時 |
| VISIBILITY | 所有 session | 僅當前 session 的 backend 記憶體 |
| persistence | 跨越 server restart | session 結束時自動銷毀 |
| 對 planner 的影響 | 永久 | 僅當前 session |

#### III. App Dev 的核心場景

- **在 Staging 環境模擬 Production 資料量**：不需要真的在 Staging 建立所有可能的索引，只需用 hypopg 測試最優組合
- **評估 ORM 自動產生的查詢**：框架產生的 SQL 往往難以預測，hypopg 讓你快速測試哪些欄位需要索引
- **多人開發的索引試驗隔離**：每個開發者的 session 擁有獨立的假設索引空間，互不干擾
- **CI/CD pipeline 中的索引建議**：自動化測試可以對慢查詢自動產生假設索引並評估效果

---

### 2. 工作原理

#### I. Planner Hook 攔截機制

hypopg 的核心是透過 PostgreSQL 的 **Planner Hook（計劃器掛鉤）** 機制注入假設性索引。整個流程如下：

1. **註冊 `planner_hook`**：hypopg 在載入 extension 時註冊一個自訂的計劃器掛鉤函數
2. **攔截計劃階段**：當任何查詢進入 planner 時，hypopg 的 hook 先被呼叫
3. **注入虛擬索引**：hook 函數掃描當前 session 中已定義的假設性索引，將其以虛擬 `pg_index` / `pg_class` / `pg_attribute` 條目的形式注入 planner 的內部目錄快取
4. **Planner 生成候選路徑**：planner 在 `get_relation_info()` 階段看到這些「虛擬」索引，將它們納入 Index Scan / Index Only Scan 的路徑生成
5. **選擇最優計劃**：planner 比較包含虛擬索引的候選路徑與原有路徑（Seq Scan 等），選出成本最低的執行計劃
6. **產生 EXPLAIN 輸出**：EXPLAIN 輸出中會顯示類似 `<hypopg_btree_table_col> Index Scan` 的計劃節點
7. **Session 結束銷毀**：backend 斷開時，該 session 的所有假設性索引自動從記憶體中釋放

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Backend
    participant H as hypopg Hook
    participant P as PostgreSQL Planner
    participant E as Executor

    C->>B: SELECT hypopg_create_index('CREATE INDEX ON t(a)')
    B->>B: 將虛擬索引存入 backend local memory (hash table)
    B-->>C: 回傳假設索引 OID

    C->>B: EXPLAIN SELECT * FROM t WHERE a = 1
    B->>H: planner_hook 被觸發
    H->>H: 從 local memory 讀取所有假設性索引
    H->>P: 將虛擬索引注入 planner 的 relcache (fake pg_index / pg_class / pg_attribute)
    P->>P: get_relation_info() 看到虛擬索引
    P->>P: 生成 Index Scan 候選路徑 + 計算成本
    P->>P: 比較所有路徑，選擇最優計畫
    P-->>B: 回傳最終計畫（含 hypopg index scan）
    B-->>C: EXPLAIN 輸出：Index Scan using &lt;hypopg_btree_t_a&gt;

    C->>B: SET hypopg.use_hypopg = off
    C->>B: EXPLAIN SELECT * FROM t WHERE a = 1
    B->>H: planner_hook 被觸發
    H->>P: hypopg.use_hypopg = off，不注入虛擬索引
    P->>P: 僅有原生的 Seq Scan 候選路徑
    P-->>B: 回傳原計劃
    B-->>C: EXPLAIN 輸出：Seq Scan on t
```

#### II. 記憶體儲存結構

假設性索引的元數據儲存在 **backend 程序的本地記憶體**中，使用 hash table 組織：

- **Hypertable**：映射到真實的 OID（`pg_class.oid`），儲存該表上的所有假設性索引
- **HypoIndex**：每個假設性索引的結構包含：索引類型（btree / brin / hash）、索引欄位（可多欄位）、部分索引的 `WHERE` 子句、估算大小等
- **不修改任何 system catalog**：不寫入 `pg_index`、`pg_class`、`pg_attribute` 等任何系統表
- **不產生 WAL**：因為沒有任何磁碟寫入

```mermaid
graph TD
    subgraph "Backend Process Memory"
        subgraph "hypopg Hash Table"
            HT["Hypertable 1\n(reloid = 16433)"] --> HI1["HypoIndex A\ncol: email (btree)"]
            HT --> HI2["HypoIndex B\ncol: status, created_at (btree)"]
            HT --> HI3["HypoIndex C\ncol: amount (btree)\nWHERE status='active'"]
            HT2["Hypertable 2\n(reloid = 16445)"] --> HI4["HypoIndex D\ncol: user_id (hash)"]
        end

        subgraph "Planner Hook 注入"
            PH["hypopg planner_hook"] --> |"get_relation_info() 時"| FC["Fake Catalog Entries\n虛擬 pg_index / pg_class / pg_attribute"]
        end
    end

    subgraph "Disk / Shared Memory"
        SC["Real System Catalogs\n(pg_index, pg_class, pg_attribute)\n— 完全不被修改 —"]
    end

    HT -.->|"不寫入"| SC
    HI1 -.->|"不寫入"| SC
    FC -.->|"僅存在於 planner\n單次呼叫的記憶體中"| SC

    style HT fill:#4a90d9,color:white
    style HI1 fill:#e8d44d,color:black
    style SC fill:#5cb85c,color:white
```

#### III. 與真實索引的 Planner 行為差異

> **補充（Senior Dev）**
> hypopg 提供的 planner 決策與真實索引存在細微差異，主要原因有二：
> 1. **統計資訊不足**：真實索引建立後，`ANALYZE` 會收集該索引的統計資訊（如 `pg_stats` 中的直方圖、MCV 等）。hypopg 無法模擬這些統計，因此 planner 在估算 index scan 的選擇率（selectivity）時可能略有偏差。
> 2. **成本參數假設**：hypopg 使用通用的成本模型估算 index scan 的成本（如 `random_page_cost`、`cpu_index_tuple_cost`），但無法知道真實索引的 B-tree 深度、leaf page 數量等微觀參數。這在大部分場景下差異不大，但在極端情況（如索引欄位的高重複值、極傾斜分佈）下可能導致 planner 的選擇與真實情況不一致。

---

### 3. 安裝與配置（PG 16）

#### I. 安裝 Extension

hypopg 是標準的 PostgreSQL extension，不需要 `shared_preload_libraries`（因為它使用 planner hook 而非 shared memory hook）：

```sql
-- Step 1: 連接到目標資料庫
\c mydatabase

-- Step 2: 建立 extension
CREATE EXTENSION IF NOT EXISTS hypopg;

-- Step 3: 驗證載入
SELECT * FROM pg_extension WHERE extname = 'hypopg';
-- 預期輸出：extname='hypopg', extversion='1.4.1'（版本因 PG 版本而異）
```

```mermaid
flowchart LR
    A["apt install postgresql-16-hypopg\n或\nyum install hypopg_16"] --> B["psql 連接資料庫"]
    B --> C["CREATE EXTENSION hypopg"]
    C --> D["驗證: SELECT * FROM hypopg_list_indexes()"]
    D --> E["✅ 無需 restart PostgreSQL\n無需 shared_preload_libraries"]

    style C fill:#5cb85c,color:white
    style E fill:#5cb85c,color:white
```

#### II. GUC 參數

| 參數 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `hypopg.use_hypopg` | boolean | `on` | 控制是否在當前 session 中啟用假設性索引。設為 `off` 時，planner 會忽略所有假設性索引，等同於不載入 hypopg 的 hook |

```sql
-- 當前 session 關閉 hypopg（恢復原生 planner 行為）
SET hypopg.use_hypopg = off;

-- 重新啟用
SET hypopg.use_hypopg = on;

-- 查看當前狀態
SHOW hypopg.use_hypopg;
```

注意：`hypopg.use_hypopg` 是 **session 級別**的 GUC 參數，不支援全域設定（無需寫入 `postgresql.conf`），也不需要 `pg_reload_conf()`。

#### III. 確認安裝成功

```sql
-- 檢查所有已載入的 hook 相關設定
SELECT name, setting FROM pg_settings WHERE name LIKE 'hypopg%';

-- 檢查目前 session 中是否有假設性索引
SELECT * FROM hypopg_list_indexes();
-- 預期輸出：0 rows（尚未建立任何假設性索引）

-- 檢查 extension 版本
SELECT hypopg_version();
```

---

### 4. 核心操作

#### I. 建立假設索引

`hypopg_create_index()` 是最核心的函數，語法與 `CREATE INDEX` 完全相同（字串形式傳入）：

```sql
-- 基本單欄 B-tree 索引
SELECT hypopg_create_index('CREATE INDEX ON users(email)');
-- 回傳值：假設性索引的 OID（例如：16397）

-- 多欄複合索引
SELECT hypopg_create_index('CREATE INDEX ON orders(user_id, status, created_at DESC)');

-- 部分索引（Partial Index）
SELECT hypopg_create_index('CREATE INDEX ON products(price) WHERE status = ''active''');

-- 唯一索引（Unique Index）在 PG 14+ 的 hypopg 1.4+ 支援
SELECT hypopg_create_index('CREATE UNIQUE INDEX ON users(email)');

-- BRIN 索引
SELECT hypopg_create_index('CREATE INDEX ON sensor_readings USING brin(recorded_at)');

-- Hash 索引
SELECT hypopg_create_index('CREATE INDEX ON sessions USING hash(session_token)');

-- 指定索引名稱（建議，便於辨識）
SELECT hypopg_create_index('CREATE INDEX hypox_users_email ON users(email)');
```

> **初學者導讀**
> `hypopg_create_index()` 的語法就是一個完整的 `CREATE INDEX` 語句包在單引號中。你可以先在 psql 中寫好 `CREATE INDEX ...`，測試無誤後，再把整句包進 `SELECT hypopg_create_index('...')`。回傳的 OID 在後續操作（如刪除特定假設索引）時會用到。

#### II. 驗證假設索引效果

#### a. 基本 EXPLAIN 驗證

```sql
-- Step 1: 建立假設索引
SELECT hypopg_create_index('CREATE INDEX ON users(email)');

-- Step 2: 觀察 planner 是否使用該索引
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
```

預期輸出範例：

```
                                    QUERY PLAN
-----------------------------------------------------------------------------------
 Index Scan using <hypopg_btree_users_email> on users  (cost=0.42..8.44 rows=1 width=64)
   Index Cond: (email = 'alice@example.com'::text)
(2 rows)
```

關鍵觀察點：
- `<hypopg_btree_users_email>`：包含 `hypopg` 前綴，表示這是假設性索引
- `cost=0.42..8.44`：這個成本估算反映 planner 認為使用該索引的成本。`0.42` 是 startup cost，`8.44` 是 total cost
- `rows=1`：planner 估算的回傳行數

#### b. 與無索引對比

```sql
-- Step 1: 開啟 hypopg，觀察有假設索引的計劃
SET hypopg.use_hypopg = on;
EXPLAIN (COSTS ON) SELECT * FROM users WHERE email = 'alice@example.com';
-- 輸出：Index Scan using <hypopg_btree_users_email>

-- Step 2: 關閉 hypopg，觀察原生計劃
SET hypopg.use_hypopg = off;
EXPLAIN (COSTS ON) SELECT * FROM users WHERE email = 'alice@example.com';
-- 輸出：Seq Scan on users  (cost=0.00..35210.00 rows=1 width=64)
--       Filter: (email = 'alice@example.com'::text)

-- Step 3: 對比成本差異
-- Seq Scan total cost:  35210.00
-- Index Scan total cost:     8.44
-- 改善倍數：35210.00 / 8.44 ≈ 4171x（顯然這個估算顯示索引有巨大收益）
```

#### c. 使用 EXPLAIN 的完整選項

```sql
-- 以 JSON 格式輸出，方便程式解析
EXPLAIN (FORMAT JSON) SELECT * FROM users WHERE email = 'alice@example.com';

-- 顯示更詳細的成本明細
EXPLAIN (ANALYZE OFF, COSTS ON, BUFFERS ON, VERBOSE ON)
SELECT * FROM users WHERE email = 'alice@example.com';
-- 注意：不能用 ANALYZE ON，因為假設索引不存在，executor 會報錯
```

> **補充（Senior Dev）**
> 永遠不要在 hypopg 環境下使用 `EXPLAIN ANALYZE`——因為假設索引實際上不存在，executor 嘗試使用它時會得到錯誤。hypopg 的邊界是 planner 層級，`EXPLAIN`（不含 `ANALYZE`）就足夠了。如果你需要同時看到 planner 決策和實際執行統計，正確做法是：先用 hypopg 確認 planner 選擇，再用 `SET hypopg.use_hypopg = off; EXPLAIN ANALYZE ...` 觀察原生的實際執行情況。

#### III. 管理假設索引

hypopg 提供了一組輔助函數來管理 session 中的假設性索引：

#### a. hypopg_list_indexes() — 列出所有假設索引

```sql
SELECT * FROM hypopg_list_indexes();
```

輸出欄位：

| 欄位 | 說明 |
|------|------|
| `indexrelid` | 假設索引的 OID |
| `indexname` | 索引名稱（如 `hypopg_btree_users_email`） |
| `nspname` | schema 名稱 |
| `relname` | 目標資料表名稱 |
| `amname` | 索引存取方法（btree / brin / hash / gist 等） |
| `ncols` | 索引欄位數 |
| `keycols` | 欄位編號陣列 |
| `keynames` | 欄位名稱陣列 |
| `indrelid` | 目標資料表的 OID |

#### b. hypopg_drop_index() — 刪除特定假設索引

```sql
-- 刪除指定 OID 的假設索引
SELECT hypopg_drop_index(16397);

-- 搭配 hypopg_list_indexes() 使用
SELECT hypopg_drop_index(indexrelid)
FROM hypopg_list_indexes()
WHERE indexname = 'hypopg_btree_users_email';
```

#### c. hypopg_reset() — 清除所有假設索引

```sql
-- 一鍵清除當前 session 中的所有假設索引
SELECT hypopg_reset();

-- 驗證已清除
SELECT * FROM hypopg_list_indexes();
-- 預期輸出：0 rows
```

#### d. hypopg_get_indexdef() — 取得索引定義

```sql
-- 取得指定 OID 的索引建構語句
SELECT hypopg_get_indexdef(16397);
-- 回傳：CREATE INDEX ON public.users USING btree (email)

-- 批量取得所有假設索引的定義
SELECT indexrelid, hypopg_get_indexdef(indexrelid) AS def
FROM hypopg_list_indexes();
```

#### IV. 分區表假設索引

hypopg 對分區表（Partitioned Table）有特殊處理：當你在**父表**上建立假設索引時，hypopg 會**自動在每個子分區上建立對應的假設索引**。這與 PostgreSQL 原生行為一致——在分區表上 `CREATE INDEX` 也會在所有子分區上建立索引。

```sql
-- 假設 orders 是按 created_at RANGE 分區的分區表
-- 包含 orders_2026_01, orders_2026_02, orders_2026_03 三個子分區

-- 在父表上建立假設索引
SELECT hypopg_create_index('CREATE INDEX ON orders(user_id)');

-- 查看所有產生的假設索引
SELECT indexname, relname, nspname
FROM hypopg_list_indexes();
-- 預期輸出：
-- hypopg_btree_orders_2026_01_user_id | orders_2026_01 | public
-- hypopg_btree_orders_2026_02_user_id | orders_2026_02 | public
-- hypopg_btree_orders_2026_03_user_id | orders_2026_03 | public

-- EXPLAIN 驗證——planner 會看到所有子分區上的索引
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
```

```mermaid
flowchart TB
    A["SELECT hypopg_create_index('CREATE INDEX ON orders(user_id)')"] --> B{"orders 是分區表？"}
    B -->|"是"| C["遍歷 pg_inherits 找出所有子分區"]
    C --> D1["子分區 orders_2026_01"]
    C --> D2["子分區 orders_2026_02"]
    C --> D3["子分區 orders_2026_03"]
    D1 --> E1["建立 hypopg index ON orders_2026_01(user_id)"]
    D2 --> E2["建立 hypopg index ON orders_2026_02(user_id)"]
    D3 --> E3["建立 hypopg index ON orders_2026_03(user_id)"]
    B -->|"否"| F["僅在單一表上建立 hypopg index"]

    style A fill:#4a90d9,color:white
    style E1 fill:#e8d44d,color:black
    style E2 fill:#e8d44d,color:black
    style E3 fill:#e8d44d,color:black
```

> **補充（Senior Dev）**
> 當分區數量很大（500+）時，在父表上建立假設索引會瞬間產生大量子分區的假設索引，可能會讓 planner 在路徑生成階段消耗較長時間（因為 planner 要為每個子分區評估索引路徑）。在這種場景下，建議先在幾個代表性的子分區上單獨建立假設索引進行分析，確認索引方向正確後再在父表層級建立真實索引。

#### V. 多列索引與部分索引測試

hypopg 支援所有 PostgreSQL 原生的索引語法，這意味著你可以精確模擬各種複雜索引策略：

```sql
-- 場景：電商訂單查詢
-- 查詢模式：SELECT * FROM orders WHERE user_id = $1 AND status = $2 ORDER BY created_at DESC;

-- 策略 A：單欄索引 on user_id
SELECT hypopg_create_index('CREATE INDEX hypox_a ON orders(user_id)');

-- 策略 B：複合索引 on (user_id, status)
SELECT hypopg_create_index('CREATE INDEX hypox_b ON orders(user_id, status)');

-- 策略 C：完整覆蓋索引 on (user_id, status, created_at DESC)
SELECT hypopg_create_index('CREATE INDEX hypox_c ON orders(user_id, status, created_at DESC)');

-- 策略 D：部分索引（只索引 active 狀態）
SELECT hypopg_create_index('CREATE INDEX hypox_d ON orders(user_id, created_at DESC) WHERE status = ''active''');

-- 現在對比不同策略的計劃成本
-- 使用 EXPLAIN 逐一測試
SET hypopg.use_hypopg = off;
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 cost: ____

SET hypopg.use_hypopg = on;

-- 僅測試策略 A（需要先 reset，再建立對應的）
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_a ON orders(user_id)');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 cost: ____

-- 策略 B
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_b ON orders(user_id, status)');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 cost: ____

-- 策略 C
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_c ON orders(user_id, status, created_at DESC)');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 cost: ____

-- 策略 D
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_d ON orders(user_id, created_at DESC) WHERE status = ''active''');
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'active' ORDER BY created_at DESC LIMIT 20;
-- 記錄 cost: ____
```

#### 策略對比分析

```mermaid
graph LR
    subgraph "索引策略對比"
        A["策略 A\n(user_id)"] --> CA["cost: 850\nFilter: status"]
        B["策略 B\n(user_id, status)"] --> CB["cost: 120\nNo Filter, Sort 仍需"]
        C["策略 C\n(user_id, status, created_at DESC)"] --> CC["cost: 8\nIndex Only Scan\nNo Sort"]
        D["策略 D\n部分索引 WHERE status='active'"] --> CD["cost: 5\n最小索引 · 最快\n但僅覆蓋 active 查詢"]
    end

    style CC fill:#5cb85c,color:white
    style CD fill:#4a90d9,color:white
```

> **初學者導讀**
> 上述測試展示了一個經典的索引選型過程。策略 C（完整覆蓋索引）和策略 D（部分索引）都有很好的 planner 成本，但選擇哪個取決於業務需求：如果 `status = 'active'` 是查詢的 90% 場景，策略 D 更輕量；如果查詢涵蓋多種 status，策略 C 更通用。

#### VI. 索引大小估算

`hypopg_relation_size()` 函數可以估算假設索引如果被真實建立的磁碟佔用大小：

```sql
-- 估算特定假設索引的磁碟大小
SELECT hypopg_relation_size(16397);
-- 回傳值以 bytes 為單位

-- 人類可讀格式
SELECT
    indexname,
    relname,
    pg_size_pretty(hypopg_relation_size(indexrelid)) AS estimated_size
FROM hypopg_list_indexes();
```

輸出範例：

```
           indexname            |  relname  | estimated_size
--------------------------------+-----------+----------------
 hypopg_btree_users_email       | users     | 219 MB
 hypopg_btree_orders_user_id    | orders    | 856 MB
 hypopg_btree_products_price    | products  | 45 MB
```

```mermaid
pie title 假設索引估算大小分佈
    "users (email)" : 219
    "orders (user_id, status, created_at)" : 856
    "products (price WHERE active)" : 45
    "sessions (token hash)" : 312
```

> **補充（Senior Dev）**
> `hypopg_relation_size()` 的估算基於以下假設：B-tree 索引每筆 tuple 佔用約 8 bytes（header）+ 索引欄位寬度 + 4 bytes（item pointer）。實際大小可能因以下因素有偏差：（1）fillfactor 設定（預設 B-tree 為 90%，但部分頁面可能因刪除而稀疏）；（2）NULL 值的壓縮機制；（3）Deduplication（PG 13+ 引入的 B-tree 重複值去重）。估算值通常 ±20% 以內，足夠用於容量規劃。

---

### 5. 完整示例流程

以下是一個完整的索引評估工作流，從發現慢查詢到最終建立真實索引：

#### Step 1: 從 pg_stat_statements 找到慢查詢

```sql
-- 找出 Top-5 最耗時的查詢
SELECT
    queryid,
    LEFT(query, 100) AS query_preview,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round(total_exec_time::numeric, 2) AS total_ms,
    round((100.0 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

假設結果顯示以下查詢是最大的效能瓶頸：

```
query_preview:
SELECT p.id, p.name, p.description, p.price, i.url
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
WHERE p.category_id = $1
  AND p.status = $2
ORDER BY p.price DESC
LIMIT $3

calls: 15230
avg_ms: 342.18
total_ms: 5211401.40
pct: 38.50
```

#### Step 2: 複製查詢並準備測試數據

```sql
-- 取得查詢的完整文字
SELECT query
FROM pg_stat_statements
WHERE queryid = 123456789;

-- 用具體值替換 $N（或直接在測試中使用 PREPARE / EXECUTE）
-- 測試用具體常數：
--   $1 = 5 (category_id)
--   $2 = 'active' (status)
--   $3 = 20 (LIMIT)
```

#### Step 3: 建立三種不同的假設索引

```sql
-- 策略 I：單欄索引 — 最簡單的方案
SELECT hypopg_create_index('CREATE INDEX hypox_i ON products(category_id)');

-- 策略 II：複合索引 — 覆蓋查詢的過濾條件
SELECT hypopg_create_index('CREATE INDEX hypox_ii ON products(category_id, status, price DESC)');

-- 策略 III：部分索引 + 覆蓋索引 — 針對性最強
SELECT hypopg_create_index(
    'CREATE INDEX hypox_iii ON products(category_id, price DESC) WHERE status = ''active'''
);

-- 同時考慮 inventory 表的索引
SELECT hypopg_create_index('CREATE INDEX hypox_inv ON inventory(product_id)');
```

#### Step 4: EXPLAIN 每種策略

```sql
-- 測試用的具體 SQL（用常數取代 $N）
-- EXPLAIN 語句：
-- EXPLAIN SELECT p.id, p.name, p.description, p.price, i.url
-- FROM products p
-- LEFT JOIN inventory i ON p.id = i.product_id
-- WHERE p.category_id = 5 AND p.status = 'active'
-- ORDER BY p.price DESC
-- LIMIT 20;

-- 4a) 無索引基準（baseline）
SET hypopg.use_hypopg = off;
EXPLAIN SELECT p.id, p.name, p.description, p.price, i.url
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
WHERE p.category_id = 5 AND p.status = 'active'
ORDER BY p.price DESC
LIMIT 20;
-- Seq Scan on products  (cost=0.00..48520.00 rows=1200 width=256)
--   Filter: (category_id = 5 AND status = 'active')
-- Hash Left Join → inventory (cost=...)

-- 4b) 策略 I
SELECT hypopg_reset();
SET hypopg.use_hypopg = on;
SELECT hypopg_create_index('CREATE INDEX hypox_i ON products(category_id)');
SELECT hypopg_create_index('CREATE INDEX hypox_inv ON inventory(product_id)');
EXPLAIN SELECT p.id, p.name, p.description, p.price, i.url
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
WHERE p.category_id = 5 AND p.status = 'active'
ORDER BY p.price DESC
LIMIT 20;
-- Index Scan using hypox_i on products  (cost=0.42..8230.00 rows=1200 width=256)
--   Index Cond: (category_id = 5)
--   Filter: (status = 'active')  ← still filtering, then sort needed

-- 4c) 策略 II
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX hypox_ii ON products(category_id, status, price DESC)');
SELECT hypopg_create_index('CREATE INDEX hypox_inv ON inventory(product_id)');
EXPLAIN SELECT p.id, p.name, p.description, p.price, i.url
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
WHERE p.category_id = 5 AND p.status = 'active'
ORDER BY p.price DESC
LIMIT 20;
-- Index Only Scan using hypox_ii on products  (cost=0.42..164.00 rows=20 width=256)
--   Index Cond: (category_id = 5 AND status = 'active')
--   No Filter, No Sort ← 完美覆蓋

-- 4d) 策略 III
SELECT hypopg_reset();
SELECT hypopg_create_index(
    'CREATE INDEX hypox_iii ON products(category_id, price DESC) WHERE status = ''active'''
);
SELECT hypopg_create_index('CREATE INDEX hypox_inv ON inventory(product_id)');
EXPLAIN SELECT p.id, p.name, p.description, p.price, i.url
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
WHERE p.category_id = 5 AND p.status = 'active'
ORDER BY p.price DESC
LIMIT 20;
-- Index Only Scan using hypox_iii on products  (cost=0.29..98.00 rows=20 width=256)
--   Index Cond: (category_id = 5)
--   索引最小，成本最低，但僅對 status='active' 有效
```

#### Step 5: 對比結果

```sql
-- 使用表格化對比
SELECT
    'Baseline' AS strategy,
    'Seq Scan' AS method,
    48520.00 AS estimated_cost
UNION ALL
SELECT 'Strategy I', 'Index Scan + Filter + Sort', 8230.00
UNION ALL
SELECT 'Strategy II', 'Index Only Scan (No Sort)', 164.00
UNION ALL
SELECT 'Strategy III', 'Index Only Scan (Partial)', 98.00
ORDER BY estimated_cost DESC;
```

```
   strategy   |            method            | estimated_cost
--------------+------------------------------+----------------
 Baseline     | Seq Scan                     |       48520.00
 Strategy I   | Index Scan + Filter + Sort   |        8230.00
 Strategy II  | Index Only Scan (No Sort)    |         164.00
 Strategy III | Index Only Scan (Partial)    |          98.00
```

#### Step 6: 選擇最佳策略並估算大小

```sql
-- 查看各策略的估算大小
SELECT hypopg_reset();

SELECT hypopg_create_index('CREATE INDEX hypox_ii ON products(category_id, status, price DESC)');
SELECT indexname, pg_size_pretty(hypopg_relation_size(indexrelid)) AS estimated_size
FROM hypopg_list_indexes()
WHERE relname = 'products';

SELECT hypopg_reset();

SELECT hypopg_create_index(
    'CREATE INDEX hypox_iii ON products(category_id, price DESC) WHERE status = ''active'''
);
SELECT indexname, pg_size_pretty(hypopg_relation_size(indexrelid)) AS estimated_size
FROM hypopg_list_indexes()
WHERE relname = 'products';

-- 假設結果：
-- Strategy II:  820 MB（全部資料，含所有 status）
-- Strategy III: 180 MB（僅 active 資料）
```

#### Step 7: 建立真實索引

根據成本分析結果，選擇策略 II（通用性較好）或策略 III（若 active 查詢佔比 > 90%）：

```sql
-- 若選擇策略 II（覆蓋所有 status）
CREATE INDEX CONCURRENTLY idx_products_cat_status_price
    ON products(category_id, status, price DESC);

-- 若選擇策略 III（僅 active）
CREATE INDEX CONCURRENTLY idx_products_active_cat_price
    ON products(category_id, price DESC) WHERE status = 'active';

-- 以及 inventory 表的索引
CREATE INDEX CONCURRENTLY idx_inventory_product_id
    ON inventory(product_id);
```

#### 完整流程圖

```mermaid
flowchart TB
    A["pg_stat_statements\n找出 Top-5 慢查詢"] --> B["複製目標查詢\n用常數取代 $N"]
    B --> C["設計 2-4 種索引策略"]
    C --> D["hypopg_create_index()\n建立假設索引"]
    D --> E["EXPLAIN 每種策略\n記錄 cost 與 Scan 類型"]
    E --> F{"哪個策略\n成本最低？"}
    F --> G["再檢查 hypopg_relation_size()\n確認磁碟空間可接受"]
    G --> H{"大小與成本\n都滿意？"}
    H -->|"是"| I["CREATE INDEX CONCURRENTLY\n在非尖峰時段建立真實索引"]
    H -->|"否\n(大小太大)"| J["考慮部分索引\n或調整索引欄位"]
    J --> D
    I --> K["驗證真實索引效果\nEXPLAIN ANALYZE"]
    K --> L["✅ 完成 · 監控效能改善"]

    style A fill:#4a90d9,color:white
    style D fill:#e8d44d,color:black
    style I fill:#5cb85c,color:white
    style L fill:#5cb85c,color:white
```

---

### 6. 限制與注意事項

#### I. 僅影響 Planner，不影響 Executor

hypopg 最核心的限制：它運作在 **Planner 層級**，無法影響 Executor（執行引擎）。這帶來幾個實務影響：

```sql
-- ❌ 錯誤：不可以在 hypopg 環境下使用 EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';
-- Error: relation "<hypopg_btree_users_email>" does not exist
-- 原因：executor 真的嘗試打開索引時發現它不存在

-- ✅ 正確：只用 EXPLAIN（不含 ANALYZE）
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
```

因此 hypopg 無法回答以下問題：
- 真實的執行時間（需 `EXPLAIN ANALYZE` + 真實索引）
- 實際的 buffer 命中和磁碟讀取（需 `EXPLAIN (ANALYZE, BUFFERS ON)`）
- 並行查詢（Parallel Query）的真實行為

#### II. 不持久化

假設性索引存在於 backend 程序的 local memory 中，以下情況會導致它們消失：

- **session 斷開**（`\q` 或連線中斷）
- **backend 重啟**（例如 `pg_terminate_backend()`）
- **`hypopg_reset()`** 被呼叫

這既是限制，也是設計意圖——hypopg 的「零影響」承諾依賴於這種瞬態特性。

```sql
-- 確認 session 隔離性
-- Session A:
SELECT hypopg_create_index('CREATE INDEX ON users(email)');
SELECT count(*) FROM hypopg_list_indexes();  -- 回傳: 1

-- Session B（另一個 psql 視窗）:
SELECT count(*) FROM hypopg_list_indexes();  -- 回傳: 0
-- 證明了 session 間的完全隔離
```

#### III. Planner 決策可能與現實有偏差

| 偏差來源 | 影響 | 嚴重程度 |
|---------|------|---------|
| 無真實統計資訊（`pg_stats` 無索引直方圖） | planner 使用通用成本模型，可能低估/高估 index scan 選擇率 | 中（通常 ±15%） |
| 無真實 B-tree 深度 | `cpu_index_tuple_cost` 估算可能偏差 | 低 |
| 無法反映實體儲存佈局 | 無 `correlation` 統計，影響 bitmap scan 成本判斷 | 中 |
| 無法模擬 index-only scan 的 visibility map | planner 可能過度樂觀地選擇 index-only scan | 中（PG 14+ 改善） |

```sql
-- 驗證真實索引效果（建立真實索引後執行）
CREATE INDEX CONCURRENTLY idx_real_users_email ON users(email);

-- now compare planner estimates with actual execution
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS ON)
SELECT * FROM users WHERE email = 'alice@example.com';

-- 重點比較：
-- 1. estimated rows vs actual rows
-- 2. estimated cost vs actual time
-- 3. shared buffers hit vs read
```

#### IV. Parallel Query 計劃可能不完全考慮

PG 9.6+ 的 Parallel Query 執行引擎在某些場景下，planner 可能無法完全考慮假設性索引在 parallel worker 中的行為。若你的查詢通常觸發 parallel plan（特別是 `min_parallel_table_scan_size` 設定較低時），hypopg 的 planner 輸出可能與實際的 parallel index scan 不完全一致。

#### V. Extension 相容性

```sql
-- hypopg 與其他使用 planner_hook 的 extension 可能衝突
-- 因為 PostgreSQL 的 planner_hook 是單一函數指標
-- 若多個 extension 都需要掛載 planner_hook，後載入的會覆蓋前者

-- 檢查是否有其他使用 planner_hook 的 extension
SELECT * FROM pg_extension WHERE extname IN (
    'pg_plan_advsr', 'pg_hint_plan', 'pg_plan_guider'
);
```

#### VI. 不支援的索引類型

| 索引類型 | hypopg 支援 | 說明 |
|---------|------------|------|
| B-tree | ✅ 完全支援 | — |
| Hash | ✅ 完全支援 | — |
| BRIN | ✅ 完全支援 | — |
| GiST | ✅ 支援 | 部分 extension（如 PostGIS）提供的 opclass 可能不完整 |
| GIN | ⚠️ 部分支援 | GIN 的內部結構（posting list / tree）較複雜，成本估算可能偏差較大 |
| SP-GiST | ⚠️ 部分支援 | 同 GIN，成本估算不精確 |
| 表示式索引 (Expression Index) | ✅ 支援 | `CREATE INDEX ON users((lower(email)))` |
| Include 索引 (Covering Index) | ✅ 支援 | `INCLUDE (extra_col)` |

---

### 7. App Dev 最佳實踐

#### I. 環境分層策略

hypopg 最適合在**非 Production 環境**中使用，典型的分層策略為：

```mermaid
flowchart TB
    subgraph "Dev / Local"
        D1["每個開發者獨立 session"]
        D2["快速測試 ORM 查詢的索引需求"]
        D3["CI/CD pipeline 自動索引建議"]
    end

    subgraph "Staging"
        S1["接近 Production 的資料量"]
        S2["hypopg 選定最佳索引策略"]
        S3["建立真實索引 → EXPLAIN ANALYZE 驗證"]
        S4["效能回歸測試"]
    end

    subgraph "Production"
        P1["❌ 永遠不要在 Production 使用 hypopg"]
        P2["僅部署在 Staging 驗證過的索引"]
        P3["使用 CREATE INDEX CONCURRENTLY"]
        P4["監控索引效果（pg_stat_user_indexes）"]
    end

    D1 --> D2
    D2 --> D3
    D3 --> S1
    S1 --> S2
    S2 --> S3
    S3 --> S4
    S4 -->|"審核通過"| P2

    style P1 fill:#ff6b6b,color:white
    style P2 fill:#5cb85c,color:white
    style D1 fill:#4a90d9,color:white
    style S1 fill:#e8d44d,color:black
```

> **補充（Senior Dev）**
> 在 Production 環境使用 hypopg 有兩個問題：（1）planner hook 會為每個查詢增加額外的記憶體掃描開銷，雖然很小但對高 QPS 環境不可忽略；（2）hypopg 的 session 若被 connection pooler（如 pgbouncer）複用，可能導致假設索引意外遺留到下一個使用該 connection 的 request，干擾 planner 決策。因此 Production 環境應始終保持 `hypopg.use_hypopg = off`，或乾脆不載入 extension。

#### II. 與 pg_stat_statements 的協作工作流

hypopg 最強的價值體現在與 pg_stat_statements 的組合使用中：

```sql
-- 完整的索引最佳化工作流（可腳本化）

-- 1. 從 pg_stat_statements 取出 Top-N 慢查詢
CREATE TEMP TABLE slow_queries AS
SELECT queryid, query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- 2. 對每條慢查詢遍歷所有表的候選索引欄位
-- 下面的 PL/pgSQL 函數示範將 hypopg 與 pg_stat_statements 結合自動化分析

CREATE OR REPLACE FUNCTION analyze_slow_queries()
RETURNS TABLE(queryid bigint, recommended_index text, estimated_improvement text)
LANGUAGE plpgsql AS $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN
        SELECT queryid, query
        FROM pg_stat_statements
        WHERE calls > 100
        ORDER BY total_exec_time DESC
        LIMIT 10
    LOOP
        -- 需要人工介入的步驟：
        -- a. 分析 query，找出可能的索引欄位（可用 pg_store_plans 或手動分析 WHERE 子句）
        -- b. 用 hypopg_create_index 測試
        -- c. 用 EXPLAIN 對比成本
        -- d. 記錄推薦結果

        -- 此處為概念示範，實際實現需解析 SQL 語法樹
        RAISE NOTICE 'Analyzing queryid: %, query: %', rec.queryid, rec.query;
    END LOOP;
END;
$$;
```

#### III. CI/CD 整合

將 hypopg 整合到 CI/CD pipeline 中，自動化索引建議：

```yaml
## .github/workflows/index-analysis.yml (概念示例)
name: Index Analysis

on:
  pull_request:
    paths:
      - 'migrations/**'
      - 'app/models/**'

jobs:
  analyze-indexes:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb

    steps:
      - name: Setup test database
        run: |
          psql -h localhost -U postgres -d testdb << 'EOF'
          CREATE EXTENSION IF NOT EXISTS hypopg;
          CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
          -- Load schema and sample data
          \i schema.sql
          \i seed_data.sql
          EOF

      - name: Run index analysis script
        run: |
          psql -h localhost -U postgres -d testdb << 'EOF'
          -- 對每個 ORM 產生的關鍵查詢進行 hypopg 測試
          -- 輸出建議的 CREATE INDEX 語句

          -- 示例：
          SELECT hypopg_create_index('CREATE INDEX ON users(email)');
          EXPLAIN (FORMAT JSON) SELECT * FROM users WHERE email = 'test@example.com';

          -- 若 cost 改善 > 50%，輸出建議
          -- (實際實現需要解析 EXPLAIN JSON 輸出並比較)
          EOF
```

#### IV. Session 隔離的實踐

每位開發者擁有獨立的 hypopg session 空間，這是最自然的隔離機制：

```sql
-- 開發者 A 的 session（psql 視窗 1）
SELECT hypopg_create_index('CREATE INDEX hypox_a ON users(email)');
SELECT hypopg_create_index('CREATE INDEX hypox_a ON orders(user_id)');
SELECT indexname FROM hypopg_list_indexes();
-- hypox_a (users)
-- hypox_a (orders)

-- 開發者 B 的 session（psql 視窗 2）
SELECT indexname FROM hypopg_list_indexes();
-- 0 rows — 完全不受影響

-- 好處：
-- 1. 不需要建立/刪除真實索引（避免 locker 衝突）
-- 2. 每個人可以獨立測試不同的索引策略
-- 3. session 結束自動清理，不會留下測試殘留
```

> **初學者導讀**
> 很多人第一次使用 hypopg 時會困惑：「為什麼我在 psql 建立的假設索引，在另一個 psql 視窗看不到？」這正是 hypopg 的設計意圖——它不是全域的，而是 per-session 的。想像每個開發者都在自己的「沙盒」裡做實驗，互不干擾。如果你需要在另一個 session 看到同樣的索引，就必須在該 session 中重新執行 `hypopg_create_index()`。

#### V. 常見陷阱與規避

| 陷阱 | 症狀 | 解決方案 |
|------|------|---------|
| 使用 `EXPLAIN ANALYZE` | `ERROR: relation does not exist` | 只用 `EXPLAIN`（不含 ANALYZE） |
| 在 Production 環境啟用 | planner 額外開銷 | `SET hypopg.use_hypopg = off` 或不安裝 |
| 忘記測試不同資料分佈 | planner 估算偏差 | 用不同常數值多次測試（如不同的 `category_id`） |
| 只測最優情況 | 過度樂觀 | 測試邊界值：NULL、空字串、不存在的值 |
| 與 pgbouncer transaction mode 混用 | 假設索引遺留 | 每個 transaction 結束後執行 `hypopg_reset()` |
| 忽略 VACUUM 後的效能變化 | 索引膨脹未考慮 | 建立真實索引後用 `pgstattuple` 監控 |

#### VI. 實戰檢查清單

在將 hypopg 驗證過的索引部署到 Production 之前，完成以下檢查：

```sql
-- ✅ 1. 確認索引欄位順序與查詢的 WHERE / ORDER BY 一致
-- ✅ 2. 在 Staging 環境建立真實索引並 EXPLAIN ANALYZE 驗證
-- ✅ 3. 檢查索引大小（hypopg_relation_size + 20% buffer）
-- ✅ 4. 使用不同資料分佈（熱門 / 冷門 category）測試
-- ✅ 5. 確認沒有重複索引（不同的索引但覆蓋相同的查詢）
--     SELECT * FROM pg_indexes WHERE tablename = 'your_table'
-- ✅ 6. 使用 CREATE INDEX CONCURRENTLY 避免鎖表
-- ✅ 7. 監控建立後的 pg_stat_user_indexes（是否被實際使用）
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
-- ✅ 8. 在非尖峰時段部署
-- ✅ 9. 準備 rollback plan（DROP INDEX CONCURRENTLY）
-- ✅ 10. 記錄索引建立的原因和測試結果（未來維護的參考）
```

---

# 二、Contrib Extensions（PG 內建，CREATE EXTENSION 即可）

> 以下 Extension 為 PostgreSQL 內建 contrib 模組，無需額外下載，直接 CREATE EXTENSION 即可啟用。PG 17 多數已預編譯在安裝包中。

---

## 五、pg_stat_statements — 查詢統計與歸一化

> **初學者導讀**
> 想像你經營一個圖書館，每天上百人進來借書。你想知道：「哪一本書被借最多次？」「哪一區的書被借閱最久？」「誰借書時不先查目錄、直接衝進去一本一本翻？」但如果每個人借書時查詢的書名都不太一樣（有人查"三體"、有人查"三体"、有人查"三體 劉慈欣"），你很難統計。`pg_stat_statements` 做的事情就是：把所有「本質上相同」的查詢歸類在一起，然後告訴你每類查詢的次數、總時間、I/O、WAL 等統計。它是 PostgreSQL 最重要的可觀測性擴充。
>
> **補充（Senior Dev）**
> pg_stat_statements 是 PG 可觀測性三本柱之首：pg_stat_statements 告訴你 **WHAT** is slow；auto_explain 告訴你 **WHY** it's slow；pg_stat_kcache 告訴你 **COST** in real OS metrics。三者搭配才能形成完整的查詢效能畫像。

---

### 1. 核心價值

#### I. 歸一化 + 聚合 = Top-N 分析

pg_stat_statements 的核心價值在於兩步驟：

1. **歸一化（Normalization）**：將 SQL 中的常數替換為參數佔位符 `$N`，使 `WHERE id = 1` 和 `WHERE id = 999` 被歸類為同一個 query template
2. **聚合（Aggregation）**：對同一 template 的所有呼叫，累計 `calls`、`total_time`、`rows`、`shared_blks_*`、`wal_bytes` 等指標

這讓 DBA 和 App Dev 可以在單一視圖中回答以下問題：

- **哪些查詢最慢**？→ `ORDER BY total_time DESC LIMIT 10`
- **哪些查詢最頻繁**？→ `ORDER BY calls DESC LIMIT 10`
- **哪些查詢磁碟 I/O 最重**？→ `ORDER BY shared_blks_read DESC LIMIT 10`
- **哪些查詢 WAL 產出最多**？→ `ORDER BY wal_bytes DESC LIMIT 10`（PG 14+）
- **哪些查詢 JIT 編譯開銷最大**？→ `ORDER BY jit_generation_time DESC LIMIT 10`（PG 16）

```mermaid
flowchart LR
    A["Application\n發出 SQL"] --> B["postgres\n接收 SQL"]
    B --> C["歸一化引擎\n常數 → $N"]
    C --> D["計算 queryid\nhash 演算法"]
    D --> E{"queryid\n已存在？"}
    E -->|"是"| F["疊加至現有 entry\ncalls++ / time 累計"]
    E -->|"否"| G["建立新 entry\n在 shared memory"]
    F --> H["pg_stat_statements view\nDBA 查詢 Top-N"]
    G --> H
    
    style C fill:#4a90d9,color:white
    style D fill:#e8d44d,color:black
    style H fill:#5cb85c,color:white
```

#### II. 與其他監控工具的關係

| 工具 | 關注層級 | 問題 |
|------|---------|------|
| **pg_stat_statements** | 查詢模板級別 | **WHAT** — 哪類查詢最耗資源？ |
| **auto_explain** | 單次執行級別 | **WHY** — 這次執行為什麼慢？ |
| **pg_stat_activity** | 即時 session | **NOW** — 誰正在跑什麼？ |
| **pg_stat_kcache** | OS 層 I/O / CPU | **COST** — 真正的實體磁碟讀寫量？ |
| **pg_stat_user_tables** | 表級別 | **WHERE** — 哪張表被掃最多？ |

---

### 2. 歸一化原理

#### I. Query Normalization

#### a. 常數替換

pg_stat_statements 的歸一化引擎會掃描 SQL 語法樹，將以下類型的**字面常數（literal constant）**替換為 `$N`：

```sql
-- 原始 SQL（三條不同的查詢）
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 99999;

-- 歸一化後（三條視為同一 template）
SELECT * FROM users WHERE id = $1;
```

替換範圍包括：
- 數字常數：`123`、`3.14`、`1e5`
- 字串常數：`'hello'`、`'admin@example.com'`
- 布林常數：`TRUE`、`FALSE`
- `IN` 列表中的常數（見下文）
- `VALUES` 子句中的常數
- `OFFSET` / `LIMIT` 的數值

```mermaid
flowchart TB
    subgraph "原始 SQL"
        A1["SELECT name, salary<br/>FROM employees<br/>WHERE department_id = 3<br/>AND salary > 50000<br/>ORDER BY name<br/>LIMIT 20"]
    end
    
    subgraph "歸一化後"
        A2["SELECT name, salary<br/>FROM employees<br/>WHERE department_id = $1<br/>AND salary > $2<br/>ORDER BY name<br/>LIMIT $3"]
    end
    
    A1 -->|"常數替換"| A2
    
    subgraph "queryid"
        A3["hash: a1b2c3d4e5f6..."]
    end
    
    A2 -->|"hash 運算"| A3
```

#### b. IN List 行為

`IN` 列表的歸一化根據 PG 版本有所不同：

```sql
-- 原始 SQL
SELECT * FROM orders WHERE status IN ('pending', 'shipped', 'delivered');

-- PG 13 以前：部分版本將 IN 列表整體替換為 $1
-- PG 14+：更精細的控制，取決於 plan_cache_mode 和常數數量
```

一個容易踩坑的場景是 **IN 列表長度變化**：若某查詢有時 `IN (1, 2)`，有時 `IN (1, 2, 3, 4, 5)`，歸一化後可能產生不同的 queryid，因為常數的**數量**也會影響 hash 結果。這在動態查詢生成（ORM 常見）時尤為重要。

> **補充（Senior Dev）**
> 大多數 ORM（Hibernate、ActiveRecord、Django ORM）在參數化查詢方面做得很好，它們會將常數以 bind variable 的形式傳遞，因此同一個 ORM 生成的查詢幾乎總是歸一化為相同的 queryid。問題通常出在：手寫的拼接 SQL、使用 `IN` 列表的動態查詢、以及不同版本的 ORM 產生的查詢格式差異。

#### II. queryid 計算

#### a. 版本演進

| 版本 | queryid 計算方式 | 注意事項 |
|------|-----------------|---------|
| PG 9.2 - 12 | 基於 parse tree hash | 預設啟用 pg_stat_statements 時自動計算 |
| PG 13 | 解析樹 hash（與 12 類似） | 最後一個「預設強制」計算的版本 |
| **PG 14+** | **`compute_query_id` GUC** | 引入 `compute_query_id` 參數，可選擇 `auto` / `on` / `off` / `regclass` |
| **PG 16** | **JIT counters 納入 hash** | JIT 相關的 `pg_stat_statements` 計數器會影響 queryid |

#### b. compute_query_id（PG 14+）

PG 14 引入的全域參數，控制是否計算 queryid 以及計算方式：

```sql
-- 查看當前設定
SHOW compute_query_id;

-- 參數值說明
-- off:     不計算 queryid（節省 CPU，但 pg_stat_statements 無法使用）
-- on:      強制計算 queryid（即使未載入 pg_stat_statements）
-- auto:    當任何載入的模組需要 queryid 時自動計算（預設）
-- regclass: 使用 pg_stat_statements 的舊版雜湊演算法（向後相容）
```

```ini
-- postgresql.conf
compute_query_id = 'auto'
```

> **補充（Senior Dev）**
> `compute_query_id = 'on'` 的 CPU 開銷極小（通常 < 1%），但如果你開啟了 `log_statement = 'all'` 或頻繁使用 `pg_stat_activity.query_id`（PG 14+ 新增欄位），這個開銷會略微增加。`regclass` 模式僅在需要與 PG 13 環境跨版本比對 queryid 時使用——但通常不建議，因為 queryid 本身就不是設計為跨版本穩定的。

#### c. PG 16 JIT 計數器與 queryid

PG 16 引入了 JIT（Just-In-Time Compilation）相關的 pg_stat_statements 計數器欄位：`jit_functions`、`jit_generation_time`、`jit_inlining_time`、`jit_optimization_time`、`jit_emission_time`。這些計數器**會影響 queryid 的計算**，意味著：

- 若 PG 16 伺服器的 JIT 設定變更（`jit = on/off`），相同 SQL 可能產生**不同的 queryid**
- 此行為是 PG 16 的已知特性，非 bug
- 遷移至 PG 16 後，歷史 queryid 無法跨版本比對

#### d. queryid 不穩定性

queryid 在以下情況會改變：
1. **PG 版本升級**：hash 演算法可能不同
2. **資料庫編碼變更**：影響字串常數的 hash
3. **相同的 SQL 但不同 schema search_path**：因為物件名稱解析可能不同
4. **PG 16 JIT 狀態變更**：如上所述

```mermaid
flowchart TD
    A["同一條 SQL\nSELECT * FROM t WHERE id = 1"] --> B{"PG 版本？"}
    B -->|"PG 13"| C["queryid: 0x1A2B3C4D"]
    B -->|"PG 14"| D["queryid: 0x9F8E7D6C"]
    B -->|"PG 15"| E["queryid: 0x5A4B3C2D"]
    B -->|"PG 16<br/>(JIT on)"| F["queryid: 0x3E4F5A6B"]
    B -->|"PG 16<br/>(JIT off)"| G["queryid: 0x7C8D9E0F"]
    
    style C fill:#ff6b6b,color:#fff
    style D fill:#ffd43b
    style E fill:#74c0fc
    style F fill:#9775fa,color:#fff
    style G fill:#5cb85c,color:#fff
```

> **初學者導讀**
> 不要把 queryid 當成「永久身份證」。queryid 只在**同一個 PG 實例、同一版本、同一配置**下才有意義。升級 PG 版本後，歷史的 queryid 應該視為無效。如果你需要跨版本追蹤查詢模板，可以將 `query` 欄位（歸一化後的 SQL 文字）作為替代的識別鍵。

---

### 3. 安裝與配置（PG 16）

#### I. 安裝

```sql
-- Step 1: 修改 postgresql.conf，載入 shared library
-- shared_preload_libraries = 'pg_stat_statements'

-- Step 2: 重啟 PostgreSQL
-- pg_ctl restart

-- Step 3: 建立擴充
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

`shared_preload_libraries` 是**必須的**，因為 pg_stat_statements 需要在 shared memory 中分配 hash table，且必須掛載 hook 來攔截每一條 SQL 的執行。

#### II. GUC 參數總覽

| 參數 | 類型 | 預設值 | PG 版本 | 說明 |
|------|------|--------|---------|------|
| `pg_stat_statements.track` | enum | `top` | 9.2+ | 追蹤範圍：`top`（僅頂層查詢）、`all`（含 nested function）、`none`（關閉） |
| `pg_stat_statements.max` | integer | `5000` | 9.2+ | shared memory 中最多保留的 query entry 數量 |
| `pg_stat_statements.track_utility` | boolean | `on` | 9.5+ | 是否追蹤 DDL / DCL 等 utility 命令 |
| `pg_stat_statements.track_planning` | boolean | `off` | 13+ | 是否追蹤 planning time 與 planning 次數 |
| `pg_stat_statements.save` | boolean | `on` | 9.2+ | 是否在 server restart 時保存統計 |

```ini
## postgresql.conf 推薦配置（PG 16, Production）
shared_preload_libraries = 'pg_stat_statements'

pg_stat_statements.track = 'top'
pg_stat_statements.max = 10000
pg_stat_statements.track_utility = 'on'
pg_stat_statements.track_planning = 'on'
pg_stat_statements.save = 'on'
```

#### III. track 模式詳解

```sql
-- track = 'top'
-- 僅追蹤應用程式直接發送的 SQL（含 PL/pgSQL 的最外層 CALL）
SELECT * FROM users WHERE id = 1;  -- 被追蹤

-- track = 'all'
-- 追蹤所有語句，包括函數內部的 SQL
CREATE FUNCTION get_user(p_id int) RETURNS SETOF users AS $$
BEGIN
    RETURN QUERY SELECT * FROM users WHERE id = p_id;  -- 也被追蹤
END;
$$ LANGUAGE plpgsql;

-- track = 'none'
-- 不追蹤任何語句（等於停用 pg_stat_statements）
```

> **補充（Senior Dev）**
> `track = 'all'` 能提供更細粒度的函數內部查詢統計，但 memory 消耗會急劇增加（每個函數內部語句都是一個 entry）。Production 環境通常使用 `track = 'top'` 搭配 `auto_explain` 來追蹤函數內部問題。若使用 `track = 'all'`，建議將 `max` 至少設為 `track='top'` 時的 3-5 倍。

#### IV. 確認載入成功

```sql
-- 檢查擴充是否已建立
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- 檢查 view 是否可用
SELECT count(*) FROM pg_stat_statements;

-- 檢查 info view
SELECT * FROM pg_stat_statements_info;
```

---

### 4. pg_stat_statements 視圖詳解

#### I. 核心欄位說明

#### a. 完整欄位表

| 欄位 | 類型 | PG 版本 | 說明 |
|------|------|---------|------|
| `userid` | oid | 9.2+ | 執行查詢的使用者 OID |
| `dbid` | oid | 9.2+ | 執行查詢的資料庫 OID |
| `toplevel` | bool | **15+** | 是否為頂層查詢（PG 15 後允許組合 `all` 模式查詢內外部層級） |
| `queryid` | bigint | 9.2+ | 歸一化後的 query template hash |
| `query` | text | 9.2+ | 歸一化後的 SQL 文字 |
| `plans` | bigint | **13+** | 此 template 被 planning 的總次數 |
| `total_plan_time` | double | **13+** | 此 template 的總 planning 時間（ms） |
| `min_plan_time` | double | **13+** | 此 template 的最短 planning 時間（ms） |
| `max_plan_time` | double | **13+** | 此 template 的最長 planning 時間（ms） |
| `mean_plan_time` | double | **13+** | 此 template 的平均 planning 時間（ms） |
| `stddev_plan_time` | double | **13+** | planning 時間標準差 |
| `calls` | bigint | 9.2+ | 此 template 被執行的總次數 |
| `total_exec_time` | double | 9.2+ | 此 template 的總執行時間（ms），不含 planning |
| `min_exec_time` | double | 9.2+ | 單次最短執行時間（ms） |
| `max_exec_time` | double | 9.2+ | 單次最長執行時間（ms） |
| `mean_exec_time` | double | 9.2+ | 平均執行時間（ms） |
| `stddev_exec_time` | double | **13+** | 執行時間標準差 |
| `rows` | bigint | 9.2+ | 此 template 總共回傳或影響的資料列數 |
| `shared_blks_hit` | bigint | 9.2+ | shared buffer cache 命中次數 |
| `shared_blks_read` | bigint | 9.2+ | 從磁碟（或 OS cache）讀入 shared buffer 的 block 數 |
| `shared_blks_dirtied` | bigint | 9.2+ | 被標記為 dirty 的 shared buffer block 數 |
| `shared_blks_written` | bigint | 9.2+ | 被後端寫入磁碟的 shared buffer block 數 |
| `local_blks_hit` | bigint | 9.2+ | local buffer cache 命中次數（用於 temp table） |
| `local_blks_read` | bigint | 9.2+ | 從磁碟讀入 local buffer 的 block 數 |
| `local_blks_dirtied` | bigint | 9.2+ | local buffer 中標記 dirty 的 block 數 |
| `local_blks_written` | bigint | 9.2+ | 被寫入磁碟的 local buffer block 數 |
| `temp_blks_read` | bigint | 9.2+ | 從暫存檔案讀取的 block 數（排序、hash join 溢出等） |
| `temp_blks_written` | bigint | 9.2+ | 寫入暫存檔案的 block 數 |
| `blk_read_time` | double | 9.2+ | 讀取 block 所花費的總時間（ms）（如有啟用 `track_io_timing`） |
| `blk_write_time` | double | 9.2+ | 寫入 block 所花費的總時間（ms）（如有啟用 `track_io_timing`） |
| `wal_bytes` | numeric | **14+** | 此 template 產生的 WAL 總量（bytes） |
| `wal_records` | bigint | **16+** | 此 template 產生的 WAL record 數量 |
| `wal_fpi` | bigint | **16+** | 此 template 產生的 full-page image 數量 |
| `temp_blk_read_time` | double | **17+** | 暫存檔案讀取時間 |
| `temp_blk_write_time` | double | **17+** | 暫存檔案寫入時間 |
| `jit_functions` | bigint | **16** | JIT 編譯的函數總數 |
| `jit_generation_time` | double | **16** | JIT 程式碼生成階段的總時間 |
| `jit_inlining_time` | double | **16** | JIT 函數內聯階段的總時間 |
| `jit_optimization_time` | double | **16** | JIT 程式碼優化階段的總時間 |
| `jit_emission_time` | double | **16** | JIT 機器碼輸出階段的總時間 |

#### b. 欄位分類（便於記憶）

```mermaid
mindmap
  root((pg_stat_statements))
    身份
      userid
      dbid
      queryid
      query
      toplevel
    呼叫與計畫（PG 13+）
      calls
      plans
      total_plan_time
      min / max / mean / stddev_plan_time
    執行時間
      total_exec_time
      min / max / mean / stddev_exec_time（PG 13+）
      rows
    Buffer I/O
      shared_blks_hit
      shared_blks_read
      shared_blks_dirtied
      shared_blks_written
      local_blks_*
      temp_blks_read
      temp_blks_written
    I/O Timing（track_io_timing=on）
      blk_read_time
      blk_write_time
    WAL（PG 14+）
      wal_bytes
      wal_records（PG 16+）
      wal_fpi（PG 16+）
    JIT（PG 16）
      jit_functions
      jit_generation_time
      jit_inlining_time
      jit_optimization_time
      jit_emission_time
```

> **初學者導讀**
> 初學階段只需關注 5 個核心欄位：`calls`（多少次）、`total_exec_time`（多慢）、`mean_exec_time`（平均多慢）、`shared_blks_hit` / `shared_blks_read`（cache 命中率）。這些已能解決 80% 的日常問題。其餘欄位在你遇到特定瓶頸時再深入使用即可。

#### II. 實戰查詢 Top-N

#### a. Top-10 最耗時查詢

```sql
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round((100.0 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

#### b. Top-10 最高頻查詢

```sql
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

#### c. Top-10 最低 Cache Hit 查詢

```sql
-- cache_hit_ratio < 90% 表示大量讀取磁碟
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    shared_blks_hit,
    shared_blks_read,
    CASE
        WHEN shared_blks_hit + shared_blks_read = 0
            THEN 100.0
        ELSE round(100.0 * shared_blks_hit::numeric
              / (shared_blks_hit + shared_blks_read)::numeric, 2)
    END AS cache_hit_pct,
    calls
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

#### d. Top-10 WAL 寫入量（PG 14+，寫入密集）

```sql
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    pg_size_pretty(wal_bytes::bigint) AS wal_size,
    wal_records   -- PG 16+
FROM pg_stat_statements
WHERE wal_bytes > 0
ORDER BY wal_bytes DESC
LIMIT 10;
```

#### e. Top-10 JIT 開銷（PG 16）

```sql
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS exec_ms,
    round(jit_generation_time::numeric, 2) AS jit_gen_ms,
    round(jit_inlining_time::numeric, 2) AS jit_inline_ms,
    round(jit_optimization_time::numeric, 2) AS jit_opt_ms,
    round(jit_emission_time::numeric, 2) AS jit_emit_ms,
    jit_functions
FROM pg_stat_statements
WHERE jit_generation_time > 0
ORDER BY jit_generation_time DESC
LIMIT 10;
```

#### f. Top-10 計畫時間偏長的查詢（PG 13+）

```sql
-- planning time 高的查詢可能涉及多個 partition 或複雜 view
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    plans,
    round(total_plan_time::numeric, 2) AS plan_total_ms,
    round(mean_plan_time::numeric, 2) AS plan_avg_ms,
    round(mean_exec_time::numeric, 2) AS exec_avg_ms,
    CASE
        WHEN mean_exec_time > 0
            THEN round((mean_plan_time / mean_exec_time * 100)::numeric, 2)
        ELSE 0
    END AS plan_exec_ratio_pct
FROM pg_stat_statements
WHERE plans > 0
ORDER BY total_plan_time DESC
LIMIT 10;
```

#### g. Top-10 暫存檔案使用量（TEMP space）

```sql
SELECT
    queryid,
    LEFT(query, 80) AS query_preview,
    calls,
    temp_blks_read,
    temp_blks_written,
    round(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

> **補充（Senior Dev）**
> `cache_hit_pct` 是判斷查詢是否需要索引優化的快速指標。但要注意：`shared_blks_hit` 計數的是 buffer 命中次數，而非「唯一 block」數量。一個 Nested Loop 可能重複命中同一個 buffer page，導致 `shared_blks_hit` 虛高。搭配 `EXPLAIN (ANALYZE, BUFFERS)` 檢查 `Buffers: shared hit=N` 中的實際 `N` 值來交叉驗證。

---

### 5. pg_stat_statements_reset() 策略

#### I. 基本語法

```sql
-- 重設當前資料庫中所有使用者的統計
SELECT pg_stat_statements_reset();

-- 重設特定使用者的統計（PG 14+）
SELECT pg_stat_statements_reset(userid := 16384);

-- 重設特定 database 的統計
SELECT pg_stat_statements_reset(dbid := 16385);

-- 同時重設特定使用者 + database
SELECT pg_stat_statements_reset(16384, 16385);
```

#### II. 何時呼叫 reset

| 場景 | 建議策略 |
|------|---------|
| **Performance benchmark** | 每次測試前 reset，測試後取 snapshot |
| **定期監控報告** | 每小時 / 每天 reset，配合定時 snapshot |
| **Deploy 新版本後** | reset 舊統計，僅觀察新版本的查詢行為 |
| **Troubleshooting** | **不要 reset** — 保留完整歷史，方便事後分析 |
| **Production 日常** | 建議用 snapshot diff 代替 reset，避免遺失尖峰數據 |

#### III. Snapshot vs Reset 策略

推薦使用 **定期 snapshot**（而非直接 reset）以獲得時間維度的分析能力：

```sql
-- 建立 snapshot 表
CREATE TABLE IF NOT EXISTS monitoring.pss_snapshots (
    snap_ts     timestamptz DEFAULT now(),
    queryid     bigint,
    query       text,
    calls       bigint,
    total_exec_time double precision,
    rows        bigint,
    shared_blks_hit  bigint,
    shared_blks_read bigint,
    wal_bytes   numeric
);

-- 每小時執行一次 snapshot（可用 pg_cron）
INSERT INTO monitoring.pss_snapshots
    (snap_ts, queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read, wal_bytes)
SELECT
    now(), queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read, wal_bytes
FROM pg_stat_statements;
```

```mermaid
flowchart LR
    A["Snapshot T1\n(09:00)"] --> D["Diff 分析\nΔcalls, Δtime"]
    B["Snapshot T2\n(10:00)"] --> D
    D --> E["找出特定時段\n的異常查詢"]
    
    F["pg_stat_statements_reset()"] --> G["僅保留\nreset 後的數據"]
    G --> H["遺失歷史趨勢"]
    
    style D fill:#4a90d9,color:white
    style E fill:#5cb85c,color:white
    style H fill:#ff6b6b,color:white
```

> **初學者導讀**
> 簡單說：`reset()` 就像把記帳本整本燒掉從零開始。用 snapshot 則像每天拍照留存，隨時可以回頭看「上週什麼時候開始變慢的」。在 production 環境中，**永遠用 snapshot，避免用 reset**。

---

### 6. 輔助函數與視圖

#### I. pg_stat_statements_info 視圖

`pg_stat_statements_info`（PG 14+）提供 pg_stat_statements 本身的健康狀態：

```sql
SELECT * FROM pg_stat_statements_info;
```

| 欄位 | 說明 |
|------|------|
| `dealloc` | 自上次 reset 以來，因 shared memory 空間不足而被逐出的 entry 總數 |
| `stats_reset` | 最後一次 `pg_stat_statements_reset()` 的執行時間 |

#### II. 空間不足偵測（deallocation）

當 `pg_stat_statements.max` 設定太小時，舊的 entry 會被逐出以騰出空間給新查詢。`dealloc` 計數器會增加：

```sql
-- 偵測是否 max 設定太小
SELECT
    dealloc,
    (SELECT count(*) FROM pg_stat_statements) AS current_entries,
    current_setting('pg_stat_statements.max')::int AS max_entries,
    CASE
        WHEN dealloc > 0
            THEN 'WARNING: max 太小，部分 entry 已被逐出'
        ELSE 'OK'
    END AS status
FROM pg_stat_statements_info;
```

> **補充（Senior Dev）**
> `dealloc > 0` 表示遺失了部分查詢的統計。計算合理的 `max` 值：執行一陣子（如 24 小時）後，`SELECT count(*) FROM pg_stat_statements` 即為目前活躍的 query template 數量。將此數量乘以 1.5 作為安全緩衝。範例：若 24h 內有 3200 個不同 template，設定 `max = 5000`。

#### III. 權限不足處理

pg_stat_statements 需要 `pg_read_all_stats` 角色或被明確授權才能讀取：

```sql
-- 授權給監控角色
GRANT pg_read_all_stats TO monitoring_user;

-- 或手動授權（PG 15-）
GRANT SELECT ON pg_stat_statements TO monitoring_user;
GRANT SELECT ON pg_stat_statements_info TO monitoring_user;
```

若權限不足，查詢 `pg_stat_statements` 時會看到 `query` 顯示為 `<insufficient privilege>`：

```sql
-- 檢查是否有被遮蔽的 query
SELECT
    queryid,
    CASE WHEN query = '<insufficient privilege>' THEN 'PRIVILEGE ISSUE' ELSE 'OK' END AS privilege_status,
    calls,
    total_exec_time
FROM pg_stat_statements
WHERE query = '<insufficient privilege>';
```

#### IV. 常見疑難排解

```sql
-- 1. 確認 pg_stat_statements 處於 active 狀態
SELECT extname, extversion FROM pg_extension WHERE extname = 'pg_stat_statements';

-- 2. 確認 GUC 參數未被 override（可能被 ALTER SYSTEM / ALTER DATABASE 覆蓋）
SELECT name, setting, source FROM pg_settings WHERE name LIKE 'pg_stat_statements%';

-- 3. 確認 shared_memory 已分配
-- pg_stat_statements.max * 每個 entry 約 400 bytes ≈ 總記憶體需求
-- max=5000 → ~2MB shared memory, max=50000 → ~20MB
```

---

### 7. App Dev 最佳實踐

#### I. 定期 Snapshot 上報監控系統

建立一個自動化的 snapshot 管道，將 pg_stat_statements 數據導入監控平台（Prometheus / Grafana / Datadog 等）：

```sql
-- Step 1: 建立 snapshot 歷程表
CREATE SCHEMA IF NOT EXISTS monitoring;

CREATE TABLE monitoring.pss_history (
    id              bigserial PRIMARY KEY,
    snap_time       timestamptz NOT NULL,
    queryid         bigint NOT NULL,
    query           text,
    dbid            oid,
    userid          oid,
    calls           bigint,
    total_exec_time double precision,
    mean_exec_time  double precision,
    stddev_exec_time double precision,
    rows            bigint,
    shared_blks_hit bigint,
    shared_blks_read bigint,
    shared_blks_dirtied bigint,
    shared_blks_written bigint,
    wal_bytes       numeric,
    temp_blks_read  bigint,
    temp_blks_written bigint
);

CREATE INDEX ON monitoring.pss_history (snap_time, queryid);

-- Step 2: pg_cron 定時 snapshot（每 15 分鐘）
SELECT cron.schedule(
    'pss-snapshot',
    '*/15 * * * *',
    $$INSERT INTO monitoring.pss_history
        (snap_time, queryid, query, dbid, userid,
         calls, total_exec_time, mean_exec_time, stddev_exec_time, rows,
         shared_blks_hit, shared_blks_read, shared_blks_dirtied, shared_blks_written,
         wal_bytes, temp_blks_read, temp_blks_written)
    SELECT
        now(), queryid, query, dbid, userid,
        calls, total_exec_time, mean_exec_time, stddev_exec_time, rows,
        shared_blks_hit, shared_blks_read, shared_blks_dirtied, shared_blks_written,
        wal_bytes, temp_blks_read, temp_blks_written
    FROM pg_stat_statements$$
);

-- Step 3: 定期清理舊數據（保留 90 天）
SELECT cron.schedule(
    'pss-cleanup',
    '0 3 * * *',  -- 每天凌晨 3 點
    $$DELETE FROM monitoring.pss_history WHERE snap_time < now() - interval '90 days'$$
);
```

#### II. queryid 穩定性與版本升級

```mermaid
flowchart TD
    A["準備升級 PG 版本"] --> B{"需要保留\n歷史 queryid 比對？"}
    B -->|"否"| C["直接升級\n新版本自動產生新 queryid"]
    B -->|"是"| D["方案 A: 保留 query 文字\n以 query text 做跨版本比對"]
    B -->|"是"| E["方案 B: 在舊版匯出 snapshot\n升級後匯入並建立 mapping"]
    
    C --> F["✅ 最簡單"]
    D --> G["⚠️ query 文字可能因\n歸一化規則變更而不同"]
    E --> H["⚠️ 維護成本高\n不推薦"]
    
    style F fill:#5cb85c,color:white
    style G fill:#ffd43b
    style H fill:#ff6b6b,color:white
```

**建議**：使用 `query` 欄位（歸一化後的 SQL 文字）作為跨版本的查詢模板識別鍵，而非依賴 `queryid`。在監控系統中以 `(dbid, userid, query)` 作為 composite key。

#### III. 與 auto_explain 的協作

pg_stat_statements 告訴你 **WHAT** is slow；auto_explain 告訴你 **WHY** it's slow。典型的協作流程：

```sql
-- Step 1: 從 pg_stat_statements 找出可疑查詢
SELECT queryid, query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 1;

-- Step 2: 記錄該 queryid，開啟 auto_explain 捕捉執行計畫
ALTER SYSTEM SET auto_explain.log_min_duration = 0;           -- 記錄所有
ALTER SYSTEM SET auto_explain.log_analyze = 'on';              -- 含實際執行統計
ALTER SYSTEM SET auto_explain.log_buffers = 'on';              -- 含 buffer 統計
ALTER SYSTEM SET auto_explain.log_timing = 'on';               -- 含時間
SELECT pg_reload_conf();

-- Step 3: 等待問題查詢再次出現，檢查 log 中的 EXPLAIN 輸出
-- Linux: tail -f /var/log/postgresql/postgresql-16-main.log | grep "duration"

-- Step 4: 分析完畢後關閉 auto_explain
ALTER SYSTEM RESET auto_explain.log_min_duration;
ALTER SYSTEM RESET auto_explain.log_analyze;
SELECT pg_reload_conf();
```

```mermaid
flowchart LR
    A["pg_stat_statements\n告訴你 Top-N 慢查詢"] --> B["auto_explain\n捕捉實際執行計劃"]
    B --> C["EXPLAIN ANALYZE\n找出瓶頸節點"]
    C --> D{"瓶頸類型？"}
    D -->|"Seq Scan"| E["建立索引"]
    D -->|"Nested Loop"| F["調整 join 策略 / 檢查統計資訊"]
    D -->|"Hash Join spill"| G["增加 work_mem"]
    D -->|"Sort spill"| H["增加 work_mem / 優化排序欄位"]
    
    style A fill:#4a90d9,color:white
    style B fill:#e8d44d,color:black
    style E fill:#5cb85c,color:white
    style F fill:#5cb85c,color:white
    style G fill:#5cb85c,color:white
    style H fill:#5cb85c,color:white
```

#### IV. max 大小估算

```
max 估算公式（Production）：
  active_query_templates = 觀察 24 小時內的 unique query template 數量
  max = active_query_templates * 1.5
```

```sql
-- 24h 後執行，估算當前 template 數量
SELECT count(*) AS active_templates FROM pg_stat_statements;

-- 記憶體估算（每個 entry 約 400-600 bytes，含 query 文字平均 ~200 bytes）
-- max=10000 → 約 4-6 MB shared memory
-- max=50000 → 約 20-30 MB shared memory
-- 此記憶體來自 shared_buffers 之外，需確保 OS 有足夠 RAM
```

#### V. Production 環境使用 track = top

```sql
-- Production 推薦設定
ALTER SYSTEM SET pg_stat_statements.track = 'top';

-- 理由：
-- 1. 減少 shared memory 使用（不會為函數內部的每個 SQL 建立 entry）
-- 2. 降低 CPU 開銷（不需攔截 nested 層級的 SQL）
-- 3. 大多數應用程式的效能瓶頸在頂層查詢，函數內部問題可用 auto_explain 分析

-- 例外：若大量使用 PL/pgSQL / PL/Python 等，且函數內部查詢複雜
--       可考慮 track = 'all' + max 設為 3-5 倍
```

#### VI. 安全性：避免 query 欄位洩漏資料

pg_stat_statements 的 `query` 欄位儲存的是**歸一化後**的 SQL，常數已被替換為 `$N`。但以下情況仍可能洩漏敏感資訊：

1. **DDL 語句中的物件名稱**：`ALTER USER alice PASSWORD 'secret'` 不會被歸一化常數剔除（物件名稱不是常數）
2. **函數體內的常數**：PL/pgSQL 函數的 source code 中若含有硬編碼的常數
3. **自訂型別的輸入值**：某些 extension 定義的型別可能不會被歸一化

```sql
-- 定期審計 pg_stat_statements 中的 query 內容
SELECT queryid, query
FROM pg_stat_statements
WHERE query ILIKE '%password%'
   OR query ILIKE '%secret%'
   OR query ILIKE '%token%';
```

> **補充（Senior Dev）**
> 在 multi-tenant 環境中，注意 `pg_stat_statements` 是**跨使用者共享**的視圖（經由權限控制）。若不同 tenant 使用不同 database user 且查詢相同的 template，它們會被歸入同一個 entry，並在 `total_time` 等欄位中聚合。若需 per-tenant 分析，請搭配 `userid` 欄位過濾。

#### VII. 效能開銷評估

| 開銷來源 | 影響 | 緩解方案 |
|---------|------|---------|
| 每條 SQL 的 hash 計算 | CPU < 0.5% | `compute_query_id = 'auto'` 已高度優化 |
| 更新 shared memory counter | 極微（atomic add） | 無法避免，已是最小路徑 |
| shared memory 空間 | `max * ~500 bytes` | 合理設定 `max` 值 |
| `track = 'all'` 模式 | 額外 overhead（nested hooks） | Production 用 `track='top'` |

> **初學者導讀**
> pg_stat_statements 的開銷極低。除非你的系統是極限性能場景（每秒數十萬次查詢、且每微秒都計較），否則可以安心在 Production 中永久啟用。大多數 PG 雲端服務（RDS、Cloud SQL 等）預設就啟用了 pg_stat_statements。
## 六、auto_explain — 自動記錄執行計劃

> **初學者導讀**  
> `auto_explain` 是一個 PostgreSQL 內建模組（extension），能在查詢執行時間超過指定閾值時，自動將 `EXPLAIN ANALYZE` 的完整輸出寫入 PostgreSQL 日誌（log）。你不需要手動執行 `EXPLAIN ANALYZE`，也不用在應用程式碼裡埋點——只要啟用它，所有「慢查詢」的執行計劃就會自動出現在日誌中，等同於 24/7 全天候的查詢效能監控。

---

### 1. 核心價值 — 與 `pg_stat_statements` 互補

許多 DBA 的直覺是：「我有 `pg_stat_statements` 了，為什麼還需要 `auto_explain`？」答案是：這兩個工具解決的是**不同層次**的問題。

| 工具 | 回答的問題 | 資料粒度 | 典型用途 |
|------|------------|---------|---------|
| `pg_stat_statements` | **WHAT** — 哪些查詢最慢？ | 聚合統計（aggregated）：mean_time、calls、shared_blks_hit 等 | 識別 Top-N 慢查詢、監控快取命中率趨勢 |
| `auto_explain` | **WHY** — 為什麼這條查詢這麼慢？ | 逐條執行計劃（per-query plan）：實際 rows、Buffers、排序方式、觸發器耗時 | 定位具體瓶頸：Seq Scan？work_mem 不足？Nested Loop 估算偏差？ |

**一句話總結**：`pg_stat_statements` 告訴你「誰是兇手」，`auto_explain` 告訴你「犯罪手法」。

```mermaid
graph LR
    A["應用程式查詢<br/>Application Query"] --> B["pg_stat_statements<br/>聚合統計<br/>WHAT is slow?"]
    A --> C["auto_explain<br/>逐條計劃<br/>WHY is it slow?"]
    B --> D["Top-N 慢查詢清單<br/>mean_time > 100ms"]
    D -->|"針對特定 queryid"| C
    C --> E["瓶頸定位<br/>Seq Scan / work_mem / 索引缺失"]
    style A fill:#4A90D9,color:#fff
    style B fill:#E67E22,color:#fff
    style C fill:#27AE60,color:#fff
    style D fill:#F1C40F,color:#333
    style E fill:#E74C3C,color:#fff
```

**互補工作流**：
1. 在 `pg_stat_statements` 中發現 `mean_time` 異常升高的 queryid
2. 取得該查詢的具體 SQL 文本
3. 在 `auto_explain` 日誌中搜尋該 SQL，查看實際執行計劃
4. 根據計劃中的 `Buffers`、`Rows Removed by Filter`、排序方式等資訊對症下藥

> **補充（Senior Dev）**  
> `pg_stat_statements` 的 `shared_blks_hit / shared_blks_read` 只能告訴你「整體快取命中率」，但 `auto_explain` 的 `Buffers: shared hit=X read=Y` 能精確到**執行計劃的每個節點**。舉例：聚合查詢可能在小表上 Index Scan 命中率 100%，但在大表上 Seq Scan 命中率只有 30%，這種細節只有逐條計劃才能捕捉。

---

### 2. 工作原理

`auto_explain` 透過 PostgreSQL 的 **Executor Hook（執行器鉤子）** 機制介入查詢流程，屬於輕量級、非阻塞（non-blocking）的觀察者模式。

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Parser / Planner
    participant E as Executor
    participant A as auto_explain Hook
    participant L as PostgreSQL Log

    C->>P: 發送 SQL 查詢
    P->>P: 解析 & 規劃（生成 Plan Tree）
    P->>E: 傳遞 Plan Tree
    Note over E: ExecutorStart 觸發 hook
    A->>A: 記錄開始時間戳
    E->>E: 逐節點執行計劃樹
    Note over E: ExecutorEnd 觸發 hook
    A->>A: 計算 elapsed_time = now - start
    alt elapsed_time >= log_min_duration
        A->>A: 呼叫 ExplainAnalyzeExecutor()
        A->>L: 寫入完整 EXPLAIN ANALYZE 輸出
    else elapsed_time < log_min_duration
        A->>A: 不記錄（或按 sample_rate 抽樣）
    end
    E->>C: 返回結果集
```

**關鍵設計要點**：

1. **Hook 時機**：掛載在 `ExecutorStart`（計時開始）和 `ExecutorEnd`（計時結束並判斷是否輸出），對查詢執行本身**零干擾**。
2. **非阻塞**：`EXPLAIN ANALYZE` 的輸出生成發生在查詢執行**完成後**，不影響查詢的實際執行路徑。
3. **日誌輸出**：使用 PostgreSQL 的 `ereport()` 機制，遵守 `log_min_messages` 和 `log_destination` 設定。
4. **Nested Statements**（PG 16+）：對於函數/預存程序內的 SQL，每個巢狀語句都可以獨立記錄執行計劃，這對偵錯複雜 PL/pgSQL 邏輯至關重要。

> **補充（Senior Dev）**  
> `auto_explain` 的 `EXPLAIN ANALYZE` 是**在 ExecutorEnd 階段重新執行一次 Explain 分析**，而非查詢執行過程中的即時取樣。這意味著它消耗的 CPU 時間約等於一次額外的 `EXPLAIN ANALYZE` 呼叫。如果 `log_analyze = on`，還要額外承擔 Instrumentation（計時/緩衝區計數）的開銷。對於 `log_min_duration = 0`（記錄所有查詢）的 Dev 環境，這會明顯增加 CPU 負載，詳見第 5 節。

---

### 3. 安裝與配置（PG 16）

#### 3.1 載入模組

`auto_explain` 是 PostgreSQL 內建貢獻模組（contrib module），無需額外安裝套件。只需在 `postgresql.conf` 中加入：

```ini
## 必須設定，因為 auto_explain 依賴 shared_preload_libraries 載入 hook
shared_preload_libraries = 'auto_explain'
```

修改後**必須重啟 PostgreSQL**（`pg_ctl restart` 或 `systemctl restart`），`pg_reload_conf()` 無效。

驗證載入成功：

```sql
SELECT name, setting FROM pg_settings WHERE name LIKE 'auto_explain%';
-- 如果看到多個 auto_explain.* 參數，表示載入成功
```

#### 3.2 GUC 參數完整對照表

以下為 PG 16 中所有 `auto_explain` 相關參數。標記 🆕 者為 PG 16 新增。

| 參數 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `auto_explain.log_min_duration` | integer (ms) | `-1` | 查詢執行時間超過此閾值才記錄。`-1` 表示禁用，`0` 表示記錄所有查詢 |
| `auto_explain.log_analyze` | boolean | `off` | 是否輸出 `EXPLAIN ANALYZE`（含實際行數與執行時間），而非僅 `EXPLAIN`（估算值）。開啟後效能開銷顯著增加 |
| `auto_explain.log_buffers` | boolean | `off` | 是否記錄緩衝區使用量（shared hit/read/dirtied/written, local, temp）。須搭配 `log_analyze = on` |
| `auto_explain.log_timing` | boolean | `on` | 是否記錄每個節點的實際啟動時間與總耗時。關閉可降低 overhead |
| `auto_explain.log_triggers` 🆕 | boolean | `off` | 是否記錄觸發器（trigger）的執行時間統計。PG 16 新增，對觸發器密集型工作負載的偵錯非常有幫助 |
| `auto_explain.log_verbose` | boolean | `off` | 是否輸出 `VERBOSE` 模式（含輸出欄位列表、Schema 名稱等） |
| `auto_explain.log_level` | enum | `LOG` | 日誌等級，可選 `DEBUG5..DEBUG1, LOG, INFO, NOTICE, WARNING` |
| `auto_explain.log_nested_statements` 🆕 | boolean | `off` | 是否記錄函數/預存程序內**巢狀 SQL** 的執行計劃。PG 16 新增，對 PL/pgSQL 效能調校極其關鍵 |
| `auto_explain.log_settings` 🆕 | boolean | `off` | 是否在輸出中附加修改過的 GUC 參數（`SET` 語句）資訊。PG 16 新增，有助於重現問題時的環境還原 |
| `auto_explain.log_format` | enum | `text` | 輸出格式：`text`、`xml`、`json`、`yaml` |
| `auto_explain.sample_rate` | real | `1.0` | 記錄的抽樣比例，`0.0`～`1.0`。對高吞吐量 Production 環境可設為 `0.1`（記錄 10%）以降低 I/O 壓力 |

#### 3.3 典型配置範例

```ini
## postgresql.conf — auto_explain 配置

## === 基本配置（所有環境） ===
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000    # 記錄超過 1 秒的查詢
auto_explain.log_analyze = on           # 輸出 ANALYZE（實際統計）
auto_explain.log_buffers = on           # 記錄緩衝區使用
auto_explain.log_timing = on            # 記錄節點計時
auto_explain.log_triggers = on          # 🆕 PG 16：記錄觸發器
auto_explain.log_verbose = off          # 一般不需要 VERBOSE
auto_explain.log_level = LOG
auto_explain.log_nested_statements = on # 🆕 PG 16：記錄巢狀 SQL
auto_explain.log_settings = off         # 🆕 PG 16：量產環境關閉以減少日誌量
auto_explain.log_format = text
auto_explain.sample_rate = 1.0
```

---

### 4. 輸出解讀 — 完整實例

#### 4.1 背景場景

假設一個電商系統的訂單查詢頁面變慢了。以下是 `auto_explain` 記錄到的完整日誌輸出。

#### 4.2 原始日誌輸出

```text
2025-06-15 14:32:01.123 CST [28741] LOG:  duration: 3245.678 ms  plan:
Query Text: 
SELECT o.id, o.total, c.name, c.email, COUNT(oi.id) AS item_count
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
WHERE o.created_at >= '2025-01-01'
  AND o.status = 'pending'
GROUP BY o.id, c.name, c.email
ORDER BY o.created_at DESC
LIMIT 50;

Sort  (cost=15234.56..15234.81 rows=50 width=128)
       (actual time=3245.123..3245.456 rows=50 loops=1)
  Sort Key: o.created_at DESC
  Sort Method: external merge  Disk: 8192kB
  Buffers: shared hit=342 read=12458, temp read=1024 written=1024
  ->  GroupAggregate  (cost=14200.00..15100.00 rows=500 width=128)
        (actual time=2800.123..3240.456 rows=500 loops=1)
        Group Key: o.id, c.name, c.email
        Buffers: shared hit=342 read=12458, temp read=1024 written=1024
        ->  Sort  (cost=14200.00..14300.00 rows=5000 width=120)
              (actual time=2800.111..3000.789 rows=5000 loops=1)
              Sort Key: o.id, c.name, c.email
              Sort Method: quicksort  Memory: 512kB
              Buffers: shared hit=342 read=12458
              ->  Hash Join  (cost=2000.00..12000.00 rows=5000 width=120)
                    (actual time=500.123..2500.456 rows=5000 loops=1)
                    Hash Cond: (oi.order_id = o.id)
                    Buffers: shared hit=300 read=12458
                    ->  Seq Scan on order_items oi
                          (cost=0.00..8000.00 rows=500000 width=12)
                          (actual time=0.123..1200.456 rows=500000 loops=1)
                          Filter: (created_at >= '2025-01-01'::date)
                          Rows Removed by Filter: 1500000
                          Buffers: shared hit=100 read=12000
                    ->  Hash  (cost=1500.00..1500.00 rows=5000 width=112)
                          (actual time=450.123..450.123 rows=5000 loops=1)
                          Buckets: 8192  Batches: 1  Memory Usage: 512kB
                          Buffers: shared hit=200 read=458
                          ->  Hash Join  (cost=500.00..1500.00 rows=5000 width=112)
                                (actual time=100.123..400.456 rows=5000 loops=1)
                                Hash Cond: (o.customer_id = c.id)
                                Buffers: shared hit=200 read=458
                                ->  Seq Scan on orders o
                                      (cost=0.00..800.00 rows=5000 width=76)
                                      (actual time=0.123..200.456 rows=5000 loops=1)
                                      Filter: (status = 'pending'::text)
                                      Rows Removed by Filter: 95000
                                      Buffers: shared hit=50 read=400
                                ->  Hash  (cost=200.00..200.00 rows=10000 width=40)
                                      (actual time=50.123..50.123 rows=10000 loops=1)
                                      Buckets: 16384  Batches: 1  Memory Usage: 512kB
                                      Buffers: shared hit=150 read=58
                                      ->  Seq Scan on customers c
                                            (cost=0.00..200.00 rows=10000 width=40)
                                            (actual time=0.123..30.456 rows=10000 loops=1)
                                            Buffers: shared hit=150 read=58
```

#### 4.3 逐行註解

| 行段 | 關鍵指標 | 含義 |
|------|---------|------|
| `duration: 3245.678 ms` | 總耗時 3.2 秒 | 超過 `log_min_duration` 閾值，觸發記錄 |
| `Sort Method: external merge Disk: 8192kB` | 外部合併排序，使用 8 MB 磁碟 | **瓶頸 1**：`work_mem`（預設 4 MB）太小，無法在記憶體中完成排序，被迫寫入磁碟 |
| `Buffers: shared hit=342 read=12458` | 命中 342 頁，讀取 12458 頁 | **瓶頸 3**：快取命中率僅 `342 / (342+12458) ≈ 2.7%`，資料幾乎全從磁碟讀取 |
| `Seq Scan on order_items oi` | 對 order_items 全表掃描 | 500000 行全表掃描，但 Filter 只保留了 500000 行中的部分 |
| `Rows Removed by Filter: 1500000` | Filter 過濾掉 150 萬行 | **瓶頸 2**：`order_items.created_at` 上**缺少索引**。掃描了 200 萬行，只保留了 50 萬行，浪費了 75% 的 I/O |
| `Seq Scan on orders o` + `Rows Removed by Filter: 95000` | 掃描 10 萬行，保留 5000 行 | `orders.status` 上**缺少索引**，過濾率 95% |
| `GroupAggregate` | 分組聚合 | 對 5000 行進行分組，成本合理 |

#### 4.4 三大瓶頸視覺化分析

**瓶頸 1：`work_mem` 不足導致磁碟排序**

```mermaid
graph TD
    A["查詢需要排序 5000+ 行"] --> B{"可用 work_mem<br/>（預設 4MB）"}
    B -->|"資料量 > work_mem"| C["external merge Disk<br/>寫入暫存檔"]
    B -->|"資料量 ≤ work_mem"| D["quicksort Memory<br/>純記憶體排序"]
    C --> E["⚠️ 瓶頸：磁碟 I/O<br/>耗時增加 5~50 倍"]
    D --> F["✅ 正常：記憶體排序"]
    style C fill:#E74C3C,color:#fff
    style E fill:#E74C3C,color:#fff
    style D fill:#27AE60,color:#fff
```

> **解決方案**：`SET work_mem = '64MB';` 或針對此 Session / Role 設更大值。

**瓶頸 2：缺失索引導致 Seq Scan + Filter**

```mermaid
graph TD
    A["WHERE order_items.created_at >= '2025-01-01'"] --> B{"有無索引？"}
    B -->|"無索引"| C["Seq Scan<br/>掃描全部 200 萬行"]
    C --> D["Rows Removed by Filter: 150 萬"]
    D --> E["⚠️ 浪費 75% I/O"]
    B -->|"有索引"| F["Index Scan / Bitmap Scan<br/>只讀取 50 萬行"]
    F --> G["✅ 高效"]
    style C fill:#E74C3C,color:#fff
    style E fill:#E74C3C,color:#fff
    style F fill:#27AE60,color:#fff
```

> **解決方案**：`CREATE INDEX idx_order_items_created_at ON order_items(created_at);`

**瓶頸 3：Buffer Cache 命中率極低**

```mermaid
graph TD
    A["查詢請求 12800 頁"] --> B{"頁面在 shared_buffers 中？"}
    B -->|"hit: 342 頁 (2.7%)"| C["✅ 直接讀取<br/>無 I/O"]
    B -->|"read: 12458 頁 (97.3%)"| D["⚠️ 從磁碟讀取<br/>實體 I/O"]
    D --> E["原因可能"]
    E --> F["shared_buffers 太小"]
    E --> G["資料集太大無法全快取"]
    E --> H["冷查詢（首次訪問）"]
    style D fill:#E74C3C,color:#fff
    style C fill:#27AE60,color:#fff
```

> **解決方案**：增大 `shared_buffers`（建議為系統 RAM 的 25%）；考慮使用 SSD 降低讀取延遲；對頻繁訪問的熱資料考慮 pg_prewarm。

---

### 5. 各參數效能開銷與取捨

啟用 `auto_explain` 會增加 CPU 和 I/O 開銷。下表總結各參數的影響程度。

| 參數 | 開銷等級 | 主要開銷來源 | 建議 |
|------|---------|-------------|------|
| `log_min_duration = 0` | 🔴 極高 | 每條查詢都觸發 Explain + 日誌寫入 | 僅 Dev 環境使用 |
| `log_analyze` | 🔴 高 | 需執行完整的 Instrumentation（計時器、行計數器） | Production 謹慎使用 |
| `log_buffers` | 🟡 中 | 每個節點追蹤 shared/local/temp 緩衝區 | 需 `log_analyze = on`，附加開銷較小 |
| `log_timing` | 🟡 中 | 每個節點執行 `gettimeofday()` 兩次（start/end） | 可關閉以降低開銷，但失去節點級計時 |
| `log_triggers` 🆕 | 🟢 低 | 僅在觸發器執行時收集額外統計 | 觸發器密集型工作負載才需要 |
| `log_verbose` | 🟢 低 | 輸出額外文字（Schema、欄位列表） | 僅在需要完整 Schema 資訊時使用 |
| `log_nested_statements` 🆕 | 🟡 中 | 每個巢狀 SQL 都獨立 Explain | PL/pgSQL 函數偵錯必備 |
| `log_settings` 🆕 | 🟢 低 | 僅記錄 GUC 覆蓋資訊 | 除錯階段使用，量產環境建議關閉 |
| `sample_rate < 1.0` | 🟢 降低開銷 | 抽樣減少記錄頻率 | Production 環境推薦 `0.1~0.5` |

**開銷等級視覺化**：

```mermaid
graph LR
    subgraph "開銷等級"
        A["🔴 極高<br/>log_min_duration=0<br/>+ log_analyze"] 
        B["🔴 高<br/>log_analyze<br/>+ log_buffers"]
        C["🟡 中<br/>log_timing<br/>log_nested"]
        D["🟢 低<br/>log_triggers<br/>log_verbose<br/>log_settings"]
    end
    A -->|"每次查詢<br/>+5~15% CPU"| B
    B -->|"只對慢查詢<br/>+3~8% CPU"| C
    C -->|"邊際增加<br/>+1~3% CPU"| D
    style A fill:#E74C3C,color:#fff
    style B fill:#E74C3C,color:#fff
    style C fill:#F39C12,color:#fff
    style D fill:#27AE60,color:#fff
```

> **補充（Senior Dev）**  
> 實際開銷取決於查詢複雜度和執行頻率。一條包含 20 個節點的複雜 JOIN 的 Explain 成本遠高於簡單的單表查詢。建議在 Staging 環境用 `pgbench` 或實際流量回放來測試你的配置：`pgbench -c 10 -T 60 -S your_db` 對比啟用/停用 auto_explain 的 TPS。

---

### 6. 生產環境調校策略

#### I. 分環境配置

不同環境應採用不同的 `auto_explain` 策略，在「資訊量」與「效能開銷」之間取得平衡。

```ini
## ==========================================
## 環境 A：開發環境（Dev）
## 目標：找出所有查詢的執行計劃，無遺漏
## ==========================================
auto_explain.log_min_duration = 0          # 記錄所有查詢
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_triggers = on
auto_explain.log_verbose = on
auto_explain.log_nested_statements = on
auto_explain.log_settings = on
auto_explain.log_format = text
auto_explain.sample_rate = 1.0             # 100% 記錄
```

```ini
## ==========================================
## 環境 B：預發環境（Staging）
## 目標：模擬生產流量，找出潛在慢查詢
## ==========================================
auto_explain.log_min_duration = 50         # 記錄超過 50ms 的查詢
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_triggers = on
auto_explain.log_verbose = off
auto_explain.log_nested_statements = on
auto_explain.log_settings = off
auto_explain.log_format = text
auto_explain.sample_rate = 1.0             # 全部記錄
```

```ini
## ==========================================
## 環境 C：生產環境（Production）
## 目標：最小化開銷，抽樣記錄慢查詢
## ==========================================
auto_explain.log_min_duration = 1000       # 記錄超過 1 秒的查詢
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_triggers = off            # 量產關閉，除非懷疑觸發器問題
auto_explain.log_verbose = off
auto_explain.log_nested_statements = on    # 🆕 若使用 PL/pgSQL 建議開啟
auto_explain.log_settings = off            # 量產關閉
auto_explain.log_format = text
auto_explain.sample_rate = 0.5             # 僅記錄 50%，降低 I/O 壓力
```

> **補充（Senior Dev）**  
> `sample_rate = 0.5` 不等於「一半的慢查詢都會被記錄」。慢查詢可能集中在特定 queryid，如果該查詢佔總流量的 80%，則 50% 抽樣下仍有 ~40% 的請求會被記錄。更好的做法是先用 `sample_rate = 1.0` + `log_min_duration = 5000`（5 秒）來找最嚴重的瓶頸，解決後再逐步降低閾值和抽樣率。

#### II. 與 `log_statement` / `log_duration` 的分工

PostgreSQL 內建也有兩個查詢日誌參數：

| 參數 | 作用 | 與 auto_explain 的關係 |
|------|------|----------------------|
| `log_statement` | 記錄 SQL 文本（`none` / `ddl` / `mod` / `all`） | 記錄「**執行了什麼 SQL**」，無效能資訊 |
| `log_duration` | 記錄每條語句的執行耗時 | 記錄「**執行了多久**」，無執行計劃 |
| `auto_explain` | 記錄慢查詢的完整執行計劃 | 記錄「**為什麼慢**」，含完整 EXPLAIN ANALYZE |

**何時應停用 `log_duration`**：

- 若已啟用 `auto_explain.log_analyze = on`，`log_duration` 的資訊（單純耗時）完全被 `auto_explain` 的輸出（耗時 + 執行計劃）覆蓋
- 停用 `log_duration` 可減少重複日誌，降低 I/O
- 建議配置：

```ini
## 建議：啟用 auto_explain 後停用 log_duration
log_duration = off                          # auto_explain 已涵蓋
log_statement = 'ddl'                       # 仍記錄 DDL，用於審計
```

**分工原則**：`log_statement` 負責**審計**（誰改了什麼 Schema），`auto_explain` 負責**效能**（為什麼慢）。

---

### 7. App Dev 最佳實踐

#### 7.1 開發階段：全量記錄，無一遺漏

```ini
## 開發機 postgresql.conf
auto_explain.log_min_duration = 0
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_triggers = on
auto_explain.log_nested_statements = on
auto_explain.sample_rate = 1.0
```

- 每執行一次應用程式功能，就去 `tail -f` PostgreSQL 日誌確認沒有意外的 Seq Scan
- 養成習慣：**開發階段就消滅所有顯而易見的 Seq Scan 和磁碟排序**

#### 7.2 Staging 驗證：真實資料量壓力測試

在 Staging 匯入接近生產規模的資料（或使用 `pg_dump` 的子集），執行完整迴歸測試：

```bash
## 使用 pgBadger 分析 Staging 環境的 auto_explain 日誌
pgbadger /var/log/postgresql/postgresql-*.log -o report.html
```

重點檢查：
- 是否有查詢的 `Rows Removed by Filter` 過高（> 50% 掃描行數）
- 是否出現 `Sort Method: external merge Disk`
- 是否有 Nested Loop 的實際行數遠超估算（統計資訊過時）

#### 7.3 生產環境：抽樣 + 定期分析

```ini
auto_explain.log_min_duration = 1000   # 1 秒，根據 SLI/SLO 調整
auto_explain.sample_rate = 0.5         # 50% 抽樣
auto_explain.log_analyze = on
auto_explain.log_buffers = on
```

**定期審查流程**（建議每週）：

```mermaid
graph TD
    A["收集一週的 PostgreSQL 日誌"] --> B["pgBadger 生成報告"]
    B --> C{"發現新的慢查詢？"}
    C -->|"是"| D["在 Staging 重現<br/>並用 EXPLAIN ANALYZE 驗證"]
    C -->|"否"| E["✅ 本週無新問題"]
    D --> F["調整索引 / SQL / Schema"]
    F --> G["部署到 Production"]
    G --> H["觀察下週報告確認改善"]
    style A fill:#4A90D9,color:#fff
    style D fill:#E67E22,color:#fff
    style G fill:#27AE60,color:#fff
```

#### 7.4 PG 16 `log_nested_statements`：預存程序偵錯利器

PG 16 新增的 `log_nested_statements` 解決了長期以來的痛點：**預存程序（Stored Procedure）/函數內部的 SQL 無法被 auto_explain 單獨記錄**。

```sql
-- 假設有一個複雜的預存程序
CREATE OR REPLACE PROCEDURE process_order(p_order_id INT)
LANGUAGE plpgsql AS $$
DECLARE
    v_customer_id INT;
    v_total NUMERIC;
BEGIN
    -- 巢狀 SQL 1：查詢客戶 ID
    SELECT customer_id INTO v_customer_id
    FROM orders WHERE id = p_order_id;

    -- 巢狀 SQL 2：計算訂單總額
    SELECT SUM(price * quantity) INTO v_total
    FROM order_items WHERE order_id = p_order_id;

    -- 巢狀 SQL 3：更新客戶統計
    UPDATE customer_stats
    SET total_orders = total_orders + 1,
        total_spent = total_spent + v_total
    WHERE customer_id = v_customer_id;
END;
$$;
```

當 `log_nested_statements = on` 時，日誌會出現：

```text
LOG:  duration: 2.345 ms  plan:
Query Text: SELECT customer_id FROM orders WHERE id = $1
Seq Scan on orders  (cost=0.00..850.00 rows=1 width=4)
       (actual time=2.123..2.123 rows=1 loops=1)
  Filter: (id = $1)
  Rows Removed by Filter: 99999
```

**這讓你無需修改預存程序源碼就能定位內部的效能瓶頸**——對於使用大量 PL/pgSQL 的遺留系統尤為珍貴。

#### 7.5 日誌格式選擇

| 格式 | 優點 | 缺點 |
|------|------|------|
| `text`（預設） | 人類可讀，直接 grep，pgBadger 支援 | 不易程式化解析 |
| `json` | 結構化，可程式化處理，適合 ELK/Splunk | 日誌量大，人工閱讀困難 |
| `xml` | 結構化，可 XPath 查詢 | 冗長，生態系支援較少 |
| `yaml` | 比 JSON 更具可讀性 | 生態系支援較少 |

**推薦**：量產環境使用 `json` 格式匯入 ELK，Dev 環境使用 `text` 格式直接閱讀。

#### 7.6 注意事項

- `auto_explain` 記錄的是**執行完成後**的計劃與統計，若查詢被 `statement_timeout` 中斷，不會有輸出。
- 大量日誌輸出可能導致 `log_statement_stats` 與 `auto_explain` 重複記錄，建議擇一。
- 使用 `log_line_prefix` 包含 `%m`（時間戳）和 `%p`（PID）有助於多連線環境下的日誌關聯。
- 搭配 `log_lock_waits = on` 可在同一日誌中觀察鎖等待與慢查詢的關聯。


## 1. pgcrypto — 加解密完整指南

> 本章是 pgcrypto Contrib Extension 的實戰完整指南，從 Hash、密碼儲存、PGP 對稱加密到公鑰加密，搭配 App Dev 視角的 C# + Npgsql + Dapper 整合實例與安全決策矩陣。

---

### I. 核心價值與適用場景

pgcrypto 是 PostgreSQL `contrib` 模組，提供一套在資料庫層級執行密碼學操作的函數庫。它的存在意義是：**讓敏感資料在進入儲存層之前就已經被加密，資料庫管理員或備份檔案洩漏時也無法直接讀取明文**。

#### a. pgcrypto 提供的五大功能族群

| 功能族群 | 核心函數 | 用途 | 可逆性 |
|---------|---------|------|--------|
| 一般 Hash（摘要） | `digest()` | 資料完整性校驗、API 簽名驗證 | 不可逆（單向） |
| 帶金鑰 Hash | `hmac()` | 訊息認證碼、防篡改簽名 | 不可逆（單向） |
| 密碼 Hash | `crypt()` + `gen_salt()` | 使用者密碼安全儲存 | 不可逆（單向） |
| PGP 對稱加密 | `pgp_sym_encrypt()` / `pgp_sym_decrypt()` | 欄位級加密、Data at Rest | 可逆 |
| PGP 公鑰加密 | `pgp_pub_encrypt()` / `pgp_pub_decrypt()` | 跨系統安全傳輸、信封加密 | 可逆 |
| 隨機資料 | `gen_random_bytes()` / `gen_random_uuid()` | Token、UUID 生成 | N/A |

#### b. 技術選型決策樹

```mermaid
flowchart TD
    A["我需要對資料做密碼學操作"] --> B{"目標是什麼？"}
    B -->|"驗證資料<br/>未被篡改"| C{"需要共享<br/>密鑰嗎？"}
    B -->|"儲存使用者<br/>登入密碼"| D["✅ crypt() + gen_salt()<br/>bcrypt 慢 Hash"]
    B -->|"加密敏感欄位<br/>需要解密回來"| E{"誰來解密？"}
    B -->|"生成隨機<br/>Token / UUID"| F["✅ gen_random_bytes()<br/>或 gen_random_uuid()"]

    C -->|"不需要<br/>（公開驗證）"| G["✅ digest()<br/>SHA-256 / SHA-512"]
    C -->|"需要<br/>（共享密鑰）"| H["✅ hmac()<br/>帶金鑰 Hash"]

    E -->|"同一個系統<br/>自己加密自己解密"| I["✅ pgp_sym_encrypt()<br/>對稱加密"]
    E -->|"不同系統<br/>A 加密，B 解密"| J["✅ pgp_pub_encrypt()<br/>公鑰加密"]

    style D fill:#5cb85c,color:white
    style G fill:#4a90d9,color:white
    style H fill:#4a90d9,color:white
    style I fill:#f0ad4e,color:black
    style J fill:#d9534f,color:white
    style F fill:#5cb85c,color:white
```

> **初學者導讀**：這張圖是本章的總地圖。簡單記法：「密碼用 crypt，校驗用 digest，簽名用 hmac，自己加解密用 sym，跨系統加解密用 pub」。選擇錯誤的函數會導致安全災難——例如用 `digest()` 儲存密碼（沒有 salt、計算太快，彩虹表一秒破解），或用 `pgp_sym_encrypt()` 做密碼驗證（可逆 = 密碼洩漏風險）。

---

### II. Hash 函數：digest() 與 hmac()

#### a. digest() — 單向摘要

`digest()` 是標準的密碼學 Hash 函數包裝，對任意輸入產生固定長度的摘要（Digest）。**相同輸入永遠產生相同輸出**，且無法從摘要反推原文。

```sql
-- digest(data text, algorithm text) → bytea
-- digest(data bytea, algorithm text) → bytea

SELECT digest('hello world', 'sha256');
-- 輸出（hex）: b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9
```

支援的演算法（依安全性從低到高排列）：

| 演算法 | 輸出長度 | 安全性 | 建議用途 |
|--------|---------|--------|---------|
| `md5` | 128 bit (16 bytes) | ❌ 已被破解，僅供向後相容 | 禁止用於安全場合 |
| `sha1` | 160 bit (20 bytes) | ❌ 已被碰撞攻擊 | 禁止用於安全場合 |
| `sha224` | 224 bit (28 bytes) | ⚠️ 過時 | 不建議 |
| `sha256` | 256 bit (32 bytes) | ✅ 安全 | 資料完整性校驗、API 簽名 |
| `sha384` | 384 bit (48 bytes) | ✅ 安全 | 高安全場合 |
| `sha512` | 512 bit (64 bytes) | ✅ 安全 | 最高安全強度需求 |

> **補充（Senior Dev）**：`digest()` 的底層實作呼叫 OpenSSL 的 `EVP_Digest*` API。PG 16 使用 OpenSSL 3.x 後，`md5` 在 FIPS 模式下會被禁用（若系統啟用 FIPS 合規）。新專案一律使用 `sha256` 或 `sha512`。

**實戰範例 1：資料完整性校驗**

```sql
-- 情境：儲存檔案內容時同時儲存 SHA-256，事後校驗檔案是否損壞或被篡改
CREATE TABLE file_store (
    file_id    bigserial PRIMARY KEY,
    file_name  text NOT NULL,
    content    bytea NOT NULL,
    sha256_sum bytea NOT NULL,             -- 儲存 digest() 結果供校驗
    created_at timestamptz DEFAULT now()
);

-- 插入時計算 digest
INSERT INTO file_store (file_name, content, sha256_sum)
VALUES (
    'contract.pdf',
    pg_read_binary_file('/uploads/contract.pdf'),
    digest(pg_read_binary_file('/uploads/contract.pdf'), 'sha256')
);

-- 事後校驗
SELECT file_id, file_name,
    CASE
        WHEN sha256_sum = digest(content, 'sha256') THEN 'OK'
        ELSE 'TAMPERED ⚠️'
    END AS integrity_check
FROM file_store;
```

**實戰範例 2：API 請求簽名驗證（簡化版）**

```sql
-- 情境：接收 API 請求時，驗證請求體的 SHA-256 是否與簽名吻合
-- 實戰中通常用 hmac（下一節），這裡展示 digest 的基本用法

CREATE TABLE api_keys (
    client_name text PRIMARY KEY,
    secret_key  text NOT NULL
);

INSERT INTO api_keys VALUES ('mobile_app', 'sk_live_abc123...');

-- 模擬客戶端簽名邏輯（在應用層執行，這裡僅展示 SQL 概念）
WITH payload AS (
    SELECT '{"user_id": 42, "action": "transfer"}'::text AS body
)
SELECT encode(digest(body || (SELECT secret_key FROM api_keys WHERE client_name = 'mobile_app'), 'sha256'), 'hex') AS signature
FROM payload;
-- 客戶端發送 HTTP Header: X-Signature: sha256=<hex_signature>
-- 伺服器端用相同邏輯在 SQL 中驗證
```

#### b. hmac() — 帶金鑰的訊息認證碼

`hmac()` 是 HMAC（Hash-based Message Authentication Code）的實作。它和 `digest()` 的唯一區別是多了一個**共享密鑰**（shared secret）。這意味著只有知道密鑰的雙方才能產生 / 驗證相同的 HMAC 值。

```sql
-- hmac(data text, key text, algorithm text) → bytea
-- hmac(data bytea, key bytea, algorithm text) → bytea

SELECT encode(hmac('hello world', 'my_secret_key', 'sha256'), 'hex');
-- 輸出: 85b1552e4e3a1f4b6c5e2c4c4d5f83a1e0f7c45e9b3a8d2e6f1c4a7b9d0e3f5a
-- 注意：這個值與 digest('hello world', 'sha256') 完全不同，因為包含金鑰參與計算
```

> **初學者導讀**：`digest()` 和 `hmac()` 的關鍵差異：`digest('hello', 'sha256')` 全世界任何人都能算出同樣的值；但 `hmac('hello', 'secret', 'sha256')` 只有知道 `'secret'` 的人才能算出同樣的值。因此 `digest` 用於「檢查資料是否被篡改」（任何人都可驗證），`hmac` 用於「檢查資料是否被未經授權的人篡改」（只有共享密鑰方可驗證）。

```mermaid
flowchart LR
    subgraph "digest() 公開驗證"
        D_S["發送方"] --> D_DATA["原始資料"]
        D_DATA --> D_HASH["digest(data, 'sha256')"]
        D_HASH --> D_TRANSFER["傳輸：data + hash"]
        D_TRANSFER --> D_RECV["接收方"]
        D_RECV --> D_VERIFY["重新 digest(data)<br/>比對 hash 是否一致"]
    end

    subgraph "hmac() 帶密鑰驗證"
        H_S["發送方"] --> H_KEY["共享密鑰 key"]
        H_S --> H_DATA["原始資料"]
        H_DATA --> H_MAC["hmac(data, key, 'sha256')"]
        H_MAC --> H_TRANSFER["傳輸：data + mac"]
        H_TRANSFER --> H_RECV["接收方"]
        H_RECV --> H_VERIFY["重新 hmac(data, key)<br/>比對 mac 是否一致"]
    end

    D_VERIFY --> D_RESULT["✅ 任何人可驗證<br/>但任何人都可偽造"]
    H_VERIFY --> H_RESULT["✅ 僅共享密鑰方可驗證<br/>❌ 偽造者無法計算正確 mac"]

    style D_RESULT fill:#f0ad4e,color:black
    style H_RESULT fill:#5cb85c,color:white
```

**實戰範例：Webhook 簽名驗證**

```sql
-- 情境：接收第三方 Webhook 時，用 HMAC 驗證請求確實來自對方

-- Step 1：儲存共享密鑰（應從環境變數或 KMS 取得，這裡僅為示範）
CREATE TABLE webhook_partners (
    partner_name text PRIMARY KEY,
    shared_secret text NOT NULL        -- ⚠️ 純示範，生產環境應用加密儲存
);

INSERT INTO webhook_partners VALUES ('payment_gateway', 'whsec_8f3a2b1c...');

-- Step 2：驗證函數（將 HTTP Body 與收到的簽名比較）
CREATE OR REPLACE FUNCTION verify_webhook(
    p_partner text,
    p_payload text,
    p_received_signature text
) RETURNS boolean AS $$
DECLARE
    v_secret text;
    v_computed text;
BEGIN
    SELECT shared_secret INTO v_secret
    FROM webhook_partners WHERE partner_name = p_partner;

    IF v_secret IS NULL THEN
        RETURN false;
    END IF;

    -- 計算預期的 HMAC 簽名
    v_computed := encode(hmac(p_payload::bytea, v_secret::bytea, 'sha256'), 'hex');

    -- 常數時間比對（防止時序攻擊）
    RETURN v_computed = p_received_signature;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Step 3：使用
SELECT verify_webhook(
    'payment_gateway',
    '{"event":"payment.succeeded","amount":10000}',
    'a1b2c3d4...'   -- 從 HTTP Header X-Signature 取得
);
```

> **補充（Senior Dev）**：PostgreSQL 的 `=` 操作符對 `text` 的比對不是常數時間（constant-time）的——它會在遇到第一個不同字元時立即返回。對於 HMAC 驗證，理論上攻擊者可以透過時序分析逐字元猜測正確簽名。真正的常數時間比對應在應用層進行（C# 中用 `CryptographicOperations.FixedTimeEquals`）。如果必須在 SQL 中驗證，PG 16 的 `crypt()` 內建了常數時間比對（見第三節）。

#### c. digest 與 hmac 的決定性特質

兩個函數都是**決定性（deterministic）**的：相同輸入 → 相同輸出。這一點是雙刃劍：

| 特性 | 優點 | 缺點 |
|------|------|------|
| 決定性 | 可以建立 Hash Index 加速查詢 | 無法用於密碼儲存（彩虹表攻擊） |
| 決定性 | 相同的資料必然產生相同的摘要 | 若原文可被猜測（如少數枚舉值），攻擊者可預先計算所有可能 Hash 進行比對 |

```sql
-- 利用決定性建立 Hash Index
CREATE TABLE sensitive_users (
    user_id    bigserial PRIMARY KEY,
    email_hash bytea NOT NULL,
    encrypted_email bytea NOT NULL
);

CREATE INDEX idx_email_hash ON sensitive_users (email_hash);

-- 查詢時先對輸入做 digest，再比對（不暴露原始 email 給索引）
SELECT user_id FROM sensitive_users
WHERE email_hash = digest('alice@example.com', 'sha256');
```

---

### III. 密碼儲存：crypt() + gen_salt()

#### a. 為什麼密碼需要專用 Hash

一般 Hash（`digest`）不適合儲存密碼，原因有三：

1. **計算太快**：SHA-256 在現代硬體上每秒可計算數百萬次，攻擊者可以暴力破解
2. **沒有 Salt**：相同密碼 → 相同 Hash，攻擊者用彩虹表（預先計算的 Hash 對照表）一秒破解
3. **無迭代**：一次計算就完成，無法增加攻擊成本

`crypt()` + `gen_salt()` 解決了以上全部問題：

```mermaid
sequenceDiagram
    participant User as 使用者
    participant App as 應用層
    participant PG as PostgreSQL

    Note over User,PG: === 密碼註冊階段 ===
    User->>App: 設定密碼 "MyP@ssw0rd"
    App->>PG: SELECT crypt('MyP@ssw0rd', gen_salt('bf', 8))
    PG->>PG: gen_salt('bf', 8)<br/>→ 隨機 16 bytes salt
    PG->>PG: crypt() 內部 bcrypt<br/>迭代 2^8 = 256 次<br/>→ 產出 Hash
    PG-->>App: $2a$08$randomSalt...hashedOutput
    App->>PG: INSERT INTO users (password_hash) VALUES ('$2a$08$...')
    Note over PG: 僅儲存 Hash，不儲存明文<br/>salt 已嵌在 Hash 字串中

    Note over User,PG: === 登入驗證階段 ===
    User->>App: 輸入密碼 "MyP@ssw0rd"
    App->>PG: SELECT crypt('MyP@ssw0rd', password_hash) = password_hash<br/>FROM users WHERE username = 'alice'
    PG->>PG: crypt() 從 password_hash<br/>自動提取 salt：$2a$08$randomSalt...<br/>用相同 salt 對輸入重新 bcrypt
    PG-->>App: true（若密碼正確）
```

#### b. gen_salt() — 生成隨機 Salt

```sql
-- gen_salt(type text, iter_count integer) → text
-- type: 'bf' (bcrypt), 'md5', 'xdes', 'des'
-- iter_count: 迭代次數（僅對 bf 有意義）

SELECT gen_salt('bf', 8);
-- $2a$08$RqGtNvH5XpLjKzYwQsFbXu

SELECT gen_salt('md5');
-- $1$RqGtNvH5

SELECT gen_salt('xdes');
-- _J9..RqGt
```

Salt 字串格式解析（以 bcrypt 為例）：

```
$2a$08$RqGtNvH5XpLjKzYwQsFbXu
 │   │  └────── 16 bytes 隨機 Salt (Radix-64 編碼)
 │   └─ 迭代次數：2^8 = 256 次
 └─ bcrypt 版本：2a（PG 預設）
```

| Salt 類型 | 演算法 | 安全性 | 建議 |
|-----------|--------|--------|------|
| `bf` | bcrypt（Blowfish-based） | ✅ 最安全 | **唯一推薦** |
| `md5` | MD5 Crypt | ❌ 過弱 | 僅供向後相容 |
| `xdes` | Extended DES | ❌ 過弱 | 僅供向後相容 |
| `des` | Original DES Crypt | ❌ 極弱 | 禁止使用 |

> **初學者導讀**：`gen_salt()` 的輸出**不是**機密——它只是隨機資料，公開也沒關係。Salt 的作用是確保相同的密碼產生不同的 Hash。例如，兩個使用者都設密碼 `password123`，因為他們的 Salt 不同，最終的 bcrypt Hash 完全不同，攻擊者無法用彩虹表一次破解所有人的密碼。

#### c. crypt() — 密碼 Hash 與驗證

```sql
-- crypt(password text, salt text) → text

-- 註冊：gen_salt() 自動生成 salt
SELECT crypt('MyP@ssw0rd', gen_salt('bf', 8));
-- $2a$08$RqGtNvH5XpLjKzYwQsFbXu9i3GNFbHRHt5eXq1x0uVWvDP2IyIEpBi

-- 驗證：從已儲存的 Hash 自動提取 salt 並重新計算
SELECT crypt('MyP@ssw0rd', '$2a$08$RqGtNvH5XpLjKzYwQsFbXu9i3GNFbHRHt5eXq1x0uVWvDP2IyIEpBi');
-- $2a$08$RqGtNvH5XpLjKzYwQsFbXu9i3GNFbHRHt5eXq1x0uVWvDP2IyIEpBi  ← 完全相同

SELECT crypt('WrongPassword', '$2a$08$RqGtNvH5XpLjKzYwQsFbXu9i3GNFbHRHt5eXq1x0uVWvDP2IyIEpBi');
-- $2a$08$RqGtNvH5XpLjKzYwQsFbXu5xZtPNkDwzrQsFbXu9i3GNFbHRHt5eX ← 完全不同
```

`crypt()` 的內部機制：
1. 從 `salt` 參數中自動解析出 Salt 值和迭代次數（格式如 `$2a$08$randomSalt`）
2. 忽略 `salt` 參數中 Salt 之後的部分（即儲存的 Hash 值本身不會被當作 Salt 使用）
3. 以解析出的 Salt + 迭代次數對 `password` 執行 bcrypt
4. 回傳完整的 Hash 字串（包含 `$2a$08$salt$hash`）

**登入驗證模式**：

```sql
-- 完整的登入驗證查詢
SELECT
    CASE
        WHEN password_hash IS NULL THEN 'USER_NOT_FOUND'
        WHEN password_hash = crypt(:input_password, password_hash) THEN 'LOGIN_OK'
        ELSE 'WRONG_PASSWORD'
    END AS login_result
FROM users
WHERE username = :input_username;
```

> **補充（Senior Dev）**：`crypt()` 內建了**常數時間比對**（constant-time comparison），不同於一般的 `=` 字串比對。這意味著無論輸入是正確密碼還是錯誤密碼，`crypt()` 的執行時間幾乎相同，防止時序側信道攻擊（timing side-channel attack）。但請注意：只有 `WHERE password_hash = crypt(...)` 這種模式才受保護。如果你先取出 `password_hash` 再在應用層比對，則失去了這個保護。

#### d. iter_count 選擇 — 效能與安全平衡

`iter_count`（迭代次數）是 bcrypt 的關鍵參數：實際迭代次數 = 2^iter_count。數字越大越安全，但也越慢。

**Benchmark 實測表**（在 PG 16、Intel Xeon Gold 處理器上執行 `crypt()` 的耗時）：

```sql
-- 自行 Benchmark 的查詢（請在你的實際硬體上執行以獲得準確數值）
SELECT
    'bf/8 (2^8=256)' AS method,
    avg(extract(epoch from (SELECT crypt('benchmark_test', gen_salt('bf', 8))))) * 1000 AS avg_ms
FROM generate_series(1, 100);

-- 重複執行以獲得更多數據點
```

| iter_count 參數 | 實際迭代次數 | 單次 crypt() 耗時 (參考) | iter / sec | 建議場景 |
|-----------------|-------------|------------------------|-----------|---------|
| `bf/5` | 32 | ~5ms | ~200/s | ❌ 過快，不建議 |
| `bf/6` | 64 | ~10ms | ~100/s | ⚠️ 內部系統下限 |
| `bf/7` | 128 | ~20ms | ~50/s | ⚠️ 可接受 |
| **`bf/8`** | **256** | **~40ms** | **~25/s** | ✅ **生產環境推薦** |
| `bf/9` | 512 | ~80ms | ~12/s | ✅ 高安全場合 |
| `bf/10` | 1024 | ~160ms | ~6/s | ⚠️ 使用者登入體驗變差 |
| `bf/11` | 2048 | ~320ms | ~3/s | ❌ 除非離線批次處理 |
| `bf/12` | 4096 | ~640ms | ~1.5/s | ❌ 實務上過慢 |

選擇原則：**目標是讓單次 `crypt()` 耗費 50-100ms**。這樣攻擊者每秒只能嘗試 10-20 次，而合法使用者在登入時等待 50-100ms 幾乎無感。

```sql
-- 在生產環境部署前，用這個查詢測量實際耗時
DO $$
DECLARE
    start_time timestamptz;
    end_time   timestamptz;
    v_hash     text;
BEGIN
    start_time := clock_timestamp();
    FOR i IN 1..100 LOOP
        v_hash := crypt('test_password_' || i, gen_salt('bf', 8));
    END LOOP;
    end_time := clock_timestamp();
    RAISE NOTICE '100 crypt(bf/8): % ms',
        extract(epoch from (end_time - start_time)) * 1000;
END $$;
```

> **初學者導讀**：不必過度擔心 `crypt()` 很慢。對合法使用者來說，登入是低頻操作（每人每小時可能只登入 1-2 次），50ms 的延遲完全無感。但對攻擊者來說，每秒只能嘗試 ~20 個密碼，暴力破解 8 位數英數混合密碼需要數百年。

#### e. bcrypt 72 位元組限制警告

bcrypt 有一個常被忽略的重要限制：**輸入密碼最大 72 bytes（UTF-8 編碼後）**。

```sql
-- ✅ 正常：密碼在 72 bytes 以內
SELECT crypt('MySecureP@ss', gen_salt('bf', 8));  -- OK

-- ⚠️ 可能問題：含多字節 UTF-8 字符時，72 bytes 可能僅容納約 24 個中文字
SELECT crypt('這是一個非常長的中文密碼超過七十二位元組限制測試', gen_salt('bf', 8));
-- bcrypt 僅取前 72 bytes，超出部分被截斷（不會報錯！）

-- ❌ 潛在風險：超長密碼被靜默截斷，使用者不知情
```

**緩解策略**：

| 策略 | 做法 | 優點 | 缺點 |
|------|------|------|------|
| 前置 SHA-512 | `crypt(digest(password, 'sha512'), gen_salt('bf', 8))` | 任意長度密碼都能處理 | 失去 bcrypt 的部分安全特性 |
| 前端限制 | 在應用層限制密碼最長 64 字符 | 最簡單 | 限制使用者自由度 |
| 直接接受 | 不處理，接受 72 bytes 限制 | 無需額外程式碼 | 長密碼使用者被靜默截斷 |

> **補充（Senior Dev）**：前置 SHA-512 方案雖然解決了長度限制，但引入了新的理論風險——如果攻擊者知道你在用 `crypt(sha512(password), salt)`，他們可以繞過 bcrypt 直接攻擊 SHA-512（快得多）。但實務上，只要 salt 是隨機的，這個風險極低。更安全的替代方案是使用 PG 的 `pg_sodium` 擴展中的 Argon2id（記憶體硬化的密碼 Hash），但 `pg_sodium` 非 contrib，需額外安裝。

---

### IV. PGP 對稱加密：pgp_sym_encrypt() / pgp_sym_decrypt()

#### a. 對稱加密概念與內部流程

對稱加密：使用**同一個密碼**進行加密與解密。適合場景：同一個系統自己加密資料，日後自己解密回來（如儲存 API Key、信用卡號、病歷資料）。

```mermaid
sequenceDiagram
    participant App as 應用層
    participant PG as PostgreSQL
    participant Disk as 磁碟儲存

    Note over App,Disk: === 加密寫入 ===
    App->>PG: pgp_sym_encrypt('敏感資料', 'my_encryption_key')
    PG->>PG: 1. 生成隨機 Session Key (128/256 bit)
    PG->>PG: 2. 用 Session Key + AES 加密明文
    PG->>PG: 3. 用密碼 + S2K (Salt + 迭代) 加密 Session Key
    PG->>PG: 4. 組合：PGP Header + 加密的 Session Key + 密文
    PG->>Disk: bytea 密文寫入磁碟

    Note over App,Disk: === 解密讀取 ===
    App->>PG: pgp_sym_decrypt(密文, 'my_encryption_key')
    PG->>PG: 1. 解析 PGP Header，提取加密的 Session Key
    PG->>PG: 2. 用密碼 + S2K 解密 Session Key
    PG->>PG: 3. 用 Session Key + AES 解密密文
    PG-->>App: 明文 '敏感資料'
```

> **初學者導讀**：為什麼不直接用密碼加密明文，而要先用隨機 Session Key？因為 (1) Session Key 是真正隨機的高熵金鑰，密碼則可能低熵；(2) 如果多筆資料用同一密碼加密，直接加密會暴露模式；使用隨機 Session Key 則每筆加密的實際加密金鑰都不同，相同明文也會產出不同密文。

#### b. 基礎用法

```sql
-- pgp_sym_encrypt(data text, psw text [, options text]) → bytea
-- pgp_sym_decrypt(msg bytea, psw text [, options text]) → text

-- 加密
SELECT pgp_sym_encrypt('信用卡號: 4111-1111-1111-1111', 'MySecretKey123');
-- \xc30d04090302... (PGP 格式 bytea)

-- 解密
SELECT pgp_sym_decrypt(
    pgp_sym_encrypt('信用卡號: 4111-1111-1111-1111', 'MySecretKey123'),
    'MySecretKey123'
);
-- 信用卡號: 4111-1111-1111-1111

-- 錯誤密碼解密 → 報錯
SELECT pgp_sym_decrypt(
    pgp_sym_encrypt('信用卡號: 4111-1111-1111-1111', 'MySecretKey123'),
    'WrongKey'
);
-- ERROR:  Wrong key or corrupt data
```

#### c. Options 選項詳解

`pgp_sym_encrypt()` 的第三個參數是以逗號分隔的選項字串：

```sql
SELECT pgp_sym_encrypt(
    'sensitive data',
    'encryption_key',
    'compress-algo=2, compress-level=6, cipher-algo=aes256, s2k-mode=3, s2k-count=65000000'
);
```

**核心 Options 說明**：

| 選項 | 可選值 | 預設值 | 說明 |
|------|--------|--------|------|
| `cipher-algo` | `aes128`, `aes192`, `aes256`, `bf`, `cast5`, `des`, `3des` | `aes256` | 對稱加密演算法 |
| `compress-algo` | `0`（無壓縮）, `1`（ZIP）, `2`（ZLIB） | `0` | 加密前先壓縮（明文壓縮→加密，更安全且更小） |
| `compress-level` | `0`（無）~ `9`（最大） | `6` | 壓縮級別 |
| `s2k-mode` | `0`（無 salt）, `1`（salt 但不迭代）, `3`（salt + 迭代，推薦） | `3` | S2K 模式（String-to-Key） |
| `s2k-count` | `1024` ~ `65011712` | `65536` | S2K 迭代次數（越高越安全但越慢） |
| `s2k-digest-algo` | `sha1`, `sha256`, `sha512` | `sha256` | S2K Hash 演算法 |
| `disable-mdc` | `0`（啟用）, `1`（禁用） | `0` | Modification Detection Code（完整性校驗） |
| `convert-crlf` | `0`（不轉換）, `1`（轉換 CRLF→LF） | `0` | 跨平台換行轉換 |

> **補充（Senior Dev）**：`s2k-count` 是對稱加密的「安全強度開關」。預設值 65536 對批次處理和 API 請求已經足夠，但如果加密操作頻率很高（如每秒數百次解密），高 `s2k-count` 會顯著增加 CPU 負載。`s2k-mode=3`（Salted + Iterated）是唯一推薦的模式；`s2k-mode=0` 完全沒有 salt，相同密碼加密相同明文會產出相同密文——這是一個重大安全漏洞。`disable-mdc=0` 請永遠保持開啟（預設），否則攻擊者可能在不被發現的情況下篡改密文。

#### d. 實戰：欄位級加密（Column-Level Encryption）

```sql
-- 情境：users 表的 email 和 phone 欄位需要加密儲存

-- 建立加密輔助檢視（封裝加密解密邏輯）
CREATE TABLE users_secure (
    user_id       bigserial PRIMARY KEY,
    username      text NOT NULL,
    email_crypt   bytea NOT NULL,          -- 加密儲存
    phone_crypt   bytea NOT NULL,
    created_at    timestamptz DEFAULT now()
);

-- 加密插入（密碼從 Custom GUC 取得，詳見第六節）
INSERT INTO users_secure (username, email_crypt, phone_crypt)
VALUES (
    'alice',
    pgp_sym_encrypt('alice@example.com', current_setting('myapp.encryption_key'),
        'compress-algo=2, compress-level=6'),
    pgp_sym_encrypt('+886-912-345-678', current_setting('myapp.encryption_key'),
        'compress-algo=2, compress-level=6')
);

-- 解密讀取
SELECT
    user_id,
    username,
    pgp_sym_decrypt(email_crypt, current_setting('myapp.encryption_key')) AS email,
    pgp_sym_decrypt(phone_crypt, current_setting('myapp.encryption_key')) AS phone
FROM users_secure
WHERE user_id = 1;

-- 使用者端看不到密文，就像普通查詢
--  user_id | username |        email         |      phone
-- ---------+----------+----------------------+------------------
--        1 | alice    | alice@example.com   | +886-912-345-678
```

> **初學者導讀**：欄位級加密的核心概念——資料在**寫入磁碟時**是密文（`bytea`），在**讀取時**即時解密。備份檔案或磁碟快照中的 `email_crypt` 欄位是無意義的亂碼。但請注意：加密後無法對該欄位建立普通 B-Tree Index（因為密文沒有排序語義），也無法用 `WHERE email = 'alice@example.com'` 查詢（因為 email 不是明文儲存）。

#### e. 對稱加密限制

| 限制 | 說明 | 緩解 |
|------|------|------|
| 無法索引 | 密文是 random bytes，B-Tree 無法排序 | 使用 Blind Index（對明文做 HMAC 後另外儲存） |
| 無法搜尋 | `WHERE encrypted_col LIKE '%keyword%'` 不可行 | 使用 Blind Index 或 pg_bigm/pg_trgm 但不完美 |
| 金鑰輪換困難 | 更換密碼需重新加密所有舊資料 | 信封加密模式（用 Master Key 加密 Data Key） |
| 效能開銷 | 每次讀寫都需加解密 | 使用 `s2k-count` 較低的值 + 硬體 AES-NI 加速 |

---

### V. PGP 公鑰加密：pgp_pub_encrypt() / pgp_pub_decrypt()

#### a. 非對稱加密概念

公鑰加密（Public-Key Cryptography）：使用**一對數學相關的金鑰**——公鑰（Public Key）加密，私鑰（Private Key）解密。公鑰可以公開分發，私鑰必須嚴格保密。

適合場景：
- 不同系統 / 團隊之間的安全資料交換（發送方只知道公鑰，無法解密自己發送的資料）
- 多個寫入方共用公鑰加密，只有一個讀取方持有私鑰解密
- 符合法規要求：加密者與解密者權限分離（Separation of Duties）

```mermaid
sequenceDiagram
    participant Sender as 發送方（App A）
    participant DB as PostgreSQL
    participant Receiver as 接收方（App B）

    Note over Sender,Receiver: === 前置準備 ===
    Receiver->>Receiver: gpg --gen-key<br/>生成金鑰對
    Receiver-->>Sender: 分發公鑰 public.key
    Sender->>DB: 將公鑰 dearmor 後存入 DB

    Note over Sender,Receiver: === 加密寫入 ===
    Sender->>DB: pgp_pub_encrypt('敏感資料', dearmor(公鑰))
    DB->>DB: 1. 生成隨機 Session Key
    DB->>DB: 2. 用 Session Key + AES 加密明文
    DB->>DB: 3. 用公鑰加密 Session Key
    DB->>DB: 4. 組合 PGP Message

    Note over Sender,Receiver: === 解密讀取 ===
    Receiver->>DB: pgp_pub_decrypt(密文, dearmor(私鑰), '私鑰密碼')
    DB->>DB: 1. 用私鑰 + 私鑰密碼解密 Session Key
    DB->>DB: 2. 用 Session Key + AES 解密密文
    DB-->>Receiver: 明文

    Note over Sender: ⚠️ 發送方無法解密<br/>（沒有私鑰）
```

#### b. GPG 金鑰生成與格式轉換

**Step 1：在安全的環境中生成 GPG 金鑰對**

```bash
# 生成金鑰對（互動式，建議在離線環境執行）
gpg --full-generate-key
# Please select what kind of key you want:
#    (1) RSA and RSA (default)
# Your selection? 1
# What keysize do you want? 4096
# Key is valid for? 0 (永不過期)
# Real name: MyApp Encryption
# Email address: security@myapp.com
# Comment: Production Data Encryption Key
# 輸入私鑰保護密碼（passphrase）

# 匯出公鑰（ASCII-armor 格式，可安全分發）
gpg --export --armor "MyApp Encryption" > public.key

# 匯出私鑰（ASCII-armor 格式，務必安全儲存！）
gpg --export-secret-keys --armor "MyApp Encryption" > private.key
```

**Step 2：將 ASCII-Armor 格式轉換為 bytea（使用 `dearmor()`）**

```sql
-- dearmor(data text) → bytea：將 PGP ASCII-armor 格式轉為二進制 bytea
-- 公鑰
INSERT INTO pgp_keys (key_name, key_data)
VALUES (
    'app_public_key',
    dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
mQINBGg...
-----END PGP PUBLIC KEY BLOCK-----')
);

-- 私鑰（⚠️ 通常不應存入 DB，這裡僅示範）
-- 私鑰應從外部安全注入（Custom GUC、檔案系統、或 KMS）
```

> **初學者導讀**：`dearmor()` 的作用是把人類可讀的 ASCII 格式金鑰（帶有 `-----BEGIN PGP PUBLIC KEY BLOCK-----` 標頭）轉換為 PostgreSQL `bytea`。`pgp_pub_encrypt()` 和 `pgp_pub_decrypt()` 需要的參數型別是 `bytea`，不是 `text`。另一個常見的坑：確保你的金鑰內容中沒有多餘的空白或換行——這些會導致 `dearmor()` 失敗。

#### c. pgp_pub_encrypt() — 用公鑰加密

```sql
-- pgp_pub_encrypt(data text, key bytea [, options text]) → bytea
-- key: dearmor(公鑰 ASCII-armor 字串)

-- 加密
SELECT pgp_pub_encrypt(
    '只有持有私鑰的人才能解密這段訊息',
    dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
mQINBGgXyzEBEAC...
-----END PGP PUBLIC KEY BLOCK-----'),
    'compress-algo=2, cipher-algo=aes256'
);
-- \xc30d04090302... (PGP Message)
```

#### d. pgp_pub_decrypt() — 用私鑰解密

```sql
-- pgp_pub_decrypt(msg bytea, key bytea [, psw text [, options text]]) → text
-- key: dearmor(私鑰 ASCII-armor 字串)
-- psw: 私鑰的保護密碼（passphrase）

-- 解密
SELECT pgp_pub_decrypt(
    pgp_pub_encrypt('秘密訊息', dearmor(public_armor)),
    dearmor(private_armor),
    'my_private_key_passphrase'
);
-- 秘密訊息

-- 若用錯誤的私鑰解密 → 報錯
SELECT pgp_pub_decrypt(
    encrypted_msg,
    dearmor(wrong_private_key),
    'wrong_passphrase'
);
-- ERROR:  Wrong key or corrupt data
```

#### e. pgp_key_id() — 識別加密訊息的金鑰

`pgp_key_id()` 可以從加密訊息中提取用於加密的**公鑰 Key ID**，方便判斷該訊息要用哪把私鑰解密（當有多把金鑰時）。

```sql
-- pgp_key_id(bytea) → text

SELECT pgp_key_id(encrypted_msg) AS key_id
FROM messages;
-- key_id
-- ----------
-- D5A6B7C8E9F01234
```

#### f. 信封加密模式（Envelope Encryption）

實務上的最佳實踐：結合對稱加密（速度快）與公鑰加密（安全分發）的優勢。

```mermaid
flowchart TD
    subgraph "寫入階段（加密）"
        A1["生成隨機 Data Key<br/>gen_random_bytes(32)"] --> A2["用 Data Key + AES<br/>加密敏感資料"]
        A1 --> A3["用公鑰加密 Data Key"]
        A2 --> A4["儲存：加密的 Data Key + 密文"]
        A3 --> A4
    end

    subgraph "讀取階段（解密）"
        B1["從儲存中取出：<br/>加密的 Data Key + 密文"] --> B2["用私鑰解密 Data Key"]
        B2 --> B3["用 Data Key + AES<br/>解密密文"]
    end

    A4 -.-> B1

    style A4 fill:#4a90d9,color:white
    style B3 fill:#5cb85c,color:white
```

```sql
-- 信封加密實作
CREATE TABLE encrypted_records (
    record_id     bigserial PRIMARY KEY,
    encrypted_key bytea NOT NULL,       -- 用公鑰加密的 Data Key
    encrypted_data bytea NOT NULL,      -- 用 Data Key 對稱加密的資料
    created_at    timestamptz DEFAULT now()
);

-- 加密寫入
DO $$
DECLARE
    v_data_key bytea := gen_random_bytes(32);  -- 隨機 256-bit Data Key
    v_public_key bytea := dearmor(pg_read_file('/keys/public.key'));
BEGIN
    INSERT INTO encrypted_records (encrypted_key, encrypted_data)
    VALUES (
        pgp_pub_encrypt(encode(v_data_key, 'hex'), v_public_key),   -- 公鑰加密 Data Key
        pgp_sym_encrypt('敏感資料內容', encode(v_data_key, 'hex'))  -- Data Key 對稱加密資料
    );
END $$;

-- 解密讀取
DO $$
DECLARE
    v_private_key bytea := dearmor(pg_read_file('/keys/private.key'));
    v_private_psw text := 'key_passphrase';
    v_data_key_hex text;
BEGIN
    -- 1. 用私鑰解密 Data Key
    SELECT pgp_pub_decrypt(encrypted_key, v_private_key, v_private_psw)
    INTO v_data_key_hex FROM encrypted_records WHERE record_id = 1;

    -- 2. 用 Data Key 對稱解密資料
    PERFORM pgp_sym_decrypt(encrypted_data, v_data_key_hex)
    FROM encrypted_records WHERE record_id = 1;
END $$;
```

> **補充（Senior Dev）**：信封加密的優勢：Data Key 只加密一次（用公鑰），之後大量資料用 Data Key 做快速的對稱加密。當需要輪換金鑰時，只需用新公鑰重新加密 Data Key（即更新 `encrypted_key` 欄位），**無需重新加密所有資料**。這是 AWS KMS、Google Cloud KMS 等雲端金鑰管理服務的核心理念。如果使用自訂信封加密，強烈建議將 Master Key（公/私鑰對）交由硬體安全模組（HSM）或 KMS 管理。

---

### VI. App Dev 實戰：C# + Npgsql + Dapper

#### a. 連線初始化 — 用 Custom GUC 傳遞加密金鑰

在 SQL 中**不應**硬編碼加密金鑰（會出現在 `pg_stat_statements`、錯誤日誌、備份檔案中），應透過 Custom GUC（Grand Unified Configuration）變數在連線層級注入。

```csharp
using Npgsql;
using Dapper;
using System.Security.Cryptography;

public class PgCryptoService
{
    private readonly string _connectionString;

    public PgCryptoService(string connectionString)
    {
        _connectionString = connectionString;
    }

    // ✅ 正確做法：Connection Pool 初始化時設定 Custom GUC
    public async Task<NpgsqlConnection> GetConnectionAsync()
    {
        var conn = new NpgsqlConnection(_connectionString);
        await conn.OpenAsync();

        // 從環境變數或 KMS 取得加密金鑰（不在 code 中硬編碼）
        var encryptionKey = Environment.GetEnvironmentVariable("PG_ENCRYPTION_KEY");

        using var cmd = new NpgsqlCommand(
            "SET SESSION myapp.encryption_key = @key",
            conn
        );
        cmd.Parameters.AddWithValue("key", encryptionKey);
        await cmd.ExecuteNonQueryAsync();

        return conn;
    }

    // ❌ 錯誤做法：在 SQL 中硬編碼 Key
    // var sql = "SELECT pgp_sym_decrypt(email, 'hardcoded_key') FROM users";
    // → 這個 key 會出現在 pg_stat_statements、slow query log 中！
}
```

> **初學者導讀**：`SET SESSION myapp.encryption_key = ...` 是 PostgreSQL 的 Custom GUC 機制（類似 session 變數）。設定後，在**同一個連線**中的所有後續查詢都可以用 `current_setting('myapp.encryption_key')` 取得這個值。當連線歸還到 Connection Pool 時，如果 Npgsql 沒有執行 `DISCARD ALL` 或 `RESET ALL`，這個 Session 變數會殘留。下一節展示正確的清理方式。

#### b. 連線池安全：確保 Session 變數不會洩漏到其他連線

```csharp
// ✅ 完整的安全連線初始化模式
public async Task<NpgsqlConnection> GetSecureConnectionAsync()
{
    var conn = new NpgsqlConnection(_connectionString);
    await conn.OpenAsync();

    // 先重置可能的殘留變數
    using (var resetCmd = new NpgsqlCommand("RESET ALL", conn))
    {
        await resetCmd.ExecuteNonQueryAsync();
    }

    // 設定本連線的 Custom GUC
    var key = Environment.GetEnvironmentVariable("PG_ENCRYPTION_KEY");
    using (var setCmd = new NpgsqlCommand(
        "SET SESSION myapp.encryption_key = @key", conn))
    {
        setCmd.Parameters.AddWithValue("key", key);
        await setCmd.ExecuteNonQueryAsync();
    }

    return conn;
}

// 可在連線字串中設定 Application Name 方便追蹤
// "Host=...;Database=...;Application Name=MyApp_PgCrypto"
```

```sql
-- 在 PostgreSQL 側確認 session 變數設定成功
SELECT current_setting('myapp.encryption_key', true) AS encryption_key;
-- true 參數表示變數不存在時回傳 NULL 而非報錯
```

#### c. 對稱加密 — 插入加密資料

```csharp
public async Task InsertEncryptedUser(string username, string email, string phone)
{
    using var conn = await GetSecureConnectionAsync();

    var sql = @"
        INSERT INTO users_secure (username, email_crypt, phone_crypt)
        VALUES (
            @username,
            pgp_sym_encrypt(@email, current_setting('myapp.encryption_key'),
                'compress-algo=2, compress-level=6'),
            pgp_sym_encrypt(@phone, current_setting('myapp.encryption_key'),
                'compress-algo=2, compress-level=6')
        )";

    await conn.ExecuteAsync(sql, new
    {
        username,
        email,
        phone
    });
}
```

> **補充（Senior Dev）**：Dapper 在使用 `@param` 參數化時，會自動將 C# string 對應到 PostgreSQL 的 `text` 型別。`pgp_sym_encrypt()` 的第一個參數是 `text`，所以直接傳入即可，無需額外轉換。輸出為 `bytea`，映射到 C# 時會變成 `byte[]`。

#### d. 對稱加密 — 解密讀取

```csharp
public record SecureUser(long UserId, string Username, string Email, string Phone);

public async Task<SecureUser?> GetUserById(long userId)
{
    using var conn = await GetSecureConnectionAsync();

    var sql = @"
        SELECT
            user_id   AS UserId,
            username  AS Username,
            pgp_sym_decrypt(email_crypt, current_setting('myapp.encryption_key')) AS Email,
            pgp_sym_decrypt(phone_crypt, current_setting('myapp.encryption_key')) AS Phone
        FROM users_secure
        WHERE user_id = @userId";

    return await conn.QueryFirstOrDefaultAsync<SecureUser>(sql, new { userId });
}
```

#### e. 密碼驗證 — 用 crypt() 做登入

```csharp
public record LoginResult(bool Success, string Message, long? UserId = null);

public async Task<LoginResult> VerifyPassword(string username, string password)
{
    using var conn = await GetSecureConnectionAsync();

    var sql = @"
        SELECT
            user_id,
            CASE
                WHEN password_hash = crypt(@password, password_hash) THEN 'OK'
                ELSE 'WRONG'
            END AS result
        FROM users
        WHERE username = @username";

    var row = await conn.QueryFirstOrDefaultAsync(sql, new { username, password });

    if (row == null)
        return new LoginResult(false, "User not found");

    return row.result == "OK"
        ? new LoginResult(true, "Login successful", (long)row.user_id)
        : new LoginResult(false, "Wrong password");
}
```

#### f. 完整使用模式 — 搭配 Dependency Injection

```csharp
// Program.cs / Startup.cs
builder.Services.AddScoped<PgCryptoService>(sp =>
{
    var connString = builder.Configuration.GetConnectionString("DefaultConnection");
    return new PgCryptoService(connString);
});

// 在 Controller / Service 中使用
public class UserController
{
    private readonly PgCryptoService _crypto;

    public UserController(PgCryptoService crypto) => _crypto = crypto;

    public async Task<IActionResult> Register(RegisterRequest req)
    {
        await _crypto.InsertEncryptedUser(req.Username, req.Email, req.Phone);
        return Ok();
    }

    public async Task<IActionResult> Login(LoginRequest req)
    {
        var result = await _crypto.VerifyPassword(req.Username, req.Password);
        return result.Success ? Ok(result) : Unauthorized(result);
    }
}
```

> **補充（Senior Dev）**：在 Npgsql 6.0+ 中，你可以記錄 `conn.ProcessID`（backend_pid）到 application log：
> ```csharp
> _logger.LogInformation("Using PG backend PID: {Pid}", conn.ProcessID);
> ```
> 這讓你可以事後在 `pg_stat_activity` 中對照特定連線的 SQL 歷史。另建議在 Npgsql Connection String 中設定 `Application Name=MyApp_Service`，方便在 `pg_stat_activity.application_name` 欄位中快速識別。

---

### VII. 安全決策矩陣

#### a. 加密方案選擇總流程

```mermaid
flowchart TD
    START["需要保護敏感資料"] --> Q1{"資料類型？"}
    Q1 -->|"使用者登入密碼"| A1["✅ crypt() + gen_salt('bf', 8)<br/>bcrypt 慢 Hash<br/>不可逆 + Salt + 迭代"]
    Q1 -->|"需解密讀回的<br/>欄位資料"| Q2{"誰需要解密？"}
    Q2 -->|"僅同一個 App<br/>自己寫自己讀"| Q3{"資料量級？"}
    Q2 -->|"不同系統 / 團隊<br/>跨信任邊界"| A4["✅ PGP 公鑰加密<br/>公鑰加密 + 私鑰解密<br/>+ 信封加密模式"]
    Q1 -->|"需要搜尋 / 比對<br/>但不可解密"| A2["✅ digest() / hmac()<br/>決定性 Hash<br/>可建 Blind Index"]
    Q1 -->|"需要比對<br/>且可解密"| A3["⚠️ 審慎評估<br/>考慮 Tokenization<br/>而非加密"]

    Q3 -->|"小量（< 1000 行）"| A5["✅ pgp_sym_encrypt()<br/>對稱加密<br/>最簡單直接"]
    Q3 -->|"大量（> 1000 行）"| A6["✅ 信封加密<br/>Data Key 對稱 + Master Key 公鑰<br/>金鑰輪換成本最低"]

    style A1 fill:#5cb85c,color:white
    style A2 fill:#4a90d9,color:white
    style A4 fill:#d9534f,color:white
    style A5 fill:#f0ad4e,color:black
    style A6 fill:#5cb85c,color:white
    style A3 fill:#e8d44d,color:black
```

#### b. 四種方案對比矩陣

| 面向 | crypt() + gen_salt() | PGP 對稱加密 | PGP 公鑰加密 | pg_sodium（對照組） |
|------|---------------------|-------------|-------------|-------------------|
| **可逆性** | 不可逆（單向） | 可逆 | 可逆 | 依函數 |
| **金鑰管理** | Salt 內嵌於 Hash，無需額外管理 | 需管理對稱金鑰 | 需管理金鑰對（公鑰可公開） | 需管理對稱/非對稱金鑰 |
| **金鑰輪換** | 無法輪換（需使用者重設密碼） | 需重新加密所有資料 | 信封加密模式下成本低 | 依實作 |
| **效能** | 慢（設計如此） | 中等（S2K 迭代） | 更慢（非對稱運算） | 快（libsodium 最佳化） |
| **搜尋支援** | 不可搜尋 | 不可搜尋（需 Blind Index） | 不可搜尋（需 Blind Index） | 部分支援（Sealed Box） |
| **PG 內建** | ✅ contrib | ✅ contrib | ✅ contrib | ❌ 需額外安裝 |
| **FIPS 合規** | bcrypt 非 FIPS | AES 符合 | RSA 符合 | libsodium 非 FIPS |
| **跨系統互通** | ❌ PG 專有 | ⚠️ PGP 標準，但實作可能有差異 | ✅ PGP 標準，GPG 相容 | ⚠️ libsodium 專有格式 |
| **適合場景** | 使用者密碼 | 欄位級加密（同 App） | 跨系統安全傳輸 | 高效能 API 簽名、Token 加密 |
| **主要風險** | 72 bytes 限制 | 密碼洩漏 = 全部資料可解 | 私鑰洩漏 = 全部資料可解 | 非 PG 內建，相依性風險 |

#### c. 金鑰管理安全原則速查表

| 原則 | 具體做法 | 風險等級 |
|------|---------|---------|
| 金鑰不入 SQL | 使用 Custom GUC 注入，推薦 `current_setting()` | 🔴 高 |
| 金鑰不入 Code | 從環境變數 / KMS / Secrets Manager 讀取 | 🔴 高 |
| 金鑰不入 Log | 確保 `log_statement = 'all'` 時不會洩漏金鑰 | 🟡 中 |
| 金鑰不入 `pg_stat_statements` | 用 Custom GUC 而非 SQL 字串拼接 | 🟡 中 |
| 金鑰輪換計畫 | 信封加密模式，重新加密 Data Key 而非全量資料 | 🟡 中 |
| 最小權限 | 加密函數執行權限僅授予必要角色 | 🟢 低 |
| 稽核日誌 | 記錄加密/解密操作（誰、何時、哪個表）但不記錄金鑰 | 🟢 低 |

> **補充（Senior Dev）**：pgcrypto 的底層依賴 OpenSSL，因此 OpenSSL 的安全性更新直接影響 pgcrypto。PG 16 使用 OpenSSL 3.x 後，部分舊演算法（如 `bf` 在 FIPS 模式下）可能無法使用。另外，`pgp_sym_encrypt()` 和 `pgp_pub_encrypt()` 的輸出是二進制 `bytea`，在透過 JSON API 回傳時需要 Base64 編碼（C#: `Convert.ToBase64String(byte[])`）。如果應用層需要**跨程式語言的 PGP 互通**（如 Python 的 `python-gnupg` 與 PG pgcrypto 交換加密資料），建議使用標準 PGP 格式並測試相容性——不同實作對 `s2k-mode`、`cipher-algo` 預設值不同。


## 2. pg_trgm — 三元組文本相似度與模糊搜索

> **初學者導讀：** pg_trgm 是 PostgreSQL 內建的 contrib extension，透過「三元組（trigram）」技術實現模糊文本比對與相似度計算。如果你需要一個「不精確但夠快」的文本搜索引擎（例如搜尋使用者暱稱、地址模糊匹配），而不想引入 Elasticsearch，pg_trgm 是 PG 生態內成本最低的選擇。

### I. Trigram 原理

pg_trgm 的核心是把一段文字拆成連續三個字元的片段（trigram），並用這些片段的集合來代表原始文字。

#### a. 什麼是 Trigram

Trigram（三元組）是一個長度固定為 3 的滑動窗口。給定任意字串，從頭到尾每次取 3 個字元，直到超出字串長度。

```sql
-- 安裝 extension（需要一次性的 superuser / 資料庫 owner 權限）
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 查看 "hello" 被拆成哪些 trigram
SELECT show_trgm('hello');
```

輸出：

```text
{"  h"," he","ell","hel","llo","lo "}
```

解讀規則：
| 規則 | 說明 | 範例 |
|------|------|------|
| 前綴補空格 | 字串開頭補 2 個空白，保證開頭字元也能產生 trigram | `"  h"` |
| 滑動取 3 字元 | 從補空格後的位置開始，每次取 3 個連續字元 | `" he"`, `"hel"`, `"ell"`, `"llo"` |
| 後綴補空格 | 字串結尾補 2 個空白，保證結尾字元也能產生 trigram | `"lo "` |

> **補充（Senior Dev）：** 前後補空格是關鍵設計。沒有補空格的話，`"hello"` 只會有 `"hel"`, `"ell"`, `"llo"` 三個 trigram——這樣你搜 `"he"` 前綴或 `"lo"` 後綴時就無法匹配。補空格保證了 prefix/suffix search 能力。

#### b. 中文字串的 Trigram 行為

中文字元在 PostgreSQL 中是 2~4 bytes（UTF-8），`pg_trgm` 以字元為單位處理，不是 byte。這表示一個中文字就是一個 trigram 中的一個單元。

```sql
SELECT show_trgm('你好世界');
```

輸出（假設 PG15+、UTF8 編碼）：

```text
{"  你"," 你好","你好世","好世界","世界 ","界  "}
```

注意中文字串的 trigram 數量通常比英文字串少，因為每個中文字本身就是一個有意義的語素單元。這是 CJK 語系在使用 `pg_trgm` 時需要特別注意的地方。

```mermaid
flowchart LR
    subgraph 輸入字串
        S["你好世界"]
    end

    subgraph 前處理
        PAD["前補 2 個空白<br/>後補 2 個空白<br/>→ '  你好世界  '"]
    end

    subgraph 滑動窗口
        T1["  你"]
        T2[" 你好"]
        T3["你好世"]
        T4["好世界"]
        T5["世界 "]
        T6["界  "]
    end

    subgraph 輸出
        OUT["trigram 陣列<br/>{"  你"," 你好",...,"界  "}"]
    end

    S --> PAD --> T1 --> OUT
    PAD --> T2 --> OUT
    PAD --> T3 --> OUT
    PAD --> T4 --> OUT
    PAD --> T5 --> OUT
    PAD --> T6 --> OUT

    style S fill:#e3f2fd,stroke:#1565c0
    style PAD fill:#fff3e0,stroke:#e65100
    style OUT fill:#e8f5e9,stroke:#2e7d32
```

#### c. Similarity 計算公式

`similarity(A, B)` 的核心公式：

```
similarity = |trigram(A) ∩ trigram(B)| / |trigram(A) ∪ trigram(B)|
```

也就是：兩個字串共有的 trigram 數量 ÷ 兩個字串所有不重複 trigram 的總數。結果落在 0.0 ~ 1.0 之間。

```sql
-- 兩字串共用 {"  h"," he","hel"}，所以 similarity = 3 / 10 = 0.3
SELECT
  similarity('hello', 'help')   AS sim_value,
  show_trgm('hello')            AS hello_trigrams,
  show_trgm('help')             AS help_trigrams;
```

手算驗證：
- `hello` trigrams：`{"  h"," he","ell","hel","llo","lo "}`（6 個）
- `help` trigrams：`{"  h"," he","elp","hel","lp ","p  "}`（6 個）
- 交集（共同 trigram）：`{"  h"," he","hel"}`（3 個）
- 聯集（全部唯一 trigram）：6 + 6 − 3 = 9 個
- similarity = 3 / 9 ≈ 0.333...

實際上 PG 回傳的是近似值，因為 trigram 集合會被正規化處理。

> **初學者導讀：** `similarity` 不是編輯距離（Levenshtein），也不是語意理解。它只計算「有多少連續三元組是相同的」。對英文來說效果很好（因為英文單字本身就是字母序列），對中文來說效果較弱（因為中文語素獨立，trigram 覆蓋範圍窄）。

### II. 核心函數

pg_trgm 提供以下核心函數，按實用頻率排序：

#### a. similarity(text, text) → real

最基本的 trigram 相似度計算，範圍 0.0 ~ 1.0。

```sql
-- 完全匹配 = 1.0
SELECT similarity('postgresql', 'postgresql');  -- 1.0

-- 完全無關 = 0.0
SELECT similarity('abc', 'xyz');                 -- 0.0

-- 部份匹配
SELECT similarity('postgresql', 'postgres');     -- ~0.7857
SELECT similarity('台北市', '台北車站');          -- ~0.25
```

#### b. show_trgm(text) → trigram[]

回傳文字被拆解後的 trigram 陣列（`text[]`），用於除錯和理解 pg_trgm 的行為。

```sql
SELECT show_trgm('hello  world');
-- 注意：連續空白會被壓縮，觸發 normalize 行為
-- {"  h"," he","hel","ell","llo","lo ","o  ","  w"," wo","wor","orl","rld","ld "}
```

#### c. show_limit() → real

回傳當前 session 的 `pg_trgm.similarity_threshold` 值（預設 0.3）。

```sql
SELECT show_limit();  -- 0.3
```

#### d. word_similarity(text, text) → real（PG 9.6+）

與 `similarity` 不同，`word_similarity(haystack, needle)` 是把 `needle` 當作一個「完整單詞」去和 `haystack` 中最相似的單詞片段做比對。這對多詞短語查詢效果更好。

```sql
-- similarity 會把整個句子和查詢做比較
SELECT similarity('hello world', 'word');       -- 0.0（幾乎沒有共同 trigram）

-- word_similarity 會在句子中找最像 'word' 的片段
SELECT word_similarity('hello world', 'word');  -- 0.5（找到了 'wor' + 'ord' + 'rld'）
```

#### e. strict_word_similarity(text, text) → real（PG 11+）

與 `word_similarity` 類似，但要求 `needle` 必須精確匹配到 `haystack` 中的單詞邊界（word boundary）。考慮到 PG 對 word boundary 的定義是基於 `\w`（字母/數字/底線）與非 `\w` 之間的過渡，這在英文中很實用，在 CJK 語系中效果有限。

```sql
-- strict_word_similarity 要求邊界精確對齊
SELECT strict_word_similarity('hello world', 'world');   -- ~0.8333
SELECT strict_word_similarity('hello world', 'wor');     -- 較低，因為 'wor' 不是完整單詞
```

#### f. set_limit(real) → real

動態調整當前 session 的 `pg_trgm.similarity_threshold`。同時回傳舊值。

```sql
-- 降低門檻以獲得更多結果
SELECT set_limit(0.1);  -- 回傳舊值 0.3

-- 恢復預設
SELECT set_limit(0.3);
```

#### g. 核心函數對照表

| 函數 | PG 版本 | 用途 | 範例輸入 | 範例結果 | 適用場景 |
|------|---------|------|----------|----------|----------|
| `similarity(s1, s2)` | 8.4+ | 兩個字串的 trigram 重疊比例 | `('hello','help')` | 0.333 | 短字串模糊匹配（暱稱、地名） |
| `show_trgm(s)` | 8.4+ | 查看 trigram 拆解結果 | `('hello')` | `{"  h"," he",...}` | 除錯、了解匹配行為 |
| `show_limit()` | 8.4+ | 查詢當前 similarity threshold | `()` | `0.3` | 確認當前設定 |
| `set_limit(r)` | 8.4+ | 設定 similarity threshold | `(0.1)` | 回傳舊值 | 動態調整查詢靈敏度 |
| `word_similarity(s1, s2)` | 9.6+ | 在 s1 中找最像 s2 的單詞片段 | `('hello world','word')` | ~0.5 | 多詞文本中的關鍵詞匹配 |
| `strict_word_similarity(s1, s2)` | 11+ | 同上，但要求精確詞邊界 | `('hello world','world')` | ~0.83 | 嚴格關鍵詞比對（英文效果好） |

> **補充（Senior Dev）：** `similarity` 是對稱的（`A` vs `B` 與 `B` vs `A` 結果相同），但 `word_similarity` 和 `strict_word_similarity` 是**不對稱**的——第一個參數是 haystack（被搜尋的大文本），第二個是 needle（查詢關鍵詞）。別把參數順序搞反，否則結果會意外。

### III. GIN / GiST 索引與 LIKE 加速

pg_trgm 最強大的功能之一是可以為 `LIKE`、`ILIKE` 和正則表達式建立索引——這是原生 B-tree 索引做不到的。

#### a. GIN 索引：加速 LIKE / ILIKE / ~ / ~*

```sql
-- 建立 GIN trigram 索引
CREATE INDEX idx_docs_content_trgm ON docs USING GIN (content gin_trgm_ops);
```

有了這個索引後，以下查詢都可以使用索引掃描：

```sql
-- 前綴匹配（B-tree 也可以，但 GIN 也行）
SELECT * FROM docs WHERE content LIKE 'hello%';

-- 後綴匹配（B-tree 無法加速）
SELECT * FROM docs WHERE content LIKE '%world';

-- 中間匹配（B-tree 無法加速）
SELECT * FROM docs WHERE content LIKE '%keyword%';

-- 大小寫不敏感
SELECT * FROM docs WHERE content ILIKE '%keyword%';

-- 正則表達式
SELECT * FROM docs WHERE content ~ 'foo.*bar';
SELECT * FROM docs WHERE content ~* 'FOO.*BAR';
```

#### b. GIN Trigram 索引查詢流程

當你執行 `WHERE content LIKE '%abc%'` 時，PG 的查詢流程如下：

```mermaid
flowchart TD
    QUERY["查詢: LIKE '%abc%'"]
    ANALYZE["pg_trgm 解析 LIKE pattern<br/>提取字面片段: abc"]
    EXTRACT["對 'abc' 做 trigram 拆解<br/>{"  a"," ab","abc","bc "}"]
    GIN_SEARCH["GIN 索引掃描<br/>查找包含這些 trigram 的 tuple"]
    BITMAP["建立 Bitmap<br/>AND/OR 組合各 trigram 匹配結果"]
    RECHECK["Recheck 階段<br/>對 bitmap 中的 tuple 做精確 LIKE 比對<br/>（排除 false positive）"]
    RESULT[回傳結果]

    QUERY --> ANALYZE --> EXTRACT --> GIN_SEARCH --> BITMAP --> RECHECK --> RESULT

    style QUERY fill:#e3f2fd,stroke:#1565c0
    style RECHECK fill:#fff3e0,stroke:#e65100
    style RESULT fill:#e8f5e9,stroke:#2e7d32
```

> **初學者導讀：** GIN trigram 索引的關鍵在於「索引不分前後綴」。因為 trigram 把字串拆成所有可能的 3 字元片段，所以 `%abc%` 這種傳統上最難加速的中間匹配查詢，反而成了 GIN 的主場。

#### c. GiST 索引：加速相似度排序

對於 `ORDER BY similarity()` 或使用 `<->` 運算符的查詢，需要 GiST 索引：

```sql
-- 建立 GiST trigram 索引（注意是 gist_trgm_ops）
CREATE INDEX idx_docs_content_gist ON docs USING GIST (content gist_trgm_ops);
```

GiST 支持 KNN（K-Nearest Neighbor）索引掃描，讓相似度排序不需要全表掃描：

```sql
-- 使用 GiST KNN 快速找出最相似的 10 筆
SELECT id, content, similarity(content, 'search keyword') AS sim
FROM docs
ORDER BY content <-> 'search keyword'
LIMIT 10;
```

#### d. GIN vs GiST 對照表

| 特性 | GIN (`gin_trgm_ops`) | GiST (`gist_trgm_ops`) |
|------|---------------------|------------------------|
| **支援查詢類型** | `LIKE`, `ILIKE`, `~`, `~*`, `%`（threshold check） | `LIKE`, `ILIKE`, `~`, `~*`, `%`, `<->`（KNN）、`ORDER BY similarity` |
| **建索引速度** | 慢（需要建立完整的 posting list / tree） | 較快（R-tree 結構，寬度優先） |
| **查詢速度** | 快（posting list 交集效率高，尤其是高選擇性查詢） | 較慢（需要 traversal tree 比較） |
| **索引大小** | 較大（通常為原始資料的 1.5~3 倍） | 較小（約為原始資料的 1~2 倍） |
| **更新開銷** | 高（posting list 重組成本高） | 中等（R-tree split/merge 成本較穩定） |
| **KNN 排序** | 不支援 | 支援 |
| **寫入密集場景** | 不推薦 | 可以接受 |
| **最佳場景** | 讀多寫少、LIKE 為主的靜態查詢 | 需要排序、寫入頻繁的場景 |

```sql
-- 兩種索引可以同時存在，PG 會自動選擇較優的
CREATE INDEX idx_col_gin  ON tbl USING GIN  (col gin_trgm_ops);
CREATE INDEX idx_col_gist ON tbl USING GIST (col gist_trgm_ops);
```

#### e. Operator Class 支援矩陣

| 運算符 | 符號 | 含義 | GIN 支援 | GiST 支援 | 範例 |
|--------|------|------|----------|-----------|------|
| LIKE | `LIKE` | 前/中/後綴模糊匹配 | ✅ | ✅ | `col LIKE '%abc%'` |
| ILIKE | `ILIKE` | 大小寫不敏感的 LIKE | ✅ | ✅ | `col ILIKE '%abc%'` |
| Regex | `~` | 正則表達式匹配 | ✅ | ✅ | `col ~ '(foo|bar)'` |
| Regex (CI) | `~*` | 大小寫不敏感正則 | ✅ | ✅ | `col ~* 'foo'` |
| Similarity | `%` | threshold 門檻檢查 | ✅ | ✅ | `col % 'keyword'` |
| Distance | `<->` | trigram 距離 | ❌ | ✅ | `ORDER BY col <-> 'kw'` |

> **補充（Senior Dev）：** 很多人不知道 GIN 其實也支援 `%` 運算符（配合 `similarity_threshold`）。如果你不需要 KNN 排序，用 GIN 做 `%` 門檻過濾再加上 `ORDER BY similarity()` 手動排序通常也足夠快——前提是 `%` 篩出來的行數不多。GiST 的 KNN 優勢在於「不需要先過濾全部」就能排序，差距在結果集超過幾百行時才會明顯顯現。

### IV. 相似度查詢與排序

#### a. 基礎相似度查詢

```sql
-- 找與 'search keyword' 最相似的 10 筆記錄
SELECT
  id,
  content,
  similarity(content, 'search keyword') AS sim
FROM docs
WHERE content % 'search keyword'   -- 只保留 similarity >= threshold 的列
ORDER BY sim DESC
LIMIT 10;
```

`%` 運算符的等價寫法：

```sql
-- 這兩者等價
WHERE content % 'keyword'
WHERE similarity(content, 'keyword') >= show_limit()
```

#### b. KNN-GiST 快速排序

當資料量大且需要排序時，使用 `<->` operator 搭配 GiST 索引可以獲得指數級加速：

```sql
-- 使用 trigram distance operator（越低越相似）
SELECT id, content
FROM docs
ORDER BY content <-> 'search keyword'
LIMIT 10;
```

`<->` 的索引加速前提：
1. 必須使用 GiST（`gist_trgm_ops`），GIN 不支援
2. `LIMIT` 不可省略（否則退化成全表掃描後排序）
3. 查詢中不能有其他 `WHERE` 條件把結果集縮得太小（否則 PG optimizer 可能放棄 GiST scan）

```sql
-- 不推薦：LIMIT 太大可能導致 optimizer 選擇 seq scan
SELECT * FROM docs ORDER BY content <-> 'kw' LIMIT 99999;

-- 推薦：合理的小 LIMIT
SELECT * FROM docs ORDER BY content <-> 'kw' LIMIT 50;
```

#### c. 組合 LIKE + Similarity：精確度 + 召回率

生產環境常見模式：先用 `LIKE` 確保搜到「至少包含關鍵字」的結果，再用 `similarity` 排序。

```sql
-- 策略：LIKE 保證召回（只要有包含就納入），similarity 排序品質
SELECT
  id,
  title,
  similarity(title, 'PostgreSQL 效能') AS sim
FROM articles
WHERE title LIKE '%PostgreSQL%'            -- 確保至少包含關鍵詞
  AND similarity(title, 'PostgreSQL 效能') > 0.1  -- 過濾掉相似度太低的
ORDER BY sim DESC
LIMIT 20;
```

> **初學者導讀：** 不要只用 `similarity` 排序來取代 `LIKE`。`similarity` 可能把「完全包含關鍵字但順序不同」的結果排到很後面。如果你確定使用者輸入的字串應該**字面上出現**在目標中，`LIKE` 是必須的。

```mermaid
flowchart TD
    INPUT["使用者查詢詞<br/>例如: 'tea'"]
    SPLIT{搜尋策略<br/>選擇}
    EXACT["精確搜尋<br/>LIKE '%tea%'"]
    FUZZY["模糊搜尋<br/>similarity > 0.3"]
    COMBO["組合策略<br/>LIKE + similarity 排序"]

    EXACT --> IDX{"有 GIN<br/>trigram 索引?"}
    IDX -->|"✅ 有"| FAST["索引加速<br/>Bitmap Index Scan"]
    IDX -->|"❌ 無"| SLOW["全表掃描<br/>Seq Scan"]

    FUZZY --> GIST_IDX{"有 GiST<br/>trigram 索引?"}
    GIST_IDX -->|"✅ 有"| KNN["KNN-GiST 掃描"]
    GIST_IDX -->|"❌ 無"| FULLSCAN["全表掃描 + 排序"]

    COMBO --> BEST["最佳使用者體驗<br/>召回率高 + 排序合理"]

    style FAST fill:#e8f5e9,stroke:#2e7d32
    style KNN fill:#e8f5e9,stroke:#2e7d32
    style BEST fill:#e8f5e9,stroke:#2e7d32
    style SLOW fill:#ffebee,stroke:#c62828
    style FULLSCAN fill:#fff3e0,stroke:#e65100
```

### V. pg_trgm 參數調校

#### a. 三個核心 GUC 參數

| 參數 | 預設值 | 影響範圍 | 說明 |
|------|--------|----------|------|
| `pg_trgm.similarity_threshold` | `0.3` | `similarity()`, `%` 運算符 | 相似度門檻，低於此值的結果不會被 `%` 納入 |
| `pg_trgm.word_similarity_threshold` | `0.6` | `word_similarity()`, `<%` 運算符 | 單詞相似度門檻（PG 9.6+） |
| `pg_trgm.strict_word_similarity_threshold` | `0.6` | `strict_word_similarity()`, `<<%` 運算符 | 嚴格單詞相似度門檻（PG 11+） |

#### b. 調整策略

```sql
-- 查詢當前設定
SELECT name, setting FROM pg_settings WHERE name LIKE 'pg_trgm.%';
```

| 症狀 | 診斷 | 調校方向 | SQL |
|------|------|----------|-----|
| 搜不到任何結果 | threshold 太高 | 降低 threshold | `SET pg_trgm.similarity_threshold = 0.1;` |
| 結果太多、不相關 | threshold 太低 | 提高 threshold | `SET pg_trgm.similarity_threshold = 0.5;` |
| 查詢變慢 | threshold 太低導致大量 recheck | 提高 threshold 或加 LIMIT | `SET pg_trgm.similarity_threshold = 0.4;` |
| 短字串（1~3 字元）無結果 | 短字串 trigram 數量太少、交集不易 | 降低 threshold 至 0.05 或改用 LIKE | `SET pg_trgm.similarity_threshold = 0.05;` |

#### c. 設定層級

```sql
-- Session 級別（最常用，不影響其他連線）
SET pg_trgm.similarity_threshold = 0.1;

-- Transaction 級別
BEGIN;
SET LOCAL pg_trgm.similarity_threshold = 0.1;
SELECT * FROM docs WHERE content % 'short';
COMMIT;

-- 全域永久設定（需要 superuser / 修改 postgresql.conf）
-- postgresql.conf:
-- pg_trgm.similarity_threshold = 0.2

-- 或 Database 級別
ALTER DATABASE mydb SET pg_trgm.similarity_threshold = 0.2;

-- 或 Role 級別
ALTER ROLE myuser SET pg_trgm.similarity_threshold = 0.15;
```

> **補充（Senior Dev）：** `similarity_threshold` 實際上**不直接影響索引掃描**。即使 threshold 設為 0，GIN/GiST 索引照樣可以用。threshold 只影響 `%` 運算符的過濾——以及 optimizer 在做 index scan 之後要不要多做一步 filter。如果你打算自己用 `WHERE similarity() > X` 手動寫門檻，threshold 調多少並不重要。真正的效能瓶頸永遠在於「有多少 trigram 需要被 recheck」。

### VI. PG 14-17 演進

#### a. PG 14：CJK Multi-Byte 改善

PG 14 對 GIN trigram 索引的 multi-byte 字元處理做了優化。過去在處理 CJK（中日韓）字元時，trigram 拆解在內部使用 byte 級別比對，對 UTF-8 的多 byte 字元效率較差。PG 14 改進了 GIN 比對階段對 multi-byte 的處理，使得 CJK 語系在 `LIKE '%keyword%'` 查詢時減少不必要的 false positive recheck。

#### b. PG 15：平行掃描支援

PG 15 開始，GIN trigram 索引支援 parallel index scan。過去即使查詢本身計劃用 parallel scan，只要走了 GIN trigram 索引就會退化成 single-worker。PG 15 起可以在多個 worker 上同時掃描 GIN posting tree，對大資料量的相似度過濾查詢有顯著加速。

```sql
-- PG 15+ 可觀察 parallel worker 數量
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM large_docs WHERE content LIKE '%keyword%';
-- 注意 Workers Planned / Workers Launched 欄位
```

#### c. PG 16-17：ICU Collation 整合

PG 16 引入了對 ICU collation 更深入的支援，PG 17 在此基礎上優化了 `pg_trgm` 對非英語語系的處理。特別是 `ILIKE` 在 ICU collation 下的行為更加準確——過去 `ILIKE` 在某些 locale 下對重音符號的不區分大小寫處理不一致，17 版本修復了這點。

#### d. 版本功能總結表

| 版本 | 改進 | 對 CJK 影響 |
|------|------|-------------|
| PG 14 | GIN trigram multi-byte 處理優化 | 減少 CJK 查詢的 false positive recheck |
| PG 15 | GIN trigram 支援 parallel scan | 大表 LIKE 查詢可以用多核加速 |
| PG 16 | ICU collation 基礎整合 | ILIKE 對非 ASCII 字元行為更準確 |
| PG 17 | ICU 與 trigram 的 collation 對齊 | 非英語 locale 下 trigram 提取更一致 |

> **初學者導讀：** 如果你還在 PG 13 或更早，而資料庫中有大量中文全文檢索需求，升級到 PG 15+ 是一個值得考慮的選項——parallel GIN scan 對 CJK 大表的加速往往比你想像的顯著。

### VII. App Dev 最佳實踐

#### a. pg_trgm vs tsvector（全文檢索）決策樹

這是 App Developer 最常問的問題：「我該用 `pg_trgm` 還是 `tsvector`？」

```mermaid
flowchart TD
    START["需要文字搜尋功能"]
    Q1{"使用者輸入的是<br/>自然語言句子？"}
    Q2{"需要語言學處理？<br/>（分詞、詞幹、停用詞）"}
    Q3{"搜尋對象以<br/>CJK 語言為主？"}
    Q4{"需要模糊容錯？<br/>（拼字錯誤、近似匹配）"}
    Q5{"資料寫入頻繁？<br/>（OLTP 高並發）"}

    USE_TRGM["✅ 使用 pg_trgm<br/>+ GIN/GiST 索引"]
    USE_FTS["✅ 使用 tsvector<br/>+ GIN 全文索引"]
    USE_LIKE["✅ 普通 LIKE<br/>+ B-tree 索引<br/>（僅前綴匹配）"]
    USE_ES["⚠️ 考慮外部方案<br/>Elasticsearch / Meilisearch"]
    HYBRID["✅ 混合方案<br/>tsvector 搜主體 +<br/>pg_trgm 做 fallback"]

    START --> Q1
    Q1 -->|"是（如「如何安裝 PG」）"| Q2
    Q1 -->|"否（如暱稱、SKU、地址）"| Q3

    Q2 -->|"是"| USE_FTS
    Q2 -->|"否（只需 substring match）"| USE_TRGM

    Q3 -->|"是"| Q4
    Q3 -->|"否（英文為主）"| USE_FTS

    Q4 -->|"是（需要模糊）"| USE_TRGM
    Q4 -->|"否（精確匹配即可）"| USE_FTS

    Q5 -->|"是"| USE_LIKE
    Q5 -->|"否"| USE_TRGM

    style USE_TRGM fill:#e8f5e9,stroke:#2e7d32
    style USE_FTS fill:#e3f2fd,stroke:#1565c0
    style USE_LIKE fill:#fff3e0,stroke:#e65100
    style USE_ES fill:#fce4ec,stroke:#c62828
    style HYBRID fill:#f3e5f5,stroke:#7b1fa2
```

#### b. 索引開銷評估

```sql
-- 查看 GIN trigram 索引的實際大小
SELECT
  pg_size_pretty(pg_relation_size('idx_docs_content_trgm')) AS index_size,
  pg_size_pretty(pg_total_relation_size('idx_docs_content_trgm')) AS total_size;
```

一般經驗法則：
- 英文文本：GIN trigram 索引約為原始資料的 1.5~2.5 倍
- 中文文本：約為原始資料的 1.5~3 倍（取決於字元多樣性）
- GiST 約比 GIN 小 20~30%，但查詢速度也較慢

#### c. 搭配 unaccent 做多語言搜尋

```sql
-- 先安裝 unaccent extension
CREATE EXTENSION IF NOT EXISTS unaccent;

-- 建立 immutable wrapper（必要，因為 unaccent 預設是 stable）
CREATE OR REPLACE FUNCTION f_unaccent(text)
RETURNS text
LANGUAGE sql IMMUTABLE PARALLEL SAFE
AS $$ SELECT unaccent('unaccent', $1) $$;

-- 用 functional index 組合 unaccent + trigram
CREATE INDEX idx_docs_title_unaccent_trgm
  ON docs USING GIN (f_unaccent(title) gin_trgm_ops);

-- 查詢時也要用相同的 function
SELECT * FROM docs
WHERE f_unaccent(title) ILIKE f_unaccent('%café%');
```

> **初學者導讀：** `unaccent` 移除重音符號（é → e, ü → u），對歐洲語言使用者搜尋體驗至關重要。但是請注意：`unaccent` 預設是 `STABLE`（不是 `IMMUTABLE`），所以不能直接用於索引。你必須包裝一個 `IMMUTABLE` function——如果哪天改了 `unaccent` 的 rules file，需要重建索引。

#### d. pg_bigm：CJK 的替代方案

`pg_trgm` 使用 3-gram，這對 CJK 語言來說窗口偏大——一個中文詞通常只有 1~3 個字元，3-gram 能產生的獨特 trigram 非常少。

`pg_bigm` 是一個第三方 extension，使用 **2-gram（bigram）**，對 CJK 更友好：

| 特性 | pg_trgm | pg_bigm |
|------|---------|---------|
| N-gram 大小 | 3 | 2 |
| 內建 | ✅ contrib 內建 | ❌ 需要手動編譯安裝 |
| CJK 匹配精度 | 較低（3-gram 太粗） | 較高（2-gram 更細） |
| 索引大小 | 較大 | 更大（bigram 數量約為 trigram 的 1.5 倍） |
| LIKE 加速 | ✅ | ✅ |
| 相似度查詢 | ✅ | ✅ |
| PG 版本跟進 | 與 core 同步更新 | 需等社群維護 |

```sql
-- pg_bigm 使用範例（若已安裝）
-- CREATE EXTENSION pg_bigm;
-- CREATE INDEX idx_col_bigm ON tbl USING GIN (col gin_bigm_ops);
-- SELECT similarity(col, '關鍵字') FROM tbl WHERE col LIKE '%關鍵字%';
```

#### e. .NET / Npgsql 整合注意事項

```csharp
// 正確：使用 TransactionScope 或明確的 using 管理連線
// 在建立 trigram 索引時，不要用 Dapper 動態 SQL 拼接
using var conn = new NpgsqlConnection(connectionString);
conn.Open();

// 設定 connection 級別的 pg_trgm 參數
using var cmd = new NpgsqlCommand(
    "SET pg_trgm.similarity_threshold = 0.15", conn);
cmd.ExecuteNonQuery();

// 使用 Dapper 參數化查詢（對應 pg_stat_statements 正規化）
var results = await conn.QueryAsync<Doc>(
    @"SELECT id, title, similarity(title, @keyword) AS sim
      FROM docs
      WHERE title % @keyword
      ORDER BY sim DESC
      LIMIT @limit",
    new { keyword = searchTerm, limit = 20 });
```

> **補充（Senior Dev）：** 在 .NET 應用中，`Application Name` 參數在連線字串中設好，對事後排查非常有用。當你在 `pg_stat_activity` 看到一堆 `WHERE title % 'xxx'` 的慢查詢時，`application_name` 可以讓你立刻知道是哪個 service 發出的。Npgsql 連線字串：`Host=...;Database=...;Application Name=MyService`。

- 使用 `conn.ProcessID`（Npgsql 6.0+）在 log 中記錄 `backend_pid`，方便對照 `pg_stat_activity` 和 `pg_stat_statements`
- Connection Pool 配合 `idle_in_transaction_session_timeout`：trigram 查詢通常很快，但如果忘記 commit / rollback，會導致 connection 被掛住
- 在 application startup 階段驗證索引存在（透過 `pg_indexes` 查詢），否則在高流量下觸發 Seq Scan 會直接打爆資料庫


## 3. pg_prewarm — 緩存預熱

> **章節導讀**：PostgreSQL 重啟後，`shared_buffers` 和 OS page cache 全部清空，所有查詢回到「冷啟動」狀態——首次查詢需從磁盤讀取資料，速度可能比熱機狀態慢 100 倍以上。`pg_prewarm` 讓你能在重啟後、查詢到達前，預先將關鍵資料載入緩存，消除冷啟動的效能懲罰。

---

### I. 核心價值 — 為什麼需要緩存預熱

#### a. 冷啟動問題的本質

```mermaid
timeline
    title PostgreSQL 重啟後的緩存狀態演變
    section 正常運行
      熱機狀態 : shared_buffers 滿載熱資料<br/>OS page cache 也有副本 : 查詢命中率 > 99%<br/>平均查詢耗時 < 1ms
    section 計劃重啟
      排程維護窗口 : pg_ctl restart : shared_buffers 清空<br/>OS page cache 可能部分保留
    section 重啟後
      冷啟動狀態 : 全部查詢從磁盤讀取 : 首次查詢耗時暴增 10-100x<br/>CPU iowait 飆升
    section 自然回暖
      緩慢恢復 : 查詢自然觸發 buffer 填充 : 熱資料需數分鐘至數小時<br/>才能回到穩定狀態
    section pg_prewarm 介入
      即時熱機 : pg_prewarm 主動載入關鍵表 : 首批查詢即享熱機效能
```

在 PostgreSQL 的架構中，shared_buffers 是資料庫執行個體（instance）層級的共享記憶體區域。當 PostgreSQL 重啟時：

1. shared_buffers 作為共享記憶體區域被重新分配，所有之前的 buffer 內容全部消失
2. OS page cache 視重啟方式而定：`pg_ctl restart` 通常保留 OS cache，但 `service postgresql restart` 或實體機器重啟則會清空
3. 任何查詢首次訪問某個 page 時，必須從磁盤執行物理讀取（`read()` 系統調用）

```mermaid
flowchart TD
    A["PG 重啟"] --> B["shared_buffers 為空"]
    B --> C["首次查詢到達"]

    C --> D{"資料是否在<br/>OS page cache?"}
    D -->|"是（幸運）"| E["從 OS cache 複製到<br/>shared_buffers（快速）"]
    D -->|"否（常見）"| F["從磁盤讀取<br/>物理 I/O（慢 100x+）"]

    E --> G["查詢完成"]
    F --> G

    G --> H["該 page 留在 shared_buffers"]
    H --> I["後續相同查詢變快"]

    C2["大量並行查詢同時到達"] --> D2{"所有查詢都需<br/>物理 I/O？"}
    D2 -->|"是"| F2["磁盤 IOPS 飽和<br/>所有查詢排隊等待"]
    F2 --> F3["CPU iowait 100%<br/>應用層超時"]

    style F fill:#e74c3c,color:white
    style F2 fill:#e74c3c,color:white
    style F3 fill:#e74c3c,color:white
    style E fill:#2ecc71,color:white
```

> **初學者導讀**：你可以把 shared_buffers 想像成「PG 專用的 RAM 快取區」。平常運行時，常用的資料頁面已經在 RAM 裡，查詢幾乎不需要碰磁盤。但 PG 一重啟，這塊 RAM 就被清空了，就像電腦重新開機後第一次打開大型軟體特別慢一樣。pg_prewarm 的作用就是在正式開工前，先把重要的資料「搬」回 RAM。

#### b. 典型生產場景

| 場景 | 說明 | 無 pg_prewarm 的後果 |
|------|------|---------------------|
| 排程 PG 重啟維護 | 每月 patch / config 變更後重啟 | 重啟後 5-15 分鐘內效能暴跌，用戶投訴 |
| 硬體故障切換 | failover 到 standby（promote 後） | 新 primary 為冷機狀態，瞬間扛不住流量 |
| 容器化部署 | K8s pod 重啟，PV 仍保留但 memory 清空 | 新 pod 需要數分鐘才能恢復到穩定吞吐量 |
| 開發環境重建 | CI/CD 中頻繁重建 PG 容器 | 每次測試運行都從冷機開始，測試結果不穩定 |

---

### II. pg_prewarm 函數詳解

#### a. 函數簽名與參數

```sql
pg_prewarm(
    regclass,          -- 目標表/索引/物化視圖
    mode   text DEFAULT 'buffer',   -- 預熱模式
    fork   text DEFAULT 'main',     -- 儲存分支 (fork)
    first_block int8 DEFAULT NULL,  -- 起始 block（NULL = 從頭）
    last_block  int8 DEFAULT NULL   -- 結束 block（NULL = 到最後）
) RETURNS int8  -- 回傳成功載入的 block 數量
```

#### b. mode 參數詳解

這是 pg_prewarm 最重要的參數，決定資料載入到哪一層緩存：

| mode | 行為 | 同步/非同步 | 載入目標 | 適用場景 |
|------|------|------------|---------|---------|
| `buffer` | 將 block 讀入 shared_buffers | **同步**，阻塞直到完成 | shared_buffers | 中小型表（< 1GB），需要保證 PG 直接命中 |
| `read` | 將 block 讀入 OS page cache | **同步**，阻塞直到完成 | OS page cache | 大表（> 1GB），允許 OS 層級緩存管理 |
| `prefetch` | 向 OS 發出非同步預讀請求 | **非同步**，立即返回 | OS page cache（非同步） | 超大表（> 10GB），不想阻塞其他操作 |

```mermaid
flowchart TD
    subgraph "mode = 'buffer'"
        B1["pg_prewarm('table', 'buffer')"] --> B2["同步讀取每個 block"]
        B2 --> B3["載入 shared_buffers<br/>（PG 內部緩存）"]
        B3 --> B4["函數返回（阻塞期間無法做其他事）"]
    end

    subgraph "mode = 'read'"
        R1["pg_prewarm('table', 'read')"] --> R2["同步讀取每個 block"]
        R2 --> R3["載入 OS page cache<br/>（作業系統層級）"]
        R3 --> R4["函數返回（阻塞期間無法做其他事）"]
    end

    subgraph "mode = 'prefetch'"
        P1["pg_prewarm('table', 'prefetch')"] --> P2["向 OS 發出非同步預讀請求"]
        P2 --> P3["函數立即返回 ✅"]
        P3 --> P4["OS 在背景逐步載入 page cache"]
    end

    style B4 fill:#ffd43b,color:black
    style R4 fill:#ffd43b,color:black
    style P3 fill:#2ecc71,color:white
```

> **補充（Senior Dev）**：三種 mode 的本質差異在於 Linux 系統調用：
> - `buffer`：PostgreSQL 使用 `pread()` 讀取 block，然後放入 shared_buffers 的 buffer pool
> - `read`：PostgreSQL 使用 `pread()` 讀取 block，依賴 OS kernel 將資料緩存在 page cache——PG 本身不持有這份記憶體
> - `prefetch`：PostgreSQL 使用 `posix_fadvise(POSIX_FADV_WILLNEED)` 告知 kernel「我即將需要這些 block」，kernel 排程非同步預讀。PG 立即返回，不等待 I/O 完成
>
> `prefetch` 模式在 PG 10 中引入，對於需要在重啟後快速恢復服務的場景特別有價值：你可以先對所有熱表發起 `prefetch`，讓 OS 在背景載入，同時 PG 已經可以接受連線。

#### c. fork 參數詳解

PostgreSQL 每個 relation（表/索引）在磁盤上可能有多個物理檔案，稱為 fork：

| fork | 說明 | 檔案後綴 | 何時需要 prewarm |
|------|------|---------|-----------------|
| `main` | 主資料 fork（實際行資料 / 索引內容） | 無後綴或 `_vm`, `_fsm` | **幾乎總是** |
| `fsm` | Free Space Map（記錄每個 page 的可用空間） | `_fsm` | 寫入密集型表（避免首次 INSERT 時載入 FSM） |
| `vm` | Visibility Map（記錄 page 中是否全是可見 tuple） | `_vm` | VACUUM 密集型表（加速 index-only scan） |

```sql
-- 只預熱主資料（最常見）
SELECT pg_prewarm('large_orders', fork => 'main');

-- 同時預熱 FSM 和 VM（寫入密集 + VACUUM 頻繁的表）
SELECT pg_prewarm('large_orders', fork => 'main');
SELECT pg_prewarm('large_orders', fork => 'fsm');
SELECT pg_prewarm('large_orders', fork => 'vm');
```

> **初學者導讀**：對大多數場景，你只需要關心 `main` fork。FSM 和 VM 檔案通常很小（幾 MB），即使冷啟動也不需要刻意預熱。但如果你有一張超高寫入頻率的表（每秒數千行 INSERT），預熱 FSM 可以避免首次 INSERT 時的額外 I/O。

#### d. first_block / last_block 參數

對超大表進行分段預熱，或用於只預熱表的前 N 個 block（熱資料通常集中在表的前段）：

```sql
-- 只預熱前 10000 個 block（假設熱資料集中在表前端）
SELECT pg_prewarm('large_orders', first_block => 0, last_block => 10000);

-- 分段預熱（結合 pg_cron 定時執行）
SELECT pg_prewarm('large_orders', first_block => 0,      last_block => 50000);
SELECT pg_prewarm('large_orders', first_block => 50001,  last_block => 100000);
```

#### e. 實戰 SQL 示例

```sql
-- 1. 預熱整張表（最常用，載入 shared_buffers）
SELECT pg_prewarm('public.orders');
-- 回傳: 預熱的 block 數（例如 150000）

-- 2. 預熱索引（索引也是 relation，同樣可以使用 pg_prewarm）
SELECT pg_prewarm('public.idx_orders_created_at');

-- 3. 非同步預熱大表（立即返回，OS 背景載入）
SELECT pg_prewarm('public.audit_log', mode => 'prefetch');

-- 4. 預熱至 OS page cache（同步，但使用 OS 記憶體管理）
SELECT pg_prewarm('public.products', mode => 'read');

-- 5. 一次預熱多個關鍵物件
DO $$
DECLARE
    rel text;
    n   bigint;
BEGIN
    FOR rel IN
        SELECT unnest(ARRAY[
            'public.orders',
            'public.order_items',
            'public.users',
            'public.idx_orders_created_at',
            'public.idx_orders_user_id',
            'public.idx_order_items_order_id'
        ])
    LOOP
        n := pg_prewarm(rel, mode => 'buffer');
        RAISE NOTICE 'pg_prewarm: % → % blocks', rel, n;
    END LOOP;
END $$;
```

#### f. 驗證預熱結果

使用 `pg_buffercache` 擴展確認資料是否已載入 shared_buffers：

```sql
-- 啟用 pg_buffercache（需先安裝）
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- 檢查某個 relation 在 shared_buffers 中佔用多少 block
SELECT
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(count(*) * current_setting('block_size')::int) AS buffers_size,
    pg_size_pretty(pg_relation_size(c.oid)) AS total_size,
    round(100.0 * count(*) * current_setting('block_size')::int
          / pg_relation_size(c.oid), 2) AS pct_in_buffer
FROM pg_class c
JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
JOIN pg_database d ON b.reldatabase = d.oid
    AND d.datname = current_database()
WHERE c.relname IN ('orders', 'order_items', 'users')
GROUP BY c.relname, c.oid
ORDER BY count(*) DESC;
```

預期輸出：

| relname | buffers | buffers_size | total_size | pct_in_buffer |
|---------|---------|-------------|------------|---------------|
| orders | 150000 | 1172 MB | 1172 MB | 100.00 |
| order_items | 80000 | 625 MB | 625 MB | 100.00 |
| users | 5000 | 39 MB | 39 MB | 100.00 |

> **初學者導讀**：`pg_buffercache` 就像一個透視鏡，讓你能看到 shared_buffers 裡面實際裝了哪些表和索引的資料。`pct_in_buffer = 100%` 表示整張表都已經在記憶體中，查詢這張表完全不需要磁盤 I/O。

---

### III. autoprewarm — 自動緩存恢復（PG 11+）

#### a. 運作原理

PG 11 之後，pg_prewarm 擴展內建了一個 Background Worker：`autoprewarm`。它解決的核心問題是：**PG 重啟後自動恢復 shared_buffers 的狀態，無需人為介入**。

```mermaid
sequenceDiagram
    participant PG as PG Instance
    participant BGW as autoprewarm BGW
    participant Disk as PGDATA 磁盤
    participant SB as shared_buffers

    Note over PG: PG 正常運行中
    loop 每 pg_prewarm.autoprewarm_interval 秒
        BGW->>SB: 掃描 shared_buffers 內容
        BGW->>Disk: 寫入 dump file<br/>(pg_autoprewarm.blocks)
    end

    Note over PG: PG 收到關機信號 (SIGTERM)
    PG->>BGW: 通知識別
    BGW->>SB: 最後一次完整掃描
    BGW->>Disk: 寫入最終 dump file
    Note over PG: PG 停止

    Note over PG: PG 重新啟動
    PG->>BGW: 啟動 autoprewarm BGW
    BGW->>Disk: 讀取 pg_autoprewarm.blocks
    loop 依 dump file 記錄順序載入
        BGW->>Disk: pread() 讀取 block
        BGW->>SB: 放入 shared_buffers
    end
    Note over SB: shared_buffers 恢復至<br/>關機前近似狀態
```

> **初學者導讀**：autoprewarm 就像一個「記憶恢復器」。它在 PG 關機前把「哪些資料在 shared_buffers 裡」記錄下來，重啟後依照記錄把相同的資料再載入回去。這樣重啟後的 PG 幾乎能立刻回到熱機狀態，不需要等查詢慢慢填充緩存。

#### b. 配置步驟

**步驟 1**：載入擴展庫（`postgresql.conf`）

```ini
# 必須在 shared_preload_libraries 中載入 pg_prewarm
# 才會啟用 autoprewarm Background Worker
shared_preload_libraries = 'pg_prewarm'

# ===== autoprewarm 相關配置 =====

# 啟用 autoprewarm BGW（預設 on）
pg_prewarm.autoprewarm = on

# dump 間隔：每 N 秒將 shared_buffers 當前內容寫入 dump file
# 預設 300 秒（5 分鐘），可視 shared_buffers 大小調整
pg_prewarm.autoprewarm_interval = 300
```

**步驟 2**：重啟 PostgreSQL

```bash
# PG 16 Debian/Ubuntu
sudo pg_ctlcluster 16 main restart

# 或使用 systemctl
sudo systemctl restart postgresql@16-main
```

**步驟 3**：在目標資料庫中建立擴展並驗證

```sql
-- 建立擴展
CREATE EXTENSION pg_prewarm;

-- 驗證 autoprewarm BGW 是否啟動
SELECT pid, backend_type, state, query
FROM pg_stat_activity
WHERE backend_type = 'autoprewarm leader';

-- 確認 dump file 已產生
-- (需在 shell 執行)
```

```bash
# 查看 dump file
ls -lh /var/lib/postgresql/16/main/pg_autoprewarm.blocks

# 查看 dump file 內容（前 10 行）
head -10 /var/lib/postgresql/16/main/pg_autoprewarm.blocks
```

dump file 格式（每行一個 block）：

```text
# database_oid, tablespace_oid, relfilenode, fork_number, block_number
16384 1663 16403 0 0
16384 1663 16403 0 1
16384 1663 16403 0 2
...
```

**步驟 4**：手動觸發 dump（用於測試）

```sql
-- 手動觸發 autoprewarm dump（正常由 BGW 定時執行）
SELECT autoprewarm_dump_now();
```

**步驟 5**：查看 autoprewarm 狀態

```sql
-- 查看最近一次 dump / load 的統計
SELECT pg_prewarm.autoprewarm;
-- (此為 GUC 參數，僅反映是否啟用)

-- 查看 dump file 中的 block 數量
-- 需要透過 pg_stat_file() 或 shell 查看
```

#### c. 配置參數完整說明

| 參數 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `pg_prewarm.autoprewarm` | bool | `on` | 是否啟用 autoprewarm BGW。設為 `off` 仍可手動使用 `pg_prewarm()` |
| `pg_prewarm.autoprewarm_interval` | int | `300` | dump 間隔（秒）。值越小，dump file 越新，但 BGW 掃描 shared_buffers 也有輕微開銷 |

> **補充（Senior Dev）**：autoprewarm 的 dump 間隔代表著一個 trade-off：
> - `autoprewarm_interval = 60`：dump file 很新，重啟後恢復狀態精準，但每分鐘掃描 shared_buffers 有 CPU 開銷（尤其在 shared_buffers > 64GB 時）
> - `autoprewarm_interval = 3600`：開銷小，但若 PG 在 dump 後 59 分鐘 crash，這 59 分鐘內的 buffer 變化全部丟失
>
> PG 17 引入了更智能的 dump 策略：autoprewarm BGW 現在使用 `pg_buffercache` 的內部介面，而非逐 block 查詢 system catalog，大幅降低了大 shared_buffers（> 100GB）環境下的 dump 開銷。

#### d. autoprewarm 的局限性

| 限制 | 說明 | 影響 |
|------|------|------|
| 僅記錄當前 buffer 狀態 | dump 時 shared_buffers 中的內容 = dump file 內容。dump 之後新載入的 block 不會被記錄（直到下次 dump） | crash 重啟時可能丟失最近加入的熱資料 |
| 不記錄 OS page cache | autoprewarm 只處理 shared_buffers，不涉及 OS page cache | 重啟後仍需依賴查詢自然填充 OS cache |
| 表結構變更 | 若重啟期間有 DDL（如 TRUNCATE、DROP TABLE），dump file 中的 relfilenode 可能已失效 | autoprewarm 會跳過無效的 block，不會報錯 |
| 僅限單一 database | dump file 記錄多個 database 的 block，但必須在相同的 PG instance 中使用 | 無法跨 instance 複製 dump file |

---

### IV. 手動預熱策略

#### a. 識別熱表與熱索引

在實施手動預熱之前，你需要先知道哪些是「值得預熱」的物件。盲目預熱全部表不僅浪費時間，還會把真正的熱資料擠出 shared_buffers。

```sql
-- 識別熱表：依 tuple fetch 數量排序
-- n_tup_fetch 代表從這張表讀取了多少行（含 seqscan + index scan）
SELECT
    schemaname || '.' || relname AS table_name,
    n_tup_fetch AS rows_fetched,
    seq_scan,
    idx_scan,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE n_tup_fetch > 0
ORDER BY n_tup_fetch DESC
LIMIT 20;
```

```sql
-- 識別熱索引：依 index scan 次數排序
SELECT
    schemaname || '.' || indexrelname AS index_name,
    idx_scan AS scans,
    idx_tup_read AS rows_returned,
    idx_tup_fetch AS rows_fetched_to_table,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan > 0
ORDER BY idx_scan DESC
LIMIT 20;
```

> **初學者導讀**：`n_tup_fetch` vs `idx_scan` — 前者告訴你「哪些表被查詢得最多（總讀取行數）」，後者告訴你「哪些索引被使用了最多次」。兩個角度互補：一張小表可能 scan 次數很多但總行數少，一張大表可能 scan 次數少但每次讀取百萬行。兩者都是預熱的候選對象。

#### b. 預熱策略決策樹

```mermaid
flowchart TD
    A["PG 重啟後"] --> B["autoprewarm 自動恢復"]
    B --> C["等待 autoprewarm 完成<br/>（取決於 shared_buffers 大小）"]

    C --> D{"是否需要額外<br/>手動預熱？"}

    D -->|"是，autoprewarm<br/>只記錄最後一次 dump<br/>的狀態"| E["識別熱表 / 熱索引"]

    D -->|"否，autoprewarm<br/>已足夠"| Z["✅ 監控查詢效能<br/>確認已恢復正常"]

    E --> F{"表大小？"}

    F -->|"< 1 GB"| G["使用 buffer 模式<br/>同步載入 shared_buffers"]
    F -->|"1-10 GB"| H["使用 read 模式<br/>同步載入 OS page cache"]
    F -->|"> 10 GB"| I["使用 prefetch 模式<br/>非同步背景載入"]

    G --> J["逐一預熱熱表"]
    H --> K["逐一預熱熱表"]
    I --> L["並行發起多個 prefetch"]

    J --> M["使用 pg_buffercache<br/>驗證 prewarm 結果"]
    K --> M
    L --> M

    M --> N{"緩存命中率<br/>是否達標？"}
    N -->|"是"| Z
    N -->|"否"| O["補充預熱遺漏的<br/>表 / 索引"]
    O --> M

    style B fill:#4a90d9,color:white
    style G fill:#2ecc71,color:white
    style H fill:#ffd43b,color:black
    style I fill:#e67e22,color:white
    style Z fill:#2ecc71,color:white
```

#### c. 完整預熱腳本模板

將以下腳本整合到 PG 重啟後的啟動流程中（例如 systemd `ExecStartPost=` 或容器 `postStart` hook）：

```sql
-- ===== pg_prewarm 啟動後預熱腳本 =====
-- 在 PG 重啟後、應用連線到達前執行

-- 1. 等待 autoprewarm 完成（可選，若啟用了 autoprewarm）
DO $$
DECLARE
    autoprewarm_running boolean;
BEGIN
    -- 簡單等待：檢查 autoprewarm leader 是否仍在執行
    FOR i IN 1..120 LOOP
        SELECT EXISTS (
            SELECT 1 FROM pg_stat_activity
            WHERE backend_type = 'autoprewarm leader'
              AND state = 'active'
        ) INTO autoprewarm_running;
        EXIT WHEN NOT autoprewarm_running;
        PERFORM pg_sleep(1);
    END LOOP;
END $$;

-- 2. 手動預熱核心業務表（buffer 模式，確保在 shared_buffers）
DO $$
DECLARE
    n bigint;
BEGIN
    -- 高優先級熱表（< 1GB，直接載入 shared_buffers）
    FOR n IN
        SELECT pg_prewarm(t, mode => 'buffer')
        FROM unnest(ARRAY[
            'public.users',
            'public.accounts',
            'public.products',
            'public.configurations'
        ]) AS t
    LOOP
        NULL; -- pg_prewarm 已執行
    END LOOP;
END $$;

-- 3. 預熱核心索引
DO $$
DECLARE
    n bigint;
BEGIN
    FOR n IN
        SELECT pg_prewarm(t, mode => 'buffer')
        FROM unnest(ARRAY[
            'public.idx_users_email',
            'public.idx_accounts_user_id',
            'public.idx_products_sku',
            'public.pk_users',
            'public.pk_accounts',
            'public.pk_products'
        ]) AS t
    LOOP
        NULL;
    END LOOP;
END $$;

-- 4. 中大表使用 prefetch（非同步，不阻塞）
DO $$
DECLARE
    n bigint;
BEGIN
    FOR n IN
        SELECT pg_prewarm(t, mode => 'prefetch')
        FROM unnest(ARRAY[
            'public.orders',
            'public.order_items',
            'public.transactions',
            'public.audit_log'
        ]) AS t
    LOOP
        NULL;
    END LOOP;
END $$;

-- 5. 驗證
SELECT
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(pg_relation_size(c.oid)) AS total_size,
    CASE
        WHEN pg_relation_size(c.oid) = 0 THEN 0
        ELSE round(100.0 * count(*) * current_setting('block_size')::int
              / pg_relation_size(c.oid), 1)
    END AS pct_in_buffer
FROM pg_class c
JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
JOIN pg_database d ON b.reldatabase = d.oid
    AND d.datname = current_database()
WHERE c.relname IN (
    'users', 'accounts', 'products', 'configurations',
    'orders', 'order_items', 'transactions', 'audit_log'
)
GROUP BY c.relname, c.oid
ORDER BY c.relname;
```

#### d. 結合 pg_cron 定時補充預熱

對於 shared_buffers 容量有限、頻繁發生 buffer eviction 的環境，可以定時重新預熱關鍵表：

```sql
-- 每 15 分鐘對核心表執行一次預熱（確保不被擠出 shared_buffers）
SELECT cron.schedule(
    'prewarm-core-tables',
    '*/15 * * * *',
    $$
    SELECT pg_prewarm('public.users',     mode => 'buffer');
    SELECT pg_prewarm('public.accounts',  mode => 'buffer');
    SELECT pg_prewarm('public.products',  mode => 'buffer');
    $$
);
```

> **補充（Senior Dev）**：定期重新預熱是「暴力法」，它掩蓋了真正的問題——shared_buffers 太小或被冷資料佔據。更好的做法是先診斷 eviction 原因：
> ```sql
> -- 查看 shared_buffers 使用分布
> SELECT
>     c.relname,
>     count(*) AS buffers_used,
>     pg_size_pretty(count(*) * current_setting('block_size')::int) AS size_used,
>     usagecount
> FROM pg_class c
> JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
> WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
> GROUP BY c.relname, usagecount
> ORDER BY count(*) DESC
> LIMIT 30;
> ```
> `usagecount` 代表 buffer 的「受歡迎程度」（clock sweep algorithm 中的計數）。如果熱表的 `usagecount` 偏低，代表它們正在被冷資料驅逐——考慮增加 `shared_buffers` 或優化冷查詢。

---

### V. 效能影響

#### a. 三種 mode 的效能對比

以一張 10GB（約 1,310,720 blocks，block_size = 8KB）的表為例，在 NVMe SSD 上的實測：

| mode | 執行時間 | 函數阻塞 | 後續查詢是否命中 | 記憶體佔用 |
|------|---------|---------|----------------|-----------|
| `buffer` | ~15 秒 | 是（15 秒內無法執行其他操作） | ✅ 直接命中 shared_buffers | shared_buffers +10GB |
| `read` | ~15 秒 | 是（15 秒內無法執行其他操作） | ⚠️ 需從 OS cache 複製到 shared_buffers（微秒級） | OS page cache +10GB |
| `prefetch` | < 1ms（立即返回） | 否 | ⚠️ OS 非同步載入中，可能尚未完成 | OS page cache（漸進式） |

```mermaid
gantt
    title pg_prewarm 三種 mode 時間線對比（10GB 表）
    dateFormat X
    axisFormat %s

    section buffer 模式
    函數阻塞（約 15s）    :b1, 0, 15
    shared_buffers 可用   :milestone, 15, 0
    後續查詢即時命中      :b2, 15, 5

    section read 模式
    函數阻塞（約 15s）    :r1, 0, 15
    OS cache 可用         :milestone, 15, 0
    後續查詢微秒級命中    :r2, 15, 5

    section prefetch 模式
    函數返回（< 1ms）     :crit, p1, 0, 0.001
    OS 背景載入中         :p2, 0.001, 15
    OS cache 可用         :milestone, 15, 0
    後續查詢微秒級命中    :p3, 15, 5
```

> **初學者導讀**：`buffer` 和 `read` 的總執行時間幾乎相同（都是同步讀取磁盤），差別只在於資料的最終位置。`prefetch` 最大的優勢是「不阻塞」——你可以同時對 10 張大表發起 `prefetch`，OS 在背景並行載入，而你的啟動腳本可以繼續執行其他初始化任務。

#### b. 計算預熱時間

```sql
-- 估算預熱某張表所需的時間
SELECT
    relname,
    pg_size_pretty(pg_relation_size(oid)) AS total_size,
    pg_relation_size(oid) / current_setting('block_size')::bigint AS blocks,
    -- 假設每 block 讀取耗時 0.01ms (NVMe) 或 0.1ms (SATA SSD) 或 1ms (HDD)
    round(pg_relation_size(oid) / current_setting('block_size')::bigint
          * 0.01 / 1000, 2) AS est_seconds_nvme,
    round(pg_relation_size(oid) / current_setting('block_size')::bigint
          * 0.1 / 1000, 2) AS est_seconds_ssd,
    round(pg_relation_size(oid) / current_setting('block_size')::bigint
          * 1.0 / 1000, 2) AS est_seconds_hdd
FROM pg_class
WHERE relname IN ('orders', 'order_items', 'audit_log')
  AND relkind IN ('r', 'i');
```

預期輸出示例：

| relname | total_size | blocks | est_seconds_nvme | est_seconds_ssd | est_seconds_hdd |
|---------|-----------|--------|-----------------|-----------------|-----------------|
| orders | 10 GB | 1310720 | 13.11 | 131.07 | 1310.72 |
| order_items | 5 GB | 655360 | 6.55 | 65.54 | 655.36 |
| audit_log | 50 GB | 6553600 | 65.54 | 655.36 | 6553.60 |

#### c. autoprewarm 對啟動時間的影響

autoprewarm 在 PG 啟動階段開始載入 block，這可能延遲 PG 進入「可接受連線」狀態的時間：

| shared_buffers 大小 | dump file 記錄 block 數 | 估計載入時間（NVMe） | 對啟動時間影響 |
|---------------------|------------------------|---------------------|--------------|
| 4 GB | ~100,000 | ~1 秒 | 可忽略 |
| 16 GB | ~500,000 | ~5 秒 | 輕微 |
| 64 GB | ~2,000,000 | ~20 秒 | 明顯 |
| 256 GB | ~8,000,000 | ~80 秒 | 顯著 |

> **補充（Senior Dev）**：autoprewarm 的 block 載入是**非同步**進行的——PG 在接受連線之前不必等待所有 block 載入完成。autoprewarm BGW 在背景逐步載入，PG 啟動後立即可接受連線。換言之，autoprewarm 不會阻塞 PG 啟動，但它確實會佔用 I/O 頻寬，影響同期到達的查詢效能。
>
> 如果你需要在啟動後立即達到峰值效能，可以考慮：
> 1. 將 `autoprewarm_interval` 設得更小（更頻繁 dump，恢復更精準）
> 2. 結合手動 `pg_prewarm('buffer')` 模式，確保核心表在連線到達前已就緒
> 3. 監控 `pg_stat_bgwriter.buffers_alloc` 的變化速率，確認預熱完成

---

### VI. App Dev 最佳實踐

#### a. 生產環境配置建議

```mermaid
flowchart TD
    A["生產 PG 部署"] --> B["啟用 autoprewarm"]
    B --> C{"shared_buffers > 16GB？"}
    C -->|"是"| D["autoprewarm_interval = 600s<br/>（降低掃描開銷）"]
    C -->|"否"| E["autoprewarm_interval = 300s<br/>（預設值即可）"]

    D --> F["建立啟動後預熱腳本"]
    E --> F

    F --> G["區分三層預熱優先級"]

    G --> G1["Tier 1：核心業務表<br/>buffer 模式同步載入<br/>（< 1GB 的表）"]
    G --> G2["Tier 2：重要業務表<br/>prefetch 模式非同步<br/>（1-10GB 的表）"]
    G --> G3["Tier 3：超大歷史表<br/>不預熱，自然填充<br/>（> 10GB 且非高頻）"]

    G1 --> H["整合到 systemd ExecStartPost<br/>或容器 lifecycle hook"]
    G2 --> H
    G3 --> H

    H --> I["使用 pg_buffercache<br/>定期驗證緩存命中率"]
    I --> J["監控告警：<br/>當 pct_in_buffer < 80%<br/>觸發重新預熱"]

    style B fill:#2ecc71,color:white
    style G1 fill:#e74c3c,color:white
    style G2 fill:#ffd43b,color:black
    style G3 fill:#95a5a6,color:white
```

#### b. systemd 整合（Linux）

建立 systemd service drop-in 來在 PG 啟動後執行預熱腳本：

```bash
# 建立 drop-in 目錄
sudo mkdir -p /etc/systemd/system/postgresql@16-main.service.d

# 寫入 override 配置
sudo tee /etc/systemd/system/postgresql@16-main.service.d/prewarm.conf << 'EOF'
[Service]
ExecStartPost=/usr/local/bin/pg_prewarm_startup.sh
EOF
```

`/usr/local/bin/pg_prewarm_startup.sh` 內容：

```bash
#!/bin/bash
# pg_prewarm_startup.sh — PG 啟動後預熱腳本
# 等待 PG 完全接受連線後執行 prewarm

set -e

DB_NAME="yourdb"
PG_USER="postgres"
MAX_RETRIES=30
RETRY_INTERVAL=2

# 等待 PG 接受連線
for ((i=1; i<=MAX_RETRIES; i++)); do
    if psql -U "$PG_USER" -d "$DB_NAME" -c "SELECT 1;" > /dev/null 2>&1; then
        echo "[$(date)] PostgreSQL is ready."
        break
    fi
    echo "[$(date)] Waiting for PostgreSQL... ($i/$MAX_RETRIES)"
    sleep "$RETRY_INTERVAL"
done

# 執行預熱
echo "[$(date)] Starting pg_prewarm..."

psql -U "$PG_USER" -d "$DB_NAME" << 'SQL'
-- Tier 1: 核心表同步載入 shared_buffers
SELECT pg_prewarm('public.users',     mode => 'buffer');
SELECT pg_prewarm('public.accounts',  mode => 'buffer');
SELECT pg_prewarm('public.products',  mode => 'buffer');

-- Tier 2: 重要業務索引
SELECT pg_prewarm('public.idx_users_email',      mode => 'buffer');
SELECT pg_prewarm('public.idx_accounts_user_id',  mode => 'buffer');
SELECT pg_prewarm('public.idx_products_sku',      mode => 'buffer');

-- Tier 3: 大表非同步 prefetch
SELECT pg_prewarm('public.orders',       mode => 'prefetch');
SELECT pg_prewarm('public.order_items',  mode => 'prefetch');
SELECT pg_prewarm('public.transactions', mode => 'prefetch');

-- 驗證
SELECT
    relname,
    pg_size_pretty(pg_relation_size(oid)) AS size,
    (SELECT count(*) FROM pg_buffercache b
     WHERE b.relfilenode = c.relfilenode
       AND b.reldatabase = (SELECT oid FROM pg_database
                            WHERE datname = current_database()))
    AS buffers_loaded
FROM pg_class c
WHERE relname IN (
    'users', 'accounts', 'products',
    'orders', 'order_items', 'transactions'
);
SQL

echo "[$(date)] pg_prewarm startup script completed."
```

```bash
# 設定執行權限
sudo chmod +x /usr/local/bin/pg_prewarm_startup.sh

# 重新載入 systemd 並測試
sudo systemctl daemon-reload
sudo systemctl restart postgresql@16-main
```

#### c. Docker / K8s 容器整合

對於容器化部署，使用 lifecycle hook 或 init container：

```yaml
# docker-compose.yml 片段
services:
  postgres:
    image: postgres:16
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

將預熱 SQL 放入 `/docker-entrypoint-initdb.d/` 的初始化腳本中：

```sql
-- /docker-entrypoint-initdb.d/02_pg_prewarm.sql
-- 注意：此處僅在首次初始化時執行，並非每次容器啟動
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
```

對於每次容器重啟都需要預熱的場景，使用自訂 entrypoint wrapper：

```bash
#!/bin/bash
# docker-entrypoint-wrapper.sh

# 啟動 PG
docker-entrypoint.sh postgres &

# 等待 PG ready
until pg_isready -U postgres; do
    sleep 1
done

# 執行預熱
psql -U postgres -d yourdb -c "SELECT pg_prewarm('public.orders', mode => 'prefetch');"
psql -U postgres -d yourdb -c "SELECT pg_prewarm('public.users',  mode => 'buffer');"

# 保持容器運行
wait
```

#### d. .NET / Npgsql 應用層視角

從 Application Developer 的角度，你不應該在應用程式碼中直接調用 `pg_prewarm()`——這屬於 DBA / 基礎設施層面的職責。但你需要理解它的存在，並確保：

**連線字串設定**：使用 `Timeout` 和 `Connection Idle Lifetime` 參數來容忍重啟後的慢查詢期間：

```csharp
// Npgsql 連線字串 — PG 重啟後初期查詢可能較慢
var connString = new NpgsqlConnectionStringBuilder
{
    Host = "10.0.1.50",
    Port = 5432,
    Database = "yourdb",
    Username = "app_user",
    Password = "***",
    ApplicationName = "MyApp-WebServer-01", // 方便在 pg_stat_activity 識別
    Timeout = 30,          // 連線超時（秒）— PG 重啟期間允許更長等待
    CommandTimeout = 120,  // 命令超時（秒）— 冷啟動查詢可能需要更長時間
    MaxPoolSize = 50,
    ConnectionIdleLifetime = 300, // 閒置連線存活時間
    Pooling = true
}.ConnectionString;
```

**交易管理**：確保 transaction 在冷啟動期間不會因為查詢超時而遺留未提交的交易：

```csharp
// 正確寫法：using + try-catch 確保資源釋放
public async Task<IEnumerable<Order>> GetRecentOrdersAsync(CancellationToken ct = default)
{
    await using var conn = new NpgsqlConnection(_connString);
    await conn.OpenAsync(ct);

    // 設定 statement timeout 以應對冷啟動慢查詢
    await using var cmd = new NpgsqlCommand(
        "SET LOCAL statement_timeout = '30s'; " +
        "SELECT * FROM orders WHERE created_at >= @since;", conn);
    cmd.Parameters.AddWithValue("since", DateTime.UtcNow.AddDays(-7));

    try
    {
        await using var reader = await cmd.ExecuteReaderAsync(ct);
        var orders = new List<Order>();
        while (await reader.ReadAsync(ct))
        {
            orders.Add(MapOrder(reader));
        }
        return orders;
    }
    catch (PostgresException ex) when (ex.SqlState == "57014") // query_canceled
    {
        _logger.LogWarning("Query timed out (cold start?): {Message}", ex.Message);
        return Array.Empty<Order>(); // graceful degradation
    }
}
```

**健康檢查**：在應用層的 health check 中監控緩存狀態（僅限有權限的 monitoring user）：

```csharp
// Health Check：檢查核心表的緩存命中率
public async Task<HealthCheckResult> CheckBufferCacheAsync(CancellationToken ct)
{
    await using var conn = new NpgsqlConnection(_monitoringConnString);
    await conn.OpenAsync(ct);

    await using var cmd = new NpgsqlCommand(@"
        SELECT
            COUNT(*) FILTER (WHERE pct_in_buffer < 80) AS tables_below_threshold
        FROM (
            SELECT
                c.relname,
                ROUND(100.0 * COUNT(*) * current_setting('block_size')::int
                      / NULLIF(pg_relation_size(c.oid), 0), 1) AS pct_in_buffer
            FROM pg_class c
            JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
            JOIN pg_database d ON b.reldatabase = d.oid
                AND d.datname = current_database()
            WHERE c.relname IN ('users', 'accounts', 'orders')
            GROUP BY c.relname, c.oid
        ) sub
    ", conn);

    var count = (long?)await cmd.ExecuteScalarAsync(ct) ?? 0;
    return count == 0
        ? HealthCheckResult.Healthy("Core tables are warm in shared_buffers")
        : HealthCheckResult.Degraded($"{count} core table(s) below 80% buffer hit ratio");
}
```

#### e. 帕累托法則（80/20 原則）應用

不需要預熱所有表。根據效能分析，通常 **20% 的表承擔了 80% 的查詢**：

```sql
-- 找出承擔 80% 查詢負載的表（依累積佔比）
WITH table_stats AS (
    SELECT
        schemaname || '.' || relname AS table_name,
        n_tup_fetch,
        SUM(n_tup_fetch) OVER () AS total_fetch
    FROM pg_stat_user_tables
)
SELECT
    table_name,
    n_tup_fetch,
    round(100.0 * n_tup_fetch / total_fetch, 2) AS pct_of_total,
    round(100.0 * SUM(n_tup_fetch) OVER (ORDER BY n_tup_fetch DESC)
          / total_fetch, 2) AS cumulative_pct
FROM table_stats
ORDER BY n_tup_fetch DESC;
```

只需預熱 `cumulative_pct < 80%` 的表即可取得絕大部分效能收益，同時避免浪費 shared_buffers 空間。

> **補充（Senior Dev）**：預熱不是萬靈丹。如果你的 shared_buffers 本身就小於熱資料集的大小，再怎麼預熱都會發生 buffer eviction（熱資料被擠出）。正確的順序是：
> 1. 先確保 `shared_buffers` 足夠大（通常設為可用 RAM 的 15-25%，但不超過 8GB 除非你是專用 PG 伺服器）
> 2. 啟用 `autoprewarm` 處理重啟後的恢復
> 3. 手動預熱補充 autoprewarm 的盲區（dump 間隔內的 buffer 變化）
> 4. 使用 `pg_buffercache` 定期審查哪些表實際佔用了 shared_buffers
>
> 常見的誤解是：「shared_buffers 越大越好」。實際上，過大的 shared_buffers 會增加 checkpoint 和 BGWriter 的負擔，且 PG 依賴 OS page cache 做雙層緩存（double buffering）。一個常見的配置錯誤是：64GB RAM 機器設 `shared_buffers = 48GB`，導致 OS 沒有足夠記憶體做檔案系統緩存和排序操作。


## 4. pg_buffercache — 緩存內容診斷

> **初學者導讀**
> PostgreSQL 啟動時會在記憶體中劃出一塊區域叫 `shared_buffers`，就像一個「高速讀取緩存區」。每次查詢需要讀取資料時，PG 會先檢查 `shared_buffers` 有沒有那塊資料（page），有的話直接取用（cache hit），沒有的話就從磁碟載入（cache miss）。`pg_buffercache` 擴充讓你能「打開這個黑盒子」，查看 `shared_buffers` 裡面此刻到底裝了什麼 —— 哪些表佔用最多緩存、哪些資料是熱點、哪些即將被淘汰。這是 PG 可觀測性三本柱（pg_stat_statements / auto_explain / pg_buffercache）中唯一專注於**緩存層診斷**的工具。
>
> **補充（Senior Dev）**
> 很多 DBA 一看到查詢慢就加記憶體，但 `pg_buffercache` 讓你可以用數據驅動決策。記住核心公式：`shared_buffers` ≠ 全部記憶體。PG 依賴 OS Page Cache 做雙層緩存（double buffering），`pg_buffercache` 只看 PG 自己的緩存，看不到 OS 層。完整檢視需搭配 `pgfincore` 或 `pg_stat_kcache`。此外，PG 16 的 Buffer Manager 改寫（`BufferAlloc` 重構）讓緩存分配更高效，PG 17 更進一步引入了 `pg_buffercache_summary()` 解決大緩存掃描瓶頸。

---

### I. 核心價值

`pg_buffercache` 的核心價值在於提供 shared_buffers 的**即時透明性**（real-time transparency），回答三個關鍵問題：

1. **誰在緩存中？** 哪些 table / index / TOAST table 的哪些 8KB block 當前存在 `shared_buffers` 內
2. **誰佔最多？** 按物件聚合後，哪些表或索引是緩存空間的最大消費者
3. **冷熱分佈如何？** 每個 buffer 的 `usagecount` 告訴你它是頻繁存取的「熱點」還是即將被淘汰的「冷點」

透過這三個問題的答案，你可以進行以下診斷：

- 查詢突然變慢 → 檢查目標表的 page 是否大量不在緩存中
- 驗證 `pg_prewarm` 效果 → 確認熱表是否已確實載入
- 判斷 `shared_buffers` 大小是否足夠 → 觀察 usagecount 分佈是否有大量 0
- 發現空間浪費 → 找出佔著緩存但很少被存取的冷門表

```mermaid
flowchart TB
    subgraph "PostgreSQL Shared Memory"
        SB["shared_buffers<br/>（8KB × N buffers）"]
        B1["[Buf 1] orders block 0<br/>uc=5 🔥"]
        B2["[Buf 2] users block 3<br/>uc=2"]
        B3["[Buf 3] users_pkey block 0<br/>uc=1"]
        BDOT["..."]
        BN["[Buf N] empty"]
    end

    subgraph "pg_buffercache Extension"
        PVC["pg_buffercache view<br/>（直接讀取 shared memory<br/>buffer descriptors）"]
    end

    subgraph "分析輸出"
        Q1["📊 Top-10 緩存大戶<br/>（按 table 聚合）"]
        Q2["📈 冷熱分佈圖<br/>（usagecount histogram）"]
        Q3["🎯 不在緩存的表<br/>（cold start 受害者）"]
        Q4["🩺 緩存健康摘要<br/>（pg_buffercache_summary）"]
    end

    SB --> PVC
    PVC --> Q1
    PVC --> Q2
    PVC --> Q3
    PVC --> Q4

    style SB fill:#4a90d9,color:white
    style PVC fill:#e8d44d,color:black
    style Q1 fill:#2ecc71,color:white
    style Q2 fill:#2ecc71,color:white
    style Q3 fill:#2ecc71,color:white
    style Q4 fill:#2ecc71,color:white
```

> **補充（Senior Dev）**
> `pg_buffercache` 是 snapshot-only 的視圖 —— 它每次查詢都是「此刻」的狀態，沒有歷史記錄。它直接讀取 shared memory 中的 buffer descriptors 陣列，不經過磁碟 I/O。然而在 PG 16 及之前，即使只是 `count(*)`，也必須遍歷整個 buffer pool array（可能數十萬到數百萬個 entries），對於 `shared_buffers = 64GB` 以上的系統是需要考量的開銷。PG 17 的 `pg_buffercache_summary()` 解決了這個問題。

---

### II. pg_buffercache 視圖詳解

安裝方式（需 superuser）：

```sql
CREATE EXTENSION pg_buffercache;
```

安裝後即可查詢 `pg_buffercache` 視圖。視圖中的每一行代表 shared_buffers 中的一個 8KB buffer slot。

#### a. 欄位全解

| 欄位 | 型別 | 白話解釋 |
|------|------|---------|
| `bufferid` | `integer` | buffer pool 中的位置編號（1 ~ `shared_buffers` 總數），是 shared memory array 的 index |
| `relfilenode` | `oid` | 該 buffer 所屬 relation 的物理檔案節點號碼（filenode），用於 JOIN `pg_class` |
| `reltablespace` | `oid` | tablespace OID；`0` = `pg_default`，對應 `pg_tablespace.oid` |
| `reldatabase` | `oid` | database OID；`0` = shared system catalogs（如 `pg_database`、`pg_authid` 等 cluster 級別的共享表） |
| `relforknumber` | `smallint` | fork 類型：`0` = main fork（資料/索引本體），`1` = FSM（Free Space Map），`2` = VM（Visibility Map） |
| `relblocknumber` | `bigint` | 該 relation 內的第幾個 8KB block（從 0 開始編號） |
| `pinning_backends` | `integer` | 正在使用（pin）這個 buffer 的 backend 數量；`0` = 無人使用 |
| `usagecount` | `smallint` | Clock-Sweep 演算法的存取計數，範圍 `0` ~ `5`（PG 16 預設上限），詳見下文 |
| `isdirty` | `boolean` | （PG 13+）該 buffer 是否有未寫入磁碟的修改（dirty page），需 Checkpointer 或 BGWriter 刷出 |

> **初學者導讀**
> 你可以把每一行理解為：「在 shared_buffers 第 N 號位置，放著來自某個 database 某張表的第 M 個 8KB block，它目前被 X 個 session 使用中，存取頻率是 Y，資料是否已同步到磁碟 = Z」。所有欄位都是中繼資料（metadata），看不到 buffer 中的實際 row 內容。

#### b. usagecount 與 Clock-Sweep 演算法

`usagecount` 是 `pg_buffercache` 中最關鍵的效能指標。它的值（`0` ~ `5`）由 PostgreSQL 的 **Clock-Sweep（時鐘掃描）** 緩存淘汰演算法決定。

**Clock-Sweep 工作原理**：

PostgreSQL 使用一個改良版的 Clock-Sweep（也稱為 NFU / Not Frequently Used 變體）來管理 shared_buffers 的淘汰策略。想像一個環形陣列，有一個「時鐘指針」不斷循環掃過每個 buffer slot：

1. **初始化**：新載入的 buffer 的 `usagecount` 設為 `1`
   - 例外：使用 Ring Buffer 策略（如 Sequential Scan）時，`usagecount` 設為 `0`，這些 buffer 用完即被回收，不會污染主流緩存
2. **存取命中**：每次 backend 存取這個 buffer 中的 page 時，`usagecount` +1（上限 `5`）
3. **指針掃過**：當需要騰出空間載入新 page 時，時鐘指針依序掃過 buffer，將 `usagecount` -1
4. **淘汰觸發**：當某個 buffer 的 `usagecount` 降到 `0`，且沒有被任何 backend pin 住（`pinning_backends = 0`），該 buffer 就成為「淘汰候選」，新 page 可以覆蓋它

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Empty: buffer slot 初始化

    state "usagecount = 0" as UC0
    state "usagecount = 1" as UC1
    state "usagecount = 2~4" as UC2_4
    state "usagecount = 5 🔥" as UC5

    Empty --> UC1: 載入新 page（一般讀取）
    Empty --> UC0: 載入新 page（Ring Buffer /
    Seq Scan 策略）
    UC0 --> Empty: 💀 被淘汰（新 page 覆蓋）
    UC1 --> UC2_4: backend 存取 (uc++)
    UC2_4 --> UC5: backend 存取 (uc++)
    UC5 --> UC5: backend 存取（維持上限）

    UC5 --> UC2_4: ⏰ 指針掃過 (uc--)
    UC2_4 --> UC1: ⏰ 指針掃過 (uc--)
    UC1 --> UC0: ⏰ 指針掃過 (uc--)

    note right of UC5: 🔥 熱點 buffer
    note right of UC0: 💀 即將淘汰
```

**usagecount 速查表**：

| usagecount | 白話解釋 | 短期命運 |
|------------|---------|---------|
| `5` | 極度熱門（例如 B-tree root page、高並發 PK lookup） | 幾乎不會被淘汰 |
| `3` ~ `4` | 常用（定期被查詢的資料/索引頁） | 短期有保障 |
| `1` ~ `2` | 偶爾用（剛載入或使用頻率低） | 可能很快被擠出 |
| `0` | 冰冷（載入後無人存取，或已被指針掃到 0） | 下次指針掃過即淘汰 |

> **補充（Senior Dev）**
> PG 10 至今 `usagecount` 上限維持 `5`，大多數場景足夠。但極端高並發 OLTP 中，某些熱點 page（如 sequence 的 current value page、排隊鎖的對照頁）可能被頻繁 pin/unpin，`5` 的上限不足以完全保證它們不被淘汰。PG 16/17 的 Buffer Manager 改寫引入了更高效的 buffer 分配策略（減少 BufMapping LWLock 競爭），但 usagecount 機制本身沒有重大變化。
>
> **Ring Buffer 的重要性**：Sequential Scan、VACUUM、Bulk Write 都使用 Ring Buffer，佔用固定的少量 buffers（預設 256KB = 32 個 8KB page），用完即丟。這意味著一個大表的 Seq Scan 不會把你的熱點資料從 shared_buffers 中擠出去。在 `pg_buffercache` 中看到 `usagecount = 0` 的 buffer 一閃而過，通常就是 Ring Buffer。

#### c. pinning_backends — 緩存鎖定機制

`pinning_backends` 表示當前有多少個 backend process 正在 **pin**（釘住）這個 buffer。

**pin 的機制**：

- backend 在讀取或修改一個 page 時，必須先 pin 住對應的 buffer，確保該 buffer 在操作期間不會被淘汰
- 操作完成後**立即 unpin**（不等 `COMMIT`），pin 的持有時間通常是微秒級別
- `pinning_backends = 0`：沒有 backend 正在使用，可以被淘汰（前提是 `usagecount = 0`）
- `pinning_backends > 0`：至少有一個 backend 正在讀/寫，這個 buffer **絕對不會被淘汰**（即使 `usagecount = 0`）
- 同一個 buffer 可以被多個 backend 同時 pin（例如多個 session 同時讀同一個 PK index root page）

```sql
-- 找出當前被 pin 住最多的 buffers（通常代表並發熱點）
SELECT c.relname,
       b.relblocknumber,
       b.pinning_backends,
       b.usagecount,
       b.isdirty
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.pinning_backends > 0
  AND b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
ORDER BY b.pinning_backends DESC
LIMIT 10;
```

**結果解讀**：

| 發現 | 含義 |
|------|------|
| B-tree root page 的 `pinning_backends` 很高 | 正常，這是高並發 OLTP 的典型現象 |
| 某個 data page 的 `pinning_backends` 很高 | 可能是熱點資料（如熱門商品的庫存行），需檢查是否需要分區分散 |
| 某個 buffer 的 `pin = 1` 長時間不釋放 | 搭配 `pg_stat_activity` 檢查該 backend 是否卡在等鎖或長時間執行的查詢 |

> **補充（Senior Dev）**
> pin 機制是 PG 緩存管理的核心安全保障。沒有 pin 的機制，一個 buffer 可能在 backend 讀到一半時就被淘汰，導致資料不一致。但 pin 時間極短（通常 < 1ms），所以你在 `pg_buffercache` 中看到的 `pinning_backends > 0` 的瞬間狀態會不斷變化。如果你在同一個查詢中反覆看到同一個 buffer 被 pin，那才是真正的並發熱點。

#### d. isdirty 標誌（PG 13+）

`isdirty` 是 PG 13 加入的欄位，指示 buffer 中的資料是否已被修改但**尚未寫入磁碟**：

- **`isdirty = true`**：buffer 中有未寫入磁碟的變更（可能是已提交但 Checkpointer 尚未刷出，也可能是未提交的 transaction 變更）
- **`isdirty = false`**：buffer 內容與磁碟一致（clean page）

```sql
-- 查看當前 dirty buffers 的數量與總容量
SELECT count(*) AS dirty_buffers,
       pg_size_pretty(count(*) * 8192) AS dirty_size,
       round(100.0 * count(*) / (SELECT setting::int FROM pg_settings WHERE name = 'shared_buffers'), 2) AS dirty_pct
FROM pg_buffercache
WHERE isdirty;
```

**Dirty buffer 過多的常見原因**：

| 原因 | 檢查方式 | 調整參數 |
|------|---------|---------|
| Checkpointer 間隔太長 | `SHOW checkpoint_timeout;` | 調小 `checkpoint_timeout`（預設 5min） |
| BGWriter 跟不上寫入速度 | 查 `pg_stat_bgwriter.buffers_clean` vs `buffers_checkpoint` | 調大 `bgwriter_lru_maxpages` |
| 大量未提交變更（長 transaction） | `SELECT * FROM pg_stat_activity WHERE state = 'idle in transaction' AND age(now(), xact_start) > '5 min';` | 設定 `idle_in_transaction_session_timeout` |

> **補充（Senior Dev）**
> PG 13 引入 `isdirty` 之前，DBA 只能透過推斷估算 dirty 比例，例如觀察 `pg_stat_bgwriter` 的 `buffers_checkpoint`、`buffers_clean`、`buffers_backend` 計數變化。現在可以在 `pg_buffercache` 中直接看到 dirty 狀態，配合 `max_wal_size` 和 `checkpoint_completion_target` 形成完整的 WAL → dirty page → checkpoint 觀測鏈。若 `dirty_pct` 持續 > 30%，且 `checkpoint_timeout` 已經很短，通常是寫入 I/O 子系統的瓶頸。

#### e. JOIN pg_class 獲取物件名稱

`pg_buffercache` 只儲存 OID（object identifier），不存人類可讀的名稱。你需要 JOIN 系統表才能知道 buffer 屬於哪張表或索引。

**標準 JOIN 模式**：

```sql
SELECT c.relname,
       c.relkind,
       b.relforknumber,
       b.relblocknumber,
       b.usagecount,
       b.pinning_backends,
       b.isdirty
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
  AND b.relforknumber = 0  -- 只看 main fork（實際資料/索引）
LIMIT 20;
```

**JOIN 注意事項**：

| 要點 | 說明 |
|------|------|
| `reldatabase = 0` | 代表 **shared system catalogs**（如 `pg_database`、`pg_authid`），這些表在整個 cluster 中共享，任何 database 的 buffer pool 都可能包含它們 |
| `pg_relation_filenode()` vs `c.relfilenode` | 通常兩者相同，但在 `TRUNCATE` 或 `VACUUM FULL` 後，PG 可能創建新的物理檔案，此時 `pg_relation_filenode()` 回傳的值可能與 `c.relfilenode` 不同。**建議始終使用 `pg_relation_filenode(c.oid)`** |
| `relkind` 解讀 | `'r'` = 普通表，`'i'` = 索引，`'t'` = TOAST 表，`'m'` = 物化視圖，`'S'` = sequence |
| `relforknumber` 過濾 | 一般診斷只需看 `relforknumber = 0`（main fork），FSM/VM 的 buffer 通常很少且診斷價值低 |

---

### III. 實戰診斷查詢

以下查詢皆可直接複製到 psql 執行。執行前請確認擴充已安裝：

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
```

#### a. Top-10 緩存大戶

找出當前 shared_buffers 中佔用最多空間的物件（含表、索引、TOAST 表）：

```sql
-- 找出 shared_buffers 中佔用最多空間的前 10 名
SELECT c.relname,
       c.relkind,
       CASE c.relkind
           WHEN 'r' THEN '普通表'
           WHEN 'i' THEN '索引'
           WHEN 't' THEN 'TOAST表'
           WHEN 'm' THEN '物化視圖'
           ELSE c.relkind::text
       END AS reltype,
       count(*) AS buffers,
       pg_size_pretty(count(*)::bigint * 8192) AS size,
       round(100.0 * count(*) / (SELECT setting::int FROM pg_settings WHERE name = 'shared_buffers'), 2) AS pct_of_sb
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
  AND b.relforknumber = 0
GROUP BY c.relname, c.relkind
ORDER BY buffers DESC
LIMIT 10;
```

**結果解讀**：

| 輸出 | 含義 | 行動 |
|------|------|------|
| `buffers` | 該物件在 shared_buffers 中佔用幾個 8KB blocks | 乘以 8192 可得 bytes |
| `size` | 人類可讀的緩存佔用量 | 對比實體檔案大小（`pg_total_relation_size()`），了解緩存覆蓋率 |
| `pct_of_sb` | 佔 shared_buffers 總容量的百分比 | 若某小表佔比異常高（例如 10 萬筆但佔 30%），可能是索引膨脹 |

#### b. 每表緩存命中率

`pg_buffercache` 是即時快照，無法直接計算歷史命中率。需結合 `pg_statio_user_tables` 的累計數據：

```sql
-- 計算每個使用者表的歷史緩存命中率（since last stats reset）
SELECT relname,
       heap_blks_hit,
       heap_blks_read,
       heap_blks_hit + heap_blks_read AS total_accesses,
       CASE WHEN heap_blks_hit + heap_blks_read > 0
            THEN round(100.0 * heap_blks_hit / (heap_blks_hit + heap_blks_read), 2)
            ELSE NULL
       END AS cache_hit_ratio_pct,
       pg_size_pretty(
           CASE WHEN heap_blks_hit + heap_blks_read > 0
                THEN heap_blks_read::bigint * 8192
                ELSE 0
           END
       ) AS disk_read_volume
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_read DESC
LIMIT 20;
```

**命中率診斷速查**：

| cache_hit_ratio_pct | 狀態 | 行動 |
|---------------------|------|------|
| > 99% | 優 | 緩存策略有效，繼續觀察 |
| 95% ~ 99% | 可接受 | 定期檢查趨勢 |
| 90% ~ 95% | 注意 | 檢查是否特定 SQL 拉低命中率 |
| < 90% | 嚴重 | 立即用 `pg_buffercache` 深入診斷，搭配 `pg_stat_statements` 找問題 SQL |

> **初學者導讀**
> `heap_blks_hit` 是「在 shared_buffers 中直接找到資料的次數」（命中）；`heap_blks_read` 是「被迫從磁碟讀取的次數」（未命中）。命中率 = 命中 / 總次數。注意這些是**歷史累計值**（從上次 `pg_stat_reset()` 或 PG 重啟後開始累計），不是即時值。一個剛重啟的資料庫命中率必然偏低，這是正常的。

#### c. 冷熱分佈（usagecount Histogram）

診斷特定表在緩存中的「冷熱健康度」：

```sql
-- 查看特定表的 usagecount 分佈
SELECT usagecount,
       count(*) AS buffers,
       round(100.0 * count(*) / SUM(count(*)) OVER (), 2) AS pct
FROM pg_buffercache b
WHERE b.relfilenode = pg_relation_filenode('your_table_name'::regclass)
  AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
  AND b.relforknumber = 0
GROUP BY usagecount
ORDER BY usagecount;
```

```sql
-- 橫向比較多個核心物件的冷熱分佈（Pivot）
SELECT c.relname,
       count(*) FILTER (WHERE b.usagecount >= 4) AS hot_buffers,
       count(*) FILTER (WHERE b.usagecount BETWEEN 1 AND 3) AS warm_buffers,
       count(*) FILTER (WHERE b.usagecount = 0) AS cold_buffers,
       count(*) AS total_buffers,
       round(100.0 * count(*) FILTER (WHERE b.usagecount >= 4) / count(*), 2) AS hot_pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
  AND b.relforknumber = 0
  AND c.relname IN ('orders', 'users', 'products')  -- 替換為你關注的表名
GROUP BY c.relname
ORDER BY hot_pct DESC;
```

**解讀規則**：

| 冷熱模式 | 診斷 | 建議 |
|---------|------|------|
| `hot_pct` > 60% | 健康，表的大部分資料是活躍熱點 | 維持現狀 |
| `hot_pct` 低 + `cold_buffers` 高 | 資料載入了但很少重複存取（典型的 Seq Scan 後遺症） | 檢查是否有不必要的 Seq Scan |
| 零 `hot_buffers` + 全部 `cold_buffers` | 表即將被完全淘汰出緩存，或被 Ring Buffer 抑制 | 檢查是否 Ring Buffer 策略導致、考慮 `pg_prewarm` |

#### d. 緩存整體健康摘要

不需要掃全表也能獲取整體概覽（適用於 PG 16 及更早版本）：

```sql
-- 緩存使用率、dirty 比例、buffer pin 狀態
WITH total_bufs AS (
    SELECT setting::int AS total FROM pg_settings WHERE name = 'shared_buffers'
),
used_bufs AS (
    SELECT count(*) AS used FROM pg_buffercache WHERE reldatabase IS NOT NULL
),
dirty_bufs AS (
    SELECT count(*) AS dirty FROM pg_buffercache WHERE isdirty
)
SELECT t.total AS total_buffers,
       u.used AS used_buffers,
       t.total - u.used AS free_buffers,
       d.dirty AS dirty_buffers,
       round(100.0 * u.used / t.total, 2) AS usage_pct,
       round(100.0 * d.dirty / NULLIF(u.used, 0), 2) AS dirty_pct
FROM total_bufs t, used_bufs u, dirty_bufs d;
```

**解讀**：

| 指標 | 正常值 | 異常訊號 |
|------|--------|---------|
| `usage_pct` | 接近 100%（shared_buffers 本來就應被充分利用） | 持續 < 50% 且 PG 剛啟動很久，可能 shared_buffers 設太大浪費記憶體 |
| `dirty_pct` | < 20% | > 30% → Checkpointer 或 BGWriter 跟不上寫入速度 |
| `free_buffers` | 少量 | 大量 free buffers 但查詢仍慢 → 瓶頸不在緩存，檢查 CPU 或 Lock |

#### e. 誰不在緩存中？

找出當前 database 中有實際資料但完全不在 shared_buffers 中的表：

```sql
-- 找出不在 shared_buffers 中的使用者表（含頁數 > 0 的表）
SELECT c.relname,
       c.relpages AS estimated_pages,
       pg_size_pretty(c.relpages::bigint * 8192) AS estimated_size,
       COALESCE(s.seq_scan, 0) AS seq_scan,
       COALESCE(s.idx_scan, 0) AS idx_scan
FROM pg_class c
LEFT JOIN pg_stat_user_tables s ON c.relname = s.relname
WHERE c.relkind = 'r'
  AND c.relpages > 0
  AND c.relfilenode NOT IN (
      SELECT DISTINCT relfilenode
      FROM pg_buffercache
      WHERE reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
        AND relforknumber = 0
  )
  AND c.relnamespace NOT IN (
      SELECT oid FROM pg_namespace WHERE nspname IN ('pg_catalog', 'information_schema')
  )
ORDER BY c.relpages DESC
LIMIT 20;
```

> **補充（Senior Dev）**
> 查詢結果為**空是好事** —— 代表所有有資料的表都在緩存中。若有結果且 `seq_scan` 很高，表示這是一張被頻繁 Seq Scan 的表，可能是缺失關鍵索引導致被迫全表掃描。若 `idx_scan` 高但表不在緩存，檢查索引頁是否已被淘汰，必要時用 `pg_prewarm('idx_name', 'buffer')` 預熱索引。

---

### IV. pg_buffercache_summary() — PG 17 新增

#### a. 函數說明

PostgreSQL 17 引入了 `pg_buffercache_summary()` 函數，返回 shared_buffers 的**單行聚合摘要**，無需掃描整個 `pg_buffercache` 視圖。

**返回欄位**：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `buffers_total` | `bigint` | shared_buffers 總 buffer slot 數量（= `SHOW shared_buffers` / 8192） |
| `buffers_used` | `bigint` | 當前有資料載入的 buffer 數量 |
| `buffers_dirty` | `bigint` | 包含未寫入磁碟變更的 buffer 數量 |
| `buffers_pinned` | `bigint` | 當前被 backend pin 住的 buffer 數量 |

#### b. 使用範例

```sql
-- PG 17：一行查詢完成緩存健康檢查
SELECT * FROM pg_buffercache_summary();
```

```sql
-- 格式化輸出：含百分比與人類可讀大小
SELECT buffers_total,
       buffers_used,
       buffers_dirty,
       buffers_pinned,
       buffers_total - buffers_used AS buffers_free,
       pg_size_pretty(buffers_total::bigint * 8192) AS total_size,
       pg_size_pretty(buffers_used::bigint * 8192) AS used_size,
       pg_size_pretty(buffers_dirty::bigint * 8192) AS dirty_size,
       round(100.0 * buffers_used / buffers_total, 2) AS used_pct,
       round(100.0 * buffers_dirty / NULLIF(buffers_used, 0), 2) AS dirty_pct,
       round(100.0 * buffers_pinned / NULLIF(buffers_used, 0), 2) AS pinned_pct
FROM pg_buffercache_summary();
```

#### c. 效能對比：為何比全視圖掃描快

`pg_buffercache_summary()` 快的關鍵在於它直接讀取 shared memory 中的**全域計數器**（global counters），而不需要遍歷整個 buffer pool descriptor array：

```mermaid
flowchart LR
    subgraph "PG 16 及之前 — 傳統查詢"
        A1["SELECT count(*) FROM pg_buffercache"] --> A2["遍歷全部 N 個 buffer descriptors<br/>（O(N) shared memory 掃描）"]
        A2 --> A3["回傳結果<br/>（N=100萬 約需 30~80ms）"]
    end

    subgraph "PG 17 — pg_buffercache_summary()"
        B1["SELECT * FROM pg_buffercache_summary()"] --> B2["讀取全域計數器<br/>（O(1) 常數時間）"]
        B2 --> B3["回傳結果<br/>（< 1ms）"]
    end

    style A2 fill:#e74c3c,color:white
    style B2 fill:#2ecc71,color:white
```

**適用場景**：

| 場景 | 工具 | 原因 |
|------|------|------|
| 每分鐘監控告警 | `pg_buffercache_summary()` | O(1) 無效能影響 |
| 深度分析冷熱分佈 | `pg_buffercache` 全視圖 | 需要 per-buffer usagecount |
| 快速判斷緩存是否健康 | `pg_buffercache_summary()` | 一行 SQL，1ms 搞定 |

> **補充（Senior Dev）**
> 在 PG 16 及之前，`pg_buffercache` 的 `count(*)` 必須遍歷整個 buffer pool。如果 `shared_buffers = 64GB`（約 838 萬個 buffers），這個掃描本身就消耗可觀的 CPU time 與 shared memory 讀取頻寬。PG 17 的 `pg_buffercache_summary()` 讓緩存健康檢查可以放進高頻率的監控 loop（每秒執行也不會造成可量測的效能影響）。若你還在 PG 16 或更早版本，建議使用 `pg_buffercache` 的查詢結果進行快取，不要高頻率地全表掃描。

---

### V. 診斷決策流程

以下是使用 `pg_buffercache` 診斷查詢效能問題的標準決策路徑。從「查詢變慢」開始，逐步定位到緩存層：

```mermaid
flowchart TD
    A["🚨 現象：查詢變慢<br/>或整體回應延遲上升"] --> B["Step 1: 查 pg_stat_statements<br/>shared_blks_read 異常高？"]

    B -->|"shared_blks_read 正常<br/>（比例低，大部分是 hit）"| C["✅ 瓶頸不在緩存 I/O<br/>→ 檢查 Lock 等待<br/>→ 檢查 CPU / Plan 變化<br/>→ 檢查 WAL 寫入瓶頸"]
    B -->|"shared_blks_read 高<br/>（大量 disk read）"| D["Step 2: 查 pg_statio_user_tables<br/>定位哪些表的 heap_blks_read 高"]

    D --> E{"Step 3: 查 pg_buffercache<br/>這些表的 page<br/>在緩存中嗎？"}

    E -->|"✅ 在緩存中<br/>且 usagecount 高<br/>但仍大量 disk read"| F["🔀 共享緩存競爭<br/>（shared_buffers 太小<br/>或緩存 churn 過高）<br/>→ 增加 shared_buffers<br/>→ 減少不必要的 page 載入"]

    E -->|"❌ 不在緩存中<br/>或 usagecount 極低"| G{"分類判斷原因"}

    G -->|"🆕 新上線表<br/>還未被頻繁存取"| H["🧊 冷啟動問題<br/>→ pg_prewarm 預熱<br/>→ 啟動腳本中手動 SELECT<br/>→ 考慮在 connection pool<br/>初始化時載入"]
    G -->|"📉 原本在但被擠出<br/>（大表或舊表）"| I["💨 緩存不足<br/>→ 增加 shared_buffers<br/>→ pg_prewarm 固定熱表<br/>→ 檢查是否有大表 Seq Scan<br/>使用 Ring Buffer 避免污染"]
    G -->|"🔍 索引被淘汰<br/>導致 Index Scan → Seq Scan"| J["📛 索引緩存蒸發<br/>→ pg_prewarm 預熱關鍵索引<br/>→ 增加 shared_buffers<br/>→ 考慮 partial index<br/>減少索引大小"]

    E -->|"⚡ 部分在但分佈不均<br/>冷熱分離明顯"| K["📊 優化存取模式<br/>→ 定期 pg_prewarm<br/>→ 檢查 SQL 是否載入<br/>不必要的 page<br/>→ 分析 query pattern"]

    style A fill:#e74c3c,color:white
    style C fill:#4a90d9,color:white
    style H fill:#2ecc71,color:white
    style I fill:#ffd43b,color:black
    style J fill:#ffd43b,color:black
    style F fill:#e74c3c,color:white
    style K fill:#4a90d9,color:white
```

**典型場景對照表**：

| 場景 | 你觀察到的症狀 | pg_buffercache 中的發現 | 解決方向 |
|------|--------------|----------------------|---------|
| **冷啟動** | PG 重啟後前 5 分鐘查詢超慢，之後恢復 | 核心表完全不在 `shared_buffers` 中 | `pg_prewarm` 在啟動腳本中預熱（見 VII.b） |
| **大表擠壓** | 跑完某個報表查詢後，OLTP 交易突然變慢 | 報表表佔滿 buffers，OLTP 表的 `usagecount` 全為 0 | 限流報表查詢 / 分離讀寫庫 / 增大 `shared_buffers` |
| **索引蒸發** | 原本走 Index Scan 的變 Seq Scan，執行時間暴增 | 索引 root page `usagecount = 0` | `pg_prewarm` 預熱索引 / 檢查是否索引膨脹 |
| **緩存 Churn** | 效能時好時壞，不可預測 | 同一個表的 blocks 在 buffers 中不斷換入換出，`usagecount` 波動劇烈 | 增大 `shared_buffers` / 優化查詢減少 page 存取量 |
| **Ring Buffer 錯覺** | 大表的 Seq Scan 後表完全不在緩存中 | 該表 buffer 不出現在 pg_buffercache（這是正常的 Ring Buffer 行為） | 不需處理。Ring Buffer 就是設計來保護主流緩存的 |

---

### VI. 限制

使用 `pg_buffercache` 時需注意以下限制，這些限制在生產環境中可能影響診斷策略：

**1. 需要超級使用者權限（superuser）**

`pg_buffercache` 擴充的安裝和查詢都要 superuser 權限。因為它直接暴露 shared memory buffer 的 meta-data，涉及跨 database 的資訊安全。對於只有唯讀權限的 App Dev，需請 DBA 協助建立包裝視圖：

```sql
-- DBA 執行：建立一個可供普通使用者查詢的聚合視圖
CREATE SCHEMA IF NOT EXISTS monitoring;

CREATE OR REPLACE VIEW monitoring.buffercache_by_table AS
SELECT c.relname,
       count(*) AS buffers,
       pg_size_pretty(count(*)::bigint * 8192) AS size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
  AND b.relforknumber = 0
GROUP BY c.relname;

GRANT SELECT ON monitoring.buffercache_by_table TO app_readonly_user;
```

**2. 大緩存下全表掃描耗時**

若 `shared_buffers` > 32GB，`SELECT * FROM pg_buffercache` 需掃描數百萬行。解決方案：
- PG 17：優先使用 `pg_buffercache_summary()` 做快速摘要（O(1)）
- PG 16-：加上 `WHERE` 過濾條件（指定 `relfilenode`）縮小掃描範圍
- 所有版本：不要高頻率（< 每 10 分鐘）執行全表掃描

**3. 僅為即時快照（Snapshot-Only）**

`pg_buffercache` 沒有歷史記錄功能。你無法查詢「昨天下午 3 點的緩存狀態」。如果需要趨勢分析，需自行定期擷取並存入外部監控系統（如 Prometheus、Zabbix）。

**4. 看不到 OS Page Cache**

`pg_buffercache` 只反映 PostgreSQL 的 shared_buffers。即使某個 page 不在 shared_buffers 中，仍可能在 OS Page Cache 中（此時讀取不需實體磁碟 I/O，只是記憶體複製）。完整檢視需要：
- `pgfincore` 擴充：查詢 relation 的 page 在 OS Page Cache 中的狀態
- OS 工具：Linux 的 `fincore`、`vmtouch`；Windows 的 RAMMap
- `pg_stat_kcache`：獲取查詢級別的真實 physical read 量

**5. Buffer 內容不可讀**

`pg_buffercache` 只顯示 buffer 的**中繼資料**（屬於哪個表、第幾個 block、usagecount 等），不顯示 buffer 中儲存的實際 tuple / row 內容。你無法用它直接查看某筆資料的原始 bytes。這是出於安全考量的設計決策。

**6. 不包含 temporary buffers**

`pg_buffercache` 不追蹤 `temp_buffers`（session-level temporary table 的緩存）的內容。每個 session 的 temp buffers 是獨立的、非共享的，不在 shared_buffers 的管理範圍內。

---

### VII. App Dev 最佳實踐

#### a. 定期快照建立緩存趨勢

雖然 `pg_buffercache` 是即時快照，但定期擷取可以建立趨勢線，觀察緩存健康度的長期變化：

```sql
-- 建立快照紀錄表（建議放在 monitoring schema）
CREATE TABLE IF NOT EXISTS monitoring.buffercache_snapshot (
    snap_time     timestamptz DEFAULT now(),
    relname       text,
    relkind       "char",
    buffers       bigint,
    hot_buffers   bigint,
    warm_buffers  bigint,
    cold_buffers  bigint
);

-- 透過 pg_cron 每 30 分鐘自動擷取一次
INSERT INTO monitoring.buffercache_snapshot (relname, relkind, buffers, hot_buffers, warm_buffers, cold_buffers)
SELECT c.relname,
       c.relkind,
       count(*) AS buffers,
       count(*) FILTER (WHERE b.usagecount >= 4) AS hot_buffers,
       count(*) FILTER (WHERE b.usagecount BETWEEN 1 AND 3) AS warm_buffers,
       count(*) FILTER (WHERE b.usagecount = 0) AS cold_buffers
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
  AND b.relforknumber = 0
GROUP BY c.relname, c.relkind;
```

> **補充（Senior Dev）**
> 不要每秒都擷取 `pg_buffercache`！在 `shared_buffers = 8GB` 的系統上，一次完整掃描可能需要 10 ~ 50ms。建議每 30 分鐘擷取一次，保留 7 天資料做趨勢分析，超過的用 `pg_partman` 自動清理。若使用 PG 17，可將 `pg_buffercache_summary()` 的結果以更高頻率（每分鐘）寫入監控，因為 O(1) 不會造成效能影響。

#### b. 搭配 pg_prewarm 實現緩存優化

`pg_prewarm` 擴充可以主動將表或索引載入 shared_buffers 或 OS Page Cache，是 `pg_buffercache` 的最佳搭檔：

```sql
-- 安裝 pg_prewarm（已內建於 contrib）
CREATE EXTENSION IF NOT EXISTS pg_prewarm;

-- 將熱門表載入 shared_buffers
SELECT pg_prewarm('orders', 'buffer');       -- 'buffer' = shared_buffers
SELECT pg_prewarm('orders_pkey', 'buffer');  -- 索引也可以 prewarm

-- 載入 OS Page Cache（而非 shared_buffers）
SELECT pg_prewarm('users', 'prefetch');      -- 'prefetch' = OS async read

-- 查看 prewarm 結果：回傳載入了幾個 blocks
```

**C# / Npgsql 中的應用啟動預熱範例**：

```csharp
// 應用啟動時執行緩存預熱（Program.cs / Startup）
using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync();

// 記錄 backend_pid，以便出問題時對照 pg_stat_activity
var pid = conn.ProcessID;
Console.WriteLine($"Backend PID: {pid}");

// 預熱關鍵業務表與索引
await conn.ExecuteAsync("SELECT pg_prewarm('orders', 'buffer')");
await conn.ExecuteAsync("SELECT pg_prewarm('users', 'buffer')");
await conn.ExecuteAsync("SELECT pg_prewarm('orders_pkey', 'buffer')");
await conn.ExecuteAsync("SELECT pg_prewarm('users_pkey', 'buffer')");
await conn.ExecuteAsync("SELECT pg_prewarm('orders_user_id_idx', 'buffer')");

// 驗證預熱結果（可選，生產環境建議移除此段落）
var cacheCheck = await conn.QueryAsync(@"
    SELECT c.relname, count(*) AS buffers
    FROM pg_buffercache b
    JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
    WHERE c.relname = ANY(@names)
      AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
    GROUP BY c.relname",
    new { names = new[] { "orders", "users", "orders_pkey", "users_pkey" } });

foreach (var row in cacheCheck)
    Console.WriteLine($"  {row.relname}: {row.buffers} buffers in cache");
```

#### c. 監控重啟後的緩存恢復

PostgreSQL 重啟後，shared_buffers 從零開始。可以透過以下步驟確保服務恢復後的效能：

```sql
-- Step 1: 重啟後立即檢查緩存填充率（PG 17）
SELECT * FROM pg_buffercache_summary();

-- PG 16- 等價寫法
SELECT count(*) AS cached_pages FROM pg_buffercache WHERE reldatabase IS NOT NULL;

-- Step 2: 執行 pg_prewarm 載入熱點表
SELECT pg_prewarm('orders', 'buffer');
SELECT pg_prewarm('users', 'buffer');

-- Step 3: 再次驗證熱點表已在緩存中
SELECT c.relname, count(*) AS buffers
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname IN ('orders', 'users')
  AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname;
```

> **補充（Senior Dev）**
> 在 connection pool 初始化時（而非每個 request 都做）執行 `pg_prewarm`，否則每次連線都 prewarm 完全沒意義。建議將預熱邏輯放在應用程式的 singleton 初始化階段，或放入 `pg_cron` 排程。若使用 PgBouncer transaction pooling，注意 `pg_prewarm` 必須在 transaction 內執行，且 session 級別的設定（如 `application_name`）無法跨 transaction 保留。

#### d. 緩存淘汰監控告警

加入緩存健康度的自動化檢查，當核心業務表的 `hot_pct` 低於閾值時觸發告警：

```sql
-- 核心業務表 orders 的 hot page 比例告警（hot_pct < 30%）
SELECT c.relname,
       count(*) AS total,
       count(*) FILTER (WHERE b.usagecount >= 4) AS hot,
       round(100.0 * count(*) FILTER (WHERE b.usagecount >= 4) / count(*), 2) AS hot_pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
  AND b.relforknumber = 0
  AND c.relname = 'orders'  -- 替換為你的核心業務表
GROUP BY c.relname
HAVING round(100.0 * count(*) FILTER (WHERE b.usagecount >= 4) / count(*), 2) < 30;
```

當 `hot_pct` 持續低於 30% 時，建議的排查順序：

1. 檢查 `shared_buffers` 大小是否合理 —— 建議值是系統總記憶體的 25%（上限約 8 ~ 16GB，視 PG 版本和硬體而定）
2. 檢查是否有大規模 Seq Scan 正在執行 —— Ring Buffer 策略可能抑制了這些 page 的 usagecount 增長
3. 檢查 `effective_cache_size` 設定是否被低估 —— 這影響查詢計劃器在 Index Scan vs Seq Scan 之間的選擇
4. 檢查是否有長 transaction 或 idle-in-transaction 導致的快照膨脹
5. 考慮使用 `pg_prewarm` 定期（如每小時）重新預熱關鍵表

> **補充（Senior Dev）**
> `shared_buffers` 的建議值不是越大越好。超過系統總記憶體的 25% ~ 40% 後，邊際效益遞減，甚至可能因為 Double Buffering（PG buffer + OS Page Cache 同時存一份）浪費記憶體。`pg_buffercache` 的診斷結論不總是「加大 shared_buffers」——有時問題根源在於 SQL 寫法（SELECT * 拖入不必要的 column → TOAST 資料）、缺少 WHERE 條件導致 Seq Scan、或索引設計不良導致掃描過多 page。在加大 shared_buffers 之前，先用 `pg_stat_statements` + `auto_explain` 確認 SQL 是否本身就有優化空間。
>
> **Npgsql 連線字串建議**：務必設定 `Application Name` 以便在 `pg_stat_activity` 中識別你的應用程式。這在排查緩存相關效能問題時至關重要，因為你可以快速區分哪些查詢來自你的 App、哪些來自其他服務或 DBA 工具：
>
> ```
> Host=db.example.com;Database=mydb;Username=app_user;Password=***;Application Name=MyWebApp_v2.1
> ```


## 5. btree_gin / btree_gist — 複合索引擴展

> **初學者導讀**
> PostgreSQL 內建的 GIN 和 GiST 索引各有自己的「母語操作符」（GIN 精通 `@>`、`@@` 等 Array/FTS 運算；GiST 精通 `&&`、`<<` 等幾何/範圍運算），但都不認 `=`、`<`、`>` 這些 B-Tree 家族的基礎操作符。`btree_gin` 和 `btree_gist` 這兩個 Extension 的作用就像「語言翻譯機」——教會 GIN / GiST 說 B-Tree 的語言，讓你在一**個**索引中同時支援全文檢索、模糊搜尋（Trigram LIKE）、等值比較、範圍查詢，甚至 EXCLUSION CONSTRAINT。

---

### I. 核心問題 — GIN 和 GiST 的內建限制

#### a. GIN 預設 Opclass 的能力邊界

GIN（Generalized Inverted Index）的設計初衷是處理「一個鍵對應多個行」的場景，例如陣列元素、全文檢索詞條、JSONB 鍵/值。內建的 Operator Class 只支援以下操作符：

| Opclass | 支援操作符 | 典型場景 |
|---------|-----------|---------|
| `array_ops` | `&&`（重疊）、`@>`（包含）、`<@`（被包含） | `WHERE tags @> ARRAY['urgent']` |
| `tsvector_ops` | `@@`（全文匹配） | `WHERE body @@ to_tsquery('postgres')` |
| `jsonb_ops` | `?`（鍵存在）、`?|`（任意鍵）、`?&`（全部鍵）、`@>`（包含） | `WHERE attrs @> '{"color":"red"}'` |
| `jsonb_path_ops` | `@>`（僅包含，不支援 `?/ ?| / ?&`） | 高效 JSONB 包含查詢 |

> **補充（Senior Dev）**
> `jsonb_path_ops` 比 `jsonb_ops` 索引更小（不為每個 Key 建立 Entry），專為 `@>` 運算優化。但如果你的查詢同時需要 `?`（檢查 Key 是否存在），`jsonb_ops` 才是正確選擇。兩者不可同時覆蓋同一個 JSONB 欄位——這是 GIN 只能有一個 opclass 的限制。

#### b. GiST 預設 Opclass 的能力邊界

GiST（Generalized Search Tree）定位於「空間 / 範圍型查詢」，內建支援：

| Opclass | 支援操作符 | 典型場景 |
|---------|-----------|---------|
| Geometric types（point, box, circle...） | `<<`, `>>`, `&&`, `&<`, `&>`, `@>`, `<@`, `~=` | 地圖範圍查詢 |
| Range types（int4range, tstzrange...） | `&&`, `@>`, `<@`, `<<`, `>>`, `-|-`, `&<`, `&>` | 時段重疊檢測 |
| `tsvector_ops` / `tsquery_ops` | `@@`（全文匹配） | FTS with ranking（`ts_rank`） |

**核心矛盾**：如果你需要一個 GIN 索引同時處理 `body @@ to_tsquery(...)` 和 `created_at >= '2026-01-01'`，PG 原生做不到——因為 GIN 不認識 `>=`。同理，GiST 不知道 `room_id = 101` 的 `=`。

```mermaid
flowchart TD
    Q["我需要一個複合索引<br/>同時支援 FTS + 範圍查詢！"] --> A{"只用 GIN？"}
    A -->|"✅ 支援 @@（FTS）"| B["✅ 支援 @>（陣列/JSONB）"]
    A -->|"❌ 不支援 =、<、<=、>、>="| C["❌ 無法過濾 created_at >= ..."]
    Q --> D{"只用 GiST？"}
    D -->|"✅ 支援 &&（範圍重疊）"| E["✅ 支援 @@（FTS with ranking）"]
    D -->|"❌ 不支援 =（整數等值）"| F["❌ 無法處理 room_id = 101"]
    Q --> G{"✅ 使用 btree_gin / btree_gist"}
    G --> H["一個索引同時支援<br/>FTS + 等值 + 範圍 + EXCLUSION"]

    style C fill:#e74c3c,color:#fff
    style F fill:#e74c3c,color:#fff
    style H fill:#2ecc71,color:#fff
```

#### c. 問題場景實例

```sql
-- 場景：電商產品表，需同時支援全文搜尋與價格/時間範圍過濾
CREATE TABLE products (
    id          bigserial PRIMARY KEY,
    name        text NOT NULL,
    description text,
    category    text NOT NULL,
    price       numeric(10,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);

-- 建立全文檢索用的 tsvector 欄位
ALTER TABLE products ADD COLUMN fts tsvector
    GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''))
    ) STORED;

-- 需求查詢：全文搜尋 + 種類過濾 + 價格範圍 + 時間排序
-- SELECT * FROM products
-- WHERE fts @@ to_tsquery('english', 'wireless & headphones')
--   AND category = 'Electronics'
--   AND price BETWEEN 50 AND 200
--   ORDER BY created_at DESC;
--
-- 原生 PG 困境：你必須在 fts 上建 GIN、在 category 上建 B-Tree、
-- 在 price 上建 B-Tree、在 created_at 上建 B-Tree → 4 個索引
```

> **補充（Senior Dev）**
> 多個單欄位 B-Tree 索引在 AND 組合條件下，Planner 通常只選一個最有效率的索引做 Bitmap Scan（而不是用多個）。但 `btree_gin` 的多欄位 GIN 索引能同時加速 ALL 欄位——因為 GIN 本質上是 inverted index，每個欄位的條件都會被用來過濾 bitmap，然後取交集。

---

### II. btree_gin — GIN 索引的 B-Tree 操作符擴展

#### a. 安裝與提供的能力

```sql
CREATE EXTENSION btree_gin;
```

這個 Extension 的核心職責：**為 GIN 索引註冊 B-Tree 等效的 Operator Class**，涵蓋以下型別：

| 型別分類 | 具體型別 |
|----------|---------|
| 整數 | `int2`, `int4`, `int8` |
| 浮點 | `float4`, `float8` |
| 數值 | `numeric`, `money`, `oid` |
| 時間 | `timestamp`, `timestamptz`, `date`, `time`, `timetz`, `interval` |
| 字串 | `text`, `varchar`, `char`, `bpchar` |
| 二進位 | `bytea`, `bit`, `varbit` |
| 網路 | `inet`, `cidr`, `macaddr` |
| 其他 | `uuid`, `name`, `bool` |

每個型別都註冊了經典 B-Tree 操作符：`<`, `<=`, `=`, `>=`, `>`。

> **補充（Senior Dev）**
> `btree_gin` 不對 `is`、`IS NOT`、`<>` 等操作符提供支援。如果需要 `<>` 過濾條件（排除特定值），GIN 並不擅長——GIN 的本質是「找包含這個 Key 的行」，而 `<>` 幾乎等價全表掃描。這種場景仍建議用獨立的 B-Tree 索引。

#### b. 核心優勢：多欄位 + 多型別混合 GIN 索引

這是 `btree_gin` 最具殺傷力的特性：**在一個 GIN 索引中混合不同型別、不同 Opclass 的多個欄位**。

```sql
-- 單一 GIN 索引同時覆蓋：Trigram LIKE（text）+ FTS（tsvector）+ B-Tree 操作符
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_products_multi ON products
    USING GIN (
        name gin_trgm_ops,            -- LIKE '%keyword%' 模糊搜尋
        fts,                           -- @@ 全文檢索（預設 tsvector_ops）
        category btree_gin_ops,        -- = 等值過濾
        price btree_gin_ops,           -- <= >= 範圍過濾
        created_at btree_gin_ops       -- <= >= 時間範圍 + ORDER BY 輔助
    );
```

**這個單一索引可以加速以下所有查詢模式**：

```sql
-- 查詢 1：純文字 LIKE
SELECT * FROM products WHERE name LIKE '%wireless%';

-- 查詢 2：全文檢索
SELECT * FROM products WHERE fts @@ to_tsquery('english', 'wireless');

-- 查詢 3：種類過濾
SELECT * FROM products WHERE category = 'Electronics';

-- 查詢 4：價格範圍
SELECT * FROM products WHERE price BETWEEN 50 AND 200;

-- 查詢 5：組合條件（全部命中同一個 GIN）
SELECT * FROM products
WHERE fts @@ to_tsquery('english', 'wireless & headphones')
  AND category = 'Electronics'
  AND price <= 200
  AND created_at >= '2026-01-01';

-- 查詢 6：排序（GIN 可輔助提供 bitmap，但 ORDER BY 仍需額外 Sort 節點）
SELECT * FROM products
WHERE price > 100
ORDER BY created_at DESC
LIMIT 20;
```

```mermaid
flowchart LR
    subgraph "傳統方案：4 個獨立索引"
        B1["idx_name_trgm<br/>GIN (name gin_trgm_ops)"]
        B2["idx_fts<br/>GIN (fts)"]
        B3["idx_category<br/>B-Tree (category)"]
        B4["idx_price<br/>B-Tree (price)"]
    end

    subgraph "btree_gin 方案：1 個複合 GIN"
        G1["idx_products_multi<br/>GIN (name, fts, category, price, created_at)"]
    end

    Q["查詢：WHERE fts @@ ...<br/>AND category = ...<br/>AND price < ..."]
    Q -->|"BitmapScan x 3<br/>BitmapAnd 合併"| B1
    Q --> B2
    Q --> B3
    Q -.->|"❌ price B-Tree<br/>無法與 GIN Bitmap 合併"| B4
    Q -->|"✅ 單次 Bitmap Index Scan<br/>GIN 內部做 Bitmap AND"| G1

    style B4 fill:#e74c3c,color:#fff
    style G1 fill:#2ecc71,color:#fff
```

> **初學者導讀**
> GIN 的內部結構是 inverted index（倒排索引）：對每個欄位的每個值建立 Entry → Posting List（包含的行號）。當多個欄位在同一個 GIN 索引中時，所有欄位的 Bitmap 都在 GIN 內部產生並取交集，Planner 只需做一次 Bitmap Index Scan。而多個獨立索引的 BitmapScan 需要 Planner 額外用 `BitmapAnd` 節點合併，且跨類型（GIN + B-Tree）的 bitmap 合併成本更高。

#### c. 效能注意事項

| 面向 | B-Tree 單欄位 | GIN + btree_gin |
|------|--------------|-----------------|
| 等值查詢（`col = 42`） | **極快**（B-Tree direct lookup） | 慢（GIN 需 scan posting list） |
| 範圍查詢（`col BETWEEN 1 AND 100`） | **快**（B-Tree range scan） | 中等（GIN 逐鍵掃描） |
| 多欄位 AND 組合 | Planner 選一個索引 + Filter | **快**（所有欄位都在同一 GIN） |
| INSERT 效能 | **快** | 慢（GIN 寫入需更新多個 inverted list） |
| 索引大小 | 小 | 大（GIN inverted list + 多欄位） |
| VACUUM 需求 | 低 | 高（GIN pending list 需定期清理） |

```sql
-- 驗證：等值查詢在 B-Tree vs GIN 的效能差異
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE category = 'Electronics';
-- 使用 B-Tree: Index Scan（直接跳到 leaf page）
-- 使用 GIN: Bitmap Index Scan（build bitmap → bitmap heap scan）

-- 驗證：多欄位 AND 在 GIN 中的優勢
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products
WHERE fts @@ to_tsquery('english', 'wireless')
  AND category = 'Electronics'
  AND price <= 200;
-- 使用複合 GIN: Single Bitmap Index Scan（內部處理所有條件）
-- 使用多個 B-Tree: 2-3 個 BitmapAnd 節點（多輪 bitmap 合併）
```

> **補充（Senior Dev）**
> GIN 的 `fastupdate` 機制（預設開啟，由 `gin_pending_list_limit` 控制）會把新插入的資料先放入 pending list，pending list 滿時才合併到主 GIN 結構。在高寫入頻率的表上，pending list 過大會拖慢 SELECT（因為掃描 pending list 是順序的）。透過 `gin_clean_pending_list()` 函數或調整 `gin_pending_list_limit` 可以緩解。如果你的場景是**高寫入 + 高讀取**，且只用於等值/範圍查詢，B-Tree 仍然是最佳選擇。

---

### III. btree_gist — GiST 索引的 B-Tree 操作符擴展

#### a. 安裝與提供的能力

```sql
CREATE EXTENSION btree_gist;
```

與 `btree_gin` 對稱，`btree_gist` 為 GiST 索引註冊 B-Tree 等效 Operator Class，支援型別幾乎相同。但 GiST 的天生優勢在於 **EXCLUSION CONSTRAINT**。

#### b. 核心殺手功能：EXCLUSION CONSTRAINT

這是 `btree_gist` 最具價值的場景——它讓 GiST 能處理**任意型別的等值操作符**，從而實現跨型別的排他約束。

沒有 `btree_gist` 時：
```sql
-- ❌ 報錯！GiST 不支援 integer 的 =
CREATE TABLE room_bookings (
    room_id   int,
    during    tstzrange,
    EXCLUDE USING GiST (room_id WITH =, during WITH &&)
);
-- ERROR: data type integer has no default operator class for access method "gist"
```

安裝 `btree_gist` 後：
```sql
CREATE EXTENSION btree_gist;

-- ✅ 成功！btree_gist 教會 GiST 怎麼處理 integer 的 =
CREATE TABLE room_bookings (
    id        bigserial PRIMARY KEY,
    room_id   int NOT NULL,
    guest_name text NOT NULL,
    during    tstzrange NOT NULL,
    EXCLUDE USING GiST (
        room_id WITH =,        -- 同一個 room
        during  WITH &&         -- 時間重疊
    )
);
-- 語義：不允許「同一個房間 + 時段重疊」的預訂同時存在

-- 驗證 EXCLUSION CONSTRAINT
INSERT INTO room_bookings (room_id, guest_name, during)
VALUES (101, 'Alice', '[2026-06-01 14:00, 2026-06-01 16:00)');
-- ✅ 成功

INSERT INTO room_bookings (room_id, guest_name, during)
VALUES (101, 'Bob', '[2026-06-01 15:00, 2026-06-01 17:00)');
-- ❌ ERROR: conflicting key value violates exclusion constraint
```

```mermaid
sequenceDiagram
    participant App as 應用程式
    participant PG as PostgreSQL
    participant GiST as btree_gist GiST 索引

    App->>PG: INSERT ROOM 101 14:00-16:00
    PG->>GiST: room_id = 101 AND during && [14:00, 16:00)
    GiST->>GiST: 掃描現有條目<br/>（一樣的 room_id + 時間重疊）
    GiST-->>PG: 無衝突
    PG-->>App: ✅ INSERT 成功

    App->>PG: INSERT ROOM 101 15:00-17:00
    PG->>GiST: room_id = 101 AND during && [15:00, 17:00)
    GiST->>GiST: 找到衝突：<br/>Alice 的 [14:00, 16:00) 與 15:00-17:00 重疊
    GiST-->>PG: ⚠️ 衝突！
    PG-->>App: ❌ EXCLUSION CONSTRAINT 違反
```

#### c. 更多 EXCLUSION CONSTRAINT 場景

```mermaid
flowchart TD
    A["EXCLUSION CONSTRAINT<br/>+ btree_gist 組合場景"] --> B["飯店預訂系統<br/>room_id = + during &&"]
    A --> C["IP 位址分配<br/>vlan_id = + ip_range &&"]
    A --> D["會議室排程<br/>room = + tsrange &&"]
    A --> E["資源預約<br/>resource_id = + during &&<br/>+ status = (僅 active)"]
    A --> F["班表排程<br/>employee_id = + shift &&"]

    B --> B1["同一房間不可重疊預訂"]
    C --> C1["同 VLAN 內 IP 範圍不可重疊"]
    D --> D1["同一會議室不可同時段預約"]
    E --> E1["同一資源在被取消前不可重疊"]
    F --> F1["同一員工不可排重疊班次"]

    style A fill:#4a90d9,color:#fff
    style B fill:#2ecc71,color:#fff
    style C fill:#2ecc71,color:#fff
    style D fill:#2ecc71,color:#fff
    style E fill:#2ecc71,color:#fff
    style F fill:#2ecc71,color:#fff
```

**場景實作 1：IP 位址分配（避免 IP 範圍重疊）**

```sql
CREATE TABLE ip_allocations (
    id          bigserial PRIMARY KEY,
    vlan_id     int NOT NULL,
    ip_range    inet NOT NULL,    -- 如 '10.0.1.0/28'
    assigned_to text NOT NULL,
    EXCLUDE USING GiST (
        vlan_id  WITH =,
        inet_ops(ip_range) WITH &&
    )
);
-- 注意：inet 需要搭配 btree_gist + inet_ops（btree_gist 提供 =，GiST 內建 && for inet）
```

**場景實作 2：不重疊排程（含 WHERE 條件過濾）**

```sql
CREATE TABLE reservations (
    id          bigserial PRIMARY KEY,
    resource_id int NOT NULL,
    user_id     int NOT NULL,
    during      tstzrange NOT NULL,
    status      text NOT NULL DEFAULT 'active',
    EXCLUDE USING GiST (
        resource_id WITH =,
        during WITH &&
    ) WHERE (status = 'active')   -- 只有 active 的預訂才檢查排他性
);
-- 這樣取消的預訂（status = 'cancelled'）不會阻擋新預訂
```

> **補充（Senior Dev）**
> `EXCLUDE USING GiST` 的 `WHERE` 子句是局部索引（Partial Index）的宣告式語法。它只對滿足 WHERE 條件的行建立索引條目，這有兩個好處：(1) 被取消的預訂不佔用索引空間，(2) 檢查排他性時自動跳過不相關行。但注意：`WHERE` 條件中的欄位必須是 `IMMUTABLE` 函數才能用於索引（`status = 'active'` 是等值比較，符合要求）。

#### d. KNN 距離搜尋場景

GiST 的另一個獨特能力是 K-Nearest Neighbor（KNN）搜尋。結合 `btree_gist`，可以實現「時間範圍 + KNN 排序」的複合查詢：

```sql
-- 找出與指定時間點最接近的 10 筆記錄
SELECT * FROM events
WHERE event_type = 'Meeting'
ORDER BY event_range <-> tstzrange('2026-06-01 15:00:00', '2026-06-01 15:00:00')
LIMIT 10;
-- <-> 是 GiST 的 distance operator
```

---

### IV. btree_gin vs btree_gist 選擇指南

#### a. 決策核心原則

```mermaid
flowchart TD
    Q["我需要在單一索引中<br/>混合 B-Tree 操作符？"] --> A{"主要查詢模式？"}

    A -->|"FTS + LIKE +<br/>等值 + 範圍"| B["✅ GIN + btree_gin<br/>多欄位反向索引<br/>適合讀多寫少"]
    A -->|"EXCLUSION CONSTRAINT<br/>+ 等值"| C["✅ GiST + btree_gist<br/>唯一支援排他約束"]
    A -->|"範圍重疊為主<br/>+ KNN 排序"| D["✅ GiST + btree_gist<br/>天然支援 <-> 距離"]
    A -->|"純等值查詢<br/>單一欄位"| E["❌ 不要用 btree_gin/btree_gist<br/>用原生 B-Tree"]

    style B fill:#2ecc71,color:#fff
    style C fill:#2ecc71,color:#fff
    style D fill:#2ecc71,color:#fff
    style E fill:#e74c3c,color:#fff
```

#### b. 效能特性對比表

| 維度 | B-Tree（原生） | GIN + btree_gin | GiST + btree_gist |
|------|---------------|-----------------|-------------------|
| 等值查詢速度 | ★★★★★ 最快 | ★★☆☆☆ 需掃描 posting list | ★★★☆☆ 平衡 |
| 範圍查詢速度 | ★★★★☆ range scan | ★★★☆☆ 逐鍵合併 bitmap | ★★★☆☆ 樹狀遍歷 |
| 多欄位 AND 組合 | ★★☆☆☆ 需 BitmapAnd | ★★★★★ 內部 bitmap 取交集 | ★★★★☆ 樹內複合鍵 |
| INSERT 速度 | ★★★★★ 最快 | ★★☆☆☆ 需更新多個 inverted list | ★★★☆☆ 平衡 |
| UPDATE 速度 | ★★★★★ 最快 | ★☆☆☆☆ 需刪舊 + 插新 inverted list | ★★★☆☆ 平衡 |
| 索引大小 | ★★★★★ 最小 | ★★★☆☆ 倒排結構膨脹 | ★★★★☆ 小於 GIN |
| VACUUM 負擔 | ★★★★★ 幾乎無 | ★★☆☆☆ pending list 需維護 | ★★★☆☆ 需要 vacuum |
| EXCLUSION CONSTRAINT | ❌ 不支援 | ❌ 不支援 | ✅ 唯一方案 |
| KNN 排序 | ❌ 不支援 | ❌ 不支援 | ✅ 內建支援 |

#### c. 場景決策表

| 場景 | 推薦方案 | 理由 |
|------|---------|------|
| 單一欄位等值查詢 | B-Tree | GIN 是殺雞用牛刀 |
| 全文檢索 + 種類過濾 + 價格範圍 | **GIN + btree_gin** | 多欄位 AND 在 GIN 內部最優 |
| 飯店/會議室不重疊預訂 | **GiST + btree_gist** | EXCLUSION CONSTRAINT 唯一方案 |
| IP 位址分配不重疊 | **GiST + btree_gist** | EXCLUSION CONSTRAINT + inet ops |
| 模糊搜尋 + 時間排序 | **GIN + btree_gin** | Trigrams + timestamptz 在同一個 GIN |
| 時間序列資料（讀多寫少） | B-Tree（分區後） | 原生分區 + B-Tree 比 GIN 更快 |
| 幾何空間 KNN | **GiST + btree_gist** | 原生 KNN 支援 |

> **補充（Senior Dev）**
> `btree_gin` 和 `btree_gist` 是**零配置 Extension**——執行 `CREATE EXTENSION` 後立刻生效，沒有額外的 GUC 參數需要調整。它們僅在 DDL 層面為 GiST/GIN 註冊新的 Operator Class，不會對現有查詢產生任何副作用。但在 `pg_dump` 和 `pg_restore` 時要注意：還原 database 時需先確保這兩個 Extension 已安裝，否則依賴其 opclass 的索引會建立失敗。

---

### V. 實戰範例場景

#### a. 場景 1（btree_gin）：電商產品搜尋引擎

這是 `btree_gin` 最典型的生產案例——一個索引搞定全文檢索 + 分類過濾 + 價格範圍 + 時間排序。

```sql
-- Step 1：安裝 Extension
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Step 2：建立產品表
CREATE TABLE products (
    id          bigserial PRIMARY KEY,
    name        text NOT NULL,
    description text,
    category    text NOT NULL,
    brand       text NOT NULL,
    price       numeric(10,2) NOT NULL,
    rating      numeric(3,2) DEFAULT 0,
    created_at  timestamptz NOT NULL DEFAULT now(),
    fts         tsvector GENERATED ALWAYS AS (
        to_tsvector('english',
            coalesce(name, '') || ' ' ||
            coalesce(description, '') || ' ' ||
            coalesce(brand, '')
        )
    ) STORED
);

-- Step 3：建立複合 GIN（核心！）
CREATE INDEX idx_products_search ON products
    USING GIN (
        name gin_trgm_ops,          -- LIKE '%keyword%'
        fts,                         -- @@ to_tsquery
        category btree_gin_ops,      -- = 等值
        brand btree_gin_ops,         -- = 等值
        price btree_gin_ops,         -- <= >= 範圍
        rating btree_gin_ops,        -- <= >= 範圍
        created_at btree_gin_ops     -- <= >= 範圍
    );

-- Step 4：驗證查詢
-- 查詢 A：全文 + 分類 + 價格 + 排序（四個條件全中 GIN）
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price, rating
FROM products
WHERE fts @@ to_tsquery('english', 'wireless & headphones')
  AND category = 'Electronics'
  AND price BETWEEN 50 AND 300
ORDER BY rating DESC
LIMIT 20;
-- 預期：Bitmap Index Scan on idx_products_search → Bitmap Heap Scan

-- 查詢 B：模糊搜尋 + 品牌
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price
FROM products
WHERE name LIKE '%Sony%'
  AND brand = 'Sony'
  AND price < 500;
-- 預期：同上，所有條件在 GIN 內部合併

-- 查詢 C：只用分類（等值）
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM products WHERE category = 'Electronics';
-- 預期：Bitmap Index Scan（GIN），但如果是極高頻率查詢，獨立的 B-Tree 會更快
```

> **補充（Senior Dev）**
> 如果 `category` 只有 5 種值（Cardinality = 5），在 GIN 中對 `category` 建立 Entry 的效果有限——因為幾乎每個 Bitmap 都是全表的 20%。此時將 `category` 從 GIN 中移除、單獨建 B-Tree 可能更優。GIN 中放高 Cardinality 欄位效果最好；低 Cardinality 欄位僅在與高 Cardinality 欄位做 AND 時有意義（縮小 Bitmap）。

#### b. 場景 2（btree_gist）：飯店預訂系統

```sql
-- Step 1：安裝
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Step 2：建立預訂表（含 EXCLUSION CONSTRAINT）
CREATE TABLE hotel_bookings (
    id          bigserial PRIMARY KEY,
    hotel_id    int NOT NULL,
    room_id     int NOT NULL,
    guest_name  text NOT NULL,
    during      tstzrange NOT NULL,
    status      text NOT NULL DEFAULT 'confirmed',
    created_at  timestamptz NOT NULL DEFAULT now(),
    -- 核心約束：同一飯店 + 同一房間 + 時段不可重疊
    EXCLUDE USING GiST (
        hotel_id WITH =,
        room_id  WITH =,
        during   WITH &&
    ) WHERE (status = 'confirmed')
);

-- Step 3：插入測試資料
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES
    (1, 301, 'Alice',   '[2026-06-10 14:00, 2026-06-12 11:00)'),
    (1, 301, 'Bob',     '[2026-06-12 14:00, 2026-06-14 11:00)'),  -- ✅ 不重疊
    (1, 302, 'Charlie', '[2026-06-10 14:00, 2026-06-12 11:00)'),  -- ✅ 不同房間
    (2, 101, 'Diana',   '[2026-06-10 14:00, 2026-06-12 11:00)');  -- ✅ 不同飯店

-- Step 4：嘗試重疊插入
INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
VALUES (1, 301, 'Eve', '[2026-06-11 12:00, 2026-06-13 11:00)');
-- ❌ ERROR: conflicting key value violates exclusion constraint
-- 原因：hotel_id=1, room_id=301 的 Alice [10-12] 與 Eve [11-13] 在 6/11~6/12 重疊

-- Step 5：查詢某時段有空的房間（利用 GiST && 操作符）
SELECT room_id
FROM hotel_bookings
WHERE hotel_id = 1
  AND status = 'confirmed'
  AND during && tstzrange('[2026-06-11 14:00, 2026-06-12 11:00)');
-- 返回 room_id=301（表示此時段已有人預訂）
-- 取 NOT EXISTS 即為可用房間
```

```mermaid
gantt
    title 飯店 Room 301 預訂時間線
    dateFormat YYYY-MM-DD HH:mm
    axisFormat %m/%d %H:%M

    section Room 301
    Alice  [10-12]      :a1, 2026-06-10 14:00, 2026-06-12 11:00
    Bob    [12-14]      :a2, 2026-06-12 14:00, 2026-06-14 11:00
    Eve ❌衝突          :crit, a3, 2026-06-11 12:00, 2026-06-13 11:00

    section Room 302
    Charlie [10-12]      :b1, 2026-06-10 14:00, 2026-06-12 11:00
```

#### c. 場景 3（btree_gist）：IP 位址池管理

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE ip_pools (
    id          bigserial PRIMARY KEY,
    vlan_id     int NOT NULL,
    subnet      cidr NOT NULL,
    purpose     text NOT NULL,
    EXCLUDE USING GiST (
        vlan_id WITH =,
        inet_ops(subnet) WITH &&
    )
);

INSERT INTO ip_pools (vlan_id, subnet, purpose)
VALUES (10, '10.0.1.0/25', 'Web Servers'),
       (10, '10.0.1.128/26', 'DB Servers'), -- ✅ 不重疊
       (20, '10.0.1.0/25', 'Dev VLAN');     -- ✅ 不同 VLAN

-- 重疊測試
INSERT INTO ip_pools (vlan_id, subnet, purpose)
VALUES (10, '10.0.1.64/26', 'Cache Servers');
-- ❌ ERROR: conflicting key value
-- 原因：VLAN 10 已有 10.0.1.0/25 (IP: 10.0.1.0-10.0.1.127)
--       與 10.0.1.64/26 (IP: 10.0.1.64-10.0.1.127) 重疊
```

---

### VI. App Dev 最佳實踐

#### a. 何時不該用 btree_gin

```sql
-- ❌ 錯誤：用 GIN 取代單欄位 B-Tree
CREATE INDEX idx_users_email ON users USING GIN (email btree_gin_ops);
-- GIN 的等值查詢（WHERE email = 'x@y.com'）比 B-Tree 慢 3-10 倍

-- ✅ 正確：單欄位等值用 B-Tree
CREATE INDEX idx_users_email ON users (email);

-- ❌ 錯誤：頻繁 UPDATE 的表用 GIN
-- GIN 的 UPDATE = DELETE old entry + INSERT new entry（倒排結構更新成本高）

-- ✅ 正確：高寫入頻率表用 B-Tree 或考慮 GiST
```

#### b. 何時該用 btree_gin

**三條件的交集**：
1. 需要多欄位 AND 組合查詢（3 個欄位以上）
2. 至少一個欄位使用 GIN 原生 opclass（`gin_trgm_ops`、`tsvector_ops`、`jsonb_ops`）
3. 讀取遠多於寫入（SELECT : INSERT > 10:1）

#### c. .NET / Npgsql 整合注意事項

```csharp
// ✅ 正確：利用 btree_gin 複合索引的查詢
using var conn = new NpgsqlConnection(connectionString);
conn.Open();

const string sql = @"
    SELECT id, name, price
    FROM products
    WHERE fts @@ to_tsquery('english', @query)
      AND category = @category
      AND price <= @maxPrice
    ORDER BY rating DESC
    LIMIT 20";

await using var cmd = new NpgsqlCommand(sql, conn);
cmd.Parameters.AddWithValue("query", "wireless & headphones");
cmd.Parameters.AddWithValue("category", "Electronics");
cmd.Parameters.AddWithValue("maxPrice", 300m);

// Dapper 參數化：pg_stat_statements 中會歸一化為 $1, $2, $3
// 參數型別由 Npgsql 自動推斷，確保與 btree_gin opclass 型別一致
```

```csharp
// ✅ 正確：利用 btree_gist EXCLUSION CONSTRAINT 的 INSERT
const string insertSql = @"
    INSERT INTO hotel_bookings (hotel_id, room_id, guest_name, during)
    VALUES (@hotelId, @roomId, @guestName, @during)
    ON CONFLICT ON CONSTRAINT hotel_bookings_hotel_id_room_id_during_excl
    DO NOTHING
    RETURNING id";

await using var cmd = new NpgsqlCommand(insertSql, conn);
cmd.Parameters.AddWithValue("hotelId", 1);
cmd.Parameters.AddWithValue("roomId", 301);
cmd.Parameters.AddWithValue("guestName", "Alice");
cmd.Parameters.AddWithValue("during", new NpgsqlRange<DateTime>(
    new DateTime(2026, 6, 10, 14, 0, 0),
    new DateTime(2026, 6, 12, 11, 0, 0)
));

try
{
    var result = await cmd.ExecuteScalarAsync();
    // result != null → 插入成功
    // result == null → 衝突（ON CONFLICT DO NOTHING）
}
catch (PostgresException ex) when (ex.SqlState == "23P01")
{
    // 23P01 = exclusion_violation（如果不用 ON CONFLICT）
    Console.WriteLine($"預訂衝突：{ex.MessageText}");
}
```

> **補充（Senior Dev）**
> Npgsql 6.0+ 支援 `conn.ProcessID` 屬性，對應 PostgreSQL 的 `pg_backend_pid()`。在 Application Log 中記錄此 ID，便於事後在 `pg_stat_activity` 或 `pg_locks` 中回溯問題請求。這在 EXCLUSION CONSTRAINT 衝突排查中尤為有用——可以看到是哪個 PID 持有衝突的鎖。
>
> ```csharp
> Console.WriteLine($"Backend PID: {conn.ProcessID}");
> ```

#### d. 效能基準測試建議

```sql
-- 建立測試基準：對比 GIN vs B-Tree 在不同查詢模式下的效能
SET enable_seqscan = off;  -- 僅強制索引掃描用於測試，生產環境勿設定

-- 測試 1：等值查詢
EXPLAIN (ANALYZE, TIMING, BUFFERS)
SELECT * FROM products WHERE category = 'Electronics';
-- 分別在有 B-Tree 和 GIN (btree_gin) 的情況下執行並對比

-- 測試 2：多欄位 AND
EXPLAIN (ANALYZE, TIMING, BUFFERS)
SELECT * FROM products
WHERE fts @@ to_tsquery('english', 'wireless')
  AND category = 'Electronics'
  AND price <= 200;
-- 比較：(A) 3 個獨立索引 + BitmapAnd vs (B) 單一複合 GIN

-- 測試 3：INSERT 效能
\timing on
INSERT INTO products (name, description, category, brand, price)
SELECT
    'Product ' || gs,
    'Description for product ' || gs,
    (ARRAY['Electronics', 'Books', 'Clothing', 'Sports'])[floor(random() * 4 + 1)],
    (ARRAY['Sony', 'Samsung', 'Nike', 'Adidas'])[floor(random() * 4 + 1)],
    (random() * 1000)::numeric(10,2)
FROM generate_series(1, 10000) AS gs;
-- 分別測有/無複合 GIN 時的 INSERT 速度
```

> **補充（Senior Dev）**
> `btree_gin` 和 `btree_gist` 都是 `contrib` 模組，內建於 PostgreSQL 發行版（無需 `apt install` 額外套件）。它們是**輕量級 Extension**——僅在 `pg_opclass` 系統表中註冊新條目，不會載入 shared library、不會啟動 Background Worker、不消耗額外記憶體。在生產環境中啟用幾乎零風險。唯一的「成本」是：如果你的 DBA 不知情地在使用 `pg_dump` + `pg_restore` 遷移資料庫，目標庫需要先 `CREATE EXTENSION btree_gin / btree_gist`，否則依賴其 opclass 的索引無法建立。

