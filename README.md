# pm-daily-brief

A Claude Code skill that generates a personalized daily intelligence brief for AI product managers. Scrapes ProductHunt, GitHub Trending, and Hacker News, filters for AI-relevant signals, weighs by your project context, and writes a Chinese-language Markdown + PDF report.

> **Built for one user's workflow.** Hardcoded paths point to a specific project root. Fork and adapt before running on your machine — see [Customization](#customization).

## What you get

- **⚡ TL;DR (60-second read)** — opportunity / risk / trend lines at the top so coffee-time triage is possible without scrolling
- **Top 3-5 deep dive** — high-signal items with PM-perspective takeaways + a `🎯 今天具体动作` field (1-2h actionable next step) when applicable
- **Full AI-core list** — every AI item from all three sources, each tagged `[跟进 / 关注 / 噪音]` for 10-second triage
- **High-heat non-AI** — breakthrough items outside AI worth knowing
- **"Today's observation"** — cross-source pattern recognition synthesized for your specific work (3 paragraphs, punchline + evidence + impact)
- **Personalized weighting** — reads your recent briefs + project files to favor topics you actually work on; surfaces the inferred profile in the report header so you can correct it
- **Markdown + PDF** double output, with elapsed scrape time in footer

See `examples/2026-05-06.md` and `examples/2026-05-06.pdf` for a real run. The PDF is the polished artifact; the Markdown is for piping into other tools.

## How it works

9-step workflow (full detail in `SKILL.md`):

1. Resolve today's date + output path; record `START_TS`
2. Parallel WebFetch from PH leaderboard / GH trending / HN front page (multi-URL fallback per source — `/browse` is no longer a fallback because PH blocks Cloudflare)
3. **(2.5) Build user interest profile** — scan recent `daily-input/*.md` and project-root `*.md`/`*.html` files to identify themes you care about; surface to header
4. AI filter + bucketing (AI core / high-heat non-AI / discard)
5. Top 3-5 selection — weighted by user profile, hard rule "don't pad to 5", each item gets a `🎯 今天具体动作` when there's a concrete 1-2h next step
6. **(4.5) TL;DR synthesis** — 3 lines: 🚀 opportunity / ⚠️ risk / 🌊 trend
7. Full-list triage — every item tagged `[跟进 / 关注 / 噪音]` based on weight match + signal strength
8. "Today's observation" — 3 paragraphs, each with bold punchline + 2 evidence sentences + impact
9. Write Markdown → generate PDF via gstack `make-pdf` → report elapsed time

Five design principles worth calling out:

- **Never fabricate.** If a source fails, mark it failed in the report. No hallucinated PH entries.
- **Weighting is a tie-breaker, not a filter.** Breakthrough non-weighted signals (new model release, policy event) always make it in.
- **Don't pad Top 5.** If only 3 items have real PM takeaways, write 3. Padding produces vacuous "this could disrupt the industry" prose.
- **Don't fake actions.** `🎯 今天具体动作` is omitted (not filled with "持续关注") if there's no concrete 1-2h next step.
- **Risk = threat, not muted opportunity.** TL;DR `⚠️` line must be threat-framed (what could disrupt you), not optimistic spin.

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

As of v3, `SKILL.md` Step 8 self-detects the gstack binary path via `$HOME` (bash) or `$HOME` (PowerShell) and resolves the Windows-style path via `cygpath`. No username hardcoding.

If your gstack install lives somewhere other than `~/.claude/skills/gstack/`, edit the `P=` and `BROWSE_BIN=` lines in Step 8.

### 5. Sources

Currently PH + GH + HN. To add HN/Reddit/Twitter/arxiv, add a parallel WebFetch in Step 2 with appropriate URL + extraction prompt.

## Why a skill, not a script

A Python script could scrape these three sources. Why is this a Claude skill?

- **Synthesis is the value, not scraping.** "Today's observation" cross-source pattern recognition + per-item PM takeaway requires LLM reasoning, not regex.
- **User-context weighting** requires reading arbitrary project files and inferring intent — also LLM reasoning.
- **Maintainability**: when ProductHunt changes its leaderboard URL (which it did between two runs while I was building this), I tell the skill in plain English to fall back to homepage. With a script, I'd debug a selector for an hour.

