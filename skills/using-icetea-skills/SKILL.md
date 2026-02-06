---
name: using-icetea-skills
description: Use when starting any conversation - maps project patterns to icetea-skills, requiring skill invocation before implementation.
---

<CRITICAL>
CHECK these patterns BEFORE any implementation:
</CRITICAL>

## Skill Routing

### Backend (Cloudflare Workers)
| Pattern | Invoke Skill |
|---------|--------------|
| `extends DurableObject`, `DurableObjectState` | `durable-objects` |
| `ctx.storage`, alarms, WebSockets, RPC | `durable-objects` |
| `wrangler.jsonc`, `wrangler.toml` | `wrangler` |
| Workers, Pages, D1, R2, KV, Queues | `cloudflare` |

### Frontend (React/Next.js)
| Pattern | Invoke Skill |
|---------|--------------|
| React components, hooks, state | (coming soon) |

### Performance
| Pattern | Invoke Skill |
|---------|--------------|
| Lighthouse, Core Web Vitals, LCP, CLS | `web-perf` |

## Red Flags
| Thought | Reality |
|---------|---------|
| "I know this already" | Skills have project-specific patterns. Check first. |
| "This is a small change" | Small changes often have big implications. Check patterns. |
| "I'll check docs later" | Skills ARE the distilled docs. Use them first. |
