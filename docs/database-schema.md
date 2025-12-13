# データベーススキーマ

## 概要

- **データベース**: Supabase (PostgreSQL)
- **リージョン**: ap-northeast-1（東京）

## Enum型

### article_category

記事のカテゴリ分類。

| 値 | 説明 |
|----|------|
| `ai_ml` | AI・機械学習 |
| `frontend` | フロントエンド |
| `backend` | バックエンド |
| `infrastructure` | インフラ・クラウド |
| `security` | セキュリティ |
| `devops` | DevOps・CI/CD |
| `mobile` | モバイル開発 |
| `database` | データベース |
| `programming` | プログラミング言語・一般 |
| `technology` | ハードウェア・ガジェット・新技術 |
| `ui_ux` | UI/UXデザイン・ユーザビリティ |
| `design` | グラフィックデザイン・デザインツール |
| `business` | テック業界ニュース・ビジネス |
| `general` | 複数分野にまたがる技術トピック |
| `tech_blog` | 技術ブログ |
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
| `name` | VARCHAR(30) | NO | - | フィード名 |
| `url` | TEXT | NO | - | RSSフィードのURL（ユニーク制約あり） |
| `site_url` | TEXT | YES | - | サイトのURL |
| `default_category` | article_category | NO | 'other' | このフィードのデフォルトカテゴリ |
| `is_active` | BOOLEAN | NO | true | 有効フラグ（falseの場合は収集対象外） |
| `created_at` | TIMESTAMPTZ | NO | now() | レコード作成日時 |
| `updated_at` | TIMESTAMPTZ | NO | now() | レコード更新日時 |

**制約**:
- `PRIMARY KEY(id)`
- `UNIQUE(url)` - URL重複を防止

### articles

収集した技術記事の保存テーブル。

| カラム | 型 | NULL | デフォルト | 説明 |
|--------|-----|------|------------|------|
| `id` | UUID | NO | gen_random_uuid() | 主キー（UUID自動生成） |
| `feed_source_id` | UUID | YES | - | 記事の取得元フィードソース（FK: feed_sources.id） |
| `guid` | TEXT | NO | - | RSSのguid（重複チェック用、ユニーク制約あり） |
| `title` | TEXT | NO | - | 表示用タイトル（常に日本語、英語記事は翻訳済み） |
| `original_title` | TEXT | NO | '' | 元のタイトル（英語記事の場合のみ、日本語記事は空文字） |
| `url` | TEXT | NO | - | 記事のURL |
| `content` | TEXT | NO | - | 記事の本文（RSSから取得した内容） |
| `published_at` | TIMESTAMPTZ | NO | - | 記事の公開日時 |
| `fetched_at` | TIMESTAMPTZ | NO | now() | システムが記事を取得した日時 |
| `category` | article_category[] | NO | - | AIが判定したカテゴリ（配列、1〜3個） |
| `keywords` | TEXT[] | NO | '{}' | AIが抽出したキーワード（配列、3〜5個） |
| `summary` | TEXT | NO | - | AIが生成した日本語要約（200〜300字） |
| `original_language` | content_language | NO | 'ja' | 記事の元言語（ja: 日本語, en: 英語） |
| `created_at` | TIMESTAMPTZ | NO | now() | レコード作成日時 |

**制約**:
- `PRIMARY KEY(id)`
- `UNIQUE(guid)` - guid重複を防止
- `FOREIGN KEY(feed_source_id) REFERENCES feed_sources(id) ON DELETE SET NULL`

## インデックス

| インデックス名 | テーブル | カラム | 用途 |
|----------------|----------|--------|------|
| `idx_articles_fetched_at` | articles | fetched_at | 時間ベースのフィルタリング |
| `idx_articles_category` | articles | category | カテゴリ別の記事取得 |
| `idx_articles_published_at` | articles | published_at | 公開日時でのソート |
| `idx_articles_feed_source_id` | articles | feed_source_id | フィードソース別の記事取得 |
| `idx_feed_sources_is_active` | feed_sources | is_active | アクティブなフィードの取得 |

## クエリ例

### カテゴリで記事をフィルタ（配列対応）

```sql
-- ai_mlカテゴリを含む記事を取得
SELECT * FROM articles 
WHERE 'ai_ml' = ANY(category);

-- PostgREST/Supabase形式
-- category=cs.{ai_ml}
```

### 過去N時間以内に取得した記事

```sql
-- 過去3時間以内
SELECT * FROM articles 
WHERE fetched_at >= NOW() - INTERVAL '3 hours';

-- PostgREST/Supabase形式
-- fetched_at=gte.2025-12-12T08:00:00Z
```

### 複合フィルタ（カテゴリ + 時間）

```sql
SELECT * FROM articles 
WHERE 'ai_ml' = ANY(category)
  AND fetched_at >= NOW() - INTERVAL '3 hours';

-- PostgREST/Supabase形式
-- category=cs.{ai_ml}&fetched_at=gte.2025-12-12T08:00:00Z
```

## ER図

```
┌─────────────────────┐       ┌─────────────────────────────┐
│    feed_sources     │       │          articles           │
├─────────────────────┤       ├─────────────────────────────┤
│ id (PK)             │◄──┐   │ id (PK)                     │
│ name                │   │   │ feed_source_id (FK) ────────┘
│ url (UNIQUE)        │   │   │ guid (UNIQUE)               │
│ site_url            │   │   │ title                       │
│ default_category    │   │   │ original_title              │
│ is_active           │   │   │ url                         │
│ created_at          │   │   │ content                     │
│ updated_at          │   │   │ published_at                │
└─────────────────────┘   │   │ fetched_at                  │
                          │   │ category[]                  │
                          │   │ keywords[]                  │
                          │   │ summary                     │
                          │   │ original_language           │
                          │   │ created_at                  │
                          │   └─────────────────────────────┘
                          │
                     1:N (nullable)
```

## 備考

### published_at vs fetched_at

| カラム | 意味 | 用途 |
|--------|------|------|
| `published_at` | 記事が元サイトで公開された日時 | 記事の新旧を判断 |
| `fetched_at` | n8nが記事を取得・保存した日時 | 配信ワークフローでのフィルタ |

配信ワークフローでは `fetched_at` を使用します。これにより、過去の記事でも新しく取得されたものは配信対象になります。

### カテゴリ配列

- 1記事に1〜3個のカテゴリを割り当て可能
- AIが関連性の高い順に選択
- フィルタには `cs`（contains）演算子を使用
