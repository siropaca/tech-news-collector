# データベーススキーマ

## 概要

- **データベース**: Supabase (PostgreSQL)
- **プロジェクトID**: `vpcpxosutscbjmwvbchi`
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
| `id` | UUID | NO | gen_random_uuid() | 主キー |
| `name` | TEXT | NO | - | フィード名 |
| `url` | TEXT | NO | - | フィードURL（UNIQUE） |
| `default_category` | article_category | YES | - | デフォルトカテゴリ |
| `is_active` | BOOLEAN | YES | true | 有効フラグ |
| `created_at` | TIMESTAMPTZ | YES | now() | 作成日時 |
| `updated_at` | TIMESTAMPTZ | YES | now() | 更新日時 |

### articles

収集した記事の保存テーブル。

| カラム | 型 | NULL | デフォルト | 説明 |
|--------|-----|------|------------|------|
| `id` | UUID | NO | gen_random_uuid() | 主キー |
| `feed_source_id` | UUID | YES | - | フィードソースID（FK） |
| `guid` | TEXT | NO | - | RSSのguid（重複チェック用） |
| `title` | TEXT | NO | - | 記事タイトル |
| `url` | TEXT | NO | - | 記事URL |
| `content` | TEXT | YES | - | 記事本文 |
| `published_at` | TIMESTAMPTZ | YES | - | 公開日時 |
| `fetched_at` | TIMESTAMPTZ | YES | now() | 取得日時 |
| `category` | article_category | YES | - | カテゴリ（AI判定） |
| `keywords` | TEXT[] | YES | - | キーワード（AI抽出） |
| `summary` | TEXT | YES | - | 要約（AI生成、日本語） |
| `original_language` | content_language | YES | 'ja' | 元の言語 |
| `created_at` | TIMESTAMPTZ | YES | now() | 作成日時 |

**制約**:
- `UNIQUE(feed_source_id, guid)` - 同一フィード内でのguid重複を防止

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
│ id (PK)             │──────<│ feed_source_id (FK)         │
│ name                │       │ id (PK)                     │
│ url (UNIQUE)        │       │ guid                        │
│ default_category    │       │ title                       │
│ is_active           │       │ url                         │
│ created_at          │       │ content                     │
│ updated_at          │       │ published_at                │
└─────────────────────┘       │ fetched_at                  │
                              │ category                    │
                              │ keywords[]                  │
                              │ summary                     │
                              │ original_language           │
                              │ created_at                  │
                              ├─────────────────────────────┤
                              │ UNIQUE(feed_source_id, guid)│
                              └─────────────────────────────┘
```

## マイグレーション

### 初期スキーマ作成

```sql
-- カテゴリのenum型
CREATE TYPE article_category AS ENUM (
  'ai_ml',
  'web_frontend', 
  'web_backend',
  'infrastructure',
  'security',
  'devops',
  'mobile',
  'database',
  'programming',
  'business',
  'other'
);

-- 言語のenum型
CREATE TYPE content_language AS ENUM (
  'ja',
  'en'
);

-- RSSフィードソース
CREATE TABLE feed_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  url TEXT NOT NULL UNIQUE,
  default_category article_category,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 収集した記事
CREATE TABLE articles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  feed_source_id UUID REFERENCES feed_sources(id),
  
  -- 記事の基本情報
  guid TEXT NOT NULL,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  content TEXT,
  published_at TIMESTAMPTZ,
  fetched_at TIMESTAMPTZ DEFAULT now(),
  
  -- AI処理結果
  category article_category,
  keywords TEXT[],
  summary TEXT,
  original_language content_language DEFAULT 'ja',
  
  created_at TIMESTAMPTZ DEFAULT now(),
  
  UNIQUE(feed_source_id, guid)
);

-- インデックス
CREATE INDEX idx_articles_fetched_at ON articles(fetched_at);
CREATE INDEX idx_articles_category ON articles(category);
CREATE INDEX idx_articles_published_at ON articles(published_at);
CREATE INDEX idx_feed_sources_is_active ON feed_sources(is_active);
```
