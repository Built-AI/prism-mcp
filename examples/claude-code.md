# Claude Code

Add Revalidata (OAuth sign-in on first use):

```sh
claude mcp add --transport http revalidata https://revalidata.com/mcp
```

Then run `/mcp` inside Claude Code, select `revalidata`, and sign in to your Revalidata account.

Add Revalidata with an API key instead (create one on the [Billing page](https://revalidata.com/prism/billing) after you sign in):

```sh
claude mcp add --transport http revalidata https://revalidata.com/mcp \
  --header "Authorization: Bearer YOUR_REVALIDATA_KEY"
```

Check it is connected:

```sh
claude mcp list
```

Remove it:

```sh
claude mcp remove revalidata
```

Set up earlier under the name `prism` at `https://prism.parad1gm.com/api/prism-mcp`? That still works; there is nothing to change.