The fundamental bet: **for personalized synthesis tasks, prompt + LLM beats scraper + template, even at the cost of 2-3 min per run.**

## Known limits

- **2-3 min runtime**, dominated by WebFetch latency on three parallel sources + 1-2 deep-read fetches. Footer reports actual elapsed time so you can decide whether daily cost is worth it.
- **WebFetch quota**: 3 parallel + 1-2 deep-reads + occasional WebSearch ≈ 5-7 net calls. Multiple runs same day may hit limits; skill skips supplemental fetches and ships partial.
- **PH anti-bot**: `/browse` (headless Chromium) gets blocked by Cloudflare on PH and is **not** a fallback path. Skill uses WebFetch on leaderboard URL as primary, falls back to homepage, then to previous-day leaderboard.
- **GH WebFetch occasionally truncates** — sometimes only get 15/25 trending repos. Skill notes the count in the header; falls back to `/trending` (no `since=`) and `/trending?since=weekly` if truncation is severe.
- **No cross-day comparison.** Each report stands alone; trend tracking is out of scope (would require structured storage). The user explicitly didn't want it for v1.
- **Single user.** Output path is hardcoded (`D:\hrdai\daily-input\`). Binary paths self-detect via `$HOME` / `cygpath` — no more `C:\Users\<name>` hardcoding. Adapt the output path as documented before running on your machine.

## Development

The skill was built using Anthropic's [skill-creator](https://github.com/anthropics/skills/tree/main/skill-creator). Iteration log:

- **v1**: Initial draft. Used `/browse` as primary scraper. PH 403'd. GH text was 99% navigation noise.
- **v2**: Switched primary to WebFetch. Added Step 2.5 user-context weighting (single biggest improvement — turned a generic RSS digest into a personalized brief). Top 5 became Top 3-5 with "don't pad" hard rule. Removed Step 1 file-overwrite confirmation (over-engineering). Added PDF generation.
- **v3** (current): PM-experience pass.
  - Added **TL;DR 60-second section** (🚀 opportunity / ⚠️ risk / 🌊 trend) so the report is usable in coffee-time without scrolling.
  - Added **`🎯 今天具体动作`** field on Top items — every PM takeaway must come with a 1-2h actionable next step or the field is omitted (no "持续关注" filler).
  - Added **`[跟进 / 关注 / 噪音]` triage tags** on every full-list item — 36 AI items become 10-second scannable.
  - **Surfaced user-profile weighting** in report header (was internal-only) so the PM can correct the inferred profile when wrong.
  - Replaced **`/browse` fallback** (known-broken on PH due to Cloudflare) with **multi-URL fallback** (homepage / previous-day / weekly) per source.
  - **De-hardcoded user paths** — Step 8 now uses `$HOME` / `cygpath` / `$USERPROFILE` self-detection for cross-machine portability.
  - Added **bash + PowerShell parallel commands** for PDF step.
  - Added **scrape elapsed time** to footer.
  - Tightened **observation section** to 3 paragraphs (was 4) — 4 dilutes signal.
  - Added **risk-framing rule** for TL;DR `⚠️` line — must be threat-framed, not optimistic spin.

Open issues / next iterations:
- **Cross-day theme tracking** (out of scope per user; may revisit if v3 makes the value obvious enough)
- **Sanity-check thresholds** for source counts (warn if GH < 15 or PH < 10, indicating scrape failure vs genuinely quiet day)
- **Auto-link to product canonical URL** vs PH/HN detail page (Step 2 prompt asks for canonical, but PH detail-page extraction is fragile — sometimes returns `/products/<slug>` instead)
- **`[跟进]` budget enforcement** — currently soft "≤ 5 per source", no hard fail if violated

## License

MIT. See `LICENSE`.

## Credit

- Skill scaffolding: [Anthropic skill-creator](https://github.com/anthropics/skills)
- Browser + PDF tooling: [gstack](https://github.com/garrytan/gstack)
- Skill review process inspired by skill-creator's eval loop
