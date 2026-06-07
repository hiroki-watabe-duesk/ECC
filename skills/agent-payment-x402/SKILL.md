---
name: agent-payment-x402
description: タスクごとの予算、支出制御、ノンカストディアルウォレットを使って AI エージェントに x402 決済実行機能を追加します。agentwallet-sdk を通じた Base と、OKX Payments / OKX Agent Payments Protocol を通じた X Layer をサポートします。
origin: community
---

# エージェント決済実行（x402）

ポリシーゲートされた決済と組み込みの支出制御で AI エージェントを有効化する。x402 HTTP 決済プロトコルと MCP ツールを使用して、カストディアルリスクなしにエージェントが外部サービス、API、または他のエージェントへの支払いを行えるようにする。

## 使用するタイミング

エージェントが API 呼び出しの支払い、サービスの購入、他のエージェントとの決済、タスクごとの支出制限の適用、またはノンカストディアルウォレットの管理が必要な場合に使用する。`cost-aware-llm-pipeline` および `security-review` スキルと自然に組み合わせられる。

## デシジョンツリー

エージェントが有料 API へのアクセスを購入するか、または他者に API を有料で提供するかに基づいてインテグレーションパスを選択する:

| ニーズ | 推奨パス |
|------|------------------|
| エージェントが Base または他の agentwallet 対応チェーン上の 402 ゲートされた API に支払う | 厳格な支出ポリシーを持つ MCP 決済サーバーとして `agentwallet-sdk` を使用する |
| エージェントが X Layer 上の 402 ゲートされた API に支払う | `okx/onchainos-skills` の OKX Agent Payments Protocol を使用する。`okx-x402-payment` は非推奨のレガシーエイリアス |
| TypeScript API がエージェントに課金する | Express、Hono、Fastify、または Next.js 向けの OKX Payments TypeScript セラー SDK ドキュメントを使用する |
| Go API がエージェントに課金する | Gin、Echo、または `net/http` 向けの OKX Payments Go セラー SDK ドキュメントを使用する |
| Rust API がエージェントに課金する | Axum 向けの OKX Payments Rust セラー SDK ドキュメントを使用する |
| Java API がエージェントに課金する | Spring Boot 2/3、Java EE、または Jakarta 向けの OKX Payments Java セラー SDK ドキュメントを使用する |
| Python API がエージェントに課金する | 実装前に現在の OKX Payments リポジトリを確認する。Python セラーガイドが利用できない可能性がある |

## サポートされているネットワーク

- `agentwallet-sdk`: 本番使用前にパッケージドキュメントで現在のネットワークカバレッジを確認する。Base Sepolia が最も安全な開発デフォルト。Base メインネットが元のスキルで示された本番パス。
- OKX Payments / X Layer: 現在のセラードキュメントは X Layer（`eip155:196`）と USDT0 決済を対象としている。決済パッケージとファシリテーターの動作は急速に変化する可能性があるため、本番コードを生成する前に現在の SDK ドキュメントを取得する。

## 仕組み

### x402 プロトコル
x402 は HTTP 402（支払い必要）をマシン交渉可能なフローに拡張する。サーバーが `402` を返すと、エージェントの決済ツールは価格を交渉し、予算を確認し、トランザクションに署名し、オーケストレーターが設定したポリシーと確認境界内でのみ再試行する。

### 支出制御
全ての決済ツール呼び出しは `SpendingPolicy` を適用する:
- **タスクごとの予算** — 単一のエージェントアクションの最大支出
- **セッションごとの予算** — セッション全体の累積制限
- **許可リストに登録された受取人** — エージェントが支払える住所/サービスを制限する
- **レート制限** — 分/時間あたりの最大トランザクション数

### ノンカストディアルウォレット
エージェントは ERC-4337 スマートアカウントを介して独自のキーを保持する。オーケストレーターは委任前にポリシーを設定し、エージェントは範囲内でのみ支出できる。プールされた資金なし、カストディアルリスクなし。

## MCP インテグレーション

決済レイヤーは任意の Claude Code またはエージェントハーネスセットアップに組み込める標準 MCP ツールを公開している。

