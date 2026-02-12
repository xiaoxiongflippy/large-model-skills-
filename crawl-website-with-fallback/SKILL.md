---
name: crawl-website-with-fallback
description: Crawl website pages to PDF/HTML/JSON with robust fallback handling and explicit blocked-result reporting. Use when a user asks to crawl or export web content from one or multiple URLs, especially if the site may trigger anti-bot checks, redirects, dynamic rendering, or partial-content failures.
---

# Crawl Website With Fallback

Use this skill to run deterministic crawl-export flows and classify outcomes as `success` or `blocked` with evidence.

## Workflow

1. Run baseline crawl with conservative settings.
2. Validate content quality in output JSON.
3. Retry with fallback settings when blocked or unstable.
4. Classify final status and report next actions.

## Step 1: Baseline Crawl

Use one-page baseline first:

```powershell
node src/index.js --url "<URL>" --output "output/<slug>.pdf" --strategy next --max-pages 1 --wait-ms 2000 --save-html --save-json
```

For section crawls:

```powershell
node src/index.js --url "<URL>" --output "output/<slug>.pdf" --strategy bfs --max-pages <N> --path-prefix "<PATH_PREFIX>" --save-html --save-json
```

## Step 2: Validate Output Quality

Mark as suspicious if any condition is true:

- `title` is empty or `Untitled`
- `textLen` very short for expected article pages (for example `< 250`)
- HTML/text contains block markers:
  - `403`, `captcha`, `verify`, `access denied`, `forbidden`
  - `请求异常`, `限制访问`, `机器人`, `人机验证`
- crawler logs include navigation/context reset errors

Quick check:

```powershell
@'
const fs=require('fs');
const d=JSON.parse(fs.readFileSync('output/<slug>.json','utf8'));
const p=d[0]||{};
const text=(p.text||'');
const merged=(p.html||'') + text;
const blocked = (p.title||'') === 'Untitled' ||
  text.length < 250 ||
  /403|captcha|verify|access denied|forbidden|请求异常|限制访问|机器人|人机验证/i.test(merged);
console.log({ blocked, title:p.title||'', textLen:text.length });
'@ | node
```

## Step 3: Fallback Retry

When suspicious/blocked, retry with stronger rendering:

```powershell
node src/index.js --url "<URL>" --output "output/<slug>-retry.pdf" --strategy next --max-pages 1 --wait-ms 5000 --show-browser --save-html --save-json
```

If still blocked and user wants full content, request login-cookie support or authenticated HTML input.

## Step 4: Report Status

Always report:

1. `status`: `success` or `blocked`
2. captured page count
3. output file paths
4. if blocked, concrete next options

## Output Naming Convention

- baseline: `output/<slug>.pdf|html|json`
- retry: `output/<slug>-retry.pdf|html|json`

## Git Publishing Notes

When preparing public repo docs, present crawler as domain-agnostic:

1. Mention supported modes: single article + section BFS.
2. Document known limits: anti-bot/captcha/paywall/login.
3. Show one successful sample and one blocked-case sample.
4. Explain ethical and legal compliance: respect robots.txt and site terms.

Do not claim guaranteed bypass of anti-bot systems.
