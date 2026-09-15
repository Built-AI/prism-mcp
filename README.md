# Prism MCP server

> The [MIT license](LICENSE) in this repository covers its documentation and examples only. It does not cover the Prism service.

[Prism](https://prism.parad1gm.com) reads contracts -- residential and commercial leases, mortgages, insurance policies and HOA documents -- and returns every deadline (renewal notices, cancellation windows, rent reviews, option dates) with its exact date, what happens if it is missed, and the sentence of the contract it came from. Dates are computed from the contract's own rules, not estimated. This repository documents how to connect an AI agent to the hosted Prism MCP server.

## Hosted endpoint

| | |
|---|---|
| URL | `https://prism.parad1gm.com/api/prism-mcp` |
| Transport | MCP Streamable HTTP |
| Auth | OAuth sign-in, or a Prism API key as a bearer token |
| Registry name | `com.parad1gm/prism` |

No API key is needed to start. The first time your client connects, it signs in to a Prism account with OAuth and you choose what the agent may do.

**OAuth details**

- OAuth 2.1 authorization code flow with PKCE (`S256`); public clients, no client secret.
- Dynamic client registration is supported, so MCP clients register themselves.
- Scopes:
  - `prism:read` -- see your saved contracts, their deadlines and the risks Prism found.
  - `prism:write` -- send contracts for Prism to read, using your credits.
- Protected resource metadata: `https://prism.parad1gm.com/.well-known/oauth-protected-resource/api/prism-mcp`
- Authorization server metadata: `https://prism.parad1gm.com/.well-known/oauth-authorization-server`

**Without OAuth**

Create an API key on your [Billing page](https://prism.parad1gm.com/prism/billing) and send it as a bearer token:

```
Authorization: Bearer YOUR_PRISM_KEY
```

**Machine-readable descriptions**

- MCP server card: `https://prism.parad1gm.com/.well-known/mcp/server-card.json`
- A2A agent card: `https://prism.parad1gm.com/.well-known/agent-card.json`
- Summary for LLMs: `https://prism.parad1gm.com/llms.txt`

## Connect

### Claude (claude.ai and Claude Desktop)

1. Open **Settings**, then **Connectors**.
2. Choose **Add custom connector**.
3. Paste `https://prism.parad1gm.com/api/prism-mcp` and sign in to Prism when prompted.

See [examples/claude-desktop.md](examples/claude-desktop.md).

### Claude Code

```sh
claude mcp add --transport http prism https://prism.parad1gm.com/api/prism-mcp
```

See [examples/claude-code.md](examples/claude-code.md).

### Cursor

Add to `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "prism": { "url": "https://prism.parad1gm.com/api/prism-mcp" }
  }
}
```

See [examples/cursor-mcp.json](examples/cursor-mcp.json).

### VS Code

Add to `.vscode/mcp.json` in your workspace:

```json
{
  "servers": {
    "prism": {
      "type": "http",
      "url": "https://prism.parad1gm.com/api/prism-mcp"
    }
  }
}
```

See [examples/vscode-mcp.json](examples/vscode-mcp.json).

### Windsurf and other MCP clients

Any client that supports remote MCP servers needs three values:

- URL: `https://prism.parad1gm.com/api/prism-mcp`
- Transport: Streamable HTTP
- Auth: OAuth (the client discovers it from the server), or an `Authorization: Bearer YOUR_PRISM_KEY` header

Most clients accept the same `mcpServers` shape shown for Cursor. Check your client's documentation for the exact key name for a remote URL (Windsurf, for example, uses `serverUrl`).

## Tools

| Tool | What it does |
|---|---|
| `prism_review_contract` | Send a lease, mortgage, insurance policy, HOA document or commercial lease as exactly one of `url`, `text` or `fileBase64` (a PDF or photo, up to 25MB). Returns a `contractId` at once; reading takes about a minute. |
| `prism_get_contract` | One contract's status and results by `contractId`: every fact if it is paid for, otherwise the free 3-fact preview. Never spends credits. |
| `prism_unlock_contract` | Every deadline and risk for a contract that has finished reading. Spends 1 credit ($0.50) unless it is already paid for. With no credits it returns the preview with `code: "insufficient_credits"`. |
| `prism_upcoming_deadlines` | Open deadlines across every contract in the account, soonest first, within `withinDays` (default 90). |
| `prism_list_contracts` | Every contract in the account, newest first, with its status and whether full results are unlocked. Paginated. |
| `prism_get_account` | Credits remaining, prices, whether an Unlimited subscription covers everything, and contracts added this month. `creditStatus` is `ok`, `low` (2 or fewer left), `empty` or `unlimited`, and `purchaseOptions` says whether buying needs your user (a checkout link) or the agent can pay itself. |
| `prism_buy_credits` | Ways to buy 20-2000 credits at $0.50 each: a Stripe checkout link for your user and, when available, an MPP purchase URL the agent can pay itself. |
| `prism_send_feedback` | Tell the Prism team a result was wrong, something was missing, or what price would work. A person reads every message. Free. A wrong result the team reproduces earns the account 10 free credits. |

Full input and output schemas are in the [server card](https://prism.parad1gm.com/.well-known/mcp/server-card.json).

A typical run: call `prism_review_contract`, wait about a minute, then call `prism_get_contract`. If the contract is not paid for, `prism_unlock_contract` spends one credit for the full results. Sending the same document again returns the same contract.

**Running out of credits without surprises.** Check `creditStatus` from `prism_get_account` before a batch. When it is `low` or `empty` -- or an unlock returns `code: "insufficient_credits"` -- call `prism_buy_credits`. Any connected agent may call it on its own: it returns a checkout link for your user and, when `purchaseOptions.agentPayment` is true, a purchase URL the agent can pay itself over the Machine Payments Protocol (MPP). Only the checkout link needs a person.

## A2A

Prism also speaks the Agent2Agent (A2A) protocol, JSON-RPC binding, versions 1.0 and 0.3.

- Agent card: `https://prism.parad1gm.com/.well-known/agent-card.json`
- Skills: `read-contract` (send a contract as a message, then poll the task) and `unlock-contract` (reply `unlock` on a task in `input-required` state).

A2A uses the same OAuth server and tokens as the MCP server, and a Prism API key also works as a bearer token. Prices are the same.

## Pricing

**Agents** (MCP and A2A)

- $0.50 per contract (1 credit), charged once, when its full results are first delivered.
- Every new account starts with 3 free credits.
- Previews, re-reads, and documents Prism cannot read are free. The same document is never charged twice.
- Credits are bought 20 or more at a time (minimum $10). Your agent can hand you a checkout link, or pay itself with an MPP wallet when that option is available.
- Contracts an agent unlocks get the same email reminders as contracts you add yourself: 90, 30 and 7 days before each deadline.
- Report a result Prism got wrong with `prism_send_feedback` (the contract and the date or clause you expected). When the team reproduces it, the account gets 10 free credits.

**People** (on the website)

- $1 once per contract, or
- $9 a month (Unlimited) for up to 50 contracts a month.

## Privacy and data

- A connected agent only reaches your own Prism account. You can allow it to read without sending contracts (`prism:read` only), and disconnect it at any time from [Security](https://prism.parad1gm.com/prism/security).
- Prism does not sell your data. Documents are shared only with the service providers needed to run Prism: the AI providers that read your documents, the email provider that sends alerts, Stripe for payment, and the cloud host.
- Payment details are handled by Stripe. Prism never sees or stores card numbers.
- Uploading a document does not give Prism any ownership of it. Only upload documents you have the right to share.
- Data is kept while your account is open. Email support@parad1gm.com to export your data or delete your account.
- Prism explains documents. It is not legal advice, and it can make mistakes: check anything important against the document itself.

Read the full [Privacy policy](https://prism.parad1gm.com/privacy) and [Terms](https://prism.parad1gm.com/terms). Prism is operated by Built AI, Inc.

## Support

Email support@parad1gm.com. For problems connecting a client, you can also [open an issue](https://github.com/Built-AI/prism-mcp/issues/new/choose). Do not post contract contents, personal details, API keys or tokens in an issue.

## About this repository

This repository holds documentation and client configuration for the hosted Prism MCP server, plus the [`server.json`](server.json) published to the MCP Registry. The Prism service itself is closed source; there is no server code here.
