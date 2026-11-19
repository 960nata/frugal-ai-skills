---
name: social-autopost-pipeline
description: Build an automated pipeline that turns published articles into Threads posts and Instagram carousels via the official Graph APIs — AI-rewritten captions within platform limits, slot-based scheduling with a content mix, post-once dedupe, long-lived token refresh, and kill switches. Use when auto-posting to Threads or Instagram, generating carousel slides, scheduling social content, or debugging Graph API publish errors.
---

# Social auto-post pipeline (Threads + Instagram)

## Shared design
- **Post once per item**: a log table (`threads_post`, `ig_post`) with a unique key on
  the source item id. Insert-or-skip before calling the API.
- **Kill switch per platform**: `THREADS_DISABLED=1`, `IG_DISABLED=1` disables without
  deleting tokens.
- **Long-lived tokens last 60 days**: store the current token in a DB table and refresh
  it weekly with a scheduled job; env holds only the bootstrap token.
- **User ID is the numeric ID** from `/me`, never the @username.
- Caption written by a small/cheap model with the brand voice (see `persona-voice-prompt`)
  and run through `llm-fact-guard` (numbers must come from the article).

## Threads
- Max **500 chars including the link**. Trim the caption, not the URL.
- Only **one topic tag** per post — send it via `topic_tag`, not extra `#` in the text.
- Use `link_attachment` for the article URL; the card image comes from the article's
  Open Graph tags — no need to upload media.
- Mentions only from an allowlist env (`THREADS_MENTIONS`), never model-invented handles.
- Flow: `POST /{user-id}/threads` (create container) → `POST /{user-id}/threads_publish`.

## Instagram carousel
1. Render slides to **JPEG** (e.g. `next/og`/Satori → PNG → JPEG). IG only fetches JPEGs
   from **public URLs**.
2. Serve slides from your own site (temporary DB table + route) if object storage may be
   locked by egress quota — your domain is the most reliable host.
3. One container per image with `is_carousel_item=true`, then a `CAROUSEL` container with
   `children` + `caption`.
4. Poll container `status_code` until `FINISHED` (with timeout), then `media_publish`.
5. Clean up temporary slides after publish.

## Scheduling
- Fixed slots per day (e.g. 6 slots) defined in **one constant** that must match the cron
  schedule; comment the UTC↔local mapping.
- Content mix as a repeating pattern, e.g. `['news','edu','meme','news','edu']` cycled
  from an epoch date, so every 5 consecutive slots have the exact ratio and the meme hour
  shifts day to day.
- Admin override: a slot table where an editor pins a specific item; pinned = news.
- Out of fresh items → pick the closest related item (same ticker/category), don't skip.

## Debugging
- `(#10) Application does not have permission` → missing scope or app not live.
- Container stuck `IN_PROGRESS` → image URL not public / not JPEG / too large.
- `Invalid OAuth access token` → token expired; check the refresh job.
- Use the diagnostics endpoint to print: token age, user id, last 5 publish results.

## Checklist
- [ ] Unique dedupe key + kill switches
- [ ] Token refresh job + alert before day 60
- [ ] Text length and single-tag rules enforced in code, not prompt only
- [ ] Slot constant matches cron schedule
