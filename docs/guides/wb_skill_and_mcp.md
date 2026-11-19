# Using the W&B Skill and MCP Server

Two complementary ways to make an AI coding assistant W&B-aware:

- **Skills** teach the agent to *write W&B code* — instrumenting training scripts with the W&B SDK, tracing agents with Weave, setting up sweeps and evaluations.
- **MCP server** lets the agent *query live W&B data* — runs, metrics, traces, artifacts, registry — and search the W&B docs.

Use both: Skills when the agent is editing your code, MCP when it's answering questions about your projects.

Official docs: [AI Assistants overview](https://docs.coreweave.com/products/wandb/platform/ai-assistants) · [Skills](https://docs.coreweave.com/platform/wb-skills) · [MCP server](https://docs.coreweave.com/platform/mcp-server)

## Prerequisites

An API key from [forge.coreweave.com/settings#apikeys](https://forge.coreweave.com/settings#apikeys) (a "Forge API Key" — works anywhere a `WANDB_API_KEY` is expected). For this repo it's already in `.env`.

## W&B Skill

Install from [`wandb/skills`](https://github.com/wandb/skills) (needs Node.js for `npx`):

```bash
# all agents, all projects
npx skills add wandb/skills --skill '*' --yes --global

# Claude Code only
npx skills add wandb/skills --agent claude-code --skill '*' --yes --global
```

Drop `--global` to install into the current project only. Works with Claude Code, Codex, Cursor, Gemini CLI, VS Code, Claude Desktop, and LeChat.

The skills cover two domains:

1. **Model training** — log metrics/media, track and compare experiments, analyze runs, tune hyperparameters.
2. **Agent building** — trace agentic apps with Weave, classify failure modes, run evaluations, set up production monitoring.

Specific prompts work best: "Log training metrics for my PyTorch model to W&B", not "help me with W&B".

## W&B MCP Server

Use the hosted server — nothing to install. Claude Code:

```bash
claude mcp add --transport http wandb https://mcp.withwandb.com/mcp \
  --header "Authorization: Bearer $WANDB_API_KEY"
```

Add `--scope user` to make it available in every project. Other clients (Cursor, VS Code, Codex, Claude Desktop) follow the same pattern: HTTP server at `https://mcp.withwandb.com/mcp` with the API key as a bearer token — see the [MCP server docs](https://docs.coreweave.com/platform/mcp-server) for per-client snippets. On Dedicated Cloud / Self-Managed the URL is `https://[YOUR-INSTANCE]/mcp` (a 404 means hosted MCP isn't enabled — ask support).

If you need a local server instead (e.g. to pin a release — hosted tracks main):

```bash
claude mcp add wandb -e WANDB_API_KEY=$WANDB_API_KEY \
  -- uvx --from git+https://github.com/wandb/wandb-mcp-server wandb_mcp_server
```

### What it can do

| Area | Tools |
| --- | --- |
| Discover | list entities/projects, probe a project's metrics/configs, infer a Weave trace schema |
| Runs | query runs/sweeps/configs (GraphQL), metric history, compare runs, diagnose convergence/overfitting/NaNs |
| Weave | query/count traces, resolve trace roots, summarize evaluations |
| Artifacts & Registry | list registries/collections/versions, artifact details, compare versions |
| Reports | create W&B reports, log analysis back to W&B |
| Docs | search W&B documentation |

### Getting good results

- Always name the entity and project explicitly (`your-team/your-project`); for this repo use the values in `.env`.
- On an unfamiliar project, have the agent probe the project (runs) or infer the trace schema (Weave) before querying.
- Ask for metric history rather than raw GraphQL for time-series; start trace queries at schema/summary detail and drill into full only when needed.

Example prompts: "Show me the top 5 runs by `val_metrics/mae` in this project." · "What versions of the checkpoint artifact exist, and what changed between v2 and v3?" · "Generate a W&B report comparing the sweep's best trials."

### Troubleshooting

- **401** — regenerate the key at forge.coreweave.com/settings#apikeys and check the `Bearer` header format.
- **Empty results** — almost always a wrong entity/project name, or the key lacks access.
- **429 on Dedicated Cloud** — instance rate limit; contact support.
