# pm-daily-brief

A Claude Code skill that generates a personalized daily intelligence brief for AI product managers. Scrapes ProductHunt, GitHub Trending, and Hacker News, filters for AI-relevant signals, weighs by your project context, and writes a Chinese-language Markdown + PDF report.

> **Built for one user's workflow.** Hardcoded paths point to a specific project root. Fork and adapt before running on your machine — see [Customization](#customization).

## What you get

- **Top 3-5 deep dive** — high-signal items with PM-perspective takeaways (not engineer-perspective)
- **Full AI-core list** — every AI item from all three sources with one-line commentary
- **High-heat non-AI** — breakthrough items outside AI worth knowing
- **"Today's observation"** — the most valuable section: cross-source pattern recognition synthesized for your specific work
- **Personalized weighting** — reads your recent briefs + project files to favor topics you actually work on
- **Markdown + PDF** double output

See `examples/2026-05-06.md` and `examples/2026-05-06.pdf` for a real run. The PDF is the polished artifact; the Markdown is for piping into other tools.

## How it works

8-step workflow (full detail in `SKILL.md`):

1. Resolve today's date + output path
2. Parallel WebFetch from PH leaderboard / GH trending / HN front page
3. **(2.5) Build user interest profile** — scan recent `daily-input/*.md` and project-root `*.md`/`*.html` files to identify themes you care about
4. AI filter + bucketing (AI core / high-heat non-AI / discard)
5. Top 3-5 selection — weighted by user profile, hard rule "don't pad to 5"
6. Synthesis: "today's observation" cross-source pattern recognition
7. Write Markdown
8. Generate PDF via gstack `make-pdf`

Three design principles worth calling out:

- **Never fabricate.** If a source fails, mark it failed in the report. No hallucinated PH entries.
- **Weighting is a tie-breaker, not a filter.** Breakthrough non-weighted signals (new model release, policy event) always make it in.
- **Don't pad Top 5.** If only 3 items have real PM takeaways, write 3. Padding produces vacuous "this could disrupt the industry" prose.

## Install

This is a Claude Code user-level skill. Drop into your skills directory:

```bash
git clone https://github.com/<your-username>/pm-daily-brief-skill.git ~/.claude/skills/pm-daily-brief
```

Or on Windows PowerShell:

```powershell
git clone https://github.com/<your-username>/pm-daily-brief-skill.git "$HOME\.claude\skills\pm-daily-brief"
```

Restart Claude Code to register the skill.

### Dependencies

- **Claude Code** with WebFetch tool
- **gstack** (for `/browse` fallback + `make-pdf`): https://github.com/garrytan/gstack
  - Install: `git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup --team`
  - Required for PDF output. If you don't want PDF, the skill will still produce Markdown.

## Use

In Claude Code, just say one of:

- `/pm-daily-brief`
- 今天有啥新的
- AI 日报
- 刷一下今天
- 行业热点
- daily brief
- 扫一下今天

The skill takes ~2-3 minutes (parallel WebFetch + 1-2 deep-read fetches + writing).

## Customization

The skill is hardcoded for one user's workflow. To adapt:

### 1. Output path

`SKILL.md` Step 1, hardcoded `<your-project>/daily-input/`. Replace with your actual project root.

### 2. User-context weighting source (Step 2.5)

`SKILL.md` Step 2.5. The skill reads:

- Recent files in `<your-project>/daily-input/*.md` (your own past reports)
- Top-level files in `<your-project>/*.md` and `<your-project>/*.html` (your project descriptions)

Change the glob roots to wherever your work lives. Don't point at `node_modules/` or anything with secrets — the skill explicitly skips `.git/`, `.env`, credentials.

### 3. Output language

The skill writes Chinese with English technical terms. To switch to English: rewrite the "写作风格" section in `SKILL.md` and the output template's headers.

### 4. PDF binary path (Windows-specific)

`SKILL.md` Step 7 hardcodes a Windows path to the gstack `browse.exe` (e.g. `C:\Users\<you>\.claude\skills\gstack\browse\dist\browse.exe`). Replace with your username, or modify to detect dynamically.

### 5. Sources

Currently PH + GH + HN. To add HN/Reddit/Twitter/arxiv, add a parallel WebFetch in Step 2 with appropriate URL + extraction prompt.

## Why a skill, not a script

A Python script could scrape these three sources. Why is this a Claude skill?

- **Synthesis is the value, not scraping.** "Today's observation" cross-source pattern recognition + per-item PM takeaway requires LLM reasoning, not regex.
- **User-context weighting** requires reading arbitrary project files and inferring intent — also LLM reasoning.
- **Maintainability**: when ProductHunt changes its leaderboard URL (which it did between two runs while I was building this), I tell the skill in plain English to fall back to homepage. With a script, I'd debug a selector for an hour.

The fundamental bet: **for personalized synthesis tasks, prompt + LLM beats scraper + template, even at the cost of 2-3 min per run.**

## Known limits

- **3-min runtime**, dominated by WebFetch latency on three parallel sources + 1-2 deep-read fetches.
- **PH anti-bot**: `/browse` (headless Chromium) gets blocked by Cloudflare on PH. Skill uses WebFetch on the leaderboard URL as primary path. Fallback to homepage if leaderboard is empty.
- **GH WebFetch occasionally truncates** — sometimes only get 15/25 trending repos. Acceptable; skill notes the count in the header.
- **No cross-day comparison.** Each report stands alone; trend tracking is out of scope (would require structured storage). The user explicitly didn't want it for v1.
- **Single user.** Hardcoded paths. Adapt as documented above before running on your machine.

## Development

The skill was built using Anthropic's [skill-creator](https://github.com/anthropics/skills/tree/main/skill-creator). Iteration log:

- **v1**: Initial draft. Used `/browse` as primary scraper. PH 403'd. GH text was 99% navigation noise.
- **v2** (current): Switched primary to WebFetch. Added Step 2.5 user-context weighting (single biggest improvement — turned a generic RSS digest into a personalized brief). Top 5 became Top 3-5 with "don't pad" hard rule. Removed Step 1 file-overwrite confirmation (over-engineering). Added PDF generation.

Open issues / next iterations:
- TL;DR section at top (60-120 字 hard summary for 30-second reads)
- Full-list deduplication (Top items appearing again in full list with `→ see Top #N` is already there, but could be tighter)
- Cross-day theme tracking (out of scope per user)
- Sanity-check thresholds for source counts (warn if GH < 15 or PH < 10, indicating scrape failure vs genuinely quiet day)

## License

MIT. See `LICENSE`.

## Credit

- Skill scaffolding: [Anthropic skill-creator](https://github.com/anthropics/skills)
- Browser + PDF tooling: [gstack](https://github.com/garrytan/gstack)
- Skill review process inspired by skill-creator's eval loop
