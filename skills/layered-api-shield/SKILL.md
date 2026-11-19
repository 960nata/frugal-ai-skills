---
name: layered-api-shield
description: Protect a public API on a quota-billed stack (Upstash/Redis, serverless DB, LLM keys) with cheapest-first defense layers — free pattern shield, remembered blocklist, single-command rate limit, and a global daily spend ceiling. Use when adding rate limiting, bot/scanner blocking, abuse protection, or LLM budget caps, or when a Redis/KV or AI quota is being drained by traffic.
---

# Layered API shield (cheapest layer first)

On quota-billed infra, the cheapest exploit is not a breach — it is making you spend
your own quota rejecting junk. A scanner sending 5,000 requests that each cost one
Redis command to reject has already won. So layers are ordered by cost:

| # | Layer | Cost | Job |
| - | ----- | ---- | --- |
| 1 | **Shield** | 0 commands | Pure pattern match on path, query, headers |
| 2 | **Blocklist** | 0 → 1 command | Remember attackers (memory first, then KV) |
| 3 | **Rate limit** | 1 command | Per-caller fixed window |
| 4 | **Budget** | 1 command | Global daily ceiling for expensive work (LLM) |

Never use an expensive layer to reject something a free layer can reject.

## 1. Shield (no I/O)
- Block scanner paths your stack never serves: `.php`, `wp-admin`, `.env`, `.git/`, `phpmyadmin`, `cgi-bin`.
- Block attack-tool user agents (`sqlmap`, `nikto`, `nuclei`, `masscan`…), traversal (`../`, `%2e%2e`), obvious injection in query strings.
- Return `{ verdict: 'allow' | 'suspect' | 'block', reason, evidence }` with a stable reason code and trimmed evidence that is safe to log.
- **Do not read the request body in middleware/proxy** — it consumes the stream the route needs. Bodies are validated per route (zod).

## 2. Blocklist
- In-process `Map` first (warm serverless instances reuse memory → zero commands).
- KV ban key claimed once per period with `SET key 1 NX EX ttl` — not rewritten per hit.
- Escalate: a 7-day offence counter makes repeat offenders get longer bans.
- KV keys use a **salted hash** of the IP. Store full IP/UA only for actually-blocked requests, in a separate incidents table, for firewall reports.

## 3. Rate limit — one EVAL instead of GET+INCR+EXPIRE
```ts
const GATE = `
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end
return { c, redis.call('EXISTS', KEYS[2]) }`

const [count, banned] = await redis.eval(GATE, [windowKey, `ban:${ipHash}`], [String(windowSec)])
```
- Window key = `rl:${scope}:${caller}:${Math.floor(now / windowSec)}`.
- Caller = authenticated user id if any, else hashed IP from the platform-set header (on Vercel, `x-forwarded-for` is set by the edge; elsewhere make sure you trust only your proxy).
- **Fail closed-ish:** if KV is down or out of quota, count in memory — do not open the gates.
- Rule tiers: `read` loose (≈120/min), `ops` tight (≈20/min), `ai` very tight.

## 4. Budget (global ceiling)
A per-caller limit does not answer "how much may the whole world spend today?" — 100 IPs
each obeying their quota can still drain every free key by noon.
- One daily counter per unit of work (e.g. "one AI meeting", not tokens — tokens are unknown before the call, and the reject decision must happen before).
- Ceiling from env (`LLM_DAILY_CEILING`), conservative default.
- Public traffic gets a **share** (e.g. 60 %); scheduled jobs use the rest and never check the public share, so the core daily job never fails because the site was busy.

## Checklist
- [ ] Layer order enforced in middleware/route helper
- [ ] Junk requests cost 0 KV commands (test it)
- [ ] One KV round-trip per legit request for the gate
- [ ] KV outage → memory fallback, not unlimited
- [ ] No raw IPs in KV keys or normal logs
