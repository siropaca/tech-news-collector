# n8n ワークフロー設計

## ワークフロー一覧

| ワークフロー名 | 実行間隔 | 目的 |
|----------------|----------|------|
| Tech News Collector - Fetch RSS | 定期実行 | RSS収集・サブWF呼び出し |
| Tech News Collector - Process RSS Articles | - | 記事処理・AI分析・保存 |
| Tech News Collector - Send (ai_ml) | 定期実行 | AI・機械学習カテゴリの記事配信 |
| Tech News Collector - Send (frontend) | 定期実行 | フロントエンドカテゴリの記事配信 |
| Tech News Collector - Send (business) | 定期実行 | ビジネスカテゴリの記事配信 |
| Tech News Collector - Send (tech_blog) | 定期実行 | 技術ブログカテゴリの記事配信 |
| Notification Error | - | エラー通知用ワークフロー |

---

## Tech News Collector - Fetch RSS（メインワークフロー）

### フロー図

```
[Schedule Trigger]
    │
    ▼
[Set Start Time]
    │
    ▼
[Get Feed Sources]
    │
    ▼
[Loop Feed Sources]
    │
    ├─► (done) ─► [Aggregate Feed Sources] → [Get New Articles] → [Notify Completion]
    │
    └─► [Fetch RSS Feed]
            │
            ▼
        [Is Valid Feed?]
            │
            ├─► (true) ─► [Limit Articles] → [Add Feed Info] → [Process RSS Articles (SubWF)] → [Loop に戻る]
            │
            └─► (false) ─► [Notify Invalid Feed] → [Loop に戻る]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Schedule Trigger | scheduleTrigger | 定期実行 |
| Set Start Time | code | ワークフロー開始時刻を記録 |
| Get Feed Sources | supabase | `feed_sources`から`is_active=true`のレコードを取得 |
| Loop Feed Sources | splitInBatches | フィードを1件ずつループ処理 |
| Fetch RSS Feed | rssFeedRead | 各フィードのURLからRSSを取得 |
| Is Valid Feed? | if | フィードが正常に取得できたか判定 |
| Limit Articles | limit | 取得する記事数を制限 |
| Add Feed Info | code | フィードソースIDを各記事に付与 |
| Process RSS Articles (SubWF) | executeWorkflow | サブワークフローを呼び出し |
| Notify Invalid Feed | slack | 無効なフィードをSlackに通知 |
| Aggregate Feed Sources | aggregate | フィードソースを集約 |
| Get New Articles | supabase | 今回追加された記事を取得（`fetched_at >= startTime`） |
| Notify Completion | slack | 完了通知を送信 |

---

## Tech News Collector - Process RSS Articles（サブワークフロー）

### フロー図

```
[Execute Workflow Trigger]
    │
    ▼
[If test] ─► (テスト時) ─► [Get Feed Source (Test)] → [Fetch RSS (Test)] → [Add Feed Info] ─┐
    │                                                                                        │
    └─► (本番) ──────────────────────────────────────────────────────────────────────────────┘
                │
                ▼
        [Loop Articles]
            │
            ├─► (done) ─► [Done]
            │
            └─► [Normalize RSS Data]
                    │
                    ▼
                [Check Duplicate]
                    │
                    ▼
                [Is Duplicate?]
                    │
                    ├─► (yes) ─► [Skip Duplicate] → [Loop に戻る]
                    │
                    └─► (no) ─► [Restore Normalized Data] → [Analyze with AI] → [Build Article Data] → [Save Article] → [Loop に戻る]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Execute Workflow Trigger | executeWorkflowTrigger | サブWFの起点（passthrough mode） |
| If test | if | テスト実行かどうかを判定 |
| Loop Articles | splitInBatches | 記事を1件ずつループ処理 |
| Normalize RSS Data | code | RSSデータを正規化 |
| Check Duplicate | supabase | `articles`テーブルでguidの重複をチェック |
| Is Duplicate | if | 重複判定（guidが存在するか） |
| Skip Duplicate | noOp | 重複記事をスキップ |
| Restore Normalized Data | code | 正規化データを復元 |
| Analyze with AI | openAi | OpenAIで記事を分析 |
| Build Article Data | code | AI結果と元データを結合 |
| Save Article | supabase | `articles`テーブルに保存 |
| Done | noOp | 処理完了 |

---

## Tech News Collector - Send (*)（配信ワークフロー）