> **セキュリティ注意**: パッケージバージョンを常にピン留めすること。このツールは秘密鍵を管理する — 固定されていない `npx` インストールはサプライチェーンリスクをもたらす。

### オプション A: agentwallet-sdk（Base / マルチチェーン）

```json
{
  "mcpServers": {
    "agentpay": {
      "command": "npx",
      "args": ["agentwallet-sdk@6.0.0"]
    }
  }
}
```

### 利用可能なツール（エージェント呼び出し可能）

| ツール | 目的 |
|------|---------|
| `get_balance` | エージェントウォレットの残高を確認する |
| `send_payment` | アドレスまたは ENS に支払いを送る |
| `check_spending` | 残り予算を照会する |
| `list_transactions` | 全ての支払いの監査証跡 |

> **注意**: 支出ポリシーは**オーケストレーター**がエージェントに委任する前に設定する — エージェント自身では設定しない。これによりエージェントが自身の支出制限を引き上げるのを防ぐ。ポリシーはオーケストレーションレイヤーまたはタスク前フックの `set_policy` で設定する。エージェント呼び出し可能なツールとして設定しない。

### オプション B: OKX Agent Payments Protocol（X Layer）

X Layer x402、Multi-Party Payment（MPP）、セッション決済、チャージ、A2A チャージフローにはこのパスを使用する。

バイヤー側のエージェントフローの場合:

1. 現在の `okx/onchainos-skills` リポジトリをインストールまたは参照する。
2. `skills/okx-agent-payments-protocol/SKILL.md` をディスパッチャーとして使用する。
3. `skills/okx-x402-payment/SKILL.md` は非推奨の互換エイリアスとして扱い、正規のスキルとしては扱わない。
4. ウォレットステータスの確認や決済アクションの前にユーザーの明示的な確認を必要とする。汎用ツール呼び出しの後ろに決済実行を隠さない。

セラー側の API フローの場合、コードを生成する前に最新の言語固有のガイドを取得する:

| ランタイム | 現在のガイド |
|---------|---------------|
| TypeScript | `https://raw.githubusercontent.com/okx/payments/main/typescript/SELLER.md` |
| Go | `https://raw.githubusercontent.com/okx/payments/main/go/x402/SELLER.md` |
| Rust | `https://raw.githubusercontent.com/okx/payments/main/rust/x402/SELLER.md` |
| Java | `https://raw.githubusercontent.com/okx/payments/main/java/SELLER.md` |

現在の OKX リポジトリを確認せずに古いドキュメントの例をコピーしない。現在の OKX ガイダンスでは `okx-agent-payments-protocol` をディスパッチャーとして使用しており、Java セラードキュメントも現在利用可能。

## 例

### MCP クライアントでの予算適用

agentpay MCP サーバーを呼び出すオーケストレーターを構築する場合、有料ツール呼び出しをディスパッチする前に予算を適用する。

> **前提条件**: MCP 設定を追加する前にパッケージをインストールする — 非インタラクティブ環境では `-y` なしの `npx` が確認を求めてサーバーがハングする: `npm install -g agentwallet-sdk@6.0.0`

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  // 1. Validate credentials before constructing the transport.
  //    A missing key must fail immediately — never let the subprocess start without auth.
  const walletKey = process.env.WALLET_PRIVATE_KEY;
  if (!walletKey) {
    throw new Error("WALLET_PRIVATE_KEY is not set — refusing to start payment server");
  }

  // Connect to the agentpay MCP server via stdio transport.
  // Whitelist only the env vars the server needs — never forward all of process.env
  // to a third-party subprocess that manages private keys.
  const transport = new StdioClientTransport({
    command: "npx",
    args: ["agentwallet-sdk@6.0.0"],
    env: {
      PATH: process.env.PATH ?? "",
      NODE_ENV: process.env.NODE_ENV ?? "production",
      WALLET_PRIVATE_KEY: walletKey,
    },
  });
  const agentpay = new Client({ name: "orchestrator", version: "1.0.0" });
  await agentpay.connect(transport);

  // 2. Set spending policy before delegating to the agent.
  //    Always verify success — a silent failure means no controls are active.
  const policyResult = await agentpay.callTool({
    name: "set_policy",
    arguments: {
      per_task_budget: 0.50,
      per_session_budget: 5.00,
      allowlisted_recipients: ["api.example.com"],
    },
  });
  if (policyResult.isError) {
    throw new Error(
      `Failed to set spending policy — do not delegate: ${JSON.stringify(policyResult.content)}`
    );
  }

  // 3. Use preToolCheck before any paid action
  await preToolCheck(agentpay, 0.01);
}

