---
tags: [hacking, box, writeup]
platform: TryHackMe/HackTheBox
difficulty: Easy/Medium/Hard
status: completed/in-progress
date: YYYY-MM-DD
skills:
  -
---

# [Box名]

**プラットフォーム**: TryHackMe / HackTheBox
**難易度**: Easy / Medium / Hard
**OS**: Linux / Windows

---

## 情報収集

### Nmap
```bash
nmap -sC -sV -oN nmap/initial <IP>
```

**開いているポート:**
| ポート | サービス | バージョン |
|-------|---------|----------|
|       |         |          |

### ディレクトリ列挙
```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**発見したパス:**
-

---

## 脆弱性発見

### 発見した脆弱性
-

### 調査メモ
-

---

## 侵入 / Initial Access

```bash

```

**ユーザーフラグ**: `user.txt`

---

## 権限昇格 / Privilege Escalation

### 調査
```bash
# sudo -l, SUID, cron, etc.
```

### 昇格手順
-

**ルートフラグ**: `root.txt`

---

## 習得スキル・学び
- スキル1
- スキル2

## ハマったポイント
-

## 参考にしたリソース
-
