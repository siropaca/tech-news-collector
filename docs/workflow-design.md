# n8n ワークフロー設計

## ワークフロー一覧

| ワークフロー名 | 実行間隔 | 目的 |
|----------------|----------|----------|------|
| Tech News Collector - Read RSS | 1時間ごと | RSS収集・サブWF呼び出し |
| Tech News Collector - Process Articles | - | 記事処理・AI分析・保存 |

---

## Tech News Collector - Read RSS（メインワークフロー）

### フロー図

```
[Schedule Trigger: 1時間ごと]
    │
    ▼
[Get Feed Sources: Supabaseからアクティブなフィード取得]
    │
    ▼
[Loop Feeds: 各フィードをループ]
    │
    ├─► (ループ完了) ─► [Send a message: Slack完了通知]
    │
    └─► [Read RSS: RSSフィード取得・パース]
            │
            ▼
        [Process Articles: サブワークフロー呼び出し]
            │
            └─► [Loop Feedsに戻る]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Schedule Trigger | scheduleTrigger | 1時間ごとに実行 |
| Get Feed Sources | supabase | `feed_sources`から`is_active=true`のレコードを取得 |
| Loop Feeds | splitInBatches | フィードを1件ずつループ処理 |
| Read RSS | rssFeedRead | 各フィードのURLからRSSを取得 |
| Process Articles | executeWorkflow | サブワークフローを呼び出し |
| Send a message | slack | 完了通知をSlackに送信 |

---

## Tech News Collector - Process Articles（サブワークフロー）

### フロー図

```
[Execute Workflow Trigger: 記事データを受け取る]
    │
    ▼
[Loop Over Items: 記事を1件ずつループ]
    │
    ├─► (ループ完了) ─► [Finished]
    │
    └─► [Format Input: データ正規化]
            │
            ▼
        [Check Duplicate: Supabaseで重複チェック]
            │
            ▼
        [If Exists: 重複判定]
            │
            ├─► (既存) ─► [Skip] ─► [Loop Over Itemsに戻る]
            │
            └─► (新規) ─► [Get Original Data: 元データ取得]
                            │
                            ▼
                        [AI Process: OpenAIで分析]
                            │
                            ▼
                        [Build Article Data: 保存用データ構築]
                            │
                            ▼
                        [Save Article: Supabaseに保存]
                            │
                            └─► [Loop Over Itemsに戻る]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Execute Workflow Trigger | executeWorkflowTrigger | サブWFの起点（passthrough mode） |
| Loop Over Items | splitInBatches | 記事を1件ずつループ処理 |
| Format Input | code | RSSデータを正規化（guid, title, url, content, published_at） |
| Check Duplicate | supabase | `articles`テーブルでguidの重複をチェック |
| If Exists | if | 重複判定（guidが存在するか） |
| Skip (Already Exists) | noOp | 重複記事をスキップ |
| Get Original Data | code | Format Inputの元データを取得 |
| AI Process | openAi | OpenAIで記事を分析 |
| Build Article Data | code | AI結果と元データを結合して保存用データを構築 |
| Save Article | supabase | `articles`テーブルに保存 |

---

## Format Input（コード）

RSSフィードのアイテムを正規化するコード。

```javascript
const item = $input.item.json;

return {
  guid: item.guid || item.id || item.link,
  title: item.title,
  url: item.link,
  content: item.content || item.contentSnippet || item.summary || item.description || '',
  published_at: item.isoDate || item.pubDate
};
```

---

## AI Process（システムプロンプト）

```
あなたは技術記事を分析するアシスタントです。
渡された記事情報を分析し、指定されたJSON形式で結果を返してください。

## 分析タスク

1. **title（タイトル）**: 日本語のタイトル（英語の場合は翻訳）
2. **category（カテゴリ）**: 記事の内容に最も適したカテゴリを1つ選択
3. **keywords（キーワード）**: 記事の重要なキーワードを3〜5個抽出
4. **summary（要約）**: 記事の要点を日本語で要約
5. **original_language（元言語）**: 記事の元の言語を判定

## タイトルのルール

- 日本語記事: 元のタイトルをそのまま使用
- 英語記事: 日本語に翻訳（固有名詞・サービス名・技術用語は英語のまま維持）

## カテゴリ一覧（必ずこの中から選択）

- ai_ml: AI・機械学習・LLM関連
- web_frontend: Webフロントエンド（React, Vue, CSS等）
- web_backend: Webバックエンド（API, サーバーサイド等）
- infrastructure: インフラ・クラウド（AWS, GCP, Azure, Kubernetes等）
- security: セキュリティ・脆弱性
- devops: DevOps・CI/CD・自動化
- mobile: モバイルアプリ開発（iOS, Android, Flutter等）
- database: データベース（SQL, NoSQL等）
- programming: プログラミング言語・開発手法一般
- business: テック業界ニュース・企業動向・買収等
- tech_news: 複数分野にまたがる技術ニュース
- other: 上記に該当しないもの

## 言語一覧（必ずこの中から選択）

- ja: 日本語
- en: 英語

## 要約のルール

- 日本語で200〜300字で作成
- 追加の推測や脚色は禁止（記事に書かれている事実のみ）
- 重要な数値・比較・結論は必ず残す
- 一段落で出力（箇条書き禁止）
- 具体的なソースコードやコマンドは記載禁止
- ですます調で統一（例：〜いる→〜います、〜だ→〜です）
- 「この記事では」「この記事は」「本記事では」などのメタ表現で始めない
- 主語（企業名、サービス名、技術名など）から直接書き始める

## 出力形式

以下のJSON形式のみを出力してください。説明文やマークダウンの装飾は不要です。

{
  "title": "日本語タイトル",
  "category": "カテゴリ値",
  "keywords": ["キーワード1", "キーワード2", "キーワード3"],
  "summary": "日本語での要約文",
  "original_language": "言語コード"
}
```

---

## Build Article Data（コード）

AI処理結果と元データを結合して保存用データを構築するコード。

```javascript
// Format Inputのデータを取得
const formatInput = $('Format Input').item.json;

// AI Processの出力からテキストを取得してパース
const aiOutput = $input.item.json.output[0].content[0].text;
const aiResult = JSON.parse(aiOutput);

return {
  json: {
    // Format Inputからの基本情報
    guid: formatInput.guid,
    original_title: formatInput.title,
    url: formatInput.url,
    content: formatInput.content,
    published_at: formatInput.published_at,
    
    // AI Processからの分析結果
    title: aiResult.title,
    category: aiResult.category,
    keywords: aiResult.keywords,
    summary: aiResult.summary,
    original_language: aiResult.original_language
  }
};
```

---

## 設定値

| 項目 | 値 | 備考 |
|------|-----|------|
| 実行間隔 | 1時間 | Schedule Trigger |
| AIモデル | OpenAI | - |
| 要約文字数 | 200〜300字 | AI生成 |
| キーワード数 | 3〜5個 | AI抽出 |

---

## エラーハンドリング

### Read RSS WF
- **RSS取得失敗**: スキップして次のフィードへ
- **サブWF呼び出し失敗**: ログ出力

### Process Articles WF
- **重複記事**: スキップしてループ継続
- **AI APIエラー**: ワークフロー停止（要改善）
- **DB保存エラー**: ワークフロー停止（要改善）

---

## 今後の改善予定

- [ ] エラーハンドリングの強化（リトライ機構）
- [ ] アウトプットワークフローの実装（定期配信）
- [ ] スコアリング機能の追加
