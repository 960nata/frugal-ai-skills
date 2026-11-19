# frugal-ai-skills — index for agents

This repository is a collection of agent skills. Each lives in `skills/<name>/SKILL.md`.
When a task matches a trigger below, read that SKILL.md fully and follow its steps and checklist.

| Skill | Load when the task involves |
| ----- | --------------------------- |
| `skills/llm-key-rotation` | multiple LLM API keys, 429 / quota errors, provider fallback |
| `skills/fetch-timeout-guard` | outbound HTTP calls, hanging functions, serverless timeouts |
| `skills/layered-api-shield` | rate limiting, bot/scanner blocking, Redis/KV quota drain, AI spend caps |
| `skills/llm-fact-guard` | hallucinated numbers, verdict/confidence checking, banned wording |
| `skills/multi-agent-council` | multi-agent debate, committees, sequential role agents |
| `skills/serverless-cron-dispatcher` | cron limits, scheduled jobs, idempotent retries, timezones |
| `skills/html-allowlist-sanitizer` | rendering LLM/scraped HTML, XSS, dangerouslySetInnerHTML |
| `skills/audio-lipsync-frames` | talking characters, lip-sync, Remotion narration |
| `skills/social-autopost-pipeline` | Threads / Instagram auto-posting, carousels, Graph API |
| `skills/persona-voice-prompt` | brand voice, casual/Gen Z tone, consistent AI copy |

Rules when applying any skill:
- Adapt code to the project's existing stack and conventions; do not add dependencies the skill does not require.
- Never write secrets or full API keys to code, logs, or commits.
- Run the skill's checklist before declaring the task done.
