# n8n ワークフロー設計

## ワークフロー一覧

| ワークフロー名 | 実行間隔 | 目的 |
|----------------|----------|------|
| Tech News Collector - Fetch RSS | 定期実行 | RSS収集・サブWF呼び出し |
| Tech News Collector - Process RSS Articles | - | 記事処理・AI分析・保存 |
| Tech News Collector - Send Slack (*) | 定期実行 | カテゴリ別記事配信 |

---

## Tech News Collector - Fetch RSS（メインワークフロー）

### フロー図

```
[Schedule Trigger]
    │
    ▼
[Get Active Feed Sources]
    │
    ▼
[Loop Feed Sources]
    │
    ├─► (done) ─► [Notify Completion]
    │
    └─► [Fetch RSS Feed] → [Add Feed Info] → [Process RSS Articles (SubWF)] → [Loop に戻る]
```

### ノード詳細

| ノード名 | タイプ | 説明 |
|----------|--------|------|
| Schedule Trigger | scheduleTrigger | 定期実行 |
| Get Active Feed Sources | supabase | `feed_sources`から`is_active=true`のレコードを取得 |
| Loop Feed Sources | splitInBatches | フィードを1件ずつループ処理 |
| Fetch RSS Feed | rssFeedRead | 各フィードのURLからRSSを取得 |
| Add Feed Info | code | フィードソースIDを各記事に付与 |
| Process RSS Articles (SubWF) | executeWorkflow | サブワークフローを呼び出し |
| Notify Completion | - | 完了通知を送信 |

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

## Tech News Collector - Send Slack (*)（配信ワークフロー）

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
    └─► (yes) ─► [Extract Article Fields] → [Aggregate Articles] → [Generate Message] → [Send Message]
                                                                          │
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
| Generate Message | agent | AIでトレンド要約・おすすめ記事を生成 |
| OpenAI Chat Model | lmChatOpenAi | GPT-5.1モデル |
| Send Message | - | 配信先にメッセージ送信 |
| No Articles (Skip) | noOp | 記事がない場合はスキップ |

### Supabaseフィルタ設定

配列型カラムのフィルタには `cs`（contains）演算子を使用：

```
category=cs.{カテゴリ名}&fetched_at=gte.{{ $now.minus({hours: N}).toUTC().toISO() }}
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
- business: テック業界ニュース・企業動向・買収等
- tech_news: 複数分野にまたがる技術ニュース
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

## 出力形式

以下のJSON形式のみを出力してください。説明文やマークダウンの装飾は不要です。

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

## Generate Message（システムプロンプト）

カテゴリに応じてシステムプロンプトを調整してください。以下はai_mlカテゴリの例です。

```
あなたはAI・機械学習分野の技術トレンドを分析するアシスタントです。
渡された記事リストを分析し、通知用のメッセージを生成してください。

## 出力形式

以下の形式で出力してください。マークダウンやコードブロックの装飾は不要です。

### 出力構成

1. **トレンド要約**（4行程度）
   - 記事全体から読み取れるトレンドやトピックを簡潔にまとめる
   - 句点（。）ごとに改行する
   - 具体的な技術名やサービス名を含める

2. **おすすめ記事**（5件程度）
   - ユーザーの関心に合った記事を選定
   - Discord形式のリンク [タイトル](<URL>) で記載（カード非表示のため`<>`でURLを囲む）
   - 選定理由は不要

### トレンド要約のスタイル

- ですます調を基本とする
- 絵文字は句点（。）の代わりに文末に置く（「〜です 🔥」「〜そうです 👀」のように）
- 絵文字の前には必ず半角スペースを入れる
- 絵文字は文脈や内容に合ったものを自由に選ぶ（同じ絵文字を繰り返し使わず、バリエーションを持たせる）
- 英語と日本語の間には必ず半角スペースを入れる（例：「Google が MCP 周りの整備を進めています」）
- 「〜ですね」「〜かも」などの口語表現は使わず、「〜です」「〜します」で終える
- 硬すぎるニュース文体ではなく、社内の技術共有チャンネルに投稿するような親しみやすいトーンで書く
- 開発者が「お、気になる」と思えるような書き方を意識する

### トレンド要約の書き方

- 新サービスや新モデルの発表は「〜が発表されました」「〜がリリースされました」のように、まずニュースとして伝える
- いきなり詳細な説明から入らず、何が起きたのかを最初に簡潔に述べる
- 読者が前提知識なしでも理解できるよう、固有名詞が初出の場合は簡単な補足を入れる
- 複数のトピックがある場合は、関連性のある話題をまとめて自然な流れで紹介する

### おすすめ記事の選定基準（優先度順）

1. **複数媒体で取り上げられているテーマ** - 異なるソースで重複して報じられているトピックは注目度が高いため必ず含める
2. **ユーザーの関心に合致する記事** - ユーザーが指定した興味・関心に沿った内容
3. **業界への影響が大きいニュース** - 新サービス発表、標準化動向、大手企業の発表など

### 出力例

OpenAI が新モデル GPT-5.2 を発表しました 🎉
専門的な知識業務で人間の専門家レベルを目指すモデルで、ChatGPT や Microsoft 365 Copilot に順次展開される予定です 📚
Google も MCP 周りの整備を進めており、Cloud API Registry で MCP サーバの管理が一元化できるようになります 🛠️
AI エージェントと外部 API の連携がクラウドレベルで整ってきており、実務での活用がかなり現実的になってきています 🚀

📌 おすすめ記事
• [OpenAI、フラグシップモデル「GPT-5.2」を発表](<https://example.com/article1>)
• [Google、MCPサーバの発見や管理のためのレジストリ「Cloud API Registry」プレビュー公開](<https://example.com/article2>)
• ...
```

### ユーザープロンプト

カテゴリに応じてユーザーの関心を調整してください。以下はai_mlカテゴリの例です。

```
以下の記事リストを分析してください。

## ユーザーの関心
普段はWebアプリケーションの開発をしています。
最近はAIエージェントやLLMを使った開発に興味があり、特にMCPやマルチエージェントシステムに注目しています。
実務で使えるAIツールやAPIの情報などを追いかけたいです。

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
- **RSS取得失敗**: スキップして次のフィードへ
- **サブWF呼び出し失敗**: ログ出力

### Process RSS Articles WF
- **重複記事**: スキップしてループ継続
- **AI APIエラー**: ワークフロー停止（要改善）
- **DB保存エラー**: ワークフロー停止（要改善）

### Send WF
- **記事なし**: スキップ（No Articles）
- **AI APIエラー**: ワークフロー停止（要改善）
