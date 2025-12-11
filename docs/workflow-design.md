# n8n ワークフロー設計

## ワークフロー一覧

| ワークフロー名 | 実行間隔 | 目的 |
|----------------|----------|------|
| インプットWF | 1時間ごと | RSS収集・分類・保存 |
| アウトプットWF | 6時間ごと | 記事選定・要約・配信 |

---

## インプットワークフロー

### フロー図

```
[Schedule Trigger: 1時間ごと]
    │
    ▼
[Supabase: アクティブなfeed_sources取得]
    │
    ▼
[Loop: 各フィードに対して]
    │
    ├─► [HTTP Request: RSS取得]
    │       │
    │       ▼
    │   [XML Parser: パース]
    │       │
    │       ▼
    │   [Loop: 各記事に対して]
    │       │
    │       ├─► [Supabase: 重複チェック (guidで)]
    │       │       │
    │       │       ▼
    │       │   [IF: 新規記事のみ続行]
    │       │       │
    │       │       ▼
    │       │   [OpenAI: カテゴリ判定・キーワード抽出・要約・翻訳]
    │       │       │
    │       │       ▼
    │       │   [Supabase: 記事保存]
    │       │
    │       └─► [次の記事へ]
    │
    └─► [次のフィードへ]
```

### ノード詳細

#### 1. Schedule Trigger
- **設定**: 1時間ごとに実行
- **Cron**: `0 * * * *`

#### 2. Supabase: フィードソース取得
```sql
SELECT * FROM feed_sources WHERE is_active = true
```

#### 3. HTTP Request: RSS取得
- **Method**: GET
- **URL**: `{{ $json.url }}`（feed_sourcesから）

#### 4. XML Parser
- RSSフィードをJSONにパース

#### 5. Supabase: 重複チェック
```sql
SELECT id FROM articles 
WHERE feed_source_id = '{{ feed_source_id }}' 
  AND guid = '{{ item.guid }}'
```

#### 6. OpenAI: AI処理

**プロンプト例**:
```
以下の記事を分析してください。

【タイトル】
{{ title }}

【本文】
{{ content }}

以下の形式でJSONを返してください：
{
  "category": "カテゴリ（ai_ml, web_frontend, web_backend, infrastructure, security, devops, mobile, database, programming, business, otherのいずれか）",
  "keywords": ["キーワード1", "キーワード2", "キーワード3"],
  "summary": "日本語での要約（100文字程度）",
  "original_language": "ja または en"
}
```

#### 7. Supabase: 記事保存
```sql
INSERT INTO articles (
  feed_source_id, guid, title, url, content, 
  published_at, category, keywords, summary, original_language
) VALUES (...)
```

---

## アウトプットワークフロー

### フロー図

```
[Schedule Trigger: 6時間ごと]
    │
    ▼
[Supabase: 過去6時間の記事取得]
    │
    ▼
[Code: スコアリング・ソート]
    │
    ▼
[Code: 上位N件を抽出]
    │
    ▼
[OpenAI: まとめ要約を生成]
    │
    ▼
[Slack: 通知送信]
```

### ノード詳細

#### 1. Schedule Trigger
- **設定**: 6時間ごとに実行
- **Cron**: `0 */6 * * *`

#### 2. Supabase: 記事取得
```sql
SELECT * FROM articles 
WHERE fetched_at > NOW() - INTERVAL '6 hours'
ORDER BY published_at DESC
```

#### 3. スコアリング（Code Node）

```javascript
// importance_score の算出例
const now = new Date();

for (const item of items) {
  let score = 0;
  
  // 新しさ
  const publishedAt = new Date(item.json.published_at);
  const hoursAgo = (now - publishedAt) / (1000 * 60 * 60);
  if (hoursAgo < 24) score += 30;
  else if (hoursAgo < 48) score += 15;
  
  // キーワードマッチ（注目キーワード）
  const hotKeywords = ['AI', 'ChatGPT', 'LLM', 'セキュリティ', '脆弱性'];
  const keywords = item.json.keywords || [];
  for (const kw of keywords) {
    if (hotKeywords.some(hot => kw.includes(hot))) {
      score += 10;
    }
  }
  
  item.json.importance_score = score;
}

return items.sort((a, b) => b.json.importance_score - a.json.importance_score);
```

#### 4. OpenAI: まとめ要約

**プロンプト例**:
```
以下の記事リストを元に、技術トレンドのダイジェストを作成してください。

【記事リスト】
{{ articles }}

以下の形式で出力してください：
- 全体の傾向（2-3文）
- 注目トピック（箇条書き3-5件）
```

#### 5. Slack: 通知送信

**メッセージフォーマット（Phase 1: シンプル版）**:
```
📰 技術ニュースダイジェスト

【注目記事】
1. {{ title1 }}
   {{ summary1 }}
   {{ url1 }}

2. {{ title2 }}
   {{ summary2 }}
   {{ url2 }}

...
```

---

## 設定値

| 項目 | 値 | 備考 |
|------|-----|------|
| インプット実行間隔 | 1時間 | 調整可能 |
| アウトプット実行間隔 | 6時間 | 調整可能 |
| 配信記事数 | 5-10件 | スコア上位 |
| 要約文字数 | 100文字程度 | AI生成 |

---

## エラーハンドリング

### インプットWF
- **RSS取得失敗**: スキップして次のフィードへ
- **パースエラー**: スキップして次の記事へ
- **AI APIエラー**: リトライ（最大3回）
- **DB保存エラー**: ログ出力、アラート

### アウトプットWF
- **記事0件**: 配信スキップ（または「新着なし」通知）
- **Slack送信失敗**: リトライ（最大3回）