### フロー図

```
[Schedule Trigger]
    │
    ▼
[Get Articles]
    │
    ▼
[Has Articles?]
    │
    ├─► (no) ─► [No Articles (Skip)]
    │
    └─► (yes) ─► [Extract Article Fields] → [Aggregate Articles] → [Generate Slack Message] → [Send to Slack]
                                                                          │                  → [Send to Discord]
                                                                   [OpenAI Chat Model]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Schedule Trigger | scheduleTrigger | 定期実行 |
| Get Articles | supabase | 直近のカテゴリ別記事を取得 |
| Has Articles? | if | 記事が存在するか判定 |
| Extract Article Fields | splitOut | 必要なフィールドのみ抽出 |
| Aggregate Articles | aggregate | 全記事を1つのリストに集約 |
| Generate Slack Message | agent | AIでトレンド要約・おすすめ記事を生成 |
| OpenAI Chat Model | lmChatOpenAi | GPT-5.1モデル |
| Send to Slack | slack | Slackにメッセージ送信（無効化中） |
| Send to Discord | discord | Discordにメッセージ送信 |
| No Articles (Skip) | noOp | 記事がない場合はスキップ |

### カテゴリ別ワークフロー

| ワークフロー名 | 対象カテゴリ | 出力形式 |
|----------------|--------------|----------|
| Send (ai_ml) | ai_ml | トレンド要約 + おすすめ記事 |
| Send (frontend) | frontend | トレンド要約 + おすすめ記事 |
| Send (business) | business | トレンド要約 + おすすめ記事 |
| Send (tech_blog) | tech_blog | おすすめブログ記事（ひとこと紹介付き） |

### Supabaseフィルタ設定

配列型カラムのフィルタには `cs`（contains）演算子を使用：

```
category=cs.{カテゴリ名}&published_at=gte.{{ $now.minus({hours: N}).toUTC().toISO() }}
```

- `cs.{カテゴリ名}`: category配列に指定カテゴリが含まれる
- `gte`: 以上（greater than or equal）
- `.toUTC().toISO()`: タイムゾーン問題を回避するためUTC形式で出力
- `N`: 実行間隔に合わせて調整

---

## Normalize RSS Data（コード）

RSSフィードのアイテムを正規化するコード。

```javascript
const item = $input.item.json;

return {
  guid: item.guid || item.id || item.link,
  title: item.title,
  url: item.link,
  content: item.content || item.contentSnippet || item.summary || item.description || '',
  published_at: item.pubDate || item.isoDate || item.date,
  feed_source_id: item.feedSourceId
};
```

---

## Analyze with AI（システムプロンプト）

```
あなたは技術記事を分析するアシスタントです。
渡された記事情報を分析し、指定されたJSON形式で結果を返してください。

## 分析タスク

1. **title（タイトル）**: 日本語のタイトル（英語の場合は翻訳）
2. **category（カテゴリ）**: 記事の内容に該当するカテゴリを1〜3個選択（配列で返す）
3. **keywords（キーワード）**: 記事の重要なキーワードを3〜5個抽出
4. **summary（要約）**: 記事の要点を日本語で要約
5. **original_language（元言語）**: 記事の元の言語を判定

## タイトルのルール

- 日本語記事: 元のタイトルをそのまま使用
- 英語記事: 日本語に翻訳（固有名詞・サービス名・技術用語は英語のまま維持）

## カテゴリ一覧（必ずこの中から選択）

- ai_ml: AI・機械学習・LLM関連
- frontend: フロントエンド（React, Vue, CSS等）
- backend: バックエンド（API, サーバーサイド等）
- infrastructure: インフラ・クラウド（AWS, GCP, Azure, Kubernetes等）
- security: セキュリティ・脆弱性
- devops: DevOps・CI/CD・自動化
- mobile: モバイルアプリ開発（iOS, Android, Flutter等）
- database: データベース（SQL, NoSQL等）
- programming: プログラミング言語・開発手法一般
- technology: ハードウェア・ガジェット・新技術（CPU, GPU, デバイス, 量子コンピュータ等）
- ui_ux: UI/UXデザイン・ユーザビリティ・アクセシビリティ
- design: グラフィックデザイン・デザインツール・デザインシステム
- business: テック業界ニュース・企業動向・買収等
- general: 複数分野にまたがる技術トピック・総合的な技術情報
- tech_blog: 技術ブログ
- other: 上記に該当しないもの

