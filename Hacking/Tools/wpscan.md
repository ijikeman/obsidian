---
tags: [hacking, tool, web, wordpress]
category: Web
date: 2026-01-15
---

# wpscan

## 概要
WordPress 専用のセキュリティスキャナー。バージョン・テーマ・プラグイン・ユーザを列挙できる。

## 基本コマンド

```bash
# 基本スキャン
wpscan --url http://target/

# HTTPS (TLS検証無効)
wpscan --url https://target/ --disable-tls-checks

# ユーザ列挙
wpscan --url http://target/ -e u

# プラグイン列挙
wpscan --url http://target/ -e ap

# テーマ列挙
wpscan --url http://target/ -e at

# 全列挙
wpscan --url http://target/ -e u,ap,at

# API Token付き（脆弱性データを取得）
wpscan --url http://target/ --api-token YOUR_TOKEN
```

## 重要な発見ポイント

| 発見内容 | 意味 |
|---------|------|
| WordPress バージョン | 古いバージョンは既知脆弱性あり |
| テーマ・バージョン | テーマ固有の CVE を調べる |
| プラグイン | 脆弱なプラグインは攻撃ベクター |
| XML-RPC 有効 | ブルートフォース攻撃に悪用可能 |
| ユーザ名 | BF 攻撃のターゲット |

## よく見つかる脆弱テーマ・プラグイン

| 名前 | バージョン | CVE | 内容 |
|-----|----------|-----|------|
| Bricks Builder | < 1.9.6 | CVE-2024-25600 | 認証不要 RCE |
| Elementor | 複数 | 複数 | XSS, RCE |
| WPForms | 複数 | 複数 | SQLi |

## 関連スキル
- [[Skills/WordPress-Exploitation]]
