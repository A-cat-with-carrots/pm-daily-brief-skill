# pm-daily-brief

> **AI 产品经理每日情报简报 Claude Code Skill**
> 一句"今天有啥新的" → 三源信号融合 + 用户画像加权 + TL;DR 60 秒摘要的精炼日报，Markdown + PDF 双格式

[![Skill](https://img.shields.io/badge/Claude-Skill-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen)]()

---

## 它是什么

在 Claude Code 里随手说一句——

> "今天有啥新的"

——技能自动跑下面这套 9 步流水线，2-3 分钟产出一份给 **AI PM 喝咖啡时间** 看的精炼简报：

- **⚡ TL;DR 60 秒摘要**（🚀 机会 / ⚠️ 风险 / 🌊 趋势 三行）
- **🌟 Top 3-5 深读**（PM 视角拆解 + `🎯 今天具体动作` 1-2h 可执行下一步）
- **🤖 AI 核心全列表**（三源所有 AI 条目 + `[跟进/关注/噪音]` 10 秒扫读 tag）
- **🌶️ 高热度非 AI**（破圈热度 Top 3-5）
- **🔭 今日观察**（跨源 pattern 综合段，3 段 punchline + 证据 + 影响）
- **用户画像加权**（自动扫历史报告 + 项目文件，推断你在做什么，header 显式 surface 让你纠错）
- **Markdown + PDF 双输出**，footer 带抓取耗时

不是"通用新闻聚合器"，是 **正在做 AI 产品的 PM 的私人情报官**——三源融合 + 个性化加权 + 跨源 pattern 综合，信号优先，噪音少。

输出样例见 [`examples/2026-05-06.md`](examples/2026-05-06.md) / [`examples/2026-05-06.pdf`](examples/2026-05-06.pdf)。PDF 是成品交付物，Markdown 用于喂下游工具。

---

## 为什么做它

AI 圈一天 50+ 新产品/repo/讨论，PM 没时间逐条看。市面上的 RSS / 情报工具普遍存在：

| 通病 | 本 skill 改进 |
|------|--------------|
| 只是 RSS 聚合，不做语义判定 | LLM 过滤 + AI 分桶（核心 / 高热非 AI / 丢弃），漏比错严重 |
| 一刀切热度排序，跟你做啥无关 | **Step 2.5 用户画像加权**：扫近期报告 + 项目文件，命中你做的方向优先 |
| Top 5 死板硬凑 | **硬约束"凑不出 5 条强的就只写 3 条"**，禁止泛话 takeaway |
| PM takeaway 写成"持续关注" | `🎯 今天具体动作` 字段强制 1-2h 可执行，写不出就空着 |
| 无 60 秒摘要 | **TL;DR 三行**（机会/风险/趋势），喝咖啡时间内决定要不要往下读 |
| 全列表无 triage，36 条 AI 看头大 | 每条加 `[跟进/关注/噪音]` 方括号 tag，10 秒扫完 |
| 抓不到数据就编造 | 失败明确标注"⚠️ X 源抓取失败"，**绝不编造条目** |
| 风险项写成"机会" | TL;DR `⚠️` 强制威胁视角（对手/技术/政策可能颠覆你），不是优化包装 |
| 加权完全主导选品，新模型也漏 | **加权是 tie-breaker，不是 filter**——突破性非加权信号必入 |
| 路径硬编码 `C:\Users\xxx` | `$HOME` / `cygpath` / `$USERPROFILE` 自检测，跨机器可移植 |

底层赌注：**对个性化情报综合任务，prompt + LLM 推理 > scraper + 模板**，即使每次跑 2-3 分钟。

---

## 安装

### 方法 1：克隆到 Claude Code 全局 skills 目录

```bash
# Windows PowerShell
git clone https://github.com/A-cat-with-carrots/pm-daily-brief-skill.git "$HOME\.claude\skills\pm-daily-brief"

# macOS / Linux
git clone https://github.com/A-cat-with-carrots/pm-daily-brief-skill.git ~/.claude/skills/pm-daily-brief
```

### 方法 2：作为项目本地 skill

```bash
cd <your-project>
mkdir -p .claude/skills
git clone https://github.com/A-cat-with-carrots/pm-daily-brief-skill.git .claude/skills/pm-daily-brief
```

重启 Claude Code（或新开会话），技能自动加载。

### 依赖

- **Claude Code** with WebFetch tool
- **gstack**（用于 `make-pdf` PDF 输出）：https://github.com/garrytan/gstack
  - 装机：`git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup --team`
  - 没装 gstack 也能跑，只产 Markdown，没 PDF

---

## 用法

在 Claude Code 里说一句：

```
> 今天有啥新的
```

或任意触发词：

- `/pm-daily-brief`
- AI 日报 / 每日简报 / daily brief
- 刷一下今天 / 扫一下今天
- 行业热点 / 产品情报 / PM brief
- 今天有什么新产品 / 今天 AI 圈

技能自动触发，按 9 步流程跑（每步给一行进度更新）：

```
Step 1：确认日期 + 输出路径                 ─┐
Step 2：并行 WebFetch PH/GH/HN（多 URL fallback）│
Step 2.5：扫历史报告 + 项目文件 → 用户画像加权    ├─→ 2-3 分钟跑完
Step 3：AI 过滤 + 三桶分类                       │
Step 4：Top 3-5 深读（5 字段 + 🎯 今天具体动作）  │
Step 4.5：TL;DR 提炼（🚀/⚠️/🌊 三行）            │
Step 5：全列表加 [跟进/关注/噪音] triage tag      │
Step 6：今日观察（3 段 pattern + 证据 + 影响）    │
Step 7：写 Markdown                              │
Step 8：调 gstack make-pdf 生成 PDF              │
Step 9：终端汇报（路径 + 抓取数 + 耗时 + TL;DR）  ─┘
```

---

## 三源抓取

| 源 | 主路径 | Fallback 1 | Fallback 2 |
|---|---|---|---|
| ProductHunt | `/leaderboard/daily/Y/M/D`（月份不补零）| homepage `/` | 前一日 leaderboard |
| GitHub Trending | `/trending?since=daily` | `/trending`（无 since）| `/trending?since=weekly` |
| Hacker News | `/` | `/news?p=1` | `/newest` |

**关键工程决策**：
- ✅ **WebFetch 主路径**——PH 被 Cloudflare 挡 403，但 leaderboard URL 可绕过；GH 比 headless browser 干净 10 倍
- ❌ **不用 `/browse`**——PH 端 Cloudflare 直接 ban headless Chromium，确认无效已退场
- 🔁 **多 URL fallback**——任一源主路径失败自动重试 fallback，最终全失败就标"⚠️ X 源抓取失败"，**绝不编造**
- 📊 **WebFetch quota**：3 并行 + 1-2 deep-read + 偶尔 WebSearch ≈ 5-7 次 net call，撞限额跳补充用已抓数据出报告

---

## 9 步流程速览

### Step 1：日期 + 路径
- CLAUDE.md 注入 `currentDate` 优先，fallback 到 `date +%Y-%m-%d`
- 跨午夜（差 > 12h）显式问用户用哪个日期
- 同日重跑直接覆盖（用户每天调一次，重跑就是想要新版）

### Step 2：并行 WebFetch 三源
- 一回合发 3 个 WebFetch
- PH/GH/HN 各拉 Top 20-30
- 明确要求"不要导航菜单噪音"——否则 WebFetch 也可能塞导航

### Step 2.5：用户画像加权（**单一最大改进**）
- **来源 1**：`Glob daily-input/*.md` 取最近 5 份，Grep `## 🌟 Top` + `## 🔭 观察` 段，提炼连续多日上 Top 的主题
- **来源 2**：`Glob 项目根/*.{md,html}` 不递归，识别用户在做什么产品（`competitive-report` / `battlecard` 文件名 = 该垂直加权）
- **不扫敏感目录**（`.git/` / `.env` / credentials）
- 输出 `high-weight` / `mid-weight` 列表，**显式写到报告 header** 让 PM 纠错
- **加权是 tie-breaker，不是 filter**——突破性非加权信号（新模型、政策事件）必入

### Step 3：AI 过滤
三桶分类：
- 🤖 **AI 核心**：LLM/agent/RAG/embedding/diffusion/TTS-STT/CV/AI infra/AI 应用层/AI 框架/推理引擎/向量库/AI 硬件
- 🌶️ **高热度非 AI**：单源 Top 3 或跨源出现
- 🗑️ **丢弃**：中低热度非 AI

不确定**倾向纳入**（漏比错严重）。

### Step 4：Top 3-5 深读
**硬约束：凑不出 5 条强的就只写 3 条，不补位降标准。**

每条 5 字段：
- **是什么**（PM 视角不是开发者视角）
- **差异化**（vs 现有方案）
- **技术/打法**
- **PM takeaway**（条件式优先："如果做 X 类产品，则 Y"）
- **🎯 今天具体动作**（1-2h 内能做完；写不出就**省略**，禁止"持续关注"废话）
- **延伸**（可选 cross-source 对照）

**抓取策略**：默认基于 tagline + PM 知识写，仅 1-2 条真高价值的 WebFetch 补原文（HN >300pts 长文 / 政策事件类）。

### Step 4.5：TL;DR（60 秒摘要）
3 行各 ≤ 30 字：
- **🚀 机会**：跟用户做的事最相关的最强信号
- **⚠️ 风险**：威胁视角（对手/技术/政策可能颠覆你），**禁止包装成机会**
- **🌊 趋势**：跨源浮出的最强 pattern

### Step 5：全列表 triage
每条开头方括号 tag：
- `[跟进]` ← high-weight 命中 + 信号强 / 直接竞品
- `[关注]` ← 相关但非急
- `[噪音]` ← 列出供完整性，可跳

每段 `[跟进]` ≤ 5 条（跟进太多等于没跟进）。

### Step 6：今日观察
**整篇最有价值的部分**。3 段（不是 4 段，4 段稀释）：
- 每段 punchline 粗体开头
- 后接 2 句证据 + 1 句对用户产品的影响
- 基于真实数据，**禁止"AI 在加速"这种废话**

### Step 7-9：写 MD → 生成 PDF → 汇报
- `gstack make-pdf` 调本地二进制（skill 调用慢，直接 bash）
- `$HOME` / `cygpath` / `$USERPROFILE` 自检测，**不硬编码 `C:\Users\xxx`**
- PDF 失败不阻塞 MD 交付，报告"PDF 生成失败，md 已就绪"
- 汇报含三源抓取数 + 抓取耗时 + TL;DR 三行浓缩

---

## 输出模板

```markdown
# AI PM Daily Brief — YYYY-MM-DD

> 三源抓取: ProductHunt N · GitHub Trending N · Hacker News N
> 入选: AI 核心 N · 高热非AI N
> 用户画像加权: theme1, theme2, theme3

## ⚡ TL;DR (60秒)

- **🚀 机会**: ...
- **⚠️ 风险**: ...
- **🌊 趋势**: ...

## 🌟 Top 3-5 深读
### 1. [产品名](官网) (PH ▲527, 当日榜首)
- **是什么**: ...
- **差异化**: ...
- **技术/打法**: ...
- **PM takeaway**: ...
- **🎯 今天具体动作**: ...

## 🤖 AI 核心(全列表)
### ProductHunt / GitHub Trending / Hacker News
- `[跟进]` [产品名](官网) — tagline (▲upvotes) — 一句话 PM 视角点评

## 🌶️ 高热度非 AI
## 🔭 今日观察
**[Pattern punchline]**。证据 1。证据 2。对用户产品的影响。
...

---
*生成时间: ... | 抓取耗时: Xm Ys | 抓取源: PH/GH/HN | skill: pm-daily-brief v3*
```

---

## 写作风格铁律

- **中文为主**，产品名/技术术语保留英文
- 一句话点评**信息密度高**——不是"这是个不错的工具"，而是"用 MCP 让 Claude 直接操作 Figma，解决设计稿到代码的 last-mile"
- **PM 视角 ≠ 开发者视角**：关心"用户问题/市场切口/可防御性"，不只关心"用了什么算法"
- 不堆形容词。事实 + 判断，禁止"令人激动""革命性"
- 链接全 markdown `[text](url)` 格式
- TL;DR 三行各 ≤ 30 字（长了就不是 TL;DR 是 lengthy DR）
- PM takeaway 优先条件式（"如果 X 则 Y"）而非断言

---

## 自定义

技能默认硬编码 `D:\hrdai\daily-input\` 输出路径，给 fork 用户改：

### 1. 输出路径
`SKILL.md` Step 1，改 `D:\hrdai\daily-input\` 为你的项目根。

### 2. 用户画像加权来源（Step 2.5）
默认读：
- 最近 `daily-input/*.md`（你过去的报告）
- 项目根 `*.md` / `*.html`（你的项目描述）

改 Glob 根到你的工作区。**不要指向 `node_modules/` 或带敏感数据目录**——技能已显式跳 `.git/` `.env` credentials。

### 3. 输出语言
默认中文+英文术语。要切英文：改 SKILL.md "写作风格" 段 + 输出模板表头。

### 4. PDF 二进制路径（Windows）
v3 起 `SKILL.md` Step 8 已自检测 `$HOME` + `cygpath`，**无用户名硬编码**。
gstack 装在非默认位置就改 `P=` 和 `BROWSE_BIN=` 两行。

### 5. 加源
当前 PH + GH + HN。要加 Reddit/Twitter/arxiv：Step 2 加一个并行 WebFetch + 抽取 prompt。

---

## 文件结构

```
pm-daily-brief/
├── SKILL.md                # 主入口（9 步编排 + 输出模板 + 写作风格 + 失败模式）
├── README.md               # 本文件
├── LICENSE                 # MIT
└── examples/
    ├── 2026-05-06.md       # 真实跑出来的样例
    └── 2026-05-06.pdf
```

---

## 为什么是 skill，不是 Python 脚本

三源 scraping 一个 Python 脚本就能搞定，但本 skill 不是 scraping 工具：

- **综合是价值，scraping 不是**——"今日观察" 跨源 pattern + 每条 PM takeaway 需要 LLM 推理，不是正则
- **用户画像加权**——读任意项目文件 + 推断意图，也是 LLM 推理
- **可维护性**——PH 改 leaderboard URL 时（构建期间真改过），用自然语言告诉 skill 回退到首页就行；用脚本得调一小时 selector

底层赌注：**对个性化综合任务，prompt + LLM > scraper + 模板**，即使每次 2-3 分钟。

---

## 已知限制

- **2-3 分钟运行时**，主要是 WebFetch 延迟 + 1-2 次 deep-read。footer 报实际耗时让你判断每日成本是否值得
- **WebFetch quota**：3 并行 + 1-2 deep-read + 偶尔 WebSearch ≈ 5-7 net call，日内多次跑可能撞限额
- **PH 反爬**：`/browse`（headless Chromium）被 Cloudflare ban，**不是 fallback 路径**
- **GH 偶尔截断**：有时只拿到 15/25 trending repo，header 标 count，截断严重时回退到 `/trending` 或 weekly
- **无跨日趋势追踪**：每份报告独立，跨日 pattern 追踪需结构化存储（v1 explicit 不要）
- **单用户**：默认输出路径 `D:\hrdai\daily-input\` 硬编码，二进制路径已自检测——按 [自定义](#自定义) 改输出路径

---

## 版本历史

- **v1**：`/browse` 主路径（被 PH Cloudflare 挡 + GH 噪音），无加权，Top 5 硬凑
- **v2**：WebFetch 主路径，加 Step 2.5 用户上下文加权（**单一最大改进**——把通用 RSS 摘要变成个性化简报），Top 3-5 + 不补位硬约束，PDF 生成
- **v3**（当前）：PM 体验大改
  - **TL;DR 60 秒摘要**（🚀/⚠️/🌊 三行），喝咖啡时间可用
  - **`🎯 今天具体动作`** 字段——每条 PM takeaway 必带 1-2h 可执行下一步，写不出就省略（禁止"持续关注"）
  - **`[跟进/关注/噪音]` triage tag**——36 条 AI 全列表 10 秒扫读
  - **加权列表 surface 到 header**（之前内部用），PM 能纠错
  - **多 URL fallback** 替换无效 `/browse` fallback
  - **路径自检测**——去 `C:\Users\xxx` 用户名硬编码
  - **bash + PowerShell 双 shell** PDF 命令
  - **抓取耗时 footer**
  - **观察段 4 段 → 3 段**（4 段稀释信号）
  - **TL;DR 风险项强制威胁视角**（对手/技术/政策颠覆你），禁止优化包装

---

## 贡献

PR 欢迎。重点方向：

- **加源**（Reddit / Twitter / arxiv / Anthropic blog / OpenAI announcement RSS）
- **跨日趋势追踪**（v1 explicit out of scope，v3 验明价值后可重启）
- **健康度阈值**（GH < 15 或 PH < 10 时警告抓取失败 vs 真清淡日）
- **`[跟进]` budget 硬限制**（当前软 ≤ 5/源，无 hard fail）
- **PH 详情页规范化**（Step 2 prompt 要 canonical URL，但 PH 详情页抽取偶尔返回 `/products/<slug>` 而非官网）

---

## 致谢

- skill 脚手架：[Anthropic skill-creator](https://github.com/anthropics/skills/tree/main/skill-creator)
- 浏览器 + PDF 工具链：[gstack](https://github.com/garrytan/gstack)
- skill review 流程灵感：skill-creator eval loop

---

## License

MIT — 见 [LICENSE](LICENSE)
