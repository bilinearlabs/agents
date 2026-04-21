# Bilinear Labs - For Agents

A Claude Code (and cross-AI) skill for querying EVM blockchain events with SQL through [BilinearLabs](https://bilinearlabs.io). Covers the BYOABI signature format, ClickHouse quirks, the full CRUD API for queries and dashboards, and a minimal inline dashboard template.

## Install — Claude Code

```
/plugin marketplace add bilinearlabs/agents
/plugin install bilinearlabs@agents
```

And use it. You can get a `sk_YOUR_API_TOKEN` [here](http://bilinearlabs.io/tokens).

```
/bilinearlabs create a dashboard of USDC monthly circulating supply with sk_YOUR_API_TOKEN
```

## Install — other AIs (Cursor, ChatGPT, Codex, Aider, Zed, …)

The skill is plain markdown — point any AI tool at the raw `SKILL.md`:

```
https://raw.githubusercontent.com/bilinearlabs/agents/main/bilinearlabs/skills/bilinearlabs/SKILL.md
```

**Cursor / Zed / Aider / Codex** — fetch into your project's `AGENTS.md` or equivalent context file:

```bash
curl -O https://raw.githubusercontent.com/bilinearlabs/agents/main/bilinearlabs/skills/bilinearlabs/SKILL.md
```

**ChatGPT / Claude API / raw LLMs** — paste the file contents into your system prompt, Custom GPT knowledge, or assistant instructions.

## What's inside

- **What BilinearLabs is** — product, BYOABI, supported chains, what it can and can't do, FAQ.
- **Quickstart** — a no-auth `curl` that returns live USDC transfers.
- **Authentication & hosts** — API key flow, `api.bilinearlabs.io` vs `bilinearlabs.io` (and the SPA verification gotcha).
- **Before writing a query** — researching contract addresses and event ABIs from authoritative sources.
- **Writing queries** — signature grammar, event format, table schema, 6 worked SQL examples.
- **ClickHouse quirks & performance** — wide-int decoding gotchas, window-function bugs, blocked features, useful patterns.
- **Creating dashboards** — customization philosophy, security rules, traceability, minimal inline HTML template, hosted flow.
- **API reference** — complete CRUD surface for queries and dashboards.
- **Common traps** — 10 things that will bite you.

## License

MIT
