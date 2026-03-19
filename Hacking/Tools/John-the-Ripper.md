---
tags: [hacking, tool, password-attacks]
category: Password Attacks
date: 2026-01-15
---

# John the Ripper

## 概要
パスワードハッシュ・暗号化ファイルのクラッキングツール。

## 基本コマンド

```bash
# ワードリストで解析
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# 解析結果を表示
john hash.txt --show

# 対応フォーマットの確認
john --list=formats
```

## よく使うパターン

### SSH 秘密鍵のパスフレーズ解析

```bash
# 1. 秘密鍵をハッシュ形式に変換
ssh2john id_rsa > id_rsa.hash

# 2. John で解析
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt

# 3. 結果確認
john id_rsa.hash --show
```

### /etc/shadow のハッシュ解析

```bash
# unshadow で passwd と shadow を結合
unshadow /etc/passwd /etc/shadow > combined.txt

john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### ZIP/RAR ファイル

```bash
zip2john secret.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### ハッシュ形式を指定

```bash
john hash.txt --format=md5crypt --wordlist=...
john hash.txt --format=bcrypt --wordlist=...
john hash.txt --format=NT --wordlist=...     # Windows NTLMハッシュ
```

## 関連ツール

- [[Tools/Hashcat]] ← GPU使用・より高速
- [[Skills/Password-Attacks]]
