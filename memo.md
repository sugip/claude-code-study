# Claude Code 学習メモ

## ドキュメント読み

### [Claude Code の仕組み](https://code.claude.com/docs/ja/how-claude-code-works)

#### ブランチ間で作業する

セッションはディレクトリに結び付けられているため、git worktrees を使用して並列 Claude Code セッションを実行できる。

**活用メモ:** レビュー用 worktree を作成すれば、作業用のメインディレクトリと異なるセッションで作業させられる。

#### セッションを再開またはフォークする

`claude --continue --fork-session` で元のセッションに影響を与えずに別のアプローチを試すことができる。

**活用メモ:** 複数のターミナルで同一セッションを使って同じ開始点から並列作業する場合は、`--fork-session` を使用して各ターミナルに独自のクリーンなセッションを与えるのが良い。

#### skills と subagents でコンテキストを管理する

手動で起動する skill（= AI に起動させたくない）は `disable-model-invocation: true` をつけるのが良い。

**理由:** skill の説明が context に読み込まれて context を逼迫するのを防ぐため。

#### チェックポイントで変更を元に戻す

ESC を2回押すことで変更内容を元に戻せる。

#### 中断して操舵する

間違った道を進んでいる場合は、修正を入力して Enter キーを押すだけ。完了を待たずにそのまま入力して良い。

#### Claude が検証するものを与える

ビジュアル作業の場合、デザインのスクリーンショットを貼り付けて、Claude にその実装と比較するよう求める。

**活用メモ:** UI 構築時に使える。まずは仕様ベースで実装してもらい、後でデザインとの比較をやるのが良さそう。

### [Claude Code を拡張する](https://code.claude.com/docs/ja/features-overview)

#### Rules（モジュール化されたルール）

`.claude/rules/*.md` にトピック別のMarkdownファイルを置くことで、CLAUDE.md を分割して管理できる。

```
.claude/
├── CLAUDE.md
└── rules/
    ├── code-style.md
    ├── testing.md
    └── security.md
```

CLAUDE.md と同じ優先度で、セッション開始時に自動で全て読み込まれる。

YAMLフロントマターの `paths` フィールドで特定ファイルにのみ適用するルールを定義できる。

```markdown
---
paths: src/api/**/*.ts
---

# API開発ルール
- 全APIエンドポイントに入力バリデーションを含める
```

**Rules とリファレンス skill の使い分け:**

| | Rules | リファレンス skill |
|---|---|---|
| 読み込み | 常時（セッション開始時） | オンデマンド（呼び出し時のみ） |
| 用途 | 常に守るべき指示・制約 | 必要なときに参照する知識・ドキュメント |
| 例 | テスト規約、セキュリティ要件 | API スタイルガイド、DB スキーマ |

**活用メモ:** CLAUDE.md が 500 行を超えてきたらリファレンスコンテンツは skill へ、指示・制約は rules へ切り出すのが良い。

#### subagent の呼び出しパターン

skill から subagent を呼び出す方法は2種類ある。

**パターン1：自然言語でアドホックに指示する**

skill 本文に自然言語で書くだけ。Claude が Task ツールを使って自動的に subagent を生成する。

```markdown
1. Use a Haiku agent to check if the pull request is closed...
4. Then, launch 5 parallel Sonnet agents to independently code review...
```

**パターン2：`agents/` ディレクトリに定義ファイルを置く**

`agents/*.md` に専用ファイルを用意し、skill から名前で参照する。

```markdown
---
name: code-reviewer
description: Use this agent when...  ← 自動呼び出しの判断基準
model: sonnet                         ← 使用するモデル
tools: Glob, Grep, Read, Bash         ← 使えるツールを絞り込める
color: red
---

You are an expert code reviewer...   ← subagent へのシステムプロンプト
```

**2つのパターンの比較:**

| | パターン1（アドホック） | パターン2（定義ファイル） |
|---|---|---|
| 定義場所 | skill 本文に自然言語で記述 | `agents/*.md` に専用ファイル |
| 再利用 | できない | 他の skill からも呼び出せる |
| モデル指定 | 自然言語で `"Use a Haiku agent"` | フロントマターの `model:` フィールド |
| 自動呼び出し | なし | `description:` を見て Claude が判断 |

**活用メモ:** 同じエージェントを複数の skill から使い回すなら `agents/` に定義ファイルを置く。1回限りのシンプルなタスクならアドホックで十分。

