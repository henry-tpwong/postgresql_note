# PostgreSQL JSON/JSONB Deep Dive: 類型構造到陣列查詢

> 本文合併兩篇 PostgreSQL JSON/JSONB 技術筆記，按照從基礎到應用的順序組織：
>
> - **第一章**（JSON Type Construction）：從 JSONB 的 scalar type 規範開始，深入 parser 內部源碼（`parse_scalar`、`json_lex`、escape handler），再到 type I/O function 機制、`format() + jsonb_in()` 構造法、BYTEA escape 限制等。這是理解 JSONB 在 PostgreSQL 內部如何運作的基礎。
> - **第二章**（Array Extraction & Querying）：從 `->` / `->>` 操作符開始，覆蓋 2016 年經典的 `json_array_elements + ARRAY()` 方案、function-based GIN index 加速、以及 PG 12–17 的現代化方案（JSONPath、SQL/JSON `json_value`/`json_table`），最後給出三種方案的選擇矩陣與版本演進總表。
>
> 建議按順序閱讀——第一章的 parser 知識能幫助你在遇到第二章的查詢錯誤時理解底層原因。

---

# 一、JSON/JSONB Value Types 與構造方法

> 來源：[digoal - PostgreSQL json jsonb 支持的value数据类型，如何构造一个jsonb (2015-09-24)](https://github.com/digoal/blog/blob/master/201509/20150924_03.md)

---

## 1. JSON 支援的 Scalar Types（區分大小寫）

PostgreSQL JSON 的 value 支援以下 5 種 scalar types，**嚴格區分大小寫**：

| Scalar Type | 合法寫法 | 非法寫法 |
|-------------|---------|---------|
| string | `"abc"` | — |
| number | `10.001` | — |
| boolean true | `true` | `TRUE`、`True` |
| boolean false | `false` | `FALSE`、`False` |
| null | `null` | `NULL`、`Null` |

實測：

```sql
SELECT jsonb '{"a": true}';       -- OK
SELECT jsonb '{"a": TRUE}';       -- ERROR: Token "TRUE" is invalid
SELECT jsonb '{"a": false}';      -- OK
SELECT jsonb '{"a": NULL}';       -- ERROR: Token "NULL" is invalid
SELECT jsonb '{"a": null}';       -- OK
SELECT jsonb '{"a": 10.001}';     -- OK
SELECT jsonb '{"a": "10.001"}';   -- OK (string, not number)
```

這源於 `src/backend/utils/adt/json.c` 中 `parse_scalar()` 的實作。Parser 對每個 token 做精確的 byte-level 匹配，`true` 必須是 4 bytes、`false` 必須是 5 bytes、`null` 必須是 4 bytes，且大小寫敏感：

```c
static inline void
parse_scalar(JsonLexContext *lex, JsonSemAction *sem)
{
    char       *val = NULL;
    json_scalar_action sfunc = sem->scalar;
    char      **valaddr;
    JsonTokenType tok = lex_peek(lex);

    valaddr = sfunc == NULL ? NULL : &val;

    /* a scalar must be a string, a number, true, false, or null */
    switch (tok)
    {
        case JSON_TOKEN_TRUE:
            lex_accept(lex, JSON_TOKEN_TRUE, valaddr);
            break;
        case JSON_TOKEN_FALSE:
            lex_accept(lex, JSON_TOKEN_FALSE, valaddr);
            break;
        case JSON_TOKEN_NULL:
            lex_accept(lex, JSON_TOKEN_NULL, valaddr);
            break;
        case JSON_TOKEN_NUMBER:
            lex_accept(lex, JSON_TOKEN_NUMBER, valaddr);
            break;
        case JSON_TOKEN_STRING:
            lex_accept(lex, JSON_TOKEN_STRING, valaddr);
            break;
        default:
            report_parse_error(JSON_PARSE_VALUE, lex);
    }
    if (sfunc != NULL)
        (*sfunc) (sem->semstate, val, tok);
}
```

Parser context 的 enum（只用於 parser 狀態管理，與 JSONB 內部存儲無關）：

```c
typedef enum  /* contexts of JSON parser */
{
    JSON_PARSE_VALUE,
    JSON_PARSE_STRING,
    JSON_PARSE_ARRAY_START,
    JSON_PARSE_ARRAY_NEXT,
    JSON_PARSE_OBJECT_START,
    JSON_PARSE_OBJECT_LABEL,
    JSON_PARSE_OBJECT_NEXT,
    JSON_PARSE_OBJECT_COMMA,
    JSON_PARSE_END
} JsonParseContext;
```

> 補充（Senior Dev）：JSONB 內部實際上是 binary format（`JsonbContainer` / `JsonbValue`），並非真正的純字串。文章說「JSONB 內部就是一個有一定規則的字串」是指 **從 text 解析為 jsonb 時** Postgres 只做 token 匹配，不保留原始 type metadata。例如 `10` 和 `10.0` 存入 jsonb 後都是 numeric scalar，無法區分原始是 integer 還是 float。這與 MongoDB BSON 保留 `int32` / `int64` / `double` 的設計不同。

---

## 2. jsonb 的內部類型本質

jsonb 內部**沒有**類型概念。Parser 看到的 TOKEN 只是字串匹配。關鍵理解：

- `JSON_TOKEN_TRUE` / `JSON_TOKEN_FALSE` / `JSON_TOKEN_NULL` / `JSON_TOKEN_NUMBER` / `JSON_TOKEN_STRING` 是 **parser token type**，不是 JSONB 的內部 type
- JSONB 存儲的是 binary representation，但在 text↔jsonb 轉換過程中，Postgres 不保留原始 PG type 信息
- 如果你需要把 `int8range`、`geometry`、`timestamptz` 等 PG 內部 type 塞進 jsonb 並**能原樣取回**，必須透過 type I/O function 做 text 橋接

> 補充（Senior Dev）：MongoDB BSON 有 `$type` operator 可查 field 的 binary type（string / int / double / date / objectid 等）。Postgres jsonb 沒有對等機制——`jsonb_typeof()` 只返回 `string` / `number` / `boolean` / `null` / `object` / `array`，無法區分 `number` 是 int 還是 float。這在跨系統數據交換時是需要注意的精度風險點。

---

## 3. Type I/O Function 機制

Postgres 每個 type 都有 `typinput` 和 `typoutput` function（定義在 `pg_type` catalog），負責 text ↔ binary 的轉換：

```sql
SELECT oid, typname, typinput, typoutput
FROM pg_type WHERE typname = 'timestamptz';
```

| 屬性 | 值 | 作用 |
|------|-----|------|
| `oid` | 1184 | type OID |
| `typinput` | `timestamptz_in` | text → binary |
| `typoutput` | `timestamptz_out` | binary → text |

用法：

```sql
SELECT timestamptz_out(now());
-- 2015-09-24 19:44:16.076233+08

SELECT timestamptz_in('2015-09-24 19:44:16.076233+08', 1184, 1);
-- 2015-09-24 19:44:16.1+08      (typmod=1, precision truncated)

SELECT timestamptz_in('2015-09-24 19:44:16.076233+08', 1184, 6);
-- 2015-09-24 19:44:16.076233+08 (typmod=6, full precision)
```

`typmod`（第三個參數）控制精度/長度（如 `varchar(N)` 的 N、`numeric(P,S)` 的 P,S、`timestamptz(P)` 的 P）。對 `jsonb_in()` 來說，typmod 固定為 -1（無限制）。

來自 `src/backend/utils/adt/varlena.c` 中的 `text_format()`：

```c
/*
 * Returns a formatted string
 */
Datum
text_format(PG_FUNCTION_ARGS)
{
    ...
}
```

> 補充（Senior Dev）：這個 I/O function 機制是 Postgres type system 的核心——每個 type 的 text ↔ binary 轉換都是 reversible（`typoutput ∘ typinput = identity`）。因此你可以把任何 type **經由 text** 塞進 jsonb 再取出，只要 text representation 沒變。這也是為什麼 jsonb 不存內部 type：它依賴 text 的 self-describing 特性。

---

## 4. format() + jsonb_in() 構造法

利用 `format()` 將任意 type 的 output 格式化為 json string，再用 `jsonb_in()` 解析為 jsonb：

### I. 基本模式

```sql
-- Step 1: 用 format 產生 json 字串
SELECT format('{"K": "%s"}', int8range(1,10));
-- {"K": "[1,10)"}

-- Step 2: jsonb_in 解析
SELECT jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring);
-- {"K": "[1,10)"}

-- Step 3: 取出 element
SELECT jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring) ->> 'K';
-- [1,10)

-- Step 4: 轉回原 type
SELECT (jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring) ->> 'K')::int8range;
-- [1,10)
```

### II. 完整建構 / 提取 / 還原 demo

```sql
-- 存入
SELECT jsonb_in(format('{"ts": "%s", "rng": "%s"}',
    now(),
    int8range(1,10)
)::cstring);
-- {"ts": "2015-09-24 19:44:16.076233+08", "rng": "[1,10)"}

-- 取出並還原
WITH src AS (
    SELECT jsonb_in(format('{"ts": "%s", "rng": "%s"}',
        now()::timestamptz,
        int8range(1,10)
    )::cstring) AS data
)
SELECT
    (data ->> 'ts')::timestamptz AS ts,
    (data ->> 'rng')::int8range AS rng
FROM src;
```

> 補充（Senior Dev）：現代 PG（9.4+）提供了更簡潔的 native 構造方式，在多數場景下應優先使用：
> ```sql
> -- PG 9.4+: jsonb_build_object
> SELECT jsonb_build_object('ts', now(), 'rng', int8range(1,10));
>
> -- PG 9.5+: 直接 concatenation
> SELECT jsonb_build_object('a', 1) || jsonb_build_object('b', 2);
>
> -- PG 11+: jsonb 支援 TRANSFORM FOR TYPE（plpgsql 中自動轉換）
> ```
> `format() + jsonb_in()` 模式的優勢在於：你**完全控制 text representation**（如 timestamp 格式、range 邊界符號），這在需要自訂序列化格式時有用。但劣勢是性能較差（多一次 text ↔ binary 轉換）且要處理 escape。

---

## 5. BYTEA 的 Escape 限制

試圖將 `bytea` 塞進 jsonb 時，會遇到 escape 衝突：

```sql
SELECT format('{"K": "%s"}', '你好'::bytea);
-- {"K": "\xe4bda0e5a5bd"}
```

`bytea_out()` 輸出的 hex format 使用 `\x` prefix，但 JSON parser 的 escape handler 不認得 `\x`：

```sql
\set VERBOSITY verbose
SELECT jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring);
-- ERROR: 22P02: invalid input syntax for type json
-- DETAIL: Escape sequence "\x" is invalid.
```

Source code 中的 escape handler（`json_lex_string()`）：

```c
static inline void
json_lex_string(JsonLexContext *lex)
{
    ...
    switch (*s)
    {
        case '"':
        case '\\':
        case '/':
            appendStringInfoChar(lex->strval, *s);
            break;
        case 'b':
            appendStringInfoChar(lex->strval, '\b');
            break;
        case 'f':
            appendStringInfoChar(lex->strval, '\f');
            break;
        case 'n':
            appendStringInfoChar(lex->strval, '\n');
            break;
        case 'r':
            appendStringInfoChar(lex->strval, '\r');
            break;
        case 't':
            appendStringInfoChar(lex->strval, '\t');
            break;
        default:
            /* Not a valid string escape, so error out. */
            lex->token_terminator = s + pg_mblen(s);
            ereport(ERROR,
                (errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
                 errmsg("invalid input syntax for type json"),
                 errdetail("Escape sequence \"\\%s\" is invalid.",
                           extract_mb_char(s)),
                 report_json_context(lex)));
    }
```

Parser 只處理 JSON standard 的 escape：`"` `\\` `/` `\b` `\f` `\n` `\r` `\t`。任何其他 escape 都會報錯。

### I. Workaround：Output Escape 模式

修改 `json_lex_string()` 後（或使用 native 函數跳過 escape），可達成存入 bytea 並提取還原：

```sql
-- 取出為 hex string（無 \x prefix）
SELECT jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring) ->> 'K';
-- e4bda0e5a5bd

-- 還原：手動補回 \x prefix
SELECT convert_from(
    byteain(('\x' || (
        jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring) ->> 'K'
    ))::cstring),
    'utf8'::name
);
-- 你好
```

> 補充（Senior Dev）：PG 9.5+ 有更好的方案。`bytea` 的 output 可設定為 `escape` 格式（`SET bytea_output = 'escape'`），此時 `\x` 變為 `\` octet 表示法。PG 9.6+ 可以直接用 `encode()` / `decode()` 配合 `base64` 來避免 escape 衝突：
> ```sql
> -- PG 9.6+: 使用 base64 encode 避開 JSON escape
> SELECT jsonb_build_object('data', encode('你好'::bytea, 'base64'));
> -- {"data": "5L2g5aW9"}
>
> SELECT decode(('{"data": "5L2g5aW9"}'::jsonb ->> 'data'), 'base64');
> -- \xe4bda0e5a5bd
> ```
> 如果你的場景是將 binary data 存進 jsonb，`base64` encode 是通用且安全的做法，跨資料庫兼容性也好。
>
> 關於 `format()` 的另一個陷阱：`%s` 會呼叫類型的 `typoutput` function，而某些類型的 output 包含 control character 或 backslash（如 `point` 類型的 `(1,2)`、`box` 類型的 `(1,2),(3,4)`），這些括號/逗號不會被 JSON parser 誤解因為它們在 quoted string 內。但若類型的 output 本身帶 `"` 字元，format 產出的 JSON 就會損壞——這是一個邊際案例，建議在 production 中優先使用 `jsonb_build_object()` 來避免手動組裝 JSON string 的風險。

---

## 6. json_lex() 完整 Tokenizer 邏輯

`json_lex()` 是 JSON parser 的 tokenizer，負責將輸入 stream 切分為 token（跳過 whitespace `' '` `'\t'` `'\n'` `'\r'`）。其 switch-case 完整覆蓋：

- `{` `}` → `JSON_TOKEN_OBJECT_START` / `END`
- `[` `]` → `JSON_TOKEN_ARRAY_START` / `END`
- `,` `:` → `JSON_TOKEN_COMMA` / `COLON`
- `"` → `json_lex_string()` → `JSON_TOKEN_STRING`
- `-` → `json_lex_number()` with leading minus → `JSON_TOKEN_NUMBER`
- `0-9` → `json_lex_number()` → `JSON_TOKEN_NUMBER`
- 其他 alpha token：匹配 `true`(4) / `false`(5) / `null`(4)，否則報錯

```c
static inline void
json_lex(JsonLexContext *lex)
{
    char       *s;
    int         len;

    /* Skip leading whitespace. */
    s = lex->token_terminator;
    len = s - lex->input;
    while (len < lex->input_length &&
           (*s == ' ' || *s == '\t' || *s == '\n' || *s == '\r'))
    {
        if (*s == '\n')
            ++lex->line_number;
        ++s;
        ++len;
    }
    lex->token_start = s;

    /* Determine token type. */
    if (len >= lex->input_length)
    {
        lex->token_start = NULL;
        lex->prev_token_terminator = lex->token_terminator;
        lex->token_terminator = s;
        lex->token_type = JSON_TOKEN_END;
    }
    else
        switch (*s)
        {
            case '{': lex->token_type = JSON_TOKEN_OBJECT_START;  break;
            case '}': lex->token_type = JSON_TOKEN_OBJECT_END;    break;
            case '[': lex->token_type = JSON_TOKEN_ARRAY_START;   break;
            case ']': lex->token_type = JSON_TOKEN_ARRAY_END;     break;
            case ',': lex->token_type = JSON_TOKEN_COMMA;         break;
            case ':': lex->token_type = JSON_TOKEN_COLON;         break;
            case '"': json_lex_string(lex);
                      lex->token_type = JSON_TOKEN_STRING;         break;
            case '-': json_lex_number(lex, s + 1, NULL);
                      lex->token_type = JSON_TOKEN_NUMBER;         break;
            case '0' ... '9':
                      json_lex_number(lex, s, NULL);
                      lex->token_type = JSON_TOKEN_NUMBER;         break;
            default:
            {
                char *p;
                for (p = s; p - s < lex->input_length - len
                     && JSON_ALPHANUMERIC_CHAR(*p); p++) /* skip */;
                if (p == s)
                    report_invalid_token(lex);
                lex->prev_token_terminator = lex->token_terminator;
                lex->token_terminator = p;
                if (p - s == 4)
                {
                    if (memcmp(s, "true", 4) == 0)
                        lex->token_type = JSON_TOKEN_TRUE;
                    else if (memcmp(s, "null", 4) == 0)
                        lex->token_type = JSON_TOKEN_NULL;
                    else
                        report_invalid_token(lex);
                }
                else if (p - s == 5 && memcmp(s, "false", 5) == 0)
                    lex->token_type = JSON_TOKEN_FALSE;
                else
                    report_invalid_token(lex);
            }
        }
}
```

> 補充（Senior Dev）：注意 `true` 和 `null` 共用 4-byte 分支、`false` 獨佔 5-byte 分支。這意味著 `tree`、`nullx` 等 4-byte 非標準 token 會進入 `true`/`null` 的 `memcmp` 檢查後才報錯（而非直接被 `JSON_ALPHANUMERIC_CHAR` 循環找到的長度觸發），`falsy` 同理。這不會影響正確性但對 custom JSON extension（如 pg_cron 的 JSON config parser）是值得注意的實現細節。

---

# 二、JSON/JSONB 陣列提取與查詢

> 來源：[digoal - 如何从PostgreSQL json中提取数组 (2016-09-10)](https://github.com/digoal/blog/blob/master/201609/20160910_01.md)
>
> 更新於 2026-05-17，補充 JSONPath / SQL/JSON 標準函數 / json_table 演進

---

## 1. JSON Value Type 與提取基礎

### I. json_typeof / jsonb_typeof

```sql
SELECT jsonb_typeof(c1), c1 FROM t3;
```

支援 type：`object`, `array`, `string`, `number`, `boolean`, `null`。

### II. -> 與 ->> 操作符

| 操作符 | 返回 type | 範例 |
|--------|----------|------|
| `->'key'` | json/jsonb（保留原始類型） | `c1->'f'` → `[1,2,3,4]` (jsonb) |
| `->>'key'` | text | `c1->>'f'` → `[1,2,3,4]` (text) |
| `->N` | json/jsonb（陣列索引） | `c1->'f'->0` → `1` |

完整演示（jsonb）：

```sql
CREATE TABLE t3 (c1 JSONB);
INSERT INTO t3 VALUES ('{
  "a":"v", "b":12, "c":{"ab":"hello"}, "d":12.3,
  "e":true, "f":[1,2,3,4], "g":["a","b"]
}');

SELECT pg_typeof(col), jsonb_typeof(col), col
FROM (SELECT c1->'a' col FROM t3) t;
--  pg_typeof | jsonb_typeof | col
-- -----------+--------------+-----
--  jsonb     | string       | "v"

SELECT pg_typeof(col), jsonb_typeof(col), col
FROM (SELECT c1->'f' col FROM t3) t;
--  pg_typeof | jsonb_typeof | col
-- -----------+--------------+--------------
--  jsonb     | array        | [1,2,3,4]
```

關鍵認知：`->` 返回的仍是 json/jsonb type（不是 native SQL type），即使 `jsonb_typeof` 判斷為 array，PG 仍視為 jsonb。

---

## 2. 2016 年方案：json_array_elements + ARRAY 構造器

### I. Step 1：JSON array → row set

```sql
SELECT jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b');
--  col
-- -----
--  1
--  2
--  3
--  4
--  5
--  6
-- pg_typeof: text
```

`jsonb_array_elements_text()` 將 JSON array 展開為 text row set。對應 json 版為 `json_array_elements_text()`。

### II. Step 2：row set → SQL array

```sql
SELECT ARRAY(
  SELECT jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b')
);
--  {1,2,3,4,5,6}
-- pg_typeof: text[]

SELECT ARRAY(
  SELECT (jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b'))::INT
);
--  {1,2,3,4,5,6}
-- pg_typeof: integer[]
```

### III. Step 3：封裝為可重複使用的 Helper Function

```sql
-- JSONB → text[]
CREATE OR REPLACE FUNCTION json_arr2text_arr(_js JSONB)
  RETURNS TEXT[] AS $$
  SELECT ARRAY(SELECT jsonb_array_elements_text(_js))
$$ LANGUAGE SQL IMMUTABLE;

-- JSON → text[]
CREATE OR REPLACE FUNCTION json_arr2text_arr(_js JSON)
  RETURNS TEXT[] AS $$
  SELECT ARRAY(SELECT json_array_elements_text(_js))
$$ LANGUAGE SQL IMMUTABLE;

-- JSONB → int[]
CREATE OR REPLACE FUNCTION json_arr2int_arr(_js JSONB)
  RETURNS INT[] AS $$
  SELECT ARRAY(SELECT (jsonb_array_elements_text(_js))::INT)
$$ LANGUAGE SQL IMMUTABLE;
```

使用：

```sql
SELECT json_arr2text_arr(c1->'g') FROM t3;  -- {a,b}
SELECT json_arr2int_arr(c1->'f') FROM t3;   -- {1,2,3,4}
```

---

## 3. 陣列查詢：@> / && 等操作符

一旦 JSON array 轉為 native SQL array，即可使用 PostgreSQL 的陣列操作符：

```sql
-- @>  全包含（contains）
SELECT * FROM t3 WHERE json_arr2text_arr(c1->'g') @> ARRAY['a'];        -- 包含 a
SELECT * FROM t3 WHERE json_arr2int_arr(c1->'f') @> ARRAY[1,2];         -- 同時包含 1,2

-- &&  相交（overlaps）
SELECT * FROM t3 WHERE json_arr2int_arr(c1->'f') && ARRAY[1,6];         -- 包含 1 或 6

-- NOT &&  不相交
SELECT * FROM t3 WHERE NOT(json_arr2int_arr(c1->'f') && ARRAY[6]);      -- 不包含 6

-- 注意 type 一致：text 陣列 vs text value / int 陣列 vs int value
SELECT * FROM t3 WHERE NOT(json_arr2text_arr(c1->'f') && ARRAY['6']);   -- 對 text[] 用 '6'
```

---

## 4. Index：GIN on Expression

```sql
-- 建立 function-based GIN index（將 JSON array 轉換結果索引化）
CREATE INDEX idx_t3_1 ON t3 USING GIN (json_arr2text_arr(c1->'f'));

SET enable_seqscan = off;

EXPLAIN SELECT * FROM t3 WHERE json_arr2text_arr(c1->'f') && ARRAY['1','6'];
--  Bitmap Heap Scan on t3  (cost=12.25..16.52 rows=1 width=32)
--    Recheck Cond: (json_arr2text_arr((c1 -> 'f'::text)) && '{1,6}'::text[])
--    ->  Bitmap Index Scan on idx_t3_1

EXPLAIN SELECT * FROM t3 WHERE json_arr2text_arr(c1->'f') @> ARRAY['1','6'];
--  Bitmap Heap Scan on t3
--    Recheck Cond: (json_arr2text_arr((c1 -> 'f'::text)) @> '{1,6}'::text[])
--    ->  Bitmap Index Scan on idx_t3_1
```

> 補充（Senior Dev）：function-based GIN index 要求 function 標記為 `IMMUTABLE`。若 function 不是 IMMUTABLE（如依賴 GUC），index 將無法建立。上面 `json_arr2text_arr` 確實是 IMMUTABLE（純 JSONB → text[] 轉換，無外部依賴）。

---

## 5. 現代化方案：JSONPath（PG 12+）

2016 年的方案仍有效，但 PG 12 引入的 JSONPath 提供更強大的表達能力：

```sql
-- 查詢條件：JSON array 中包含數字 1
SELECT * FROM t3
WHERE c1 @? '$.f[*] ? (@ == 1)';

-- 提取所有匹配的元素
SELECT * FROM jsonb_path_query(
  c1,
  '$.f[*] ? (@ > 2)'
) FROM t3;

-- 返回原生 SQL 型別（自動處理）
SELECT * FROM t3
WHERE c1 @? '$.b ? (@ > 10)';
```

> 補充（Senior Dev）：JSONPath 的 query 在 GIN index 上的加速方式有別於手動轉換：
> - `c1 @? '$.f[*] ? (@ == 1)'` 可以使用 default `jsonb_ops` GIN index（因為 `@?` 內部轉為 `jsonb_path_match`）
> - 但 JSONPath 的 `@?` 不會自動利用「提取 array → text[] → GIN index」的路徑，需手動建立。對於大量陣列包含查詢，2016 年的 function-based GIN 方案可能仍有性能優勢。

---

## 6. SQL/JSON 標準函數（PG 15+）

PG 15 引入 ISO SQL/JSON 標準函數，替代部分手動處理：

```sql
-- json_exists — 判斷值是否存在
SELECT * FROM t3 WHERE JSON_EXISTS(c1, '$.f[*] ? (@ == 1)');

-- json_value — 提取單一 scalar value（自動轉型）
SELECT JSON_VALUE(c1, '$.b' RETURNING INT) FROM t3;  -- → 12 (int)

-- json_query — 提取整個 object/array
SELECT JSON_QUERY(c1, '$.f') FROM t3;  -- → [1,2,3,4] (jsonb)
```

> 補充（Senior Dev）：`json_value` 解決了 2016 年的核心痛點——`->` 返回 jsonb 而非 native type。現在 `json_value(c1->'b' RETURNING INT)` 直接拿到 PostgreSQL integer，不需要額外 `::INT` cast。

---

## 7. json_table（PG 17+）

PG 17 引入 `json_table`（又是 ISO SQL/JSON 標準），直接在 FROM 子句中將 JSON 陣列展開為 structured column：

```sql
SELECT jt.*
FROM t3,
     JSON_TABLE(
       c1, '$.f[*]'
       COLUMNS (
         val INT PATH '$'
       )
     ) AS jt;

-- 支持多層嵌套、多 column、type coercion
SELECT jt.id, jt.name, jt.price
FROM orders,
     JSON_TABLE(
       doc, '$.items[*]'
       COLUMNS (
         id    INT     PATH '$.product_id',
         name  TEXT    PATH '$.product_name',
         price NUMERIC PATH '$.price'
       )
     ) AS jt
WHERE jt.price < 100;
```

> 補充（Senior Dev）：`json_table` 將 JSON 陣列的 row-expansion 從「select-list subquery + array constructor」的兩步操作變成單一步驟，plan execution 更高效（無需 intermediate array materialization）。對大量 JSON 集的 OLAP 場景是 game changer。

---

## 8. 三種方案的選擇矩陣

| 方案 | 適用 PG 版本 | 適用場景 | Trade-off |
|------|-------------|---------|-----------|
| `json_array_elements` + `ARRAY()` | 9.3+ | 需要為陣列建立 GIN index 進行 `@>` / `&&` 查詢 | 需 self-defined function，維護成本 |
| JSONPath `@?` / `*.?()` | 12+ | 條件查詢（不需要轉 SQL array） | 對 `@>` 語義支援弱於 native array operators |
| SQL/JSON (`json_value` / `json_table`) | 15+ / 17+ | 結構化提取、返回 native SQL type | 最新方案，生態文檔較少 |

---

## 9. 版本演進總表

| 功能 | 版本 | 說明 |
|------|------|------|
| `json_array_elements_text` | PG 9.3 | JSON array → row set |
| `jsonb` type + GIN index | PG 9.4 | 二進制 JSON + 原生 GIN 加速 |
| `jsonb_path_query` / `@?` / `@@` | PG 12 | JSONPath 表達式 |
| SQL/JSON `json_exists` / `json_value` / `json_query` | PG 15 | ISO 標準函數 |
| SQL/JSON `json_table` | PG 17 | ISO 標準 inline table function |

---

## 10. 參考

- [How to turn JSON array into Postgres array (DBA.SE)](http://dba.stackexchange.com/questions/54283/how-to-turn-json-array-into-postgres-array)
- [PostgreSQL JSON Functions](https://www.postgresql.org/docs/9.6/static/functions-json.html)
- [PostgreSQL Array Functions](https://www.postgresql.org/docs/9.6/static/functions-array.html)
