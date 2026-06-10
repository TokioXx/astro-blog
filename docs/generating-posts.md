# Generating blog posts

This blog publishes three kinds of auto-generated posts, all written to `src/content/posts/` as Markdown and deployed to Cloudflare Pages (`astro-blog-a0m.pages.dev`) on every push to `main`.

| Post type | Slug pattern | Cadence | Source of data |
| --- | --- | --- | --- |
| HN daily digest | `hn-daily-YYYY-MM-DD` | daily | **HN front page — all 30 posts** (HN official API for ranking/scores + `news-aggregator` skill for article text) |
| Portfolio research | `portfolio-YYYY-MM-DD` | trading days only | `/trading-ideas:research` per ticker + live quotes |
| Watchlist | `watchlist-YYYY-MM-DD` | ad-hoc | `/trading-ideas:research` per ticker + live quotes |

The **HN digest** is generated automatically every day by a **local macOS launchd LaunchAgent** (`com.guodong.astro-blog-daily`): it runs Claude Code headlessly at **5 PM America/Los_Angeles**, regenerates `hn-daily-DATE.md` per §1, builds, and pushes to `main`. Runner: `~/.claude/blog-daily/run.sh`; logs: `~/.claude/blog-daily/last-run.log`; schedule: `~/Library/LaunchAgents/com.guodong.astro-blog-daily.plist`. Manage with `launchctl list | grep astro-blog-daily` and `launchctl unload/load -w <plist>`. The **portfolio** and **watchlist** posts are generated manually (the portfolio needs the private cost-basis CSV, which stays local). The steps below are the same whether a human or the agent runs them.

---

## 0. Prerequisites

- **Node ≥ 22.12** — the repo pins it via `.nvmrc` (`nvm use`). The system default may be older; Astro 6 will not build on Node 20.
- **pnpm** — `corepack enable && corepack prepare pnpm@latest --activate`, then `pnpm install`.
- **news-aggregator skill** (for the HN digest) — clone + isolated venv (macOS Python is PEP 668 externally-managed):
  ```bash
  git clone --depth 1 https://github.com/cclank/news-aggregator-skill /tmp/news-agg
  python3 -m venv /tmp/news-agg/.venv
  /tmp/news-agg/.venv/bin/pip install -q requests beautifulsoup4
  ```

---

## 1. HN daily digest (`hn-daily-YYYY-MM-DD.md`)