### [一般的なワークフロー](https://code.claude.com/docs/ja/common-workflows)

#### PR にリンクされたセッションを再開する

`gh pr create` で PR を作成すると、そのセッションが PR に自動的にリンクされる。PR 番号を指定して後から再開できる。

```bash
claude --from-pr 123
```

**ユースケース:**

- **レビューコメント対応** — PR を書いたときの文脈（設計判断の経緯など）を持ったまま修正できる
- **時間を空けて作業再開** — セッション名を覚えていなくても PR 番号さえ分かれば呼び出せる
- **複数回のレビューサイクル** — レビュー対応のたびに同じ PR のセッションに戻ることで文脈の連続性が保てる

**活用メモ:** PR 単位でセッションが管理できる。「この PR の作業セッション」として紐づけておくことで、レビュー対応のたびに文脈を失わずに済む。

#### ヘッドレスモード

`-p` フラグで Claude を非インタラクティブに実行するモード。1回のクエリを実行して結果を返して終了する。

```bash
# 基本的な使い方
claude -p "コードを分析してください"

# パイプとの連携
cat build-error.txt | claude -p 'このエラーの原因を説明してください' > output.txt

# Plan Mode と組み合わせて読み取り専用分析
claude --permission-mode plan -p "認証システムを分析して改善案を提案してください"
```

出力形式の指定:

| フラグ | 用途 |
|---|---|
| `--output-format text` | プレーンテキスト（デフォルト） |
| `--output-format json` | コスト・期間などのメタデータ付き JSON |
| `--output-format stream-json` | リアルタイムでストリーミング出力 |

**活用メモ:** CI/CD パイプラインへの組み込みや Unix パイプとの連携など、「Claude をスクリプトの一部として使う」場面で使う。

### [ベストプラクティス](https://code.claude.com/docs/ja/best-practices)

#### [Chrome 拡張機能を使った UI 検証](https://code.claude.com/docs/ja/chrome)

`claude --chrome` または `/chrome` で Chrome 拡張機能と連携し、コーディングとブラウザ操作を1つのワークフローで完結させられる。

**主なユースケース:**

- **ローカル Web アプリのテスト** — `localhost:3000` を開いてフォーム操作・エラー表示を確認させる
- **デザイン検証** — Figma モックを渡して実装 → ブラウザで開いて差分確認のループを自動化
- **コンソールデバッグ** — ページロード時のコンソールエラーを読み取り → 原因コードを修正まで一気に完結
- **デモ GIF 作成** — ユーザーフローを録画して GIF として保存

**前提条件（制約）:**

- Google Chrome 専用（Brave・Arc 等の Chromium 系は未対応）
- Claude in Chrome 拡張機能 v1.0.36 以上が必要
- Anthropic 直接プランのみ（Bedrock・Vertex AI 経由では使えない）

#### ファンアウトパターンのコスト削減

ルールに基づいた一括変換など、ファンアウトパターン（複数ファイルへの並列 Claude 呼び出し）を安価に実行する方法。

| 方法 | 効果 |
|---|---|
| Haiku モデルを指定（`--model claude-haiku-4-5-20251001`） | 最もインパクトが大きい |
| `--allowedTools` でツールを絞る | 不要な探索・読み込みを削減 |
| プロンプトを短く具体的にする | 入力トークンを削減 |
| 少数ファイルで先にテストしてからスケール | 失敗コストを最小化 |
| `--permission-mode plan` で事前確認 | 実行前に意図しない動作を検証 |

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --model claude-haiku-4-5-20251001 \
    --allowedTools "Edit,Bash(git commit *)"
done
```

#### キッチンシンクセッション（失敗パターン）

1つのセッションに関係のないタスクを混在させてしまうパターン。コンテキストが汚染されてパフォーマンスが低下する。

**具体例:**

```
// 認証機能を実装中...
> implement OAuth login

// 途中で別のことを聞いてしまう
> how does React useEffect work?

// 元の作業に戻る
> ok back to the OAuth, add the callback handler
```

他にも:
- バグ修正中に別のファイルのバグも頼む
- リファクタリング中に技術的な質問を挟む

**なぜ問題か:** Claude はコンテキスト内の全会話を参照して回答するため、関係ない情報が混入すると以前の指示を「忘れる」・間違いが増える・コンテキスト消費が早まる。

**対策:** タスクが変わるたびに `/clear` でリセットする。「ちょっと聞くだけ」でも別セッションにするのが理想。
