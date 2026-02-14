---
title: "gitのちょっとした手間を解決するCLI、gutを作りました"
emoji: "🔧"
type: "tech"
topics: ["git", "cli", "ai", "typescript", "gemini"]
published: false
---

## はじめに

`git commit -m "ここに何書こう..."` で手が止まる瞬間、ありませんか？

ちゃんとしたコミットメッセージを書こうとすると意外と時間がかかります！

`feat`なのか`fix`なのか、スコープはどうするか、英語で書くか日本語で書くか...。
結局「fix bug」みたいな雑なメッセージで済ませてしまったり。

ブランチ名も同じです。issueの内容を見て、`feature/user-authentication-with-oauth2-integration`みたいな名前を考えて、タイプミスしないように気をつけながら`git checkout -b`する。地味にめんどくさい。

PRの説明文も書かないといけないし、stashに名前をつけ忘れて後で「これ何だっけ...」となることもしばしば。

これらって一つひとつは大したことないんですが、積み重なると開発のフローが止まるんですよね。

そこで、こういった「gitまわりの小さなめんどくさい」をAIで自動化するCLIツール「**gut**」を作りました。

https://github.com/gitton-dev/gut

日本語でコミットメッセージやPR説明文を生成したい場合は、以下のコマンドで設定できます：

```bash
# グローバル設定（すべてのリポジトリに適用）
$ gut config set lang ja

# 特定のリポジトリだけ日本語にしたい場合
$ gut config set lang ja --local
```

たとえば、従来の`git commit`だとこんな感じですよね。

```bash
# 変更をステージング
$ git add src/auth.ts

$ git commit -m "feat(auth): add OAuth2 login flow with Google provider"
```

gutを使うとこうなります。

```bash
# これだけ
$ gut commit

✨ Generated commit message:
feat(auth): add OAuth2 login flow with Google provider

Proceed? (Y/n) y
✅ Committed!
```

diffを解析してコミットメッセージを自動生成してくれるので、もうメッセージを考える必要がありません。

ブランチ名も同様です。従来だとissueを見ながら自分で考える必要がありました。

```bash
# issueを確認して...ブランチ名を考えて...
$ git checkout -b feature/user-authentication-with-oauth2
```

gutならissue番号を渡すだけ。

```bash
$ gut branch 123 --checkout

✨ Created and checked out: feature/123-oauth2-authentication
```

PRの説明文も、今まではテンプレートを埋めるのが手間でした。

```bash
# テンプレートを開いて、変更内容を思い出しながら書いて...
$ gh pr create --title "Add OAuth2" --body "..."
```

gutならコマンド一発です。

```bash
$ gut pr

📝 Generated PR:

Title: Add OAuth2 authentication with Google provider

Description:
──────────────────────────────────────────────────
## Summary
- OAuth2認証フローを実装
- Googleプロバイダーに対応

## Changes
- src/auth.ts: OAuth2クライアントの実装
- src/config.ts: 認証設定の追加
──────────────────────────────────────────────────

Create PR with gh CLI? (y/N) y
✓ PR created successfully!
```

## なぜgutを作ったのか

### コードを書く以外のところでAIを使いたかった

Claude CodeのようなAIアシスタントも強力ですが、単にコミットメッセージを生成したいだけなのに、ちょっと時間がかかります

gutは**git操作だけに特化**しています。コードベース全体を解析するのではなく、diffだけを見るので数秒で結果が返ります。

### サブスクリプション不要

多くのAIツールは月額課金が必要ですよね。gutは**自分のAPIキーを使う方式**なので、使った分だけの従量課金で済みます。

Gemini、OpenAI、Anthropicから好きなプロバイダーを選べます。APIキーはOSのネイティブキーチェーン（macOSならKeychain、WindowsならCredential Vault）に保存されるので、平文で保存されることはありません。

## gutでできること

### コミットメッセージの自動生成

```bash
$ gut commit

✨ Generated commit message:
feat(auth): add OAuth2 login flow with Google provider

Proceed? (Y/n)
```

