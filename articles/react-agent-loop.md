---
title: "ReAct Agent Loop — 最小実装の概念コード"
emoji: "📝"
type: "tech"
topics: ["aiエージェント", "ブロックチェーン", "web4", "llm"]
published: true
---

class SimpleReActAgent:
    """
    Reason → Act → Observe のループを回す最小エージェント。
    実際のプロダクションでは LangGraph や Claude Code の
    ツールループがこのパターンを内部的に実装している。
    """

    def __init__(self, llm, tools: dict[str, callable]):
        self.llm = llm          # LLM API クライアント
        self.tools = tools      # {"tool_name": callable}
        self.memory: list[dict] = []  # 会話履歴

    def run(self, goal: str, max_steps: int = 10) -> str:
        self.memory.append({"role": "user", "content": goal})

        for step in range(max_steps):
            # 1. Reason — LLMに現状を分析させ、次の行動を決定
            response = self.llm.chat(
                messages=self.memory,
                tools=list(self.tools.keys()),
                system="You are a helpful agent. Think step by step."
            )

            # 2. 完了判定
            if response.is_final_answer:
                return response.content

            # 3. Act — ツールを実行
            tool_name = response.tool_call.name
            tool_args = response.tool_call.arguments

            if tool_name not in self.tools:
                observation = f"Error: Unknown tool '{tool_name}'"
            else:
                try:
                    observation = self.tools[tool_name](**tool_args)
                except Exception as e:
                    observation = f"Error: {e}"

            # 4. Observe — 結果をメモリに追加し、次のループへ
            self.memory.append({
                "role": "assistant",
                "content": response.reasoning,
                "tool_call": {"name": tool_name, "args": tool_args}
            })
            self.memory.append({
                "role": "tool",
                "content": str(observation)
            })

        return "Max steps reached without resolution."
```

このコードは概念的なものですが、**すべてのエージェントフレームワークの根幹にあるループ構造**を示しています。LangGraph、CrewAI、OpenAI Agents SDK、Claude Code — いずれも内部的にはこの Reason → Act → Observe のサイクルを回しています。

### MCP と A2A — エージェント時代のAPIスタンダード

2024〜2025年にかけて、エージェント間の通信プロトコルが標準化されつつあります。

**MCP（Model Context Protocol）** — Anthropicが提唱
- エージェントが外部ツール（API、DB、ファイルシステム等）にアクセスするための統一プロトコル
- JSON-RPC 2.0ベース。ツールの発見・スキーマ取得・実行を標準化
- 従来のバラバラなFunction Calling実装を統一する役割

**A2A（Agent-to-Agent Protocol）** — Googleが提唱
- エージェント同士が直接通信するためのプロトコル
- Agent Card（能力の宣言）→ タスク依頼 → 結果返却の流れを標準化
- HTTPベースで、既存のWebインフラ上で動作

:::message
**MCPとA2Aの関係**
MCP = エージェントが「道具」を使うためのプロトコル（人とツールの接続）
A2A = エージェント同士が「会話」するためのプロトコル（エージェント間の接続）
この2つは競合ではなく補完関係にあります。
:::

---

## 3. AIエージェント — 実装パターン

### Single Agent vs Multi-Agent

エージェントシステムの設計は、大きく2つのアプローチに分かれます。

**Single Agent（単一エージェント）**
- 1つのLLMインスタンスがすべてのツールを持ち、タスクを完結
- 適用：定型的なタスク、明確なゴール、ツール数10個以下
- 例：コードレビューエージェント、カスタマーサポートBot

**Multi-Agent System（マルチエージェント）**
- 複数の専門エージェントが役割分担し、協調してタスクを遂行
- 適用：複雑な意思決定、異なる専門性が必要、組織的な業務フロー
- 例：ソフトウェア開発チーム、投資判断パイプライン

```
┌─ Single Agent ─────────────────┐
│  [LLM] → Tool A → Tool B → ✓  │
└────────────────────────────────┘

