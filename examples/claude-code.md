# Claude Code

Add Prism (OAuth sign-in on first use):

```sh
claude mcp add --transport http prism https://prism.parad1gm.com/api/prism-mcp
```

Then run `/mcp` inside Claude Code, select `prism`, and sign in to your Prism account.

Add Prism with an API key instead (create one on the [Billing page](https://prism.parad1gm.com/prism/billing)):

```sh
claude mcp add --transport http prism https://prism.parad1gm.com/api/prism-mcp \
  --header "Authorization: Bearer YOUR_PRISM_KEY"
```

Check it is connected:

```sh
claude mcp list
```

Remove it:

```sh
claude mcp remove prism
```