変更内容を解析して、Conventional Commits形式のメッセージを自動生成します。プロジェクトに`.gut/commit.md`を置けば、独自のルールにも対応します。

### PR説明文の生成

```bash
$ gut pr

📝 Generated PR:

Title: Add user authentication feature

Description:
──────────────────────────────────────────────────
## Summary
- ユーザー認証機能を追加
- セッション管理を実装

## Changes
- 新規ファイル: src/auth.ts
- 設定追加: src/config.ts
──────────────────────────────────────────────────

Create PR with gh CLI? (y/N) y
✓ PR created successfully!
```

ブランチのコミット履歴とdiffから、PRのタイトルと説明文を生成。`.github/pull_request_template.md`があれば、そのテンプレートに沿って埋めてくれます。`--copy`オプションでクリップボードにコピーも可能です。

### コードレビュー

```bash
$ gut review

📝 Code Review

Summary: Overall good implementation with minor suggestions

Issues:
🔴 critical | src/auth.ts:42
   Password is logged to console
   → Remove console.log or use secure logging

⚠️ warning | src/api.ts:15
   Missing error handling for API call
   → Wrap in try-catch block
```

変更内容をレビューして、バグやセキュリティ上の問題、改善点を指摘します。

### 変更内容からブランチを作成

```bash
$ gut checkout

✨ Generated branch name:

  feature/add-user-authentication

Create and checkout this branch? (y/N) y
✓ Created and checked out branch: feature/add-user-authentication
```

「まずコードを書き始めてしまった！」というとき、現在の変更内容を解析してブランチ名を自動生成します。`gut branch`がissue番号から生成するのに対し、`gut checkout`はdiffから生成するので、先にコードを書き始めた場合に便利です。

### コンフリクトの自動解決

```bash
$ gut merge feature/login

🔀 Merging feature/login into main...
⚠️ Found 2 conflicted files

Resolving conflicts with AI...
✅ src/auth.ts - resolved (combined)
✅ src/config.ts - resolved (theirs)

All conflicts resolved!
```

マージコンフリクトが発生した際、AIが両方の変更の意図を理解して解決案を提示します。

### 変更の説明

```bash
$ gut explain abc123

📖 Commit Explanation

Summary: Add rate limiting to API endpoints

Purpose: Prevent abuse and ensure fair usage of API resources

Changes:
- src/middleware/rateLimit.ts: New rate limiting middleware
- src/routes/api.ts: Apply rate limiting to all endpoints
```

コミット、PR、ファイルの内容を「なぜその変更が必要だったのか」まで含めて説明してくれます。

### コミットの検索

```bash
$ gut find "ログイン機能"

🔍 Found 3 matching commits:

1. abc1234 - feat(auth): add login form component
   Reason: Implements the login UI functionality

2. def5678 - fix(auth): handle login timeout error
   Reason: Fixes an issue with the login feature
```

「あの機能いつ実装したっけ？」という曖昧な検索をAIが解釈して、関連するコミットを見つけます。

## 技術的な実装

### アーキテクチャ

gutは以下の技術スタックで構築されています：

- **TypeScript**: 型安全な実装
- **Commander.js**: CLIフレームワーク
- **Vercel AI SDK**: マルチプロバイダー対応
- **Zod**: スキーマ定義と構造化出力
- **simple-git**: Git操作
- **keytar**: セキュアな認証情報管理

### Vercel AI SDKによるマルチプロバイダー対応

複数のAIプロバイダーに対応するため、Vercel AI SDKを採用しました。これにより、プロバイダーごとの差異を吸収しています：

```typescript
import { generateText, generateObject } from 'ai'
import { createGoogleGenerativeAI } from '@ai-sdk/google'
import { createOpenAI } from '@ai-sdk/openai'
import { createAnthropic } from '@ai-sdk/anthropic'

const DEFAULT_MODELS: Record<Provider, string> = {
  gemini: 'gemini-2.0-flash',
  openai: 'gpt-4o-mini',
  anthropic: 'claude-sonnet-4-20250514'
}

async function getModel(options: AIOptions) {
  const apiKey = await getApiKey(options.provider)

  switch (options.provider) {
    case 'gemini': {
      const google = createGoogleGenerativeAI({ apiKey })
      return google(modelName)
    }
    case 'openai': {
      const openai = createOpenAI({ apiKey })
      return openai(modelName)
    }
    case 'anthropic': {
      const anthropic = createAnthropic({ apiKey })
      return anthropic(modelName)
    }
  }
}
```

