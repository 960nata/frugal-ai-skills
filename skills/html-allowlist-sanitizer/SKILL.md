---
name: html-allowlist-sanitizer
description: Sanitize LLM-generated or scraped HTML with a strict allowlist (tags, attributes, URL schemes) as a pure string function that behaves identically on server and browser. Use when rendering model output or third-party content via dangerouslySetInnerHTML / v-html / innerHTML, when AI writes articles from external pages (prompt-injection → XSS risk), or when reviewing an existing blocklist-based sanitizer.
---

# HTML allowlist sanitizer

An LLM that reads external pages can be told by those pages to emit
`<img src=x onerror=...>` or `<a href="javascript:...">`. If that output is rendered
with `dangerouslySetInnerHTML`, it is stored XSS on your public site.

**Blocklists always lose one step** (unquoted attributes, entities, a forgotten tag).
Use an allowlist: keep only listed tags and attributes, rebuild every attribute from a
checked value, and allow only safe URLs.

## Rules
1. **Drop with content**: `script, style, iframe, object, embed, noscript, template, svg,
   math, form, textarea, select, button, title, head, frameset, frame, applet`.
2. **Allowed tags** (example for articles): `h1–h6, p, br, hr, div, span, strong, b, em,
   i, u, s, mark, small, sub, sup, code, pre, blockquote, q, cite, ul, ol, li, a, img,
   table, thead, tbody, tr, th, td, figure, figcaption`.
   Anything else: remove the tag, keep its text.
3. **Allowed attributes per tag**: `a[href,title]`, `img[src,alt,width,height]`,
   `th/td[colspan,rowspan]`. **No** `on*`, `style`, `srcdoc`, `formaction`.
4. **URLs**: decode entities, trim, lowercase-check the scheme. Allow only `http:`,
   `https:`, and in-page `#anchors` (optionally relative `/paths`). Everything else → drop the attribute.
5. **Rewrite, don't copy**: output `name="escaped value"` built from the validated value,
   escaping `& < > " '`.
6. Force `rel="noopener noreferrer nofollow"` and `target="_blank"` on external links.
7. **Pure string, no DOM** — the same function runs on the server before saving and in
   the browser before rendering (defense in depth).

If adding a dependency is acceptable, `sanitize-html` or DOMPurify (+ jsdom on server)
with an explicit allowlist config is also fine — the rules above still define the config.

## Test cases (must all be neutralized)
```html
<img src=x onerror=alert(1)>
<a href="javascript:alert(1)">x</a>
<a href="&#106;avascript:alert(1)">x</a>
<a href=" JaVaScRiPt:alert(1)">x</a>
<svg><script>alert(1)</script></svg>
<iframe srcdoc="<script>alert(1)</script>"></iframe>
<p style="background:url(javascript:alert(1))">x</p>
<scr<script>ipt>alert(1)</script>
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">x</a>
```
And must survive: headings, lists, tables, `https` links, `#anchor` links, `<code>`.

## Checklist
- [ ] Sanitize at write time (DB) **and** render time
- [ ] Payload test suite above runs in CI
- [ ] No `style` attribute allowed
