---
title: "MCP（Model Context Protocol）完全解説 — Claude CodeやCursorをツールに繋ぐ新標準"
emoji: "🔌"
type: "tech"
topics: ["claude", "mcp", "ai", "langchain", "claudecode"]
published: true
---

2024年11月にAnthropicが公開した**MCP（Model Context Protocol）**は、AI開発のツール統合問題を一気に解決する標準プロトコルです。Claude Code、Cursor、Windsurfなどが次々と対応し、2026年現在は事実上の業界標準になっています。

この記事では「MCPとは何か」から「自分でMCPサーバーを作る方法」まで、実装コード付きで解説します。

---

## MCPが解決する問題

MCPが登場する前、AIアプリケーションは外部ツールとの連携を**個別に実装**していました。

```
MCPなし:
Claude App → GitHub専用の実装
Claude App → Slack専用の実装
Claude App → DB専用の実装
（N個のツール = N個の独自実装が必要）

MCPあり:
Claude App → MCPプロトコル → GitHub MCPサーバー
                            → Slack MCPサーバー
                            → DB MCPサーバー
（1つの標準インターフェースで全ツールに接続）
```

**M×N問題の解決**: Mモデル × Nツール = M×N個の実装 → MCP導入後はM+N個で済む。

---

## MCPの構造

```
MCP Host（Claude Code、Cursorなど）
    ↓ JSON-RPC 2.0
MCP Client（接続管理）
    ↓
MCP Server（ツール提供）
    ↓
外部システム（ファイル、GitHub、DB...）
```

MCPサーバーが提供できる3種類のプリミティブ：

| プリミティブ | 説明 | 例 |
|------------|------|-----|
| **Tools** | AIが呼び出せる関数 | `read_file`, `search_github` |
| **Resources** | AIが読めるデータ | ファイル内容、DBレコード |
| **Prompts** | 再利用可能なプロンプトテンプレート | ワークフロースターター |

---

## Claude CodeにMCPを設定する

```json
// ~/.claude/mcp_settings.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..." }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

この設定後にClaude Codeを再起動すると、「GitHubのこのIssueを見て実装して」「このテーブルのスキーマを確認して」といった指示が直接使えるようになります。

---

## 公式MCPサーバー一覧（2026年）

| サーバー | 提供するTools |
|---------|-------------|
| filesystem | ローカルファイルの読み書き |
| github | リポジトリ、Issue、PR、コード検索 |
| postgres | SQLクエリ、スキーマ確認 |
| sqlite | ローカルDB操作 |
| slack | チャンネル投稿、ファイル送信 |
| google-drive | ファイルアクセス、検索 |
| fetch | 外部URLへのHTTPリクエスト |
| brave-search | Web検索 |
| memory | セッション横断の永続メモリ |

コミュニティ製：Supabase、Linear、Notion、Jira、Figmaなど500+。

---

## 自分でMCPサーバーを作る（TypeScript）

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "my-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ツール定義
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "get_weather",
    description: "都市名から現在の天気を取得する",
    inputSchema: {
      type: "object",
      properties: { city: { type: "string", description: "都市名（例: Tokyo）" } },
      required: ["city"]
    }
  }]
}));

// ツール実行
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "get_weather") {
    const { city } = request.params.arguments;
    return { content: [{ type: "text", text: `${city}の天気: 22°C、晴れ` }] };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

**50行以下でMCPサーバーが完成します。**

---

## Python版

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("my-server")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="search_docs",
            description="社内ドキュメントを検索する",
            inputSchema={
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "search_docs":
        results = search_internal_docs(arguments["query"])
        return [types.TextContent(type="text", text=str(results))]

async def main():
    async with stdio_server() as streams:
        await server.run(*streams, server.create_initialization_options())
```

---

## MCPと従来のFunction Calling・LangChainツールの違い

| アプローチ | 移植性 | セットアップ | エコシステム |
|---------|--------|------------|------------|
| **MCP** | あらゆるMCP対応ホストで使用可能 | npm installで数分 | 急成長中（500+サーバー） |
| Function Calling | モデル固有 | アプリごとに実装 | モデル依存 |
| LangChain Tools | Python限定 | Pythonセットアップ必須 | 大規模だがPython専用 |

**MCPの最大メリット**: 1回サーバーを書けば、Claude Code・Cursor・Windsurf・Zedすべてで使い回せる。

---

## まとめ

- MCPはAIツール統合のM×N問題を解決するオープンプロトコル
- Claude Code（ネイティブ）、Cursor、Windsurfが対応済み
- TypeScript/Python両方でサーバーを50行以下で実装可能
- 公式サーバー + コミュニティ製で500+のツールが即利用可能

MCPの詳細仕様と実装例: [ai-wikis — MCP完全ガイド](https://github.com/pyonkichi369/ai-wikis/blob/main/concepts/mcp.md)

*Claude APIを使ったMCPサーバー構築: [claude.ai/code](https://claude.ai/referral/gvWKlhQXPg)*
