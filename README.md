# SWIRL plugin for Claude Code

Deploy, configure, and integrate [SWIRL Enterprise](https://swirlaiconnect.com)
 - federated search + RAG across your organization's data, searched where it
lives - directly from Claude Code.

## Install

```bash
claude plugin marketplace add swirlai/swirl-claude-plugin
claude plugin install swirl@swirl --scope user
```

Start a new Claude Code session after installing.

## Update

Get the latest version at any time:

```bash
claude plugin marketplace update swirl
claude plugin update swirl@swirl
```

Then start a new Claude Code session. `claude plugin list` shows the
installed version; the latest release is on the
[releases page](https://github.com/swirlai/swirl-claude-plugin/releases).

## Commands

| Command | What it does |
|---|---|
| `/swirl:start` | Assess where you are and route to the right skill |
| `/swirl:install` | Guided install - Docker Compose or local - with live verification |
| `/swirl:railway` | Deploy and manage SWIRL on Railway - the one-click hosted template |
| `/swirl:provider` | SearchProvider wizard: connect a source using 25+ built-in connectors |
| `/swirl:connector` | Develop a custom connector when no built-in fits |
| `/swirl:rag` | Connect an LLM (OpenAI, Anthropic, Azure, Ollama, …) and tune grounded RAG |
| `/swirl:migrate` | Community → Enterprise migration, or version upgrades - with verification |
| `/swirl:mcp` | Wire the SWIRL MCP server into Claude so agents search through SWIRL (Enterprise) |
| `/swirl:troubleshoot` | Log-driven diagnosis of a misbehaving deployment |

Both editions are supported: skills call out the differences where they
matter (install paths, RAG configuration, and the MCP server, which is an
Enterprise feature).

## Typical flows

**New deployment:**

```
/swirl:install          # get the stack running, verified with a live search
/swirl:provider         # connect your first sources
/swirl:rag              # grounded answers with citations
```

**Give Claude access to your enterprise data:**

```
/swirl:mcp              # one token, one command - Claude searches everything
                        # SWIRL federates, with your permissions enforced
```

## Requirements

- Claude Code CLI
- Docker (for the recommended install path) or Python 3.11+
- SWIRL Community (open source) or a SWIRL Enterprise license

## Security

These skills never print API keys, tokens, client secrets, or license
contents; configuration examples use placeholders such as `<api-key>` and
`<license-json>`. SWIRL MCP resources are secret-scrubbed server-side.

## Documentation

https://docs.swirlaiconnect.com

## License

Apache 2.0 - see [LICENSE](LICENSE). © SWIRL Corporation.
