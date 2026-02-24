# Icetea AI & Robotics Skills

A collection of [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) for building applications. Originally forked from [cloudflare/skills](https://github.com/cloudflare/skills), with opinionated modifications.

## Installing

These skills work with any agent that supports the Agent Skills standard, including Claude Code, OpenCode, OpenAI Codex, and Pi.

### Claude Code

Install using the [plugin marketplace](https://code.claude.com/docs/en/discover-plugins#add-from-github):

```bash
# Step 1: Add the marketplace
/plugin marketplace add icetea-ai/skills

# Step 2: Install the plugin (via UI or CLI)
/plugin install icetea-skills@icetea-ai
# Or: /plugin → Discover → Select skills → Install
```

Once installed, all skills are automatically available and load when relevant to your conversation.

### Cursor

Install from the Cursor Marketplace or add manually via **Settings > Rules > Add Rule > Remote Rule (Github)** with `cloudflare/skills`.

### npx skills

Install using the [`npx skills`](https://skills.sh) CLI:

```
npx skills add https://github.com/icetea-ai/skills
```

### Clone / Copy

Clone this repo and copy the skill folders into the appropriate directory for your agent:

| Agent | Skill Directory | Docs |
|-------|-----------------|------|
| Claude Code | `~/.claude/skills/` | [docs](https://code.claude.com/docs/en/skills) |
| Cursor | `~/.cursor/skills/` | [docs](https://cursor.com/docs/context/skills) |
| OpenCode | `~/.config/opencode/skills/` | [docs](https://opencode.ai/docs/skills/) |
| OpenAI Codex | `~/.codex/skills/` | [docs](https://developers.openai.com/codex/skills/) |
| Pi | `~/.pi/agent/skills/` | [docs](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent#skills) |

## Skills

Skills are contextual and auto-loaded based on your conversation. When a request matches a skill's triggers, the agent loads and applies the relevant skill to provide accurate, up-to-date guidance.

| Skill | Useful for |
|-------|------------|
| cloudflare | Comprehensive platform skill covering Workers, Pages, storage (KV, D1, R2), AI (Workers AI, Vectorize, Agents SDK), networking (Tunnel, Spectrum), security (WAF, DDoS), and IaC (Terraform, Pulumi) |
| agents-sdk | AI agents on Workers using Agents SDK: stateful agents, @callable RPC, workflows, MCP servers, streaming chat, React hooks |
| durable-objects | DurableObject classes, ctx.storage, blockConcurrencyWhile, alarms, WebSockets, RPC, SQLite storage, stateful coordination |
| sandbox-sdk | Secure code execution environments: Sandbox SDK, code interpreter, file ops, preview URLs |
| wrangler | Deploying and managing Workers, KV, R2, D1, Vectorize, Queues, Workflows |
| web-perf | Auditing Core Web Vitals (FCP, LCP, TBT, CLS), render-blocking resources, network chains |
| workers-best-practices | Workers code review, anti-patterns, production best practices, type checking, config validation |
| tanstack-query-best-practices | TanStack Query (React Query) best practices: query keys, cache integrity, mutations, SSR hydration, performance, TypeScript patterns, testing |

### Auto-Discovery Limitations

Skills don't always auto-discover due to security constraints - we avoid SessionStart hooks that execute scripts. If Claude doesn't automatically invoke the right skill:

1. **Explicitly tell Claude**: "Use the durable-objects skill" or "Load icetea-skills:cloudflare"
2. **Add to CLAUDE.md**: Include skill routing in your project's `CLAUDE.md` file.

## MCP Servers

| Server | Purpose |
|--------|---------|
| cloudflare-docs | Up-to-date Cloudflare documentation and reference |
| cloudflare-bindings | Build Workers applications with storage, AI, and compute primitives |
| cloudflare-builds | Manage and get insights into Workers builds |
| cloudflare-observability | Debug and analyze application logs and analytics |
| cloudflare-api | Cloudflare API operations (account, zones, DNS, etc.) |
| chrome-devtools | Browser automation and debugging via Chrome DevTools |

## Resources

This is a fork of [Cloudflare Skills](https://github.com/cloudflare/skills) but focusing on the techstack at Icetea AI & Robotics.
