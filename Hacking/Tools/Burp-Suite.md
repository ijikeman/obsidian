---
tags: [hacking, tool, web, proxy, cheatsheet]
category: Webアタック
date: 2026-03-28
---

# Burp Suite チートシート

## 概要

WebアプリケーションのHTTPトラフィックを傍受・改ざん・リプレイするプロキシツール。

---

## 初期セットアップ

### 1. Burp Suite のプロキシポート変更

`Proxy > Proxy Settings` で以下に変更：

```
127.0.0.1:8080 → 127.0.0.1:8081
```

> ※ Tomcat等が 8080 を使っている場合に競合するため

---

### 2. Firefox の設定

#### FoxyProxy アドオンをインストール

1. Firefox のアドオン設定から「FoxyProxy」を検索して追加

#### FoxyProxy にプロキシを追加

FoxyProxy アイコン → Options → Proxies → Add

```
Title:    burp
Hostname: 127.0.0.1
Port:     8081
```

---

### 3. Burp Suite の CA 証明書を Firefox に追加

1. Firefox で `http://127.0.0.1:8081` にアクセス
2. 「CA Certificate」リンクから証明書をダウンロード
3. Firefox の設定 → Privacy & Security → Certificates → 証明書をインポート

---

### 4. 動作確認

1. Burp Suite の `Proxy` 画面で「Intercept off」→「Intercept on」に変更
2. FoxyProxy で「burp」プロファイルを選択
3. Firefox で任意のURLにアクセスし、Burp Suite にリクエストが表示されればOK

---

## よく使う機能

| 機能 | 説明 |
|------|------|
| **Proxy > Intercept** | リクエストを傍受・改ざん |
| **Proxy > HTTP History** | 通信履歴の確認 |
| **Repeater** | リクエストを手動で送り直す |
| **Intruder** | ブルートフォース・ファジング |
| **Decoder** | Base64/URL エンコード・デコード |
| **Scanner** (Pro) | 自動脆弱性スキャン |

---

## ハマったポイント

- HTTPS サイトは CA証明書を追加しないと傍受できない
- Intercept on のままにすると全リクエストが止まるので注意

---

## 関連スキル

- [[Tools/curl-cheatsheet]]
- [[Tools/ffuf]]