// Pre-tool hook: fail-closed budget enforcement with four distinct error paths.
async function preToolCheck(agentpay: Client, apiCost: number): Promise<void> {
  // Path 1: Reject invalid input (NaN/Infinity bypass the < comparison)
  if (!Number.isFinite(apiCost) || apiCost < 0) {
    throw new Error(`Invalid apiCost: ${apiCost} — action blocked`);
  }

  // Path 2: Transport/connectivity failure
  let result;
  try {
    result = await agentpay.callTool({ name: "check_spending" });
  } catch (err) {
    throw new Error(`Payment service unreachable — action blocked: ${err}`);
  }

  // Path 3: Tool returned an error (e.g., auth failure, wallet not initialised)
  if (result.isError) {
    throw new Error(
      `check_spending failed — action blocked: ${JSON.stringify(result.content)}`
    );
  }

  // Path 4: Parse and validate the response shape
  let remaining: number;
  try {
    const parsed = JSON.parse(
      (result.content as Array<{ text: string }>)[0].text
    );
    if (!Number.isFinite(parsed?.remaining)) {
      throw new TypeError("missing or non-finite 'remaining' field");
    }
    remaining = parsed.remaining;
  } catch (err) {
    throw new Error(
      `check_spending returned unexpected format — action blocked: ${err}`
    );
  }

  // Path 5: Budget exceeded
  if (remaining < apiCost) {
    throw new Error(
      `Budget exceeded: need $${apiCost} but only $${remaining} remaining`
    );
  }
}

main().catch((err) => {
  console.error(err);
  process.exitCode = 1;
});
```

## ベストプラクティス

- **委任前に予算を設定する**: サブエージェントを生成する際は、オーケストレーションレイヤーを通じて SpendingPolicy を添付する。エージェントに無制限の支出を与えない。
- **依存関係をピン留めする**: MCP 設定では常に正確なバージョンを指定する（例: `agentwallet-sdk@6.0.0`）。本番環境にデプロイする前にパッケージの整合性を確認する。
- **監査証跡**: タスク後のフックで `list_transactions` を使用して支出内容とその理由をログに記録する。
- **フェイルクローズ**: 決済ツールに到達できない場合、有料アクションをブロックする — 無課金アクセスへのフォールバックはしない。
- **security-review と組み合わせる**: 決済ツールは高い特権を持つ。シェルアクセスと同じ精査を適用する。
- **まずテストネットでテストする**: 開発には Base Sepolia を使用し、本番には Base メインネットに切り替える。

## 本番リファレンス

- **npm**: [`agentwallet-sdk`](https://www.npmjs.com/package/agentwallet-sdk)
- **NVIDIA NeMo Agent Toolkit にマージ済み**: [PR #17](https://github.com/NVIDIA/NeMo-Agent-Toolkit-Examples/pull/17) — NVIDIA のエージェント例向けの x402 決済ツール
- **プロトコル仕様**: [x402.org](https://x402.org)
- **OKX Payments SDK**: [`okx/payments`](https://github.com/okx/payments) — X Layer x402 向けの TypeScript、Go、Rust、Java セラーインテグレーション
- **OKX Agent Payments Protocol スキル**: [`okx/onchainos-skills`](https://github.com/okx/onchainos-skills/tree/main/skills/okx-agent-payments-protocol)
- **OKX Payments 概要**: [web3.okx.com/onchainos/dev-docs/payments/overview](https://web3.okx.com/onchainos/dev-docs/payments/overview)
