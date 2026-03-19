---
tags: [hacking, tool, reconnaissance, web]
category: Reconnaissance
date: 2026-03-19
---

# Gobuster

## 概要
ディレクトリ・ファイル・サブドメインをブルートフォースで列挙するツール。

## 基本コマンド

```bash
# ディレクトリ列挙
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 拡張子指定
gobuster dir -u http://<IP> -w <wordlist> -x php,html,txt

# サブドメイン列挙
gobuster dns -d <domain> -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

# HTTPS
gobuster dir -u https://<IP> -w <wordlist> -k  # -k: 証明書エラー無視
```

## よく使うワードリスト

```bash
# Kali/ParrotOS 標準
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/usr/share/wordlists/dirb/common.txt

# SecLists（別途インストール）
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

## 関連スキル
- [[Nmap]]
- [[Feroxbuster]]  ← 再帰的スキャンならこちら
