---
name: persona-voice-prompt
description: Create one reusable brand-voice prompt module (e.g. casual Indonesian Gen Z, "bahasa gaul") shared by every AI writer — articles, social captions, replies — while keeping factual and compliance rules separate and non-negotiable. Use when the user wants a consistent tone across AI-generated content, asks to make AI copy sound more casual/local/Gen Z, or when style instructions keep conflicting with accuracy rules.
---

# Persona voice prompt module

Put the voice in **one module** imported by every writer prompt. Two variants are
usually enough: long-form (articles) and social (captions/replies).

## Principle: style is flexible, substance is not
The voice controls *how* things are said. Fact rules (numbers only from source),
compliance rules (no buy/sell calls, no profit promises) and safety rules live in the
task prompt **and** in code checks (`llm-fact-guard`). The voice block must restate that
it never overrides them.

## Template

```ts
export const ARTICLE_VOICE = `VOICE (REQUIRED):
- Audience: <who>, addressed as "<you-form>". Short sentences, 2–4 sentence paragraphs.
- Explain technical terms right after using them, with everyday analogies (<local examples>).
- Slang sparingly: max 1–2 per paragraph from this list: <list>. Never forced.
- Headings may be casual questions. No emoji in body. No ALL CAPS shouting.
- FORBIDDEN: hype/transaction slang (<list>), mocking people or victims.
- Sad news (layoffs, deaths, losses): empathetic, less slang.
- SEO keyword stays at the front of the title.`

export const SOCIAL_VOICE = `VOICE (REQUIRED):
- Like telling a friend: <slang list>. Short, punchy; meme formats OK ("me: …, market: …").
- Facts and numbers exactly from the source — the humor is in the telling, not in invention.
- FORBIDDEN: <hype/transaction slang>, mocking people.`
```

### Example values — Indonesian Gen Z
- you-form: "kamu" (not "Anda"), "kita" for the editorial voice
- analogies: dompet digital, langganan streaming, harga kopi susu, tiket konser, gaji UMR, paylater
- slang: jujurly, literally, vibes, plot twist, red flag, spill, relate, valid, FOMO, overthinking, no debat, kerasa banget, nah kan, gaes, cuy
- forbidden (finance): "gas beli", "serok", "auto cuan", "to the moon", "HODL", "all in", "pom-pom"

## Steps
1. Write the module with the two variants; keep each under ~15 lines (it is sent with every call).
2. Import it into every writer prompt — never copy-paste the text.
3. Enforce the forbidden list in code too (regex check after generation).
4. Review 10 generated samples; tune the slang cap if it reads cringe.

## Checklist
- [ ] Single source of truth for voice
- [ ] Forbidden words checked in code, not just in the prompt
- [ ] Empathy rule for sensitive news
