# frugal-ai-skills

Agent skills for building **AI products that stay cheap, safe, and alive on free tiers
and serverless**. Each skill is a battle-tested pattern pulled from a production
Next.js + LLM app that runs multi-agent analysis, auto-publishing, and video rendering
on quota-limited infrastructure.

Works with any agent that reads the open **Agent Skills** format (`SKILL.md` with YAML
frontmatter): Claude Code, OpenAI Codex (GPT), Google Antigravity, Gemini CLI, and others.

## Skills

| Skill | What it gives you |
| ----- | ----------------- |
| [`llm-key-rotation`](skills/llm-key-rotation/SKILL.md) | Pool many API keys, rotate evenly, cool down on 429/quota/auth, fall back across providers |
| [`fetch-timeout-guard`](skills/fetch-timeout-guard/SKILL.md) | "No naked fetch": one wrapper with timeout + signal + readable errors |
| [`layered-api-shield`](skills/layered-api-shield/SKILL.md) | Cheapest-first defense: free shield → blocklist → 1-command rate limit → global AI budget |
| [`llm-fact-guard`](skills/llm-fact-guard/SKILL.md) | Catch invented numbers & banned words; second model may only downgrade |
| [`multi-agent-council`](skills/multi-agent-council/SKILL.md) | Role-based agent committee with independence rules, stale-data abstain, resumable turns |
| [`serverless-cron-dispatcher`](skills/serverless-cron-dispatcher/SKILL.md) | Many scheduled jobs from one hourly cron, timezone-aware, idempotent leases |
| [`html-allowlist-sanitizer`](skills/html-allowlist-sanitizer/SKILL.md) | Allowlist sanitizer for LLM/scraped HTML + XSS payload test suite |
| [`audio-lipsync-frames`](skills/audio-lipsync-frames/SKILL.md) | Loudness-driven 3-mouth lip-sync for Remotion / frame renderers |
| [`social-autopost-pipeline`](skills/social-autopost-pipeline/SKILL.md) | Threads + Instagram carousel auto-posting with dedupe, slots, token refresh |
| [`persona-voice-prompt`](skills/persona-voice-prompt/SKILL.md) | One shared brand-voice module (e.g. Indonesian Gen Z) that never overrides fact rules |

## Install

```bash
git clone https://github.com/<you>/frugal-ai-skills.git
cd frugal-ai-skills
./install.sh            # all detected agents (symlinks, so `git pull` updates them)
./install.sh claude     # or: codex | antigravity | gemini
./install.sh --copy all # copy instead of symlink
./install.sh --project /path/to/repo   # install into one project only
```

| Agent | Global folder | Per-project folder |
| ----- | ------------- | ------------------ |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| OpenAI Codex (GPT) | `~/.codex/skills/` | `.agents/skills/` |
| Google Antigravity | `~/.gemini/antigravity/skills/` | `.agent/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` |

> Agent folders change between releases. If a skill does not show up, check your
> agent's docs for the current skills path and symlink `skills/<name>` there.

**Claude.ai / ChatGPT (web):** zip a single skill folder (`cd skills && zip -r llm-key-rotation.zip llm-key-rotation`)
and upload it where your plan supports custom skills. Or paste the `SKILL.md` content
into a project's instructions.

**Any other agent:** point it at [`AGENTS.md`](AGENTS.md), which lists every skill and when to load it.

## Usage

You don't call skills by hand. Ask normally — the agent loads a skill when its
`description` matches:

- "My Gemini keys keep hitting 429, I have 12 of them" → `llm-key-rotation`
- "Add rate limiting, Upstash quota keeps running out" → `layered-api-shield`
- "The AI summary invents numbers" → `llm-fact-guard`
- "Make my mascot talk in the Remotion video" → `audio-lipsync-frames`

## Contributing

1. One folder per skill in `skills/`, kebab-case, folder name = `name:` in frontmatter.
2. `description` says **what** it does and **when** to use it (trigger words matter).
3. Keep `SKILL.md` under ~150 lines; move long material to `references/` in the same folder.
4. No secrets, account IDs, or company-specific names. Run `./scripts/validate.sh` before a PR.

---

## Bahasa Indonesia

Kumpulan skill agent AI untuk bikin produk AI yang **hemat kuota, aman, dan tahan banting**
di tier gratis/serverless. Semua diambil dari pola yang sudah jalan di production.
Bisa dipakai di Claude Code, Codex (GPT), Antigravity, dan Gemini CLI. Pasang dengan
`./install.sh`, lalu minta ke agent seperti biasa — skill dimuat otomatis kalau cocok.

## License

MIT — see [LICENSE](LICENSE).
