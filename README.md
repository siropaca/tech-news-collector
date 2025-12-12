# Tech News Collector

技術記事を自動収集し、AIで分類・要約してSlackに配信するシステムです。

## 概要

RSSフィードから技術記事を定期的に収集し、OpenAI を使用してカテゴリ分類・キーワード抽出・要約生成を行います。カテゴリ別にSlackへトレンド要約とおすすめ記事を配信します。

## 技術スタック

- **ワークフロー**: n8n
- **データベース**: Supabase (PostgreSQL)
- **AI**: OpenAI (GPT-5.1)
- **通知**: Slack

## 主な機能

### 記事収集・分析
- RSSフィードからの記事自動収集（1時間ごと）
- AIによるカテゴリ自動分類（13種類、複数カテゴリ対応）
- キーワード抽出（3〜5個）
- 日本語要約生成（200〜300字）
- 英語記事のタイトル自動翻訳
- 重複記事の自動スキップ

### Slack配信
- カテゴリ別のトレンド要約配信（3時間ごと）
- AIによるおすすめ記事選定
- ユーザーの関心に基づいたパーソナライズ

## ワークフロー

| ワークフロー名 | 実行間隔 | 目的 |
|----------------|----------|------|
| Tech News Collector - Fetch RSS | 1時間ごと | RSS収集 |
| Tech News Collector - Process RSS Articles | - | 記事分析・保存 |
| Tech News Collector - Send Slack (ai_ml) | 3時間ごと | AI/ML記事配信 |

## ドキュメント

- [システムアーキテクチャ](docs/architecture.md)
- [データベーススキーマ](docs/database-schema.md)
- [ワークフロー設計](docs/workflow-design.md)
