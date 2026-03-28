---
tags: [hacking, tool, reconnaissance, web, cheatsheet]
category: Reconnaissance
date: 2026-03-28
---

# ffuf チートシート

## 概要

高速なWebファジングツール。ディレクトリ列挙・サブドメイン探索・ログインブルートフォースに使う。

---

## ディレクトリ列挙

```bash
# 基本
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

# PHP ファイルも探す
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -e .php,.html,.txt
```

---

## フィルタリング

```bash
# ヒット数（word数）で絞り込み（不要な結果を除外）
-fw 8

# 応答文字列でマッチ絞り込み
-mr "Welcome"

# 応答サイズで除外（サイズが同じものを除外）
-fs 4605

# 複数サイズを除外
-fs 0,4605
```

---

## サブドメイン探索

```bash
# Host ヘッダーを使ったサブドメイン列挙
ffuf -u http://$IP -H "Host: FUZZ.$HOST" -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

# サイズでフィルタしてヒットを絞り込む
ffuf -u http://$IP -H "Host: FUZZ.$HOST" -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -c -fs 0,4605
```

---

## ログイン ブルートフォース

```bash
# POST ログインフォームへのパスワードブルートフォース
ffuf -u http://$IP/login.php \
  -H "Host: $HOST" \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fw 8 \
  -w /usr/share/wordlists/rockyou.txt | egrep -v ' Errors: '
```

---

## よく使うオプション

| オプション | 説明 |
|----------|------|
| `-u` | ターゲットURL（`FUZZ` がワードリストの値に置換される）|
| `-w` | ワードリストのパス |
| `-H` | ヘッダー追加 |
| `-X` | HTTPメソッド指定 |
| `-d` | POSTデータ |
| `-e` | 拡張子追加（`.php,.html`）|
| `-fw` | word数でフィルタ（除外）|
| `-fs` | サイズでフィルタ（除外）|
| `-mr` | 応答文字列でマッチ（包含）|
| `-c` | カラー出力 |
| `-t` | スレッド数（デフォルト40）|

---

## ハマったポイント

- サブドメイン列挙では `-fs` でデフォルトページのサイズを除外しないとノイズが多い
- `FUZZ` はURLだけでなく `-H` や `-d` 内にも使える

---

## 関連スキル

- [[Tools/Gobuster]]
- [[Skills/Nmap]]
