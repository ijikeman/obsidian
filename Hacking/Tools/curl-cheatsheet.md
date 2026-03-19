---
tags: [hacking, tool, web, curl]
category: Web
date: 2026-01-13
---

# curl チートシート

## 基本構文

```bash
curl [オプション] URL
```

---

## よく使うオプション

### リクエスト

```bash
# GETリクエスト（デフォルト）
curl http://target/

# HTTPメソッド指定
curl -X POST http://target/
curl -X PUT http://target/
curl -X DELETE http://target/

# POSTデータ（フォーム形式）
curl -X POST -d "username=admin&password=pass" http://target/login

# POSTデータ（JSON）
curl -X POST -H "Content-Type: application/json" -d '{"user":"admin"}' http://target/api

# ファイルをPOST
curl -X POST -F "file=@shell.php" http://target/upload
```

### ヘッダー操作

```bash
# 任意ヘッダー追加
curl -H "Authorization: Bearer TOKEN" http://target/

# User-Agent 偽装
curl -A "Mozilla/5.0" http://target/
curl -A "internalcomputer" http://target/ua_check.php

# Referer 指定
curl -e "http://legitimate-site.com" http://target/
```

### Cookie 操作

```bash
# Cookie を保存（ログイン時）
curl -c cookies.txt -d "user=admin&pass=admin" http://target/login

# 保存した Cookie を送信
curl -b cookies.txt http://target/dashboard

# Cookie を直接指定
curl -b "session=abc123; role=admin" http://target/
```

### レスポンス確認

```bash
# ヘッダーも表示
curl -i http://target/

# ヘッダーのみ表示
curl -I http://target/

# 詳細（デバッグ）
curl -v http://target/

# サイレントモード（スクリプト向け）
curl -s http://target/

# リダイレクト追従
curl -L http://target/
```

### 認証

```bash
# Basic認証
curl -u admin:password http://target/

# Bearer Token
curl -H "Authorization: Bearer TOKEN" http://target/api
```

### SSL/TLS

```bash
# 証明書エラーを無視
curl -k https://target/
```

### 出力

```bash
# ファイルに保存
curl -o output.html http://target/
curl -O http://target/file.zip  # ファイル名そのまま保存

# レスポンスコードのみ取得
curl -o /dev/null -s -w "%{http_code}" http://target/
```

---

## ペンテスト向けパターン

### ユーザ列挙・ブルートフォース

```bash
# bash ループでブルートフォース
for pass in $(cat wordlist.txt); do
  response=$(curl -s -X POST -d "user=admin&pass=$pass" http://target/login)
  if echo "$response" | grep -q "Welcome"; then
    echo "[+] Found: $pass"
    break
  fi
done
```

### レスポンスの差分確認

```bash
# 単語数でフィルタ（ffuf の -fw 相当）
curl -s -X POST -d "username=admin&password=test" http://target/login | wc -w

# ステータスコード確認
curl -o /dev/null -s -w "%{http_code}\n" -X POST -d "user=admin&pass=test" http://target/login
```

### APIエンドポイント探索

```bash
# 存在確認
curl -o /dev/null -s -w "%{http_code}" http://target/api/v1/users
```

---

## 関連ツール

- [[Tools/Burp-Suite]] - より高度なリクエスト操作
- [[Tools/ffuf]] - ブルートフォースに特化
