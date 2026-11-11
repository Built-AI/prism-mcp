# Revalidata MCP server

> The [MIT license](LICENSE) in this repository covers its documentation and examples only. It does not cover the Revalidata service.

[Revalidata](https://revalidata.com) reads contracts and lists the deadlines in them: the date, what happens if it is missed, and the passage of the contract it was read from where one was captured. It reads residential and commercial leases, mortgages, insurance policies, HOA documents, purchase and sale agreements, construction contracts, loan modifications and vendor agreements. Dates are computed from the contract's own rules, not estimated. It can miss dates; how often is measured and published at [revalidata.com/accuracy](https://revalidata.com/accuracy).

This repository documents how to connect an AI agent to the hosted Revalidata MCP server. Revalidata is a product of Built AI, Inc.

## Hosted endpoint

| | |
|---|---|
| URL | `https://revalidata.com/mcp` |
| Transport | MCP Streamable HTTP |
| Auth | OAuth sign-in, or a Revalidata API key as a bearer token |
| Registry name | `com.revalidata/contracts` |
| Claude connector directory | [Revalidata Contracts](https://claude.ai/directory/prism-mcp) |

No API key is needed to start. The first time your client connects, it signs in to a Revalidata account with OAuth and you choose what the agent may do.

**OAuth details**

- OAuth 2.1 authorization code flow with PKCE (`S256`); public clients, no client secret.
- Dynamic client registration is supported, so MCP clients register themselves.
- Scopes:
  - `prism:read`: see your saved contracts, their deadlines and the risks Revalidata found.
  - `prism:write`: send contracts for Revalidata to read, using your credits.
- Protected resource metadata: `https://revalidata.com/.well-known/oauth-protected-resource/mcp`
- Authorization server metadata: `https://revalidata.com/.well-known/oauth-authorization-server`

**Without OAuth**

Create an API key on your [Billing page](https://revalidata.com/prism/billing) after you sign in, and send it as a bearer token:

```
Authorization: Bearer YOUR_REVALIDATA_KEY
```

**Machine-readable descriptions**

- MCP server card: `https://revalidata.com/.well-known/mcp/server-card.json`
- A2A agent card: `https://revalidata.com/.well-known/agent-card.json`
- Summary for LLMs: `https://revalidata.com/llms.txt`

## Connect

### Claude (claude.ai and Claude Desktop)

Find **Revalidata Contracts** in Claude's connector directory and connect it, or add it yourself:

1. Open **Settings**, then **Connectors**.
2. Choose **Add custom connector**.
3. Paste `https://revalidata.com/mcp` and sign in to Revalidata when prompted.

See [examples/claude-desktop.md](examples/claude-desktop.md).

### Claude Code

```sh
claude mcp add --transport http revalidata https://revalidata.com/mcp
```

See [examples/claude-code.md](examples/claude-code.md).

### Cursor

Add to `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "revalidata": { "url": "https://revalidata.com/mcp" }
  }
}
```

See [examples/cursor-mcp.json](examples/cursor-mcp.json).

### VS Code

Add to `.vscode/mcp.json` in your workspace:

```json
{
  "servers": {
    "revalidata": {
      "type": "http",
      "url": "https://revalidata.com/mcp"
    }
  }
}
```

See [examples/vscode-mcp.json](examples/vscode-mcp.json).

### Windsurf and other MCP clients

Any client that supports remote MCP servers needs three values:

- URL: `https://revalidata.com/mcp`
- Transport: Streamable HTTP
- Auth: OAuth (the client discovers it from the server), or an `Authorization: Bearer YOUR_REVALIDATA_KEY` header

Most clients accept the same `mcpServers` shape shown for Cursor. Check your client's documentation for the exact key name for a remote URL (Windsurf, for example, uses `serverUrl`).

**Connected before?** The former addresses, `https://revalidata.com/api/prism-mcp` and `https://prism.parad1gm.com/api/prism-mcp`, still work, so nothing you set up earlier breaks. New setups should use `https://revalidata.com/mcp`.

## Tools

| Tool | What it does |
|---|---|
| `prism_review_contract` | Send a contract as exactly one of `url`, `text` or `fileBase64` (a PDF or photo). Returns a `contractId` at once; reading takes about a minute. Sending the same document again returns the same contract. |
| `prism_get_contract` | One contract's status and results by `contractId`: the summary, deadlines (kind, date, days until due, what happens if it is missed, the source sentence), risks and, for leases, mortgages, insurance policies, HOA documents and service agreements, an abstract of the key terms. In full for a paid contract, the free preview otherwise. Never spends credits. |
| `prism_unlock_contract` | The full results for a contract that has finished reading. Spends 1 credit ($0.50) unless it is already paid for. Ask your user before calling it. |
| `prism_upcoming_deadlines` | Open deadlines across every contract in the account, soonest first, within `withinDays` (default 90). |
| `prism_list_contracts` | Every contract in the account, newest first, with its status and whether full results are unlocked. Paginated. |
| `prism_get_account` | Credits remaining, prices, whether an Unlimited subscription covers everything, and contracts added this month. `creditStatus` is `ok`, `low` (2 or fewer left), `empty` or `unlimited`. |
| `prism_buy_credits` | A Stripe checkout link for your user to buy credits ($0.50 each, 20 to 2,000 at a time). It cannot pay: your user opens the link and pays on Stripe's page. Ask your user before calling it. |
| `prism_send_feedback` | Tell the Revalidata team a result was wrong, a document type is not supported, a feature is missing, or what price would work. A person reads every message; there is no promise of a reply or a fix. Free. A wrong result the team reproduces earns the account 10 free credits. |

Full input and output schemas are in the [server card](https://revalidata.com/.well-known/mcp/server-card.json).

A typical run: call `prism_review_contract`, wait about a minute, then call `prism_get_contract`. If the contract is not paid for, `prism_unlock_contract` spends one credit for the full results, with your user's agreement.

**Running out of credits.** Check `creditStatus` from `prism_get_account` before a batch. When it is `low` or `empty`, or an unlock returns `code: "insufficient_credits"`, ask your user, then call `prism_buy_credits` for a checkout link they pay on themselves.

## A2A

Revalidata also speaks the Agent2Agent (A2A) protocol, JSON-RPC binding, versions 1.0 and 0.3.

- Agent card: `https://revalidata.com/.well-known/agent-card.json`
- Skills: `read-contract` (send a contract as a message, then poll the task), `unlock-contract` (reply `unlock` on a task in `input-required` state) and `send-feedback`.

A2A uses the same OAuth server and tokens as the MCP server, and a Revalidata API key also works as a bearer token. Prices are the same.

## Pricing

**Agents** (MCP and A2A)

- $0.50 per contract (1 credit), charged once, when its full results are first delivered.
- Every new account starts with 3 free credits.
- Previews, re-reads, and documents Revalidata cannot read are free. The same document is never charged twice.
- Credits are bought 20 or more at a time (minimum $10) through a checkout link your user pays on.
- Contracts an agent unlocks get the same email reminders as contracts you add yourself: 90, 30 and 7 days before each deadline.

**People** (on the website)

- $1 once per contract,
- $9 a month (Unlimited) for up to 50 contracts a month, or
- $49 a month on the property-manager plan, for 5 people and up to 1,000 contracts a month.

## Privacy and data

- A connected agent only reaches your own Revalidata account. You can allow it to read without sending contracts (`prism:read` only), and disconnect it at any time from Security in your Revalidata account.
- Revalidata does not sell your data. Documents are shared only with the service providers needed to run it: the AI providers that read your documents, the email provider that sends alerts, Stripe for payment, and the cloud host.
- Payment details are handled by Stripe. Revalidata never sees or stores card numbers.
- Uploading a document does not give Revalidata any ownership of it. Only upload documents you have the right to share.
- Data is kept while your account is open. Email support@revalidata.com to export your data or delete your account.
- Revalidata explains documents. It is not legal advice, and it can make mistakes: check anything important against the document itself.

Read the full [Privacy policy](https://revalidata.com/privacy) and [Terms](https://revalidata.com/terms). Revalidata is operated by Built AI, Inc.

## Support

Email support@revalidata.com. For problems connecting a client, you can also [open an issue](https://github.com/Built-AI/prism-mcp/issues/new/choose). Do not post contract contents, personal details, API keys or tokens in an issue.

## About this repository

This repository holds documentation and client configuration for the hosted Revalidata MCP server, plus the [`server.json`](server.json) published to the MCP Registry as `com.revalidata/contracts`. The tool names keep their original `prism_` prefix so existing integrations keep working. The Revalidata service itself is closed source; there is no server code here.
