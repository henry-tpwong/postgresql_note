# PostgreSQL 中文全文檢索（zhparser + FTS）

> 來源：[digoal - PostgreSQL chinese full text search 中文全文检索 (2014-03-24)](https://github.com/digoal/blog/blob/master/201403/20140324_01.md)
>
> 更新於 2026-05-17，補充現代化分詞方案、GIN 優化、效能調校

---

## 背景：為什麼需要獨立的中文分詞？

PostgreSQL 內建 `default` text search parser 按空白與標點切詞，對英文 OK，對中文無效——中文詞與詞之間無空格，長度不固定，同一個字串在不同語境可拆分為不同詞（「發展中國家」→「發展/中國/家」vs「發展中/國家」）。

**zhparser** 是 PostgreSQL C extension，實作 `TEXT SEARCH PARSER` 所需介面（`zhprs_start` / `zhprs_getlexeme` / `zhprs_end` / `zhprs_lextype`），底層呼叫 **SCWS**（Simple Chinese Word Segmentation）引擎進行分詞。

> 補充（Senior Dev）：除了 zhparser + SCWS，現代 PostgreSQL 生態中還有其他選擇：

| 方案 | 底層引擎 | 特點 |
|------|---------|------|
| **zhparser** | SCWS | 字典分詞，支援自訂詞典，生態最成熟 |
| **pg_jieba** | Jieba (結巴分詞) | Trie tree + HMM，支援新詞發現 |
| **pg_bigm** | 2-gram | 純 bigram 切分（如「全文檢索」→「全文」「文檢」「檢索」），不需詞典，召回率高但 precision 較低 |
| **pg_trgm** | Trigram | PG 內建，支援 `ILIKE '%keyword%'` 加速，非真正 FTS 但可做模糊 substring 搜尋 |

RDS PostgreSQL / 阿里雲 PolarDB / Supabase 等雲端服務均已內建 zhparser，不需自行編譯。

---

## 安裝（CentOS + PG 9.3 原文步驟）

```bash
# SCWS 分詞庫
wget http://www.xunsearch.com/scws/down/scws-1.2.2.tar.bz2
tar -jxvf scws-1.2.2.tar.bz2
cd scws-1.2.2
./configure --prefix=/opt/scws-1.2.2
make && make install

# zhparser extension
git clone https://github.com/amutu/zhparser.git
cd zhparser
export PATH=/home/pg93/pgsql/bin:$PATH
which pg_config                          # 確認指向正確 PG 安裝
SCWS_HOME=/opt/scws-1.2.2 make
make install
```

> 補充（Senior Dev）：現代環境（PG 14+ / PG 16+ / PG 18）zhparser 已更新支援。SCWS 最新版本為 1.2.3+，zhparser GitHub repo 持續維護。若使用 Docker，可直接 `apt install postgresql-16-zhparser`（或對應 PG 版本）。Mac 上可用 `brew install scws` + 手動編譯 zhparser。

---

## 建立全文檢索配置

```sql
CREATE EXTENSION zhparser;

-- 查詢已註冊的 parser（確認 zhparser 載入成功）
SELECT * FROM pg_ts_parser;
--  prsname  | prsnamespace |  prsstart   |    prstoken     |  prsend   |  prsheadline  |  prslextype
-- ----------+--------------+-------------+-----------------+-----------+---------------+---------------
--  default  |           11 | prsd_start  | prsd_nexttoken  | prsd_end  | prsd_headline | prsd_lextype
--  zhparser |        25956 | zhprs_start | zhprs_getlexeme | zhprs_end | prsd_headline | zhprs_lextype

-- 建立使用 zhparser 的 text search configuration
CREATE TEXT SEARCH CONFIGURATION testzhcfg (PARSER = zhparser);

-- 配置 token type → dictionary mapping
ALTER TEXT SEARCH CONFIGURATION testzhcfg
  ADD MAPPING FOR n, v, a, i, e, l WITH simple;

-- 檢視 mapping
SELECT * FROM pg_ts_config_map
WHERE mapcfg = (SELECT oid FROM pg_ts_config WHERE cfgname = 'testzhcfg');
```

`pg_ts_config_map` 顯示 token type → dictionary 的對應關係。zhparser 支援的 token type（對應 SCWS 詞性標注）：

| Token Type | 含義 | 常用 mapping |
|-----------|------|-------------|
| `a` | adjective（形容詞） | `simple` |
| `b` | other（其他） | `simple` |
| `c` | conjunction（連詞） | 可忽略 |
| `d` | adverb（副詞） | `simple` |
| `e` | exclamation（嘆詞） | 可忽略 |
| `g` | root（詞根） | `simple` |
| `h` | prefix（前綴） | `simple` |
| `i` | idiom（成語） | `simple` |
| `j` | abbreviation（縮寫） | `simple` |
| `k` | tail（後綴） | `simple` |
| `l` | temp（暫用） | `simple` |
| `m` | numeral（數詞） | `simple` |
| `n` | noun（名詞） | `simple` |
| `o` | onomatopoeia（擬聲詞） | 可忽略 |
| `p` | preposition（介詞） | 可忽略 |
| `q` | quantifier（量詞） | `simple` |
| `r` | pronoun（代詞） | 可忽略 |
| `s` | space（空格） | 可忽略 |
| `t` | time（時間詞） | `simple` |
| `u` | auxiliary（助詞） | 可忽略 |
| `v` | verb（動詞） | `simple` |
| `w` | punctuation（標點） | 可忽略 |
| `x` | unknown（未知） | `simple` |
| `y` | modal（語氣詞） | 可忽略 |
| `z` | state（狀態詞） | `simple` |

> 補充（Senior Dev）：實務上建議 mapping 規則分三層：
> 1. **實詞**（n,v,a,i,l,m,t,z,j）→ `simple` dictionary（保留原始詞形）
> 2. **虛詞**（c,d,p,r,u,y,w,s,e,o）→ 不 mapping（直接丟棄，減少 tsvector 體積）
> 3. **自訂義**：特定行業術語用 `synonym` dictionary 或 `thesaurus` dictionary 做同義詞擴展

---

## 分詞效果測試

```sql
-- ts_parse：查看 parser 如何切分 token
SELECT * FROM ts_parse('zhparser',
  'hello world! 2010年保障房建设在全国范围内获全面启动，从中央到地方纷纷加大保障房建设投入力度。'
);
--  tokid |  token
-- -------+----------
--    101 | hello
--    101 | world
--    117 | !
--    101 | 2010
--    113 | 年
--    118 | 保障
--    110 | 房建      -- ※ 注意：分詞器可能將「保障房建設」拆成「保障」+「房建」，不如預期
--    118 | 设在
--    110 | 全国
--    ...

-- to_tsvector：生成全文檢索向量
SELECT to_tsvector('testzhcfg',
  '“今年保障房新开工数量虽然有所下调，但实际的年度在建规模以及竣工规模会超以往年份”'
);
--  '下调':7 '保障':1,30 '历史':21 '年度':9 '规模':11,13 '竣工':12 ...
```

> 補充（Senior Dev）：分詞品質是 FTS 的上限。上例中「保障房建設」被拆成「保障」+「房建」而非「保障房」+「建設」，這是通用詞典的侷限。解決方案：
> 1. **自訂詞典**：zhparser 支援 `zhparser.extra_dicts` 參數載入自訂詞典（`/opt/scws/etc/dict.utf8.xdb`），加入領域詞彙如「保障房、區塊鏈、微服務」
> 2. **自訂規則**：SCWS 支援 `rules.ini` 自訂分詞優先級
> 3. **詞典格式**：`word<TAB>tf<TAB>attr`（tf=詞頻，attr=詞性），例如 `保障房   14  n`

```sql
-- to_tsquery：將查詢字串轉為 tsquery
SELECT to_tsquery('testzhcfg', '保障房资金压力');
--  '保障' & '房' & '资金' & '压力'

-- 一般查詢寫法
SELECT * FROM articles
WHERE to_tsvector('testzhcfg', content) @@ to_tsquery('testzhcfg', '保障房');
```

---

## 完整部署流程（推薦）

```sql
-- Step 1: 建立 extension
CREATE EXTENSION zhparser;

-- Step 2: 建立 parser → config
CREATE TEXT SEARCH CONFIGURATION chinese_fts (PARSER = zhparser);

-- Step 3: 只 mapping 實詞（名詞、動詞、形容詞等）
ALTER TEXT SEARCH CONFIGURATION chinese_fts
  ADD MAPPING FOR n, v, a, i, l, j, t, m, z, g, h, k
  WITH simple;

-- Step 4: 可選：加入 synonym dictionary 做同義詞擴展
-- CREATE TEXT SEARCH DICTIONARY chinese_syn (
--   TEMPLATE = synonym,
--   SYNONYMS = chinese_synonyms
-- );
-- ALTER TEXT SEARCH CONFIGURATION chinese_fts
--   ALTER MAPPING FOR n, v, a WITH chinese_syn, simple;

-- Step 5: 建立 GIN index（加速全文檢索）
CREATE INDEX idx_articles_content_fts ON articles
  USING GIN (to_tsvector('chinese_fts', content));

-- Step 6: 查詢
SELECT title, ts_headline('chinese_fts', content,
  to_tsquery('chinese_fts', 'PostgreSQL & 全文检索'),
  'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20'
) AS headline
FROM articles
WHERE to_tsvector('chinese_fts', content) @@ to_tsquery('chinese_fts', 'PostgreSQL & 全文检索')
ORDER BY ts_rank(to_tsvector('chinese_fts', content),
                 to_tsquery('chinese_fts', 'PostgreSQL & 全文检索')) DESC
LIMIT 20;
```

---

## 效能調校

### GIN Index 選擇

| Index 類型 | 優勢 | 劣勢 |
|-----------|------|------|
| `GIN` | FTS 查詢極快、支援多欄位 composite tsvector | 寫入稍慢（shared buffer hit 時可忽略） |
| `GiST` | 寫入比 GIN 快、比 GIN 省空間 | 查詢比 GIN 慢 |
| `RUM`（需 extension） | 支援 ranking + phrase search + KNN 排序 | 體積最大 |

> 補充（Senior Dev）：PG 10+ GIN 支援 `fastupdate = on`（預設），將 pending list 中的 index entries 定期合併入主 index tree，大幅降低寫入 overhead。`gin_pending_list_limit`（預設 4MB）可在高寫入場景調高。

```sql
-- 若 content 欄位很長，只索引前 N 個字元可減少 index 體積
CREATE INDEX idx_articles_content_fts ON articles
  USING GIN (to_tsvector('chinese_fts', substring(content, 1, 1024)));
```

### zhparser 專屬參數

| 參數 | 預設 | 說明 |
|------|------|------|
| `zhparser.punctuation_ignore` | on | 忽略標點符號 |
| `zhparser.seg_with_duality` | on | 二元分詞（將長詞再次二元切分） |
| `zhparser.dict_in_memory` | off | 詞典載入記憶體（加快分詞但吃記憶體） |
| `zhparser.extra_dicts` | — | 載入額外自訂詞典路徑 |
| `zhparser.multi_short` | on | 短詞複合 |
| `zhparser.multi_duality` | on | 雙字詞複合 |
| `zhparser.multi_zmain` | off | 單字切分 |
| `zhparser.multi_zall` | off | 所有詞二次切分 |

```sql
-- 範例：開啟詞典記憶體快取 + 關閉二元分詞（精確度優先）
SET zhparser.dict_in_memory = on;
SET zhparser.seg_with_duality = off;
```

---

## 與其他 FTS 查詢模式的互動

```sql
-- websearch_to_tsquery（PG 11+，Google 式查詢語法）
SELECT websearch_to_tsquery('chinese_fts', 'PostgreSQL 全文检索 -旧版本');

-- phraseto_tsquery（PG 9.6+，精確詞組匹配）
SELECT phraseto_tsquery('chinese_fts', '保障房建设');

-- plainto_tsquery（全 AND）
SELECT plainto_tsquery('chinese_fts', '保障房建设资金');  -- → '保障' & '房' & '建设' & '资金'

-- ts_rank_cd (cover density ranking, 比 ts_rank 更精準)
SELECT ts_rank_cd(to_tsvector('chinese_fts', content),
                  to_tsquery('chinese_fts', '保障房')) FROM articles;
```

---

## 雲端 PostgreSQL 相容性

| 平台 | zhparser 支援 |
|------|-------------|
| 阿里雲 RDS PG | 內建（`CREATE EXTENSION zhparser`） |
| 阿里雲 PolarDB PG | 內建 |
| AWS RDS PG | 不支援外掛 C extension（可用 pg_bigm / pg_trgm 替代） |
| Supabase | 需自行編譯（managed service 不開放 C extension） |
| Google Cloud SQL PG | 不支援 C extension |
| 自建 PostgreSQL | 完全支援 |

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `phraseto_tsquery` | PG 9.6 | 精確詞組匹配 |
| GIN fastupdate | PG 10 | 寫入優化 |
| `websearch_to_tsquery` | PG 12 | Google 式查詢語法 |
| `pg_bigm` extension | PG 13+ | 2-gram CJK 分詞（不需 SCWS） |
| GIN index parallel builds | PG 15+ | 大型 GIN index 加速建立 |
| `tsvector` merge optimizations | PG 16+ | 合併效能提升 |

## 參考

1. [zhparser GitHub](https://github.com/amutu/zhparser)
2. [SCWS 分詞引擎](https://github.com/hightman/scws)
3. [zhparser 使用文檔](http://amutu.com/blog/zhparser/)
4. [amutu 部落格](http://amutu.com/blog/zhparser/)
5. [PGXN zhparser](http://pgxn.org/dist/zhparser/)
6. [德哥早期 nlpbamboo 分詞方案](https://github.com/digoal/blog/blob/master/201206/20120621_01.md)
