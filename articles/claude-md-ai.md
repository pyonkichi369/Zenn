---
title: "CLAUDE.md完全攻略 — プロジェクト専用AIを作る実践テンプレート集"
emoji: "📝"
type: "tech"
topics: ["AI", "プログラミング", "エンジニア"]
published: true
---

Claude Codeを使って「なんか微妙な出力しか出ない」と感じたことはありませんか？

原因はほぼ100%、**CLAUDE.mdを書いていない**ことです。

前回の記事「Claude Codeで開発速度が10倍になった具体的な方法」では、Claude Codeの基本的な使い方を5つ紹介しました。今回はその続編として、**CLAUDE.mdの具体的な書き方とコピペで使えるテンプレート**を公開します。

この記事を読めば、あなたのプロジェクトに最適化されたAIアシスタントを15分で構築できます。

## CLAUDE.mdとは何か

CLAUDE.mdは、プロジェクトのルートに配置する設定ファイルです。Claude Codeはこのファイルを**毎回自動で読み込み**、プロジェクト固有のルールに従って動作します。

つまり、CLAUDE.mdに書いたルールがAIの「仕事の仕方」を決めます。

書かないとどうなるか？ Claude Codeは汎用的な推測で動きます。あなたのプロジェクトのコーディング規約も、テスト方針も、ディレクトリ構造も知らないまま作業します。

## CLAUDE.mdの基本構造

効果的なCLAUDE.mdには4つのセクションがあります。

```markdown
# CLAUDE.md

## プロジェクト概要
[何を作っているか、技術スタックは何か]

## ビルド・実行コマンド
[開発者が毎日使うコマンド一覧]

## アーキテクチャ
[ディレクトリ構造、設計パターン、レイヤー分け]

## コーディング規約
[命名規則、エラー処理、テスト方針]
```

それぞれのセクションに何を書くべきか、具体例を見ていきましょう。

## テンプレート1: Pythonバックエンド（FastAPI）

FastAPIプロジェクト向けのテンプレートです。

```markdown
# CLAUDE.md

## Quick Reference
Run: `make dev` | Test: `make test` | Lint: `make lint`
DB: `make db-migrate` | Docker: `docker compose up -d`

## Architecture
- `app/routers/` — APIエンドポイント（thin router）
- `app/services/` — ビジネスロジック
- `app/models/` — SQLAlchemy ORM モデル
- `app/core/` — 設定、認証、ミドルウェア

## Rules
- Routerはリクエスト解析→サービス呼び出し→レスポンス返却のみ
- ビジネスロジックはservices/に書く（routerに直接書かない）
- テスト必須: 新しいエンドポイントには認証テスト（401/403/404/422）を書く
- エラー処理: HTTPException（4xx）、グローバルハンドラ（5xx）
- 環境変数: os.environ.get("KEY", "") — デフォルト値にシークレットを書かない
```

このテンプレートのポイントは**ルーターを薄く保つ**ルールです。Claude Codeはこのルールを読んで、新しいエンドポイントを追加するときに自動的にサービス層を作ります。

## テンプレート2: フロントエンド（React/Next.js）

```markdown
# CLAUDE.md

## Quick Reference
Dev: `npm run dev` | Build: `npm run build` | Test: `npm test`

## Architecture
- `src/components/` — 再利用可能なUIコンポーネント
- `src/features/` — 機能単位のモジュール
- `src/hooks/` — カスタムフック
- `src/lib/` — ユーティリティ、API クライアント

## Rules
- コンポーネントは関数コンポーネント + TypeScript
- 状態管理: React Query（サーバー状態）、useState/useReducer（ローカル状態）
- スタイリング: Tailwind CSS — インラインstyleは使わない
- インポート順: React → ライブラリ → 内部モジュール → 型定義
- テスト: Testing Library + Vitest（実装ではなく振る舞いをテスト）
- アクセシビリティ: ボタンにaria-label必須、img要素にalt必須
```