**Source: only [news.ycombinator.com](https://news.ycombinator.com/). Cover the ENTIRE front page — all 30 posts — in front-page ranking order (rank 1→30), _not_ sorted by points.**

**Fetch** in two parts:

1. Authoritative front-page ranking + live scores + comment counts, from HN's official API:
   ```bash
   # first 30 ids = home page (page 1), already in HN ranking order
   curl -s https://hacker-news.firebaseio.com/v0/topstories.json | python3 -c "import json,sys;print(*json.load(sys.stdin)[:30])"
   # per id: title, url, score, descendants(=comments), time(epoch), type
   curl -s https://hacker-news.firebaseio.com/v0/item/<id>.json
   ```
2. Article text for the summaries, via the skill (merge onto the API list by `url`):
   ```bash
   cd ~/.claude/skills/news-aggregator-skill   # or /tmp/news-agg
   ./.venv/bin/python scripts/fetch_news.py --source hackernews --limit 30 --deep --no-save > /tmp/hn.json
   ```

For any post the skill can't read (PDF, JS/SPA pages, **Ask HN / text posts** which have no `url`), `WebFetch` the article or the HN discussion page, or summarize from the title + public knowledge **with an explicit caveat**. Include job/Show HN/Launch HN/Ask HN posts too — _every_ front-page item.

**Write** `src/content/posts/hn-daily-DATE.md`, in **Simplified Chinese**, all 30 items in **front-page ranking order**. Each item shows points (Heat), relative time, and the **comment count** on the Discussion link. For Ask HN / text posts, link the title to the HN item. The `--deep` `content` truncates at ~3000 chars — if an item looks thin, `WebFetch` its `url` for the full article. Per-item, a **~one-minute-read** summary:

```markdown
#### N. [中文标题](url 或 hn_url)
- **Source**: Hacker News | **Time**: <相对时间> | **Heat**: 🔥 <点赞数>
- **Links**: [Discussion · <N> 条评论](hn_url)
- **Summary**: 一段约一分钟阅读（约 150–220 字）的中文摘要，覆盖原文关键事实、数据与结论，不遗漏要点（勿臆造）。站点拒绝抓取时据标题/公开资料概述并标注。
- **Deep Dive**: 💡 **Insight**: 1–2 句中文洞见（影响/价值）。
```

Frontmatter:
```yaml
---
author: Guodong Xie
pubDatetime: <ISO datetime>
title: "Hacker News 每日精读 · DATE"
slug: hn-daily-DATE
featured: false
draft: false
tags: [hacker-news, daily]
description: "抓取 Hacker News 首页全部 30 条，按首页排名顺序生成中文深度精读（每条约一分钟阅读）—— DATE。"
---
```

Reference template: the most recent `src/content/posts/hn-daily-*.md`.

---

## 2. Portfolio research (`portfolio-YYYY-MM-DD.md`)

Only generate on **US-market trading days** — skip weekends and US market holidays (2026 closed dates: 01-01, 01-19, 02-16, 04-03, 05-25, 06-19, 07-03, 09-07, 11-26, 12-25).

**Tickers** are grouped into 7 themes (see the latest `portfolio-*.md`):
`🚀 航天` RKLB, MDA · `🛡️ 国防与航空` LHX, RTX · `💻 大型科技` AMZN, TSLA · `🔬 半导体与 AI 算力` NVDA, TSM, PLTR · `⚡ 能源、电力与核电` GEV, MP, XE · `🏦 金融科技与加密` COIN, CRCL · `🧬 软件与医疗` CRM, JNJ.

**Live quotes** per ticker (for Start/End/today's move):
```bash
curl -s 'https://query1.finance.yahoo.com/v8/finance/chart/<TICKER>'
# meta.regularMarketPrice = 收盘(End); meta.chartPreviousClose = 起始(Start)
# 当日% = (End-Start)/Start*100
```

**Research** each ticker with `/trading-ideas:research <TICKER>` for the thesis, analyst rating/target, and the per-ticker `📰 个股要闻` (latest material news + source link). Theme-level `> 📰 近期要闻` callouts come from a sector news search. Theses change slowly — reuse them from the previous portfolio post and only refresh prices + news.

**Structure** (mirror the latest `portfolio-*.md` exactly):
- Snapshot table: `代码 | 起始 / 收盘（当日 %） | 我的总收益率 | 评级 | 目标价`
- Per theme: a `📰 近期要闻` callout, then per ticker a `### TICKER — Company`, a price line `**$Start / $End（当日%）**`, the Chinese thesis, and a `📰 个股要闻` line.
- The `非投资建议` disclaimer at the end.

**Privacy rules (strict):**
- **Never** print dollar holding values or share counts — only prices, percentages, ratings, targets, and news.
- `我的总收益率` = `(End − avg_cost)/avg_cost`. The per-ticker **average cost is private** and lives in the remote routine's config — it is **not** stored in this repo.

Frontmatter: `title: "投资组合深度研究：8 大主题下的 30 只持仓（YYYY 年 M 月 D 日）"` (the title **must include the full date with the day**, not just year/month), `slug: portfolio-DATE`, `tags: [portfolio, equity-research, investing]`.

---

## 3. Watchlist (`watchlist-YYYY-MM-DD.md`)

Same research format as the portfolio post, for tickers you're **watching but do not own**. Identical to §2 **except**: omit the `我的总收益率` column and any personal figures (there's no holding). Keep `代码 | 起始 / 收盘（当日 %） | 评级 | 目标价`, the thesis, and `📰 个股要闻`.

To add a ticker (e.g. **MRVL**): run `/trading-ideas:research MRVL`, fetch its live quote, place it under the right theme (MRVL → `🔬 半导体与 AI 算力`), and append the entry.

Frontmatter: `title: "观察名单 · DATE"`, `slug: watchlist-DATE`, `tags: [watchlist, equity-research]`.

---

## 4. Build, preview, publish

```bash
nvm use                 # Node 22
pnpm install
pnpm dev                # local preview at http://localhost:4321
pnpm build              # must finish with 0 errors; generates dist/posts/<slug>/
```
A post is only valid if `pnpm build` passes (it runs `astro check`). Then:
```bash
git add src/content/posts/<file>.md
git commit -m "Add <post> for DATE"
git push origin main    # GitHub Actions builds & deploys to Cloudflare Pages
```

> Note: the portfolio post currently covers **16 representative holdings**, not the full ~30-ticker account. Tickers not yet in the post: TT, PWR, SMERY, JPM, GOOG, AAPL, DUOL, HOOD, MSFT, CBRS, LRCX, ABBNY, VRT, VEEV. Add any of them by following §2.
