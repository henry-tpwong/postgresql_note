# PostgreSQL pg_shard — 資料庫分片（Sharding）

> 來源：[digoal - pg_shard PostgreSQL 數據庫分片 (2015-09-28)](https://github.com/digoal/blog/blob/master/201509/20150928_01.md)
>
> 官方：[CitusData / pg_shard v1.2.2](https://github.com/citusdata/pg_shard/tree/v1.2.2)

---

> 補充（Senior Dev）：pg_shard 是 Citus 的前身。2016 年 CitusData 將 pg_shard 合併進 Citus extension（開源版），Citus 現已成為 PostgreSQL 生態最成熟的分佈式方案。pg_shard 的架構（hash sharding + metadata table + master_copy_shard_placement）基本奠定了 Citus 的核心設計。本文整理的安裝流程和限制對理解 Citus 底層仍有參考價值。現代部署中 pg_shard 已被 Citus 取代，不建議在新項目使用。

---

## 1. 環境準備與安裝

### GCC 版本要求

pg_shard 使用了 GCC 4.6+ 的特性。若系統 GCC < 4.6，需先安裝高版本：

```bash
yum install -y gmp mpfr libmpc libmpc-devel

wget http://gcc.cybermirror.org/releases/gcc-4.9.3/gcc-4.9.3.tar.bz2
tar -jxvf gcc-4.9.3.tar.bz2
cd gcc-4.9.3
./configure --prefix=/opt/gcc4.9.3
make && make install

# 配置 linker 路徑
vi /etc/ld.so.conf
# 加入:
/opt/gcc4.9.3/lib
/opt/gcc4.9.3/lib64

ldconfig
ldconfig -p | grep gcc

# 配置 PATH
vi /etc/profile
# 加入:
export PATH=/opt/gcc4.9.3/bin:$PATH
```

### 安裝 pg_shard v1.2.2

```bash
git clone https://github.com/citusdata/pg_shard.git
cd pg_shard/
git checkout master  # v1.2.2 at commit 7e6103f

. /home/postgres/.bash_profile
make clean; make; make install
```

> 補充（Senior Dev）：現代 Citus 安裝只需 `apt install postgresql-17-citus` 或 `CREATE EXTENSION citus;`，不再需要手動編譯。但 pg_shard 時期的手編流程對理解 extension 載入機制有幫助——`shared_preload_libraries` 中載入的 extension 會在 PostgreSQL 啟動時掛載 hook（planner hook / executor hook），這是 Citus 能攔截並重寫 distributed query 的技術基礎。

---

## 2. 叢集配置

### 拓撲：1 Master + 4 Worker

```bash
# 在 Master 的 $PGDATA 中建立 worker 列表
vi pg_worker_list.conf
```

```
localhost 1922
localhost 1923
localhost 1924
localhost 1925
```

### 確保連線無密碼

所有 worker 節點的 `pg_hba.conf` 對 local 連線設為 trust：

```
local   all   all   trust
```

### Master 端載入 pg_shard

```bash
vi $PGDATA/postgresql.conf
# 加入:
shared_preload_libraries = 'pg_shard'

pg_ctl restart -m fast
```

### 建立分佈式環境

在所有節點（master + workers）建立一致的 role、database、schema。然後在 master 建立 extension：

```sql
CREATE EXTENSION pg_shard;
```

---

## 3. 建立分佈式表

### 建立測試表

```sql
CREATE TABLE customer_reviews
(
    customer_id TEXT NOT NULL,
    review_date DATE,
    review_rating INTEGER,
    review_votes INTEGER,
    review_helpful_votes INTEGER,
    product_id CHAR(10),
    product_title TEXT,
    product_sales_rank BIGINT,
    product_group TEXT,
    product_category TEXT,
    product_subcategory TEXT,
    similar_product_ids CHAR(10)[]
);
```

> 建議：在定義 worker table 前將所有 index、constraint 確定好。後續添加 index 需在所有 worker 節點手動執行。

### 設為分佈式表（Hash Sharding on customer_id）

```sql
SELECT master_create_distributed_table('customer_reviews', 'customer_id');
```

### 在 Worker 節點建立 Shard

16 個 shard，每個 shard 2 個 replica：

```sql
SELECT master_create_worker_shards('customer_reviews', 16, 2);
```

### 查看 Metadata

```sql
\dt pgs_distribution_metadata.*
```

```
         Schema           |      Name       | Type  |  Owner
---------------------------+-----------------+-------+----------
 pgs_distribution_metadata | partition       | table | postgres
 pgs_distribution_metadata | shard           | table | postgres
 pgs_distribution_metadata | shard_placement | table | postgres
(3 rows)
```

**partition 表**（記錄分佈方法）：

```sql
SELECT * FROM pgs_distribution_metadata.partition;
```

```
 relation_id | partition_method |     key
-------------+------------------+-------------
       42067 | h                | customer_id
(1 row)
```

```sql
SELECT 42067::regclass;  -- customer_reviews
```

**shard 表**（記錄每個 shard 的 hash range，int32 空間均分為 16 段）：

```sql
SELECT * FROM pgs_distribution_metadata.shard;
```

```
  id   | relation_id | storage |  min_value  |  max_value
-------+-------------+---------+-------------+-------------
 10000 |       42067 | t       | -2147483648 | -1879048194
 10001 |       42067 | t       | -1879048193 | -1610612739
 10002 |       42067 | t       | -1610612738 | -1342177284
 10003 |       42067 | t       | -1342177283 | -1073741829
 10004 |       42067 | t       | -1073741828 | -805306374
 10005 |       42067 | t       | -805306373  | -536870919
 10006 |       42067 | t       | -536870918  | -268435464
 10007 |       42067 | t       | -268435463  | -9
 10008 |       42067 | t       | -8          | 268435446
 10009 |       42067 | t       | 268435447   | 536870901
 10010 |       42067 | t       | 536870902   | 805306356
 10011 |       42067 | t       | 805306357   | 1073741811
 10012 |       42067 | t       | 1073741812  | 1342177266
 10013 |       42067 | t       | 1342177267  | 1610612721
 10014 |       42067 | t       | 1610612722  | 1879048176
 10015 |       42067 | t       | 1879048177  | 2147483647
(16 rows)
```

> 分佈方式：對 `customer_id` 做 hash → 映射到 int32 空間，按 range 均分到 16 個 shard。

**shard_placement 表**（記錄每個 shard 的 replica 位置，32 row = 16 shard × 2 replica）：

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
```

```
 id | shard_id | shard_state | node_name | node_port
----+----------+-------------+-----------+-----------
  1 |    10000 |           1 | localhost |      1922
  2 |    10000 |           1 | localhost |      1923
  3 |    10001 |           1 | localhost |      1923
  4 |    10001 |           1 | localhost |      1924
  5 |    10002 |           1 | localhost |      1924
  6 |    10002 |           1 | localhost |      1925
  7 |    10003 |           1 | localhost |      1925
  8 |    10003 |           1 | localhost |      1922
  9 |    10004 |           1 | localhost |      1922
 10 |    10004 |           1 | localhost |      1923
 11 |    10005 |           1 | localhost |      1923
 12 |    10005 |           1 | localhost |      1924
 13 |    10006 |           1 | localhost |      1924
 14 |    10006 |           1 | localhost |      1925
 15 |    10007 |           1 | localhost |      1925
 16 |    10007 |           1 | localhost |      1922
 17 |    10008 |           1 | localhost |      1922
 18 |    10008 |           1 | localhost |      1923
 19 |    10009 |           1 | localhost |      1923
 20 |    10009 |           1 | localhost |      1924
 21 |    10010 |           1 | localhost |      1924
 22 |    10010 |           1 | localhost |      1925
 23 |    10011 |           1 | localhost |      1925
 24 |    10011 |           1 | localhost |      1922
 25 |    10012 |           1 | localhost |      1922
 26 |    10012 |           1 | localhost |      1923
 27 |    10013 |           1 | localhost |      1923
 28 |    10013 |           1 | localhost |      1924
 29 |    10014 |           1 | localhost |      1924
 30 |    10014 |           1 | localhost |      1925
 31 |    10015 |           1 | localhost |      1925
 32 |    10015 |           1 | localhost |      1922
(32 rows)
```

- `shard_state = 1` 表示正常
- replica 分佈策略確保任意 worker 掛掉時，每個 shard 仍有一個健康副本在其他 worker

---

## 4. 限制（Limitations）

### 4.1 不支援子查詢

```sql
INSERT INTO customer_reviews SELECT generate_series(1, 100);
-- ERROR: 0A000: cannot perform distributed planning for the given query
-- DETAIL: Subqueries are not supported in distributed queries.
```

### 4.2 不支援 non-constant expression / 變數

```sql
INSERT INTO customer_reviews VALUES ('a', now());
-- ERROR: 0A000: cannot plan sharded modification containing values
-- which are not constants or constant expressions
```

### 4.3 不支援 Prepared Statement / Bind Variable

```sql
PREPARE a (text) AS INSERT INTO customer_reviews VALUES ($1);
-- ERROR: 0A000: PREPARE commands on distributed tables are unsupported
```

使用 `pgbench -M extended`（extended query protocol）會直接失敗：

```
Client 2 aborted in state 1: ERROR: unrecognized node type: 2100
```

必須使用 `-M simple`（simple query protocol）才能正常寫入。

### 4.4 效能測試

`pgbench -M simple` 寫入測試（8 client / 8 thread / 1000s）：

```
progress: 2.8 s, 0.7 tps,   lat 2568.361 ms
progress: 3.2 s, 68.3 tps,  lat 578.633 ms
progress: 3.2 s, 264.5 tps, lat 8.263 ms
progress: 4.0 s, 1193.6 tps, lat 8.561 ms
progress: 5.0 s, 1255.6 tps, lat 6.376 ms
progress: 6.0 s, 1277.5 tps, lat 6.263 ms
```

暖機後穩定在 ~1200-1400 tps。

### 4.5 完整限制清單

**中長期（架構決定，不打算支援）：**

- 跨 shard 的分散式 transaction（如 A 轉帳到 B，shard key 不同）
- 非 distribution column 的 Unique constraint / Foreign Key constraint
- 分散式 JOIN

> 原文官方說明：If you'd like to run complex analytic queries, please consider upgrading to CitusDB.

**短期（未來版本預計支援）：**

- 不支援 DDL（ALTER TABLE）：需手動在所有 worker 節點 propagate
- 不支援 DROP TABLE（未來版本將加入 shard cleanup command）
- 不支援 `INSERT INTO ... SELECT ...`

**單點瓶頸：Master 無法做對等備援**

Master 同時處理查詢路由和 metadata 管理，容易成為 network bandwidth 和 CPU 瓶頸。德哥認為連接池代理（connection pool proxy）方案更好——輕量、易於對等部署、效能線性增長。

> 德哥觀點：JDBC 9.4（支援 load balance + failover）+ plproxy 可完美實現真正效能線性增長的資料庫分片，前提是不需要跨庫 transaction / JOIN / 唯一約束。
>
> 參考：[JDBC 9.4 load balancing and failover](http://blog.163.com/digoal@126/blog/static/16387704020158241250463/)

### 4.6 官方 Limitations 原文

> pg_shard is intentionally limited in scope during its first release, but is fully functional within that scope. We classify pg_shard's current limitations into two groups:
>
> **Medium-term (architectural):**
> - Transactional semantics for queries that span across multiple shards
> - Unique constraints on columns other than the partition key, or foreign key constraints
> - Distributed JOINs — If you'd like to run complex analytic queries, please consider upgrading to CitusDB
>
> **Short-term:**
> - Table alterations: customers accomplish them by using a script that propagates such changes to all worker nodes
> - DROP TABLE does not have any special semantics on a distributed table. An upcoming release will add a shard cleanup command
> - Queries such as `INSERT INTO foo SELECT bar, baz FROM qux` are not supported
>
> We keep an open discussion on GitHub issues to hear what you have to say.

> 補充（Senior Dev）：這些限制大部分在 Citus 6.0+ 已解決。分散式 JOIN 在 Citus 中通過 co-located join（相同 distribution column 的表可本地 JOIN）和 broadcast join（小表廣播到所有 worker）實現；分散式 transaction 通過 2PC 實現；subquery 在 Citus 的 adaptive executor 中支援。但 single coordinator 瓶頸在 Citus 中仍然存在——大規模部署需要 Citus MX（每個 worker 也可接受查詢）或 Citus 11+ 的 "query from any node" 特性。

---

## 5. Shard 修復（Repair）

### 5.1 模擬 Worker 故障

關閉 worker node `localhost:1922`：

```bash
pg_ctl stop -m fast -D /data01/pg_root_1922
```

查詢仍然成功（pg_shard 自動路由到健康 replica），但會輸出一系列連線失敗警告：

```sql
SELECT count(*) FROM customer_reviews;
```

```
WARNING:  Connection failed to localhost:1922
DETAIL:  Remote message: could not connect to server: Connection refused
...
 count
-------
  6296
(1 row)
```

### 5.2 查看 Shard 狀態

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
```

掛掉的 worker 上的所有 shard 狀態變為 `3`（需要修復）：

```
 id | shard_id | shard_state | node_name | node_port
----+----------+-------------+-----------+-----------
  2 |    10000 |           1 | localhost |      1923
  3 |    10001 |           1 | localhost |      1923
 ...
  8 |    10003 |           3 | localhost |      1922   -- 掛了
 32 |    10015 |           3 | localhost |      1922   -- 掛了
  9 |    10004 |           3 | localhost |      1922   -- 掛了
 ...
```

`shard_state = 1` 健康，`shard_state = 3` 需修復。

### 5.3 在故障期間繼續寫入（造成資料不一致）

```bash
pgbench -M simple -n -r -P 1 -f ./test.sql -c 8 -j 8 -T 1000
```

```
WARNING: Connection failed to localhost:1922 ...
progress: 1.0 s, 167.9 tps
progress: 2.0 s, 1306.0 tps
progress: 3.0 s, 1451.7 tps
...
```

這段時間寫入的資料不會同步到掛掉的 worker 1922 上的 shard 副本，造成資料落後。

### 5.4 重啟故障 Worker 並修復

```bash
pg_ctl start -D /data01/pg_root_1922
```

在所有 worker 節點建立 extension（修復函數 `master_copy_shard_placement` 需要在目標端也存在 pg_shard）：

```bash
psql -h 127.0.0.1 -p 1922 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1923 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1924 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1925 -c "CREATE EXTENSION pg_shard;"
```

### 5.5 找出需修復的 Shard（從健康副本複製到故障副本）

找出 `shard_state = 3` 的 shard 及其健康副本配對：

```sql
SELECT t1.shard_id, t1.node_name, t1.node_port, t2.node_name, t2.node_port
FROM pgs_distribution_metadata.shard_placement t1,
     pgs_distribution_metadata.shard_placement t2
WHERE t1.shard_id = t2.shard_id
  AND (t1.node_name || t1.node_port) <> (t2.node_name || t2.node_port)
  AND t1.shard_state = 3;
```

```
 shard_id | node_name | node_port | node_name | node_port
----------+-----------+-----------+-----------+-----------
    10003 | localhost |      1922 | localhost |      1925
    10004 | localhost |      1922 | localhost |      1923
    10008 | localhost |      1922 | localhost |      1923
    10011 | localhost |      1922 | localhost |      1925
    10012 | localhost |      1922 | localhost |      1923
    10015 | localhost |      1922 | localhost |      1925
(6 rows)
```

### 5.6 執行修復

```sql
-- master_copy_shard_placement(
--   shard_id, source_node, source_port, target_node, target_port
-- )
-- 注意：source = 健康副本，target = 故障副本，不能搞反

SELECT master_copy_shard_placement(
    t1.shard_id, t2.node_name, t2.node_port,  -- 從健康副本 t2
    t1.node_name, t1.node_port                 -- 複製到故障副本 t1
)
FROM pgs_distribution_metadata.shard_placement t1,
     pgs_distribution_metadata.shard_placement t2
WHERE t1.shard_id = t2.shard_id
  AND (t1.node_name || t1.node_port) <> (t2.node_name || t2.node_port)
  AND t1.shard_state = 3;
```

若方向搞反會報錯（pg_shard 有保護）：

```
ERROR: 22023: source placement must be in finalized state
```

修復完成後所有 shard 狀態恢復為 `1`：

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
-- 全部 32 row, shard_state = 1
```

### 5.7 驗證資料一致性

```sql
SELECT count(*) FROM customer_reviews;
-- 17729 (與故障前一致)
```

> 補充（Senior Dev）：pg_shard 的修復機制是基於 `master_copy_shard_placement` 的 full-copy 策略——直接從健康副本把整個 shard 的數據複製到故障節點，沒有 incremental / WAL-based 同步。這在 shard 很大時非常耗時（全量傳輸），且修復期間故障 shard 的寫入會被跳過。現代 Citus 支援基於 logical replication 的 shard rebalancing（`citus_rebalance_start()`）和 streaming replication 的 worker HA，本質上解決了這個問題。

---

## 參考

1. [pg_shard GitHub v1.2.2](https://github.com/citusdata/pg_shard/tree/v1.2.2)
2. [pg_shard Quick Start Guide](https://www.citusdata.com/citus-products/pg-shard/pg-shard-quick-start-guide)
3. [JDBC 9.4 load balancing and failover](http://blog.163.com/digoal@126/blog/static/16387704020158241250463/)
