---
name: multi-agent-council
description: Design a multi-agent "committee" where role-based LLM agents take turns over one shared fact sheet, stay independent instead of echoing each other, abstain on stale data, and persist each turn so a serverless timeout can resume mid-meeting. Use when building multi-agent debate, review panels, investment/analysis committees, or any sequential agent discussion that ends in a verdict.
---

# Multi-agent council

Several agents speak in turn over **one** fact block. Each turn sees the full transcript,
so the strategist really answers the analyst and the chair really weighs the risk
officer — not four separate opinions glued together.

## Design rules

1. **Facts first, deterministically.** Gather and compute all metrics in plain code
   (pure functions, no I/O inside the math) before any model call. Format them as one
   `FACTS:` block reused by every seat.
2. **Feasibility gate before spending tokens.** If data is stale (e.g. > 2 days for
   24/7 markets, > 5 days for weekday markets) or too short (< 30 points), stop and
   return `abstain`. A meeting on stale data does not produce a worse verdict — it
   produces one that looks *just as convincing* as a correct one.
3. **Roles are data**, not code: `{ id, name, model preference, system prompt, output contract }`.
   Typical: analyst → strategist → risk officer → chair (verdict JSON).
4. **Independence rule** appended to every seat (models trained to be agreeable will
   otherwise copy the first seat, including its wrong numbers):
   ```
   INDEPENDENT:
   - Judge from evidence, not from colleagues. A claim without a number or source is unproven.
   - Agree only with evidence: "AGREE: <evidence>". If a colleague is wrong: "CORRECTION: <claim> → <what the data says>".
   - Blunt, no flattery; attack the claim, not the speaker. Correct beats polite.
   - Not enough data → say so. Do not force a conclusion.
   ```
5. **Persist per turn.** Write each message to the DB as soon as it is produced
   (`session_id, turn, role, content`). On retry (queue redelivery, timeout), load the
   transcript and continue from the next missing turn — never re-run paid turns.
6. **Reuse sessions**: before opening a new meeting for the same subject/day, look for a
   reusable open/finished session.
7. **Chair output is structured** (zod schema): verdict, confidence 0–100, rationale,
   key risk, invalidation condition. Then run it through `llm-fact-guard`.
8. **Mix model families** across seats where possible; the checker must differ from the chair.

## Skeleton

```ts
const facts = await gatherFacts(subject)          // deterministic
if (facts.staleDays > STALE_LIMIT[facts.market]) return abstain('stale data')

const session = await findReusable(subject) ?? await openSession(subject)
const transcript = await loadTranscript(session.id)

for (const role of COUNCIL.slice(transcript.length)) {
  const reply = await complete({
    system: `${role.prompt}\n\n${INDEPENDENT}`,
    messages: [{ role: 'user', content: factsToPrompt(facts) }, ...asMessages(transcript)],
  })
  await recordMessage(session.id, role.id, reply)   // persist before next turn
  transcript.push({ role: role.id, content: reply })
}
const verdict = guardVerdict(parseChair(transcript.at(-1)!), facts)
await closeSession(session.id, verdict)
```

## Checklist
- [ ] Gate runs before the first token is spent
- [ ] Every turn persisted immediately; resume tested by killing mid-run
- [ ] Independence rule in every seat prompt (keep it short — it is sent every turn)
- [ ] Chair JSON validated + guarded
