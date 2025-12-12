# データベーススキーマ

## 概要

- **データベース**: Supabase (PostgreSQL)
- **リージョン**: ap-northeast-1

## Enum型

### article_category

記事のカテゴリ分類。

| 値 | 説明 |
|----|------|
| `ai_ml` | AI・機械学習 |
| `web_frontend` | Webフロントエンド |
| `web_backend` | Webバックエンド |
| `infrastructure` | インフラ・クラウド |
| `security` | セキュリティ |
| `devops` | DevOps・CI/CD |
| `mobile` | モバイル開発 |
| `database` | データベース |
| `programming` | プログラミング言語・一般 |
| `business` | テック業界ニュース・ビジネス |
| `tech_news` | 技術ニュース全般 |
| `other` | その他 |

### content_language

記事の元言語。

| 値 | 説明 |
|----|------|
| `ja` | 日本語 |
| `en` | 英語 |

## テーブル

### feed_sources

RSSフィードソースの管理テーブル。

| カラム | 型 | NULL | デフォルト | 説明 |
|--------|-----|------|------------|------|
| `id` | UUID | NO | gen_random_uuid() | 主キー（UUID自動生成） |
| `name` | TEXT | NO | - | フィード名（例: Publickey, web.dev） |
| `url` | TEXT | NO | - | RSSフィードのURL（ユニーク制約あり） |
| `default_category` | article_category | NO | 'other' | このフィードのデフォルトカテゴリ |
| `is_active` | BOOLEAN | NO | true | 有効フラグ（falseの場合は収集対象外） |
| `created_at` | TIMESTAMPTZ | NO | now() | レコード作成日時 |
| `updated_at` | TIMESTAMPTZ | NO | now() | レコード更新日時 |

### articles

収集した記事の保存テーブル。

| カラム | 型 | NULL | デフォルト | 説明 |
|--------|-----|------|------------|------|
| `id` | UUID | NO | gen_random_uuid() | 主キー（UUID自動生成） |
| `guid` | TEXT | NO | - | RSSのguid（重複チェック用、ユニーク制約あり） |
| `title` | TEXT | NO | - | 表示用タイトル（常に日本語、英語記事は翻訳済み） |
| `original_title` | TEXT | NO | '' | 元のタイトル（英語記事の場合のみ、日本語記事は空文字） |
| `url` | TEXT | NO | - | 記事のURL |
| `content` | TEXT | NO | - | 記事の本文（RSSから取得した内容） |
| `published_at` | TIMESTAMPTZ | NO | - | 記事の公開日時 |
| `fetched_at` | TIMESTAMPTZ | NO | now() | システムが記事を取得した日時 |
| `category` | article_category | NO | - | AIが判定したカテゴリ |
| `keywords` | TEXT[] | NO | '{}' | AIが抽出したキーワード（配列） |
| `summary` | TEXT | NO | - | AIが生成した日本語要約（200〜300字） |
| `original_language` | content_language | NO | 'ja' | 記事の元言語（ja: 日本語, en: 英語） |
| `created_at` | TIMESTAMPTZ | NO | now() | レコード作成日時 |

**制約**:
- `UNIQUE(guid)` - guid重複を防止

## インデックス

| インデックス名 | テーブル | カラム |
|----------------|----------|--------|
| `idx_articles_fetched_at` | articles | fetched_at |
| `idx_articles_category` | articles | category |
| `idx_articles_published_at` | articles | published_at |
| `idx_feed_sources_is_active` | feed_sources | is_active |

## ER図

```
┌─────────────────────┐       ┌─────────────────────────────┐
│    feed_sources     │       │          articles           │
├─────────────────────┤       ├─────────────────────────────┤
│ id (PK)             │       │ id (PK)                     │
│ name                │       │ guid (UNIQUE)               │
│ url (UNIQUE)        │       │ title                       │
│ default_category    │       │ original_title              │
│ is_active           │       │ url                         │
│ created_at          │       │ content                     │
│ updated_at          │       │ published_at                │
└─────────────────────┘       │ fetched_at                  │
                              │ category                    │
                              │ keywords[]                  │
                              │ summary                     │
                              │ original_language           │
                              │ created_at                  │
                              └─────────────────────────────┘
```