## テンプレート3: Vanilla JavaScript（フレームワークなし）

フレームワークを使わないプロジェクト向けです。

```markdown
# CLAUDE.md

## Architecture
- `public/modules/` — ES6モジュール（named export必須）
- `public/styles/` — CSSファイル

## Rules
- named function exportのみ使う（default exportは禁止）
- モジュールレベルのクロージャで状態を管理（グローバル変数禁止）
- DOM要素へのアクセスはdomCacheを使う
- CSS: カスタムプロパティ（design tokens）のみ使用、ハードコードした色やフォント禁止
- ファイルは400行以内
```

## テンプレート4: データ分析・機械学習

```markdown
# CLAUDE.md

## Quick Reference
Jupyter: `jupyter lab` | Test: `pytest tests/`
Env: `conda activate ml-project`

## Architecture
- `notebooks/` — 探索的分析（番号付き: 01_, 02_...）
- `src/data/` — データ取得・前処理
- `src/models/` — モデル定義・学習
- `src/evaluation/` — 評価指標・可視化

## Rules
- notebookは探索用、本番コードはsrc/に移す
- データパスはハードコードしない（config.yamlから読む）
- 実験結果はMLflowまたはログファイルに記録
- ランダムシードは必ず固定する（再現性）
- 大きなデータファイルはgitignoreに追加
```

## やってはいけない3つのアンチパターン

### NG1: ルールが曖昧すぎる

```markdown
# ❌ 悪い例
## Rules
- きれいなコードを書いてください
- テストを書いてください
```

```markdown
# ✅ 良い例
## Rules
- 関数は50行以内。超えたら分割する
- 新しいAPIエンドポイントには認証テスト（401/403）を必ず書く
```

**具体的な数値や条件**を入れると、AIは明確に従います。

### NG2: CLAUDE.mdが長すぎる

1,000行のCLAUDE.mdは逆効果です。AIのコンテキストウィンドウを圧迫して、肝心のコードに使える容量が減ります。

理想は**50〜150行**。それ以上になったら、`.claude/rules/`ディレクトリにファイルを分割しましょう。

```
.claude/
  rules/
    development.md    # コーディング規約
    architecture.md   # 設計方針
    testing.md        # テスト方針
```

Claude Codeは`.claude/rules/`内のファイルも自動で読み込みます。

### NG3: コマンドを書かない

開発者が毎日使うコマンドをCLAUDE.mdに書かないと、AIは推測でコマンドを実行します。

```markdown
# ✅ Quick Reference（必ず書く）
Dev: `npm run dev` | Test: `npm test` | Lint: `npm run lint`
Build: `npm run build` | DB: `npx prisma migrate dev`
```

これだけで、AIが正しいコマンドを使うようになります。

## 効果測定: Before / After

私のプロジェクトでのCLAUDE.md導入前後の比較です。

| 指標 | 導入前 | 導入後 |
|------|--------|--------|
| AIの出力が1回で使える確率 | 40% | 85% |
| コードレビューでの指摘数 | 5-8件 | 1-2件 |
| テストカバレッジ | 30% | 80%+ |
| 新機能の実装時間 | 4-8時間 | 1-2時間 |

最も大きな変化は「やり直し」が激減したことです。CLAUDE.mdがあると、AIは最初からプロジェクトの規約に沿ったコードを書きます。

## まとめ

CLAUDE.mdを書くだけで、Claude Codeの出力品質は劇的に向上します。

1. **4セクション構成**: 概要、コマンド、アーキテクチャ、規約
2. **具体的なルール**: 「きれいに」ではなく「50行以内」
3. **50-150行**: 長すぎたら`.claude/rules/`に分割
4. **コマンド一覧**: Quick Referenceは必須

この記事のテンプレートをコピーして、あなたのプロジェクトに合わせてカスタマイズしてください。15分の投資で、開発効率が変わります。

---