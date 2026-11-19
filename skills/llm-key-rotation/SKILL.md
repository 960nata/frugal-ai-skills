---
name: llm-key-rotation
description: Build a resilient LLM client that pools many API keys per provider, rotates them evenly, cools down keys that hit 429/quota/auth errors, and falls back across providers. Use when the user has several free-tier or rate-limited API keys, sees 429 "quota exceeded" errors, wants provider fallback (Gemini → Groq → OpenAI-compatible → paid), or asks how to multiply free LLM quota safely.
---

# LLM key rotation + provider fallback

Free tiers limit quota **per key**, so ten keys give ten times the quota — but only
if they are used evenly. "Use key 1 until it breaks, then key 2" keeps key 1 hot and
the rest idle. This skill builds a two-level retry: another key on the same provider
first, then the next provider.

## Steps

1. **Collect keys from env** with the convention `PREFIX`, `PREFIX_1`, `PREFIX_2` … `PREFIX_100`.
   - Skip gaps (a deleted `_5` must not hide `_6`).
   - Skip placeholders (`<...>`, `changeme`) and duplicates.
   - Store `{ value, index, fingerprint, envName }`. `fingerprint` = first 8 hex of SHA-256.
2. **Never log or persist a full key.** Logs, DB rows and admin pages show only
   `#index` and `fingerprint`. A key in a log is a published key.
3. **Pick keys from a random start offset**, skipping cooling keys. A random start gives
   the same statistical spread as a shared cursor with zero reads/writes.
4. **Cool down by reason**, not a fixed time:
   | Reason | Cooldown |
   | --- | --- |
   | `rate_limited` (429 per-minute) | 60 s |
   | `daily_quota` | until the provider's real daily reset (e.g. midnight in the provider's timezone) + 1 min |
   | `auth` (401/403) | 24 h — it will not heal itself |
5. **Share cooldowns cheaply**: keep a local in-memory map (works inside one warm
   instance) plus one row-per-provider read from your DB/KV, cached ~10 s. Never one
   KV command per key per attempt — 40 dead keys × every attempt burns quota.
6. **Provider chain**: order providers by quota size and latency, paid last. Try up to
   N (≈4) keys per provider, then move on. If a provider returns 5xx/network error,
   **trip** it in memory for ~60 s so the next requests skip it.
7. **Classify errors once** in the adapter (`rate_limited | daily_quota | auth | server | bad_request`).
   Only key-related errors penalise the key; `bad_request` must not rotate (it will fail on every key).
8. Record telemetry per call: provider, key fingerprint, latency, outcome.

## Reference implementation (TypeScript)

```ts
import { createHash } from 'node:crypto'

export interface PooledKey { value: string; index: number; fingerprint: string; envName: string }
type Reason = 'rate_limited' | 'daily_quota' | 'auth'

export function collectKeys(prefix: string, env = process.env): PooledKey[] {
  const seen = new Set<string>(); const out: PooledKey[] = []
  const add = (envName: string) => {
    const v = env[envName]?.trim()
    if (!v || v.startsWith('<') || seen.has(v)) return
    seen.add(v)
    out.push({ value: v, index: out.length, envName,
      fingerprint: createHash('sha256').update(v).digest('hex').slice(0, 8) })
  }
  add(prefix); for (let n = 1; n <= 100; n++) add(`${prefix}_${n}`)
  return out
}

const cooling = new Map<string, number>() // `${provider}:${fp}` -> until(ms)
const SECONDS: Record<Reason, () => number> = {
  rate_limited: () => 60,
  daily_quota: () => secondsUntilMidnight('America/Los_Angeles') + 60,
  auth: () => 86_400,
}

export function penalise(provider: string, key: PooledKey, reason: Reason) {
  cooling.set(`${provider}:${key.fingerprint}`, Date.now() + SECONDS[reason]() * 1000)
  console.warn(`[keys] ${provider} #${key.index} (${key.fingerprint}) cooled: ${reason}`)
}

export function nextKey(provider: string, pool: PooledKey[]): PooledKey | null {
  const start = Math.floor(Math.random() * pool.length)
  for (let i = 0; i < pool.length; i++) {
    const k = pool[(start + i) % pool.length]
    if ((cooling.get(`${provider}:${k.fingerprint}`) ?? 0) <= Date.now()) return k
  }
  return null
}

function secondsUntilMidnight(timeZone: string, now = new Date()): number {
  const p = new Intl.DateTimeFormat('en-US', { timeZone, hour: 'numeric', minute: 'numeric',
    second: 'numeric', hourCycle: 'h23' }).formatToParts(now)
  const g = (t: string) => Number(p.find((x) => x.type === t)?.value ?? 0)
  return 86_400 - (g('hour') * 3600 + g('minute') * 60 + g('second'))
}
```

The outer loop: `for provider of chain → skip if tripped → up to 4 × nextKey → call →
on key error penalise + continue; on server error trip provider + break → next provider`.
Throw one aggregated error listing `provider#index:reason` when everything fails.

## Checklist
- [ ] No full key in logs, errors, DB, or client responses
- [ ] Unit test: gaps in env numbering, duplicates, placeholders
- [ ] Unit test: all keys cooling → `null`, falls through to next provider
- [ ] Daily-quota cooldown matches the provider's real reset time
- [ ] Paid provider is last and guarded by a budget (see `layered-api-shield`)
