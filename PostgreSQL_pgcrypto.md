# PostgreSQL pgcrypto — 數據加密全面指南

> 來源：[digoal - 固若金湯 — PostgreSQL pgcrypto 加密插件 (2016-07-27)](https://github.com/digoal/blog/blob/master/201607/20160727_02.md)

---

## 1. 加密架構：Server-Side vs Client-Side

pgcrypto 提供的是 **Server-Side 加密**——加密/解密在資料庫服務端執行。

| 加密位置 | 網路傳輸內容 | 防禦範圍 |
|----------|-------------|---------|
| Server-Side（pgcrypto） | 明文 | Data-at-rest（存儲層） |
| Client-Side（應用層加密） | 密文 | Data-at-rest + Data-in-transit |
| **Server-Side + TLS** | 加密通道傳輸明文 | Data-at-rest + Data-in-transit（完整防禦） |

> 德哥建議：服務端加解密 + 傳輸層加密（TLS/SSL）是生產環境的標準組合。單純依靠 pgcrypto 無法防止網路竊聽。

> 補充（Senior Dev）：現代 PostgreSQL（PG 10+）預設啟用 SCRAM-SHA-256 密碼驗證和 TLS 1.3。建議 `ssl = on` + `ssl_min_protocol_version = 'TLSv1.2'` + `pgcrypto` 形成纵深防禦。若只加密個別敏感欄位（如 SSN、信用卡號），pgcrypto 是合適方案；若全表加密需求，考量 PG TDE（PG 16+ `pg_tde` extension 或雲端 RDS 自帶）更透明。

---

## 2. Hash 函數：digest() 與 hmac()

```sql
CREATE EXTENSION pgcrypto;

-- digest(data, algorithm) → bytea
-- 支援: md5, sha1, sha224, sha256, sha384, sha512
-- 若編譯時帶 --with-openssl 可支援更多算法
SELECT digest('I am digoal.', 'md5');
-- \xc3b0fb1147858d2259d92f20668fc8f9

-- hmac(data, key, algorithm) → bytea
-- 比 digest 多一個 key 參數：相同 data + 不同 key → 不同 hash
-- 用於防止 data + hash 同時被篡改（攻擊者無 key 無法重算 hash）
SELECT hmac('I am digoal.', 'this is a key', 'md5');
-- \xc70d0fd2af2382ea8e0a7ffd9edcbd58
```

**重要**：`digest()` 和 `hmac()` 有 text 和 bytea 兩個 overload。若參數類型不明確會調用不同 function，結果不同：

```sql
SELECT digest('\xffffff'::bytea, 'md5');
-- \x8597d4e7e65352a302b63e07bc01a7da
SELECT digest('\xffffff', 'md5');
-- \xd721f40e22920e0fd8ac7b13587aa92d   ← 不同結果！調用了 digest(text, text)
```

> `hmac` 若 key 長度超過 hash 算法的 block size，key 會被先 hash 一次再使用。

> 補充（Senior Dev）：`digest()` / `hmac()` 每次輸入相同 → 輸出相同（deterministic）。這意味著它們**不適合儲存密碼**——rainbow table 攻擊可逆向。應使用 `crypt()` + `gen_salt()`。但 `hmac(key, data, 'sha256')` 適合用於 API Token / HMAC 簽名校驗。

---

## 3. 密碼加密：crypt() + gen_salt()

`crypt()` + `gen_salt()` 的核心設計目標是**加大破解難度**（slow hash + random salt）。每次同一密碼產生的 hash 都不同。

### 3.1 使用

```sql
-- gen_salt(type [, iter_count]) → salt
-- type: des, xdes, md5, bf
-- iter_count: xdes 需為奇數 (1-16777215, default 725); bf (4-31, default 6)
SELECT gen_salt('bf', 10);
-- $2a$10$qY0amXGalzj14rooSpTf5e

SELECT crypt('this is a pwd source', gen_salt('bf', 10));
-- $2a$10$tvNK2H9mPu1tU5L6oAHdSeze5Hlz7G0y4oEKNg9SlGa06J2sywZHu

SELECT crypt('this is a pwd source', gen_salt('bf', 10));
-- $2a$10$cgJiTAs55vMBqYR1kGMGXuMKZI4dsayna4wgEL4K7duYkD0g25ufW  ← 不同！
```

### 3.2 密碼驗證

`crypt(password, stored_hash)` → 若密碼正確返回與 `stored_hash` 相同的值。驗證方式：

```sql
SELECT crypt('this is a pwd source', pwd) = pwd
FROM userpwd WHERE userid = 1;
-- 正確密碼 → t, 錯誤密碼 → f
```

**原理**：`crypt()` 從 `stored_hash` 的第二個參數位擷取 salt（格式：`$algorithm$iter$salt$hash`），用相同的 algorithm / iter / salt 重算 hash，與儲存值比對。

### 3.3 算法對比

| Algorithm | Max Pwd Len | Adaptive | Salt Bits | 說明 |
|-----------|------------|----------|-----------|------|
| bf (Blowfish) | 72 | ✓ | 128 | **推薦**，iter 可控 |
| md5 | unlimited | ✗ | 48 | 次選 |
| xdes | 8 | ✓ | 24 | iter 需奇數 |
| des | 8 | ✗ | 12 | 不建議，古老 |

### 3.4 破解速度基準

| Algorithm | Hashes/sec (Pentium 4 1.5GHz) | 破解 8-char \[a-z\] | 破解 8-char \[A-Za-z0-9\] |
|-----------|------------------------------|---------------------|--------------------------|
| crypt-bf/8 (iter=256) | 28 | 246 年 | 251,322 年 |
| crypt-bf/7 | 57 | 121 年 | 123,457 年 |
| crypt-bf/6 | 112 | 62 年 | 62,831 年 |
| crypt-bf/5 | 211 | 33 年 | 33,351 年 |
| crypt-md5 | 2,681 | 2.6 年 | 2,625 年 |
| crypt-des | 362,837 | 7 天 | 19 年 |
| sha1 | 590,223 | 4 天 | 12 年 |
| md5 | 2,345,086 | 1 天 | 3 年 |

**iter_count 選擇原則**：以當前 CPU 實測為準，讓 `crypt()` 執行時間在 **4-100 ms** 之間（每秒 10-250 次），平衡安全性與 user login latency。

> 補充（Senior Dev）：
> - `bf` (Blowfish/bcrypt) 的 iter_count 對應的是 2^iter 次迭代。iter=10 → 1024 次。現代 CPU 上 iter=10 約 50-100ms，iter=12 約 200-400ms。PG 14+ 可考慮 `scram-sha-256`（內建密碼驗證機制，不需要 pgcrypto 處理密碼欄位）
> - `crypt()` 的 salt 長度：bf 為 128-bit (22 char Base64)，md5 為 48-bit。salt 太短會降低抗碰撞性
> - **Timing Attack 注意**：`crypt()` 使用 constant-time comparison，但若應用層用 `SELECT ... WHERE pwd = crypt(...)` 而非 `crypt(...) = stored_hash`，可能洩漏 timing info。務必使用 constant-time 比對方式

---

## 4. PGP 對稱加密

遵循 OpenPGP (RFC 4880)。對稱加密用同一密鑰加解密。

```sql
-- pgp_sym_encrypt(data, psw [, options]) → bytea
-- pgp_sym_decrypt(msg, psw [, options]) → text
```

```sql
SELECT pgp_sym_encrypt(
  'i am digoal', 'pwd',
  'cipher-algo=aes256, compress-algo=2, compress-level=9'
);

SELECT pgp_sym_decrypt(
  '\xc30d040403024581...'::bytea,
  'pwd'
);
-- → i am digoal
```

**options** 支援：`cipher-algo`（aes256 / aes128 / bf / 3des / cast5）、`compress-algo`（0=none, 1=ZIP, 2=ZLIB）、`compress-level`（0-9）、`convert-crlf`、`disable-mdc`、`s2k-mode`、`s2k-digest-algo`、`s2k-cipher-algo`。

> 補充（Senior Dev）：PGP symmetric encrypt 內部流程：隨機產生 session key → 用 session key 對稱加密 data → 用 password + S2K（String-to-Key）派生 key 加密 session key → 組裝 PGP message。這意味著即使兩次用相同 password 加密相同 data，輸出也不同（因為 session key 是隨機的）。預設 `cipher-algo=aes128`；建議生產用 `cipher-algo=aes256`。

---

## 5. PGP 公鑰加密

### 5.1 GPG 金鑰生成

```bash
gpg --gen-key
# 選擇: (1) DSA and Elgamal (default)
# keysize: 2048
# expire: 0 (never)
# Real name: digoal

# 導出公鑰和私鑰
gpg -a --export digoal > public.key
gpg -a --export-secret-keys digoal > secret.key
```

### 5.2 金鑰格式轉換（dearmor）

GPG 匯出的金鑰是 ASCII-armor 格式（`-----BEGIN PGP PUBLIC KEY BLOCK-----`），pgcrypto 需要 bytea 格式。使用 `dearmor()` 轉換：

```sql
SELECT dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v1.4.5 (GNU/Linux)

mQGiBFGgIDgRBADALXrWA...
=XGEV
-----END PGP PUBLIC KEY BLOCK-----');
-- → \x9901a20451a02038110400c02d7ad... (raw bytea)
```

### 5.3 加解密

```sql
-- 公鑰加密
SELECT pgp_pub_encrypt('i am digoal', '<public_key_bytea>');

-- 私鑰解密
SELECT pgp_pub_decrypt('<encrypted_bytea>', '<private_key_bytea>');
```

若 GPG 生成私鑰時設了 passphrase，解密需傳入：

```sql
pgp_pub_decrypt(msg bytea, key bytea, psw text [, options text])
```

### 5.4 輔助函數

```sql
-- 查詢加密數據使用對稱密鑰還是公鑰
SELECT pgp_key_id('<encrypted_bytea>');
-- SYMKEY 或公鑰 ID (如 1B437BCCD670D845)

-- bytea ↔ ASCII-armor 轉換
SELECT armor(data bytea);     -- → text (PGP ASCII-armor)
SELECT dearmor(data text);    -- → bytea

-- 產生隨機加密用 bytea
SELECT gen_random_bytes(32);  -- 32 random bytes
```

> 補充（Senior Dev）：
>
> **公鑰加密的生產考量**：
> 1. **Key Rotation**：公鑰/私鑰應定期輪換。需維護 key version 欄位與歷史私鑰的安全備份（解密歷史數據需要舊私鑰）
> 2. **私鑰存儲**：絕對不能存在同一資料庫中。推薦方案：私鑰存在 HashiCorp Vault / AWS KMS / Azure Key Vault，應用層呼叫 KMS decrypt → 拿到明文後才送入 PG（或由應用層直接解密不回傳私鑰給 PG）
> 3. **效能**：PGP 公鑰加密/解密比對稱加密慢約 10-50×（RSA/Elgamal 運算）。大批量加密場景建議用 envelope encryption 模式（generate random symmetric key → encrypt data with symmetric key → encrypt symmetric key with public key）
> 4. **現代替代**：`pg_sodium` / `pgsodium` extension 基於 libsodium，提供 `crypto_box()` / `crypto_secretbox()` 等更現代的 AEAD（Authenticated Encryption），比 pgcrypto 的 PGP 實現更新、更安全、更快。但 pgcrypto 是 contrib 內建，相容性最好
>
> **對稱加密的 Key Management**：
> - 絕對不要把 key 寫死在 SQL / stored procedure 中
> - 使用 `SET SESSION myapp.encryption_key = '...'`（Custom GUC，PG 9.2+）在 connection pool 初始化時設定，function 內用 `current_setting('myapp.encryption_key')` 讀取，避免 key 出現在 query log / pg_stat_activity 中
> - 或將加密 key 存在另一張受限權限的 table 中，搭配 Row-Level Security 限制訪問
>
> **gen_random_bytes 的 entropy 來源**：PG 10+ 使用 OpenSSL 的 CSPRNG（`RAND_bytes()`），在 Linux 上從 `/dev/urandom` 獲取 entropy；PG 16+ 改用内置 CSPRNG，不再依赖 OpenSSL。

---

## 6. 三種加密方案適用場景總結

| 方案 | 適用場景 | 特點 |
|------|----------|------|
| `crypt()` + `gen_salt()` | 密碼儲存 | Slow hash + random salt，每次不同輸出；高破解難度；適合短字串（密碼 ≤ 72 chars） |
| PGP 對稱加密 | 大量數據加密（資料欄位、文件） | 效能好；需管理對稱密鑰（建議存在外部 KMS / custom GUC） |
| PGP 公鑰加密 | 高安全場景（僅應用層持私鑰） | 加密與解密分離；私鑰不存 DB；適合合規需求（PCI-DSS、HIPAA） |

---

## 參考

1. [PostgreSQL pgcrypto 官方文檔](http://www.postgresql.org/docs/9.3/static/pgcrypto.html)
2. [GnuPG Manual](http://www.gnupg.org/gph/en/manual.html)
3. [德哥 blog — pgcrypto 相關系列](http://blog.163.com/digoal@126/blog/static/163877040201342233131835/)