┌─ Multi-Agent System ───────────────────────────┐
│  [CEO Agent] ─── 指示 ──→ [CTO Agent]         │
│       │                        │               │
│       │                   [Impl Agent]         │
│       │                        │               │
│       ↓                        ↓               │
│  [CFO Agent] ←── 結果 ── [QA Agent]           │
└────────────────────────────────────────────────┘
```

### エージェントフレームワーク比較（2026年3月時点）

| フレームワーク | 提供元 | 特徴 | 適用場面 |
|--------------|--------|------|---------|
| **LangGraph** | LangChain | グラフベースのワークフロー、状態管理が強い | 複雑な条件分岐、ループ、人間介入 |
| **CrewAI** | CrewAI | 役割ベース、チーム構成が直感的 | マルチエージェントの素早いプロトタイプ |
| **AutoGen** | Microsoft | 会話ベース、研究用途に強い | エージェント間の自由な対話実験 |
| **OpenAI Agents SDK** | OpenAI | ハンドオフパターン、Guardrails内蔵 | OpenAIエコシステム内のプロダクション |
| **Claude Code** | Anthropic | CLIベース、コード生成特化、ツールループ | 開発者向け自律コーディング |

### 実例：AEGIS — 61エージェント組織

筆者が開発しているAEGIS（Autonomous Enterprise Governance & Intelligence System）は、61のAIエージェントが企業組織のように構成されたマルチエージェントシステムです。

:::details AEGISのアーキテクチャ概要
```
Executive Layer:  CEO, CTO, CFO, CRO, CMO, CPO, CDO（経営判断）
Review Layer:     Technical, Security, Business, Ethics, Risk（品質管理）
Specialist Layer: QA, Researcher, DataScientist, PromptEngineer（専門家）
Implementation:   Primary, Secondary, DataPipeline（実装）
Creative:         Screenwriter, FilmDirector, VideoEditor（映像制作）
Marketing:        GrowthHacker, SocialMediaManager, ContentStrategist（マーケ）
Infrastructure:   SRE, SecurityEngineer, MLOpsEngineer（インフラ）
Legal:            LegalCounsel, ComplianceOfficer, DataPrivacyOfficer（法務）
```

全エージェントにガバナンスルール（Rule Engine）が適用され、リスクレベルに応じて自動承認 → 通知 → 人間確認 → 緊急停止の4段階でエスカレーションされます。

重要なのは、**これはコンセプトではなく、実際に動作しているシステム**だということです。98,000行のEngineコード、20,000行のAPI、12,000行のUIで構成されています。
:::

### Agent Wallet パターン — ERC-4337 Account Abstraction

AIエージェントが経済活動を行うには**ウォレット**が必要です。しかし、通常のEOA（Externally Owned Account）は秘密鍵管理が前提で、エージェントには不向きです。

**ERC-4337（Account Abstraction）** がこの問題を解決します。

```
従来のEOA:
  秘密鍵 → 署名 → トランザクション送信
  問題: 秘密鍵の管理、ガス代の支払い、単純な署名のみ

ERC-4337 Smart Account:
  UserOperation → Bundler → EntryPoint Contract → Smart Account
  利点:
    - カスタム署名検証（マルチシグ、時間制限、金額制限）
    - ガス代の第三者負担（Paymaster）
    - バッチ処理（複数操作を1トランザクションに）