ユーザーは`--provider`オプションで好きなプロバイダーを選べます：

```bash
$ gut commit --provider openai
```

### Zodによる構造化出力

AIからの出力を型安全に扱うため、Zodスキーマを定義しています：

```typescript
const CodeReviewSchema = z.object({
  summary: z.string().describe('Brief overall assessment'),
  issues: z.array(
    z.object({
      severity: z.enum(['critical', 'warning', 'suggestion']),
      file: z.string(),
      line: z.number().optional(),
      message: z.string(),
      suggestion: z.string().optional()
    })
  ),
  positives: z.array(z.string()).describe('Good practices observed')
})
```

Vercel AI SDKの`generateObject`を使うことで、AIの出力を直接このスキーマに沿った型付きオブジェクトとして取得できます：

```typescript
const result = await generateObject({
  model,
  schema: CodeReviewSchema,
  prompt: `You are an expert code reviewer...`
})

// result.object は CodeReview 型として型安全に使える
```

### セキュアな認証情報管理

APIキーの管理には**keytar**を使用しています。keytarはOSのネイティブ認証情報ストレージ（macOSのKeychain、WindowsのCredential Vault、LinuxのSecret Service API）を利用するため、平文でキーを保存することなくセキュアに管理できます：

```typescript
import keytar from 'keytar'

const SERVICE_NAME = 'gut-cli'

export async function saveApiKey(provider: Provider, key: string): Promise<void> {
  await keytar.setPassword(SERVICE_NAME, provider, key)
}

export async function getApiKey(provider: Provider): Promise<string | null> {
  return await keytar.getPassword(SERVICE_NAME, provider)
}
```

### プロジェクト固有の設定

gutはプロジェクトごとにカスタマイズ可能です。`.gut/`ディレクトリにテンプレートファイルを置くことで、AIの出力をプロジェクトのルールに合わせられます：

| ファイル | 用途 |
|---------|------|
| `.gut/commit.md` | コミットメッセージのルール |
| `.gut/pr.md` | PRの説明文テンプレート |
| `.gut/branch.md` | ブランチ命名規則 |
| `.gut/merge.md` | コンフリクト解決戦略 |
| `.gut/checkout.md` | diffからのブランチ名生成ルール |

例えば、コミットメッセージに日本語を使いたい場合：

```markdown
<!-- .gut/commit.md -->
# コミットメッセージ規約

- 日本語で記述する
- 形式: [種別] 説明
- 種別: 機能追加、バグ修正、リファクタリング、ドキュメント
- 例: [機能追加] ログイン機能を実装
```

また、グローバルに日本語出力を設定することも可能です：

```bash
$ gut config set lang ja
```

## インストールと使い方

```bash
# インストール
npm install -g gut-cli

# APIキーの設定（Geminiの場合）
gut auth login --provider gemini

# 使ってみる
gut commit
```

### プロジェクトにテンプレートを追加

`gut init`コマンドで、プロジェクトに`.gut/`フォルダとテンプレートファイルを自動生成できます：

```bash
$ gut init

Created /path/to/project/.gut
  Copied: commit.md
  Copied: pr.md
  Copied: branch.md
  ...

✓ 12 template(s) initialized in .gut/
```

言語設定が日本語（`gut config set lang ja`）の場合、テンプレートも自動的に日本語に翻訳されます。生成されたテンプレートはプロジェクトに合わせて自由にカスタマイズできます。

## まとめ

gutは「gitワークフローの小さな面倒をAIで解消する」というコンセプトで作りました。既存のAIコーディングツールとは異なり、git操作に特化することで高速かつ実用的なツールになっています。

サブスクリプション不要で、自分のAPIキーさえあれば使えます。ぜひ試してみてください！

https://github.com/gitton-dev/gut

