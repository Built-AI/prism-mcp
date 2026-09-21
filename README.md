# Prism by Parad1gm

> The [MIT license](LICENSE) in this repository covers its documentation and examples only. It does not cover the Prism service.

Prism reads leases (residential, office, retail, industrial, ground), mortgages and loan notes and loan modifications, insurance policies and certificates, HOA documents, purchase and sale agreements, construction contracts, and vendor and property service agreements, and returns every deadline and critical date with its exact date, what happens if it is missed, and the sentence of the document it came from. Dates are computed by code from the document's stated dates and rules, never estimated by the model. For a paid contract it also returns a structured abstract of key terms (about 35 for a lease: parties, premises, area, rent and steps, deposit, permitted use, options, insurance, default, holdover), each with a verbatim quote, and a term the document does not state is left empty. An amendment is a contract of its own: sent with `amendsContractId`, it is read together with the contract it changes and its earlier amendments. Each deadline of a paid contract gets email reminders 90, 30 and 7 days before it falls due. Prism is not legal advice.

This repository documents how to connect an AI agent to the hosted Prism MCP server. The service itself is closed source; there is no server code here.

## Hosted endpoint

| | |
|---|---|
| URL | `https://prism.parad1gm.com/api/prism-mcp` |
| Transport | MCP Streamable HTTP |
| Auth | OAuth 2.1 with dynamic client registration, or a Prism API key as a bearer token |
| Registry name | `com.parad1gm/prism` |
| Setup page | [prism.parad1gm.com/agents](https://prism.parad1gm.com/agents?utm_source=github&utm_medium=readme) |

No API key is needed to start. The first time your client connects, it signs in to a Prism account with OAuth and you choose what the agent may do. Every new account starts with 3 free contracts.

## Quick start

### Claude (claude.ai and Claude Desktop)

1. Open **Settings**, then **Connectors**.
2. Choose **Add custom connector**.
3. Name: `Prism`. URL: `https://prism.parad1gm.com/api/prism-mcp`.
4. Add the connector, then connect it and sign in to your Prism account when prompted.

See [examples/claude-desktop.md](examples/claude-desktop.md).

### Claude Code

```sh
claude mcp add --transport http prism https://prism.parad1gm.com/api/prism-mcp
```

Then run `/mcp` inside Claude Code, select `prism`, and sign in. See [examples/claude-code.md](examples/claude-code.md) for the API-key form.

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

### API key instead of OAuth

Create an API key on your [Billing page](https://prism.parad1gm.com/prism/billing?utm_source=github&utm_medium=readme) and send it as a bearer token:

```
Authorization: Bearer YOUR_PRISM_KEY
```

### OAuth details

- OAuth 2.1 authorization code flow with PKCE (`S256`); public clients, no client secret.
- Dynamic client registration is supported, so MCP clients register themselves.
- Scopes: `prism:read` (see your saved contracts, their deadlines and the risks Prism found) and `prism:write` (send contracts for Prism to read, using your credits).
- Protected resource metadata: `https://prism.parad1gm.com/.well-known/oauth-protected-resource/api/prism-mcp`
- Authorization server metadata: `https://prism.parad1gm.com/.well-known/oauth-authorization-server`

### Machine-readable descriptions

- MCP server card: `https://prism.parad1gm.com/.well-known/mcp/server-card.json`
- A2A agent card: `https://prism.parad1gm.com/.well-known/agent-card.json`
- Summary for LLMs: [prism.parad1gm.com/llms.txt](https://prism.parad1gm.com/llms.txt?utm_source=github&utm_medium=readme)

## Tools

| Tool | What it does |
|---|---|
| `prism_review_contract` | Send one document as exactly one of `url`, `text` or `fileBase64` (a PDF or photo, up to 25MB); add `amendsContractId` to have an amendment read together with the contract it changes and its earlier amendments. Returns a `contractId` at once; reading takes about a minute. |
| `prism_get_contract` | One contract's status and results by `contractId`: every deadline with its date, consequence and source sentence, plus, for a paid contract, the structured abstract of key terms with a verbatim quote for each; otherwise the free 3-fact preview. Never spends credits. |
| `prism_unlock_contract` | Every deadline, risk and abstract term for a contract that has finished reading. Spends 1 credit ($0.50) unless it is already paid for. With no credits it returns the preview with `code: "insufficient_credits"`. |
| `prism_upcoming_deadlines` | Open deadlines across every contract in the account, soonest first, within `withinDays` (default 90). |
| `prism_list_contracts` | Every contract in the account, newest first, with its status and whether full results are unlocked. Paginated. |
| `prism_get_account` | Credits remaining, prices, whether an Unlimited subscription covers everything, and contracts added this month. `creditStatus` is `ok`, `low` (2 or fewer left), `empty` or `unlimited`; `purchaseOptions` says whether buying needs your user (a checkout link) or the agent can pay itself. |
| `prism_buy_credits` | Ways to buy 20-2000 credits at $0.50 each: a Stripe checkout link for your user and, when available, an MPP purchase URL the agent can pay itself. |
| `prism_send_feedback` | Tell the Prism team a result was wrong, something was missing, or what price would work. A person reads every message. Free. A wrong result the team reproduces earns the account 10 free credits. |

Full input and output schemas are in the [server card](https://prism.parad1gm.com/.well-known/mcp/server-card.json).

### A typical run

1. Call `prism_review_contract` with the document. Keep the `contractId`.
2. Wait about a minute, then call `prism_get_contract`.
3. If the contract is not paid for, `prism_unlock_contract` spends one credit for the full results.
4. For an amendment, call `prism_review_contract` again with the amendment and `amendsContractId` set to the original contract's id. The amendment is read against the original and any earlier amendments, so superseded terms are dropped rather than repeated.

Sending the same document again returns the same contract and is not charged twice.

**Running out of credits without surprises.** Check `creditStatus` from `prism_get_account` before a batch. When it is `low` or `empty`, or an unlock returns `code: "insufficient_credits"`, call `prism_buy_credits`. Any connected agent may call it on its own: it returns a checkout link for your user and, when `purchaseOptions.agentPayment` is true, a purchase URL the agent can pay itself over the Machine Payments Protocol (MPP). Only the checkout link needs a person.

## What it does not do

- It is not legal advice. Prism explains documents; for advice on what to do, talk to a lawyer.
- It does not fill gaps. A term the document does not state is left empty in the abstract, and a deadline the document gives no rule for is not estimated.
- It is not deterministic across runs. The reading step uses a language model, so two reads of the same document can differ in which facts are found and how they are worded. The date arithmetic is code and does not vary once the inputs are fixed.
- It is not perfect. On our own synthetic test set of difficult contracts (two runs each, production model, 21 September 2026): expected deadline dates were found 58% of the time in a single document and 75% when an amendment was read together with its contract; deleted or superseded terms were correctly left out 95-97% of the time; lease abstract fields were correct 84-93% on most leases. Check anything important against the document itself, and report a wrong result with `prism_send_feedback`.
- It does not run locally. Documents are sent to the hosted service; see [Privacy and data](#privacy-and-data).

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

**People** (on the website, [prism.parad1gm.com](https://prism.parad1gm.com/?utm_source=github&utm_medium=readme))

- $1 once per contract, or
- $9 a month (Unlimited) for up to 50 contracts a month.

## Privacy and data

- A connected agent only reaches your own Prism account. You can allow it to read without sending contracts (`prism:read` only), and disconnect it at any time from [Security](https://prism.parad1gm.com/prism/security?utm_source=github&utm_medium=readme).
- Prism does not sell your data. Documents are shared only with the service providers needed to run Prism: the model providers that read your documents, the email provider that sends reminders, Stripe for payment, and the cloud host.
- Payment details are handled by Stripe. Prism never sees or stores card numbers.
- Uploading a document does not give Prism any ownership of it. Only upload documents you have the right to share.
- Data is kept while your account is open. Email support@parad1gm.com to export your data or delete your account.

Read the full [Privacy policy](https://prism.parad1gm.com/privacy?utm_source=github&utm_medium=readme) and [Terms](https://prism.parad1gm.com/terms?utm_source=github&utm_medium=readme). Prism is operated by Built AI, Inc.

## Links

- Setup page for agents: [prism.parad1gm.com/agents](https://prism.parad1gm.com/agents?utm_source=github&utm_medium=readme)
- Summary for LLMs: [prism.parad1gm.com/llms.txt](https://prism.parad1gm.com/llms.txt?utm_source=github&utm_medium=readme)
- [Privacy policy](https://prism.parad1gm.com/privacy?utm_source=github&utm_medium=readme)
- [Terms](https://prism.parad1gm.com/terms?utm_source=github&utm_medium=readme)
- Official MCP Registry: `com.parad1gm/prism`

## Support

Email support@parad1gm.com. For problems connecting a client, you can also [open an issue](https://github.com/Built-AI/prism-mcp/issues/new/choose). Do not post contract contents, personal details, API keys or tokens in an issue.

## About this repository

This repository holds documentation and client configuration for the hosted Prism MCP server, plus the [`server.json`](server.json) published to the MCP Registry. The Prism service itself is closed source; there is no server code here.
