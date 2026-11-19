---
name: llm-fact-guard
description: Guard LLM output against invented numbers, overconfidence, and forbidden phrases using deterministic rules first and a second model that may only downgrade. Use when an LLM writes reports, verdicts, financial/medical/news content, or summaries from source data and you need to catch hallucinated figures, cap confidence, or block banned wording before publishing.
---

# LLM fact guard

A valid JSON verdict from one model is not a checked verdict. This skill adds two layers
**before** anything is saved or published.

## Layer 1 — deterministic rules (no model, cannot be rate-limited or persuaded)

### a. Number audit
Every number in the output must trace back to the source/evidence block.
- Compare digit sequences ignoring separators: `11,57`, `11.57`, `-11,57%` are the same; `18.027` = `18027`.
- Strip trailing decimal zeros (`3,50` = `3,5`).
- Skip trivial tokens: ≤ 2 digits and 4-digit years.
- Accept rounding: `12,3%` matches `12,34` in the facts (tolerance ≈ 0.5 %).
- Locale ambiguity: `1.234` may be 1234 or 1.234 — try every reading.

```ts
const NUMBER = /\d[\d.,]*\d|\d/g
function digits(t: string) {
  const [whole, frac] = t.split(/[.,](?=\d{1,2}$)/)
  return (whole.replace(/[.,]/g, '') + (frac ? frac.replace(/0+$/, '') : '')).replace(/^0+(?=\d)/, '')
}
const trivial = (t: string) => t.replace(/[.,]/g, '').length <= 2 || /^(19|20)\d{2}$/.test(t)

export function unknownNumbers(text: string, evidence: string): string[] {
  const known = new Set([...evidence.matchAll(NUMBER)].map((m) => digits(m[0])))
  const out = new Set<string>()
  for (const m of text.replace(/<[^>]*>/g, ' ').matchAll(NUMBER))
    if (!trivial(m[0]) && !known.has(digits(m[0]))) out.add(m[0])
  return [...out]
}
```

### b. Banned words
A regex list of phrases that must never appear in public output (e.g. direct buy/sell
calls, guaranteed-profit claims, medical prescriptions). Return the matches.

### c. Hard caps
Rules such as "history shorter than N data points → confidence < 40".

## Layer 2 — second model from a different family
Send the output, the evidence, and Layer-1 findings. Ask for strict JSON:
`{ "supported": boolean, "max_confidence": 0-100, "issues": string[] }`.

Parse defensively: strip ```` ```json ```` fences, slice from first `{` to last `}`,
remove trailing commas, coerce `"true"`/`"72"` strings, validate with zod. Broken JSON
= unreachable.

## Policy: the checker may only DOWNGRADE
- Lower confidence, or turn a directional verdict into neutral/"hold".
- **Never** raise confidence or flip direction. Two models disagreeing means the
  evidence is balanced, not that the second model is right.
- Checker unreachable (quota gone) → cap confidence (e.g. 60). Disputed → cap lower (e.g. 40).
- Banned words / untraced numbers → regenerate once, else strip or downgrade.

Keep the policy a **pure function** (no I/O) so it is unit-testable without calling a model.

## Checklist
- [ ] Facts gathered deterministically before any model call
- [ ] Number audit + banned words run on every output
- [ ] Checker uses a different model family than the writer
- [ ] Merge function is pure and only lowers
- [ ] Tests cover locale formats, rounding, years, fenced JSON