## カテゴリ選択のルール

- 最も関連性の高いカテゴリを1〜3個選択する
- 1つで十分な場合は1つだけでよい
- 最大3個まで（4個以上は選択しない）
- 優先度の高い順に並べる

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

## キーワードのルール

- 配列内の各キーワードは必ずダブルクオート（"）で囲むこと
- 日本語のキーワードも必ずダブルクオートで囲むこと
- 例: ["Claude Code", "CLAUDE.md", "コンテキスト", "プロジェクトルール"]

## 出力形式

以下のJSON形式のみを出力してください。説明文やマークダウンの装飾は不要です。
すべての文字列値は必ずダブルクオートで囲んでください。

{
  "title": "日本語タイトル",
  "category": ["カテゴリ1", "カテゴリ2"],
  "keywords": ["キーワード1", "キーワード2", "キーワード3"],
  "summary": "日本語での要約文",
  "original_language": "言語コード"
}
```

---

## Build Article Data（コード）

AI処理結果と元データを結合して保存用データを構築するコード。

```javascript
// Normalize RSS Dataのデータを取得
const formatInput = $('Normalize RSS Data').item.json;

// Analyze with AIの出力からテキストを取得してパース
const aiOutput = $input.item.json.output[0].content[0].text;
const aiResult = JSON.parse(aiOutput);

return {
  json: {
    // 基本情報
    guid: formatInput.guid,
    original_title: formatInput.title,
    url: formatInput.url,
    content: formatInput.content,
    published_at: formatInput.published_at,
    feed_source_id: formatInput.feed_source_id,
    
    // AI分析結果
    title: aiResult.title,
    category: aiResult.category,
    keywords: aiResult.keywords,
    summary: aiResult.summary,
    original_language: aiResult.original_language
  }
};
```

---

## Generate Slack Message（プロンプト）

### 共通ルール

すべてのSendワークフローで以下のルールを適用：

- **見出し**: `## {現在日時}のニュース` 形式（tech_blogは「おすすめブログ記事」）
- **日本語優先**: 同程度の内容であれば日本語記事を優先して選定
- **言語コード表示**: 日本語以外の記事には末尾に `(en)` などを付与
- **リンク形式**: `[タイトル](<URL>)` または `[タイトル](<URL>) (en)`
- **絵文字**: 2行に1つ程度、控えめに添える

### カテゴリ別プロンプト概要

| カテゴリ | 役割 | 出力形式 |
|----------|------|----------|
| ai_ml | AI・機械学習トレンド分析 | トレンド要約4行 + おすすめ記事5件 |
| frontend | フロントエンドトレンド分析 | トレンド要約4行 + おすすめ記事5件 |
| business | ビジネス動向分析 | トレンド要約4行 + おすすめ記事5件 |
| tech_blog | 技術ブログ選定 | おすすめブログ記事（ひとこと紹介付き）最大10件 |

### ユーザープロンプト（共通構造）

```
以下の記事リストを分析してください。

## 現在日時
{{ $now.setZone('Asia/Tokyo').toFormat('yyyy/MM/dd HH') }}時

## ユーザーの関心
（カテゴリに応じた関心事を記載）

## 記事リスト
{{ JSON.stringify($json) }}
```

---

## 設定値

| 項目 | 値 | 備考 |
|------|-----|------|
| RSS収集間隔 | 定期実行 | Fetch RSS |
| 配信間隔 | 定期実行 | Send |
| AIモデル（記事分析） | OpenAI GPT-5.1 | Process RSS Articles |
| AIモデル（配信生成） | OpenAI GPT-5.1 | Send |
| 要約文字数 | 200〜300字 | AI生成 |
| キーワード数 | 3〜5個 | AI抽出 |
| カテゴリ数 | 1〜3個 | AI判定 |

---

## エラーハンドリング

### Fetch RSS WF
- **RSS取得失敗（無効なフィード）**: Slackに通知してスキップ、次のフィードへ
- **サブWF呼び出し失敗**: ログ出力

### Process RSS Articles WF
- **重複記事**: スキップしてループ継続
- **AI APIエラー**: ワークフロー停止（要改善）
- **DB保存エラー**: ワークフロー停止（要改善）

### Send WF
- **記事なし**: スキップ（No Articles）
- **AI APIエラー**: ワークフロー停止（要改善）

### 共通
- **エラー発生時**: Notification Errorワークフローで通知
