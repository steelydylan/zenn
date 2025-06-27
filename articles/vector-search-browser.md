---
title: "マイグレーションもベクトル検索もブラウザだけで完結！PGlite凄すぎる..."
emoji: "📔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pglite","drizzleorm","openai","react","typescript"]
published: true
---

## はじめに

最近、PGLite（ブラウザで動くPostgreSQL互換DB）をつかって、サーバー側にデータをもたないプライベート重視のChatGPTライクなサービスを作ってます！


https://app.aidy.jp/


**特に最近、Aidyでは、「書きたい人」をサポートするAIライティングに力を入れていて、**\
構成の提案から文章の下書き、図の挿入まで、執筆をサポートしています。

Mermaid図、マインドマップ、インフラ構成図の生成や、マークダウンエディタとの連携で、記事作成がよりリッチになります！

https://youtu.be/2KO7GUi0meg

この動画を作った時はAll in One AI アプリという立ち位置で**Better ChatGPT**のようなサービスを目指していたのですが、ManusやGensparkのような競合が増えてきたので方向性を変えて技術記事やこだわりのあるライターに向けた**ライティング特化のサービス**に切り替え中です！

完全自動じゃなくて「ちょっとAIにライティングを手伝ってほしい」派にぴったりのサービスを目指しています。

今回このサービスのチャット検索などにサーバーレスで動くベクトル検索システムを入れようと思い実装方針を考えました！

結果まとまったので、実装の流れやハマりどころ、気づいたことなんかをまとめてみます。

「サーバーいらずで、フロントエンドだけでベクトル検索できたら最高じゃない？」みたいなノリで始めたんですが、意外と実用的なレベルまで持っていけたので、同じようなことやってみたい人の参考になれば嬉しいです。

## 技術スタックざっくり紹介


今回使った主な技術はこんな感じです。

* PGLite（PostgreSQL互換DB、IndexedDBに永続化）

* OpenAI API（text-embedding-ada-002で埋め込み生成）

* Drizzle ORM（型安全なDB操作）

* React + TypeScript（UIまわり）

* Vite（ビルドツール）


PGLiteはブラウザーで動くPostgreSQLで機能をかなり再現してくれてます。そこにpgvector相当の拡張も入れられるので、ベクトル検索もできちゃいました

## データベーススキーマとベクトル型

まずはDBスキーマ。Drizzle ORMで書くとこんな感じでいつものようにDBスキーマを定義します！

```typescript
import { pgTable, serial, text, vector, timestamp } from 'drizzle-orm/pg-core';

export const documents = pgTable('documents', {
  id: serial('id').primaryKey(),
  content: text('content').notNull(),
  embedding: vector('embedding', { dimensions: 1536 }),
  createdAt: timestamp('created_at').defaultNow(),
});
```

ポイントは`vector`型。OpenAIのtext-embedding-ada-002が1536次元なので、そこに合わせてます。

これでベクトルをそのままDBに突っ込めるのが地味に嬉しいポイントです！

## PGLiteの初期化とVector拡張


pgliteはPostgreSQLのWASM実装でフロントエンドでも動作します！

僕はそのpgliteを便利に使うためのelectric-sqlというライブラリを使っています！


https://electric-sql.com/


このライブラリはORMの `drizzle-orm` と組み合わせるのがとても楽で採用しています！


### pg-vectorの活用


PostgresSQLでベクトル検索する場合は `pg-vector`互換のある `@electric-sql/pglite/vector` を使います！

やり方はこんな感じ。

フロントエンドでもpgvectorがちゃんと動作します！


```typescript
import { PGlite } from '@electric-sql/pglite';
import { vector } from '@electric-sql/pglite/vector';
import { drizzle } from 'drizzle-orm/pglite';

export const client = new PGlite('idb://test', {
  extensions: { vector }
});

export const db = drizzle(client);
```

これでIndexedDBにデータが永続化されるし、pgvector相当の機能も使えます。

ブラウザでここまでできる時代になったんだなぁとしみじみ。

## フロントエンドでのマイグレーション

ここがちょっと面白いポイント。普通はサーバーでDBマイグレーションやると思うんですが、今回は全部フロントエンドで完結させたいので、ビルド時にマイグレーションファイルをJSON化してバンドル→ブラウザで実行、という流れにしました。


これは事前に、ビルドしてマイグレーション用のjsonファイルを生成しておきます！

```typescript
// scripts/compile-migrations.mts
import { readMigrationFiles } from 'drizzle-orm/migrator';
import { writeFile } from 'node:fs/promises';

const migrations = readMigrationFiles({ migrationsFolder: './drizzle' });
await writeFile('./src/db/migrations.json', JSON.stringify(migrations));
console.log('migrations.json generated');
```

で、実際にブラウザでマイグレーションを走らせるのがこちら。

