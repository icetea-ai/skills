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

### npx skills

Install using the [`npx skills`](https://skills.sh) CLI:

```
npx skills add https://github.com/icetea-ai/skills
```

## Skills

Skills are contextual and auto-loaded based on your conversation. When a request matches a skill's triggers, the agent loads and applies the relevant skill to provide accurate, up-to-date guidance.

| Skill | Useful for |
|-------|------------|
| cloudflare | Workers, Pages, Queues, KV, D1, R2, Workers AI, Vectorize, Hyperdrive, Containers, Workflows |
| durable-objects | DurableObject classes, ctx.storage, blockConcurrencyWhile, alarms, WebSockets, RPC, SQLite storage, stateful coordination |
| wrangler | Deploying and managing Workers, KV, R2, D1, Vectorize, Queues, Workflows |
| web-perf | Auditing Core Web Vitals (FCP, LCP, TBT, CLS), render-blocking resources, network chains |
| using-icetea-skills | Maps project patterns to skills, guides which skill to invoke before implementation |

### Auto-Discovery Limitations

Skills don't always auto-discover due to security constraints - we avoid SessionStart hooks that execute scripts. If Claude doesn't automatically invoke the right skill:

1. **Explicitly tell Claude**: "Use the durable-objects skill" or "Load icetea-skills:cloudflare"
2. **Add to CLAUDE.md**: Include skill routing in your project's `CLAUDE.md` file.

## Resources

This is a fork of [Cloudflare Skills](https://github.com/cloudflare/skills) but focusing on the techstack at Icetea AI & Robotics.