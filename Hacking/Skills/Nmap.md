---
tags: [hacking, skill, reconnaissance]
category: Reconnaissance
difficulty: beginner
learned_from: TryHackMe
date: 2026-03-19
---

# Nmap

## 概要
ネットワークスキャンツール。開いているポートやサービスを列挙する。

## 使いどころ
- 最初の情報収集（必ず最初に実行）
- サービスのバージョン確認
- OS検出

## 基本コマンド

```bash
# 基本スキャン
nmap <IP>

# バージョン・スクリプトスキャン（よく使う）
nmap -sC -sV <IP>

# 全ポートスキャン
nmap -p- <IP>

# 高速スキャン → 詳細スキャン（2段階）
nmap -p- --min-rate 5000 <IP>
nmap -sC -sV -p <発見したポート> <IP>

# 結果をファイル保存
nmap -sC -sV -oN nmap/initial <IP>
nmap -sC -sV -oA nmap/full <IP>   # 3形式で保存
```

## 主要オプション

| オプション | 説明 |
|----------|------|
| `-sC` | デフォルトスクリプト実行 |
| `-sV` | バージョン検出 |
| `-sU` | UDPスキャン |
| `-p-` | 全65535ポートスキャン |
| `-p 80,443,8080` | 特定ポートのみ |
| `-A` | OS検出・スクリプト・トレースルート |
| `-O` | OS検出 |
| `--min-rate 5000` | 速度指定（CTFで有効） |
| `-oN` | 通常形式で保存 |
| `-oA` | 全形式で保存 |

## よくあるパターン

### CTF / Box攻略での標準フロー
```bash
# Step1: 高速スキャンで開いているポートを把握
nmap -p- --min-rate 5000 -T4 <IP> -oN nmap/ports

# Step2: 発見したポートを詳細スキャン
nmap -sC -sV -p 22,80,443 <IP> -oN nmap/detailed
```

### UDP含む詳細スキャン
```bash
nmap -sC -sV -sU -p- <IP>
```

## ハマったポイント
- `-p-` は時間がかかるので `--min-rate` と組み合わせる
- ファイアウォール環境では `-Pn`（ ping不要）が必要な場合がある

## 参考リンク
- [TryHackMe: Nmap Room](https://tryhackme.com/room/furthernmap)

## 関連スキル
- [[Gobuster]] - ディレクトリ列挙
- [[Enumeration]]