```typescript
import migrations from './migrations.json';
import { db, client } from './database'

export async function runMigrate() {
  // extensionを有効化
  await client.exec("CREATE EXTENSION IF NOT EXISTS vector;");
  // @ts-expect-error drizzleのdialectから直接呼べます
  await db.dialect.migrate(
    migrations,
    db.session,
    { migrationsTable: 'drizzle_migrations' },
  );
}
```

このやり方、最初は「本当に動くの？」と半信半疑でしたが、意外とすんなり動いて感動しました笑

`db.dialect.migrate` というメソッドは型としてはエラーになるのですがdbから生えているので使いましょう！

## OpenAI Embeddingsの統合

VercelのAI SDKを使うと、OpenAIの埋め込みAPIもかなりシンプルに扱えます。

```typescript
import { embed } from "ai";
import { createOpenAI } from "@ai-sdk/openai";

const openai = createOpenAI({
  apiKey: import.meta.env.VITE_OPENAI_API_KEY,
});

export async function generateEmbedding(value: string) {
  const { embedding } = await embed({
    model: openai.embedding("text-embedding-ada-002"),
    value,
  });
  return embedding;
}
```

APIキーはViteの`VITE_`プレフィックスで管理して、クライアントサイドから呼び出せるようにしています。

::: message alert
ここで注意したいのは、サンプルではOpenAIのAPIをフロントエンドから直接呼び出していますが、実際の運用ではAPIキーがクライアント側に露出してしまうため、セキュリティ上おすすめできません。APIキーは必ずサーバー側で管理し、フロントエンドからは直接呼ばず、プロキシサーバー経由でリクエストを中継する形が安全です。
:::


Aidyでもこの方式を採用しており、APIキーはバックエンドで秘匿しつつ、フロントエンドからはプロキシ経由で埋め込み生成を行っています。

##### ベクトル検索の実装


DrizzleのSQLビルダーを使って、コサイン類似度で検索できるようにしています。ざっくりこんな感じ。

```typescript
export class VectorSearchService {
  async addDocument(content: string): Promise<void> {
    const embedding = await generateEmbedding(content);
    await db.insert(documents).values({
      content,
      embedding,
    });
  }

  async searchSimilar(query: string, limit: number = 5) {
    const queryEmbedding = await generateEmbedding(query);
    const similarity = sql<number>`1 - (${cosineDistance(
      documents.embedding,
      queryEmbedding
    )})`;

    const result = await db.select({
      id: documents.id,
      content: documents.content,
      createdAt: documents.createdAt,
      similarity: similarity,
    }).from(documents)
      .where(sql`${similarity} > 0.7`)
      .orderBy(sql`${similarity} DESC`)
      .limit(limit);

    return result;
  }
}
```

類似度の閾値（ここでは0.7）を超えたものだけ返すようにしています。コサイン類似度の計算も全部ブラウザ内で完結しています


## コードサンプル

UIはTailwindCSSでサクッと作りました。コードはここで公開しています！


https://github.com/steelydylan/browseronly-vector-search


##### セットアップ方法


1. 依存パッケージをインストール

```bash
npm install
```

1. `.env`ファイルにOpenAIのAPIキーを設定

```
VITE_OPENAI_API_KEY=sk-your-api-key-here
```

1. 開発サーバーを起動

```bash
npm run db:generate
npm run dev
```

## 実際に使ってみて感じたメリット


たとえば、以下のようなシーンで特に威力を発揮するなと感じました


* &#x20;オフライン環境でのドキュメント検索やナレッジ管理（ネット接続不要）&#x20;
* 個人用のメモアプリや日記アプリでの類似ノート検索&#x20;
* ブラウザ拡張やPWAなど、サーバーを持たない軽量ツールの全文検索
* ユーザーごとに独立したデータ管理が必要な場合（プライバシー重視）
* プロトタイピングなど、サーバー構築コストをかけたくない場面 &#x20;


特に、ユーザーのローカル環境だけで完結するため、プライバシーやコスト面でも大きなメリットがあります。

## パフォーマンスや制約について


ただし、全部ブラウザでやるので、データ量が多くなるとパフォーマンスは落ちます。数千件くらいまでが現実的かなと思います。あと、IndexedDBの容量制限や、OpenAI APIのレート制限も要注意です。


## まとめ

PGLiteとOpenAI Embeddingsを組み合わせることで、サーバーなしでもかなり本格的なベクトル検索システムが作れることが分かりました。プロトタイピングや小規模ツール、オフライン重視のアプリなんかには特におすすめです。

従来はサーバーサイドでしかできなかったことが、フロントエンドだけで実現できる時代になったのは本当にすごいなと思います。今後もこの手の技術はどんどん進化していくと思うので、引き続きウォッチしていきたいです！

AI時代の一つの選択肢としてプライベート重視のベクトル検索システムを検討してみるのも面白いかもしれません。

## 参考リンク

mizchiさんのこちらの記事からインスピレーションをウケました！
Denoでできるならブラウザでもできるだろうという発想で試してみました！

https://zenn.dev/mizchi/articles/pglite-vector-search