```

エージェント × ERC-4337の組み合わせが強力な理由：

1. **支出制限**: スマートアカウントに「1日あたり最大0.1 ETH」のような制約をコントラクトレベルで実装可能
2. **マルチシグ承認**: 高額取引は人間の署名を追加で要求
3. **Paymaster**: エージェントがガス代を持たなくても、サービス提供者が代払い
4. **セッションキー**: 一時的な権限委譲により、エージェントに限定的な操作権限を付与

:::message alert
**セキュリティ上の注意**
AIエージェントにウォレットの秘密鍵を直接渡すのは非常に危険です。ERC-4337のセッションキーや支出制限を必ず実装し、エージェントの権限を最小限に制限してください。秘密鍵は常にHSM（Hardware Security Module）やSecret Managerで管理すべきです。
:::

---

## 4. 交差する理由 — 技術的必然性

### なぜHTTP API + Stripeでは不十分なのか

「エージェント間の決済なら、既存のStripe APIでいいのでは？」

これは合理的な疑問ですが、以下の3つの理由で不十分です。

**1. アイデンティティの問題**
Stripeのアカウントは人間（または法人）に紐づきます。AIエージェントは法人格を持たず、KYC（本人確認）を通過できません。ブロックチェーンのアドレスは暗号鍵ペアから生成されるため、法人格は不要です。

**2. マイクロペイメントの問題**
Stripeの最低決済額は約$0.50で、手数料は2.9% + $0.30です。エージェントが1回のAPI呼び出しに$0.001を支払うユースケースでは、手数料が元本の300倍になります。L2ブロックチェーンなら$0.001以下のトランザクションが$0.001以下の手数料で可能です。

**3. プログラマビリティの問題**
Stripe APIは人間が設計した決済フローを実行します。スマートコントラクトは**決済ロジック自体がプログラマブル**です。「AIエージェントAがタスクを完了し、AIエージェントBが品質を検証し、AIエージェントCが承認した場合にのみ支払う」といった複雑な条件を、コントラクトとして直接表現できます。

### プロトコルスタック — Web 4.0のアーキテクチャ

これら3つの技術が交差する地点で、新しいプロトコルスタックが形成されつつあります。

```
┌─────────────────────────────────────────────────┐
│  Application Layer                               │
│  エージェントアプリケーション（AEGIS等）            │
├─────────────────────────────────────────────────┤
│  Communication Layer                             │
│  A2A Protocol（エージェント間通信）                │
├─────────────────────────────────────────────────┤
│  Tool Layer                                      │
│  MCP（外部ツール・API接続）                       │
├─────────────────────────────────────────────────┤
│  Identity & Wallet Layer                         │
│  ERC-4337（アカウント抽象化 + セッションキー）     │
├─────────────────────────────────────────────────┤
│  Settlement Layer                                │
│  L2 Rollup（Optimistic / ZK）                    │
├─────────────────────────────────────────────────┤
│  Consensus Layer                                 │
│  Ethereum L1（最終的なセキュリティ保証）           │
└─────────────────────────────────────────────────┘
```

このスタックの各層は、従来のWeb（TCP/IP → HTTP → REST → Application）と構造的に対応しています。違いは、**人間ではなくエージェントが主体的に利用する**ことを前提に設計されている点です。

### 市場規模 — 数字で見る収束

この3技術の交差は、市場データにも明確に現れています。

| 領域 | 2024年 | 2030年予測 | CAGR |
|------|--------|-----------|------|
| AIエージェント市場 | 約8,000億円 | 約7.39兆円 | ~44% |
| ブロックチェーン市場 | 約2.7兆円 | 約12兆円 | ~28% |
| DeFi TVL | 約15兆円 | — | — |

注目すべきは、AIエージェント市場の**CAGR 44%**という異常な成長率です。これは単純なチャットBot市場ではなく、自律的に意思決定・実行するエージェントへの投資が急増していることを反映しています。

そしてこれらのエージェントが経済活動を始めた瞬間、**決済・契約・信頼のインフラ**としてブロックチェーンが不可欠になります。「AIが賢くなる」と「お金が動く」の交差点に、ブロックチェーンが位置しているのです。

---

## 5. 連載予告 — 「Web 4.0への道」

本連載では、エンジニアの視点から「AIエージェント × ブロックチェーン」の世界を実装レベルで掘り下げていきます。

| # | タイトル（予定） | 内容 |
|---|----------------|------|
| 0 | **本記事** | 3技術の俯瞰（ブロックチェーン × AI × エージェント） |
| 1 | なぜAIエージェントに暗号資産ウォレットが必要か | EOA vs ERC-4337、セッションキー実装 |
| 2 | エージェント間マイクロペイメントの設計 | L2上のPayment Channel、Paymaster |
| 3 | スマートコントラクトによるエージェントガバナンス | On-chain Rule Engine、マルチシグ |
| 4 | MCP × A2A × DeFi — 統合アーキテクチャ | プロトコルスタックの実装 |
| 5 | Web 4.0の全体像と残された課題 | 規制、セキュリティ、スケーラビリティ |

各回とも**コード例付き**で、「読んだら手を動かせる」内容を目指します。

:::message
**次回予告**
第1回「なぜAIエージェントに暗号資産ウォレットが必要か」では、ERC-4337 Account Abstractionの仕組みと、エージェント向けウォレットの実装パターンを解説します。
:::

---

## まとめ

- **ブロックチェーン**は「信頼不要の合意プロトコル」。L2 Rollupにより、エージェント間マイクロペイメントが現実的なコストで実行可能に
- **AI / LLM**は「テキスト生成」から「自律的な行動ループ（ReAct）」へ進化。MCP・A2Aにより通信が標準化
- **AIエージェント**はSingle AgentからMulti-Agent Systemへ。ERC-4337により、エージェント固有のウォレットが実現可能に
- 3技術が交差する理由は**技術的必然性** — エージェント経済にはアイデンティティ、マイクロペイメント、プログラマブルな決済が不可欠
- 市場は2030年に7兆円超へ。この波に乗るか見送るかは、今のうちの技術理解にかかっている

---

## 免責事項

本記事は技術的な解説を目的としており、特定の暗号資産やプロジェクトへの投資を推奨するものではありません。ブロックチェーン技術やAIエージェントの実装にはリスクが伴います。特にスマートコントラクトのセキュリティ、秘密鍵管理、規制対応については、専門家への相談を推奨します。

コード例は概念説明のための擬似コードであり、プロダクション環境でそのまま使用しないでください。

**知識日付**: 2026-03-19
**筆者**: AIエージェントシステム（AEGIS）の開発者。61エージェント組織を運用中。