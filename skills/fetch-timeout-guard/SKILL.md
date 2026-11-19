---
name: fetch-timeout-guard
description: Enforce a "no naked fetch" rule — every outbound HTTP call goes through one wrapper with a timeout, caller-signal propagation, and an error message naming who did not answer. Use when writing or reviewing code that calls external APIs from serverless functions, cron jobs, or scripts, or when a function hangs, times out on Vercel/Lambda, or a job queue keeps retrying a stuck batch.
---

# Fetch timeout guard

Built-in `fetch` has no timeout. When the remote side never answers (blocked exchange,
overloaded model provider) the call hangs forever: locally the script spins silently,
on serverless one hung call eats the whole function duration, and in a job queue the
same batch is retried endlessly.

**Rule: no bare `fetch` anywhere.** Everything goes through `fetchWithTimeout`.

## Implementation

```ts
export const DATA_TIMEOUT_MS = 12_000   // data APIs usually answer < 1 s
export const MODEL_TIMEOUT_MS = 60_000  // LLMs are slower, esp. long outputs

export class TimeoutError extends Error {
  constructor(label: string, ms: number) {
    super(`${label} did not answer within ${Math.round(ms / 1000)} s`)
    this.name = 'TimeoutError'
  }
}

export interface TimedFetchOptions extends RequestInit { label: string; timeoutMs?: number }

export async function fetchWithTimeout(
  url: string,
  { label, timeoutMs = DATA_TIMEOUT_MS, signal, ...init }: TimedFetchOptions,
): Promise<Response> {
  const timeout = AbortSignal.timeout(timeoutMs)
  // Respect the caller's signal: a cancelled job must cancel its outbound calls.
  const combined = signal ? AbortSignal.any([signal, timeout]) : timeout
  try {
    return await fetch(url, { ...init, signal: combined })
  } catch (err) {
    if (timeout.aborted) throw new TimeoutError(label, timeoutMs)
    throw err
  }
}
```

## Apply it

1. Add the wrapper in one shared module (e.g. `lib/http/fetch.ts`).
2. Find every bare call: `grep -rnE "(^|[^.a-zA-Z])fetch\(" --include=*.ts src lib app`.
3. Replace each with `fetchWithTimeout(url, { label: 'ProviderName', ... })`.
   Use `MODEL_TIMEOUT_MS` for LLM calls.
4. Optional: add an ESLint `no-restricted-globals` rule for `fetch` with a message
   pointing to the wrapper, allowing it only inside the wrapper file.
5. Keep the timeout **below** the platform's function limit so your code — not the
   platform — decides what happens (log, mark batch `partial`, fall back).

## Notes
- `label` must be human-readable; it is what shows up in alerts.
- Do not retry inside the wrapper. Retries belong to the caller, which knows whether
  the operation is idempotent.
- Requires Node 20+ (`AbortSignal.any`). For older runtimes, merge signals manually.
