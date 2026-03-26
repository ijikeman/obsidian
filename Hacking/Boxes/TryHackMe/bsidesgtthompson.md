---
tags: [hacking, box, writeup]
platform: TryHackMe
difficulty: Easy
status: in-progress
date: 2026-03-27
skills:
  - Ghostcat (CVE-2020-1938)
  - AJP13
  - Apache Tomcat
---

# bsidesgtthompson

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux

---

## 情報収集

### Nmap

```bash
nmap -sC -sV -T4 10.144.129.213
```

**開いているポート:**
| ポート | サービス | バージョン |
|-------|---------|----------|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 8009/tcp | AJP13 | Apache Jserv Protocol v1.3 |
| 8080/tcp | HTTP | Apache Tomcat 8.5.5 |

---

## 脆弱性発見

### 発見した脆弱性

- **Ghostcat (CVE-2020-1938)** - AJP13 (8009) が外部公開されている
  - Apache Tomcat 8.5.5 は脆弱なバージョン（修正版: 8.5.51+）
  - 認証なしで任意のファイル読み取り可能
  - `WEB-INF/web.xml` 等の設定ファイルから認証情報取得の可能性
  - 条件次第でRCE（リモートコード実行）も可能

### AJP (Apache JServ Protocol) とは

```
[ブラウザ] --HTTP--> [Apache httpd] --AJP--> [Tomcat]
```

WebサーバーとJavaアプリケーションサーバー間の通信プロトコル。
通常は内部通信用で外部公開すべきでない。

### 調査メモ

- Tomcat Manager (`/manager/html`) は認証あり → デフォルト認証情報を試す予定
- AJP13ポートが開放されているためGhostcat攻撃の優先度高

---

## 侵入 / Initial Access

> ※ 作業中

---

## 権限昇格 / Privilege Escalation

> ※ 未着手

---

## 習得スキル・学び

- AJP13プロトコルの役割と危険性
- Ghostcat (CVE-2020-1938) の概要

## ハマったポイント

-

## 参考にしたリソース

-
