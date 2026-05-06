---
name: pm-daily-brief
description: AI产品经理每日情报简报。手动每天调用一次,自动抓取当天 ProductHunt 榜单、GitHub Trending(daily)、Hacker News 热门,以 AI 相关为主、其他破圈热度也带,会读用户最近报告 + 项目文件做主题加权,混合深度(全列表一句话 + Top 3-5 深读),自主补充相关资讯,中文 Markdown + PDF 双格式输出到 D:\hrdai\daily-input\YYYY-MM-DD.{md,pdf}。Use when 用户说"今天有啥新的"、"今天 AI 圈"、"AI 日报"、"每日简报"、"daily brief"、"刷一下今天"、"行业热点"、"今天值得看的"、"产品情报"、"PM brief"、"今天有什么新产品"、"扫一下今天",或调用 /pm-daily-brief。
---

# PM Daily Brief

为 AI 产品经理生成每日情报简报。目标读者: 你自己。目标用途: 30 分钟内吸收当天产品/技术/趋势信号,辅助产品决策。

## 触发与定位

用户每天手动调一次。每次都生成"当天"的报告(以系统当前日期为准,从 CLAUDE.md 注入或 `date` 命令获取)。

**不是**通用新闻聚合器。是给一个**正在做 AI 产品**的 PM 看的精炼简报 — 信号优先,噪音少。

## 输出语境

报告是写给**单一用户(你自己)看**的私人备忘,不是发布物。可以:

- 直接用真实产品名/项目名(`[Project-A]` 这种占位是 sample 公开版才用)
- 在 `🎯 今天具体动作` 字段里 reference 用户具体文件路径或 repo
- 在观察段里 reference 用户在做什么

仅当用户表示要分享报告(推 GitHub / 发同事)时, 用 sed 批量脱敏成占位符。

## 工作流

按顺序执行下列步骤。每步给用户一行进度更新(如"抓取 ProductHunt..."),保持简短。

### Step 1: 确认日期与输出路径

- 日期: 优先用 CLAUDE.md 注入的 `currentDate`,fallback 到 `date +%Y-%m-%d` (bash,跨平台)。Windows-only 环境也可用 PowerShell `Get-Date -Format yyyy-MM-dd`。**若两者差超过 12 小时(跨午夜调用), 显式问用户用哪个日期。**
- 输出: `D:\hrdai\daily-input\YYYY-MM-DD.md`
- **覆盖策略**: 文件已存在直接覆盖, 不要问用户、不要 diff (用户每天调一次, 同日重跑就是想要新版本)。Step 8 汇报时若覆盖了同日旧版, 一句话提示"(覆盖了 N 分钟前的同日报告)"。

记录 `START_TS=$(date +%s)`,Step 8 计算抓取耗时。

### Step 2: 抓取三源(WebFetch 优先,多 URL 重试)

**默认用 WebFetch,不要用 /browse。** 实测 (2026-05-06):
- ProductHunt 首页被 Cloudflare 挡 403,headless browser 进不去。WebFetch 走 leaderboard URL 可绕过。
- GitHub Trending 用 browse 抓回的 text 99% 是导航菜单噪音。WebFetch 输出干净 10 倍。
- HN 两种都行,WebFetch 更快。

并行抓三源(同一回合发三个 WebFetch 调用):

1. **ProductHunt 当日 leaderboard**
   - URL: `https://www.producthunt.com/leaderboard/daily/YYYY/M/D` (注意月份不补零, e.g. `/2026/5/6`)
   - prompt 要求: 每个产品返回名称、tagline、upvotes、tags、PH 链接、**产品官网链接(从详情页抓)**, top 20。
2. **GitHub Trending (daily)**
   - URL: `https://github.com/trending?since=daily`
   - prompt 要求: 每个 repo 返回 owner/repo、完整 GitHub URL、描述、语言、总 stars、当日 stars。**明确要求"不要包含导航菜单或语言选择器"** — 否则 WebFetch 也可能塞导航。Top 25。
3. **Hacker News 首页**
   - URL: `https://news.ycombinator.com/`
   - prompt 要求: 标题、分数、评论数、外链 URL、域名、HN 讨论页 URL。Top 30。

**Fallback (按顺序试,/browse 已确认无效不再用)**:

| 源 | 主路径 | Fallback 1 | Fallback 2 |
|---|---|---|---|
| PH | `/leaderboard/daily/Y/M/D` | homepage `/` | 前一日 leaderboard (慢一天数据 > 没数据) |
| GH | `/trending?since=daily` | `/trending` (无 since) | `/trending?since=weekly` (退一档) |
| HN | `/` | `/news?p=1` | `/newest` |

任一源最终全失败,报告头部标"⚠️ X 源抓取失败 (尝试: 主+fallback1+fallback2)",**绝不编造条目**。

**WebFetch quota 注意**: 三源并行 + Step 4 的 1-2 次 deep-read fetch + 偶尔 Step 4.5 WebSearch ≈ 5-7 次 net call。日内多次跑可能撞限额。撞了直接跳过补充, 用已抓数据出报告。

### Step 2.5: 读用户上下文(主题加权依据)

抓完三源后, 在过滤前先建 **用户兴趣画像**, 让本日报告偏向用户实际在做的方向。

**读两类来源**:

1. **最近 3-5 份 daily brief**: `Glob D:\hrdai\daily-input\*.md`, 取最近 5 个(按文件名日期排), Grep 它们的 `## 🌟 Top 3-5 深读` 段标题 + `## 🔭 今日观察` 段。
   - 提炼: 哪些主题连续多日上 Top? (e.g. agent 经济、context 工程、voice agent)
   - 提炼: 用户最近反复出现的关键词?

2. **项目根快扫**: `Glob D:\hrdai\*.{md,html}` (限制根目录, 不递归, 不读 `.claude/` `.git/` `node_modules/`)。识别用户在做什么产品。
   - 文件名/目录名带 `competitive-report` / `battlecard` / `competitor` → 用户在做该方向竞品研究, 该垂直加权
   - 文件名/目录名暗示产品代号 → 读对应 `README.md` 第一段 + tagline 提取产品定位
   - 看 `CLAUDE.md` 注入的项目描述

**不要扫描敏感目录** (`.git/`, `.env`, credentials)。如果 Glob 误中, 跳过不读。

第一次跑 (`daily-input/` 还没历史) 跳过来源 1, 只用项目文件。

**输出**: 维护一个 **加权列表**, 类似:
```
high-weight: ["团队管理AI", "HR tech", "agent 经济", "context 工程"]
mid-weight: ["voice agent", "video gen", "Skills/插件分发"]
```

**该列表必须显式写到报告 header**(模板里 `用户画像加权:` 那行), 让用户看到 skill 推断出的画像, 错了能反馈。

**怎么用**: 在 Step 3 AI 判定时, 命中 high-weight 主题的边缘条目倾向纳入; 在 Step 4 Top N 选品时, 同分情况优先 high-weight; 在 Step 5 全列表 triage tag 时, 命中 high-weight 默认 `[跟进]`。

**不要做的事**:
- 不要让加权完全决定选品 — 突破性的非加权信号 (如新模型发布) 必须保留。**加权是 tie-breaker, 不是 filter。**
- 不要每天都强行塞 high-weight 主题。当天没相关条目就不塞, 不要硬找。

### Step 3: AI 过滤与分类

把所有条目过一遍,分三桶:

- **🤖 AI 核心**: LLM、agent、RAG、embedding、diffusion、TTS/STT、CV/多模态、AI infra/eval、prompt 工具、AI 应用层(代码助手、设计、写作、客服、search、教育...)、AI 框架/SDK、推理引擎、向量库、AI 硬件
- **🌶️ 高热度非 AI**: 不属于 AI 但单源排名 Top 3 或跨源出现 — 用户要求"很火的其他东西也带"
- **丢弃**: 中低热度的非 AI 条目

判定不确定时倾向**纳入**(漏比错严重)。AI 判定基于描述语义,不只关键词 — 比如一个"chrome extension for X"如果 X 是 AI 用例,算 AI。

### Step 4: Top 3-5 深读

从三桶里挑 **3 到 5 条最值得 PM 关注** 的(优先 AI 核心 + Step 2.5 high-weight 主题, 看产品差异化/技术新颖度/赛道动向, 不只看热度)。

**硬约束**: **凑不出 5 条强的就只写 3 条, 不要为补位降标准。**今天可能就是只有 2-3 个真信号, 那就 3 条; 偶尔信号特别多也可以 4-5。第 4、5 条的 PM takeaway 写不出洞见时, 直接把它降级到全列表去, 不要硬凑泛话。

每条提炼 5 个字段(`🎯 今天具体动作` 凑不出就空着, 不要硬写"持续关注"这种废话):

- **是什么** (1-2 句, PM 视角不是开发者视角)
- **差异化** (vs 现有方案有什么新)
- **技术/打法** (用了什么模型、架构、商业模式)
- **PM takeaway** (对你做 AI 产品的启发 — 一句话, 可以是条件式 "如果 ... 则 ...")
- **🎯 今天具体动作** (1-2 小时内能做完的下一步; 例: "拆 X 的 README + 通信协议", "把 Y 写成 Skill 上 GitHub 看一周 star")。**写不出具体动作就省略此字段**。
- **延伸** (可选, cross-source 对照, 不每条都要)

**抓取策略 — 不要每个 Top 5 都 fetch 原文**:
- 默认基于已有 tagline + 描述 + 行业知识写。20 字 tagline + PM 知识足够支撑前 4 个字段。
- **只对 1-2 个真正高价值的**条目用 WebFetch 补充原文。判定标准: HN 高分长文 (>300pts 且非简短公告)、有清晰长 URL 的 manifesto/技术博客、引出大趋势的政策/事件类。
- 产品官网/PH/GH README 通常**不必 fetch** — tagline + tag 足够,fetch 多了拖时间且收益边际递减。

**链接规范**: 标题用产品官网链接 (Step 2 已要求 PH 抓时带), 旁边括号里给来源标记 `(PH ▲527, 当日榜首)`。HN 类目链接给原文 URL,PH 给产品官网,GH 给 repo URL。让 PM 一跳到位,不再多一层 PH/HN 详情页。

**自主补充(WebSearch)**: 仅当某条目引出值得跟进的更大趋势(新模型、新范式、政策事件),且原始信息不足时,用 WebSearch 补 1 条上下文加到"延伸"字段。**保持信噪比** — 一个报告里"延伸"出现 2-3 次就够,不要每条都补。

### Step 4.5: TL;DR 提炼 (60 秒摘要)

写 Top 3-5 之后、之前的 header 之后,产出 **TL;DR 段**:

- **🚀 机会** (一句话, 跟用户做的事最相关的最强信号)
- **⚠️ 风险** (一句话, 该警觉的对手/技术/政策, 强调威胁性而非机会)
- **🌊 趋势** (一句话, 跨源浮出的最强 pattern, 通常 Step 5 观察段第一段的 punchline)

**为什么必须有**: PM 喝咖啡时间内就该知道今天有没有大事。3 行 60 秒读完, 决定要不要往下读全文。如果 TL;DR 写不出"风险"项, 想想是不是真的没风险还是没认真找 — 大多数日子都有。

### Step 5: 全列表加 triage tag

每条全列表点评开头加方括号 tag, 让 PM 10 秒扫完决定看哪条:

- `[跟进]` — 命中 high-weight 加权 + 信号强 / 直接竞品 / 该立项跟踪
- `[关注]` — 相关但非急, 知道有这事就行
- `[噪音]` — 列出供完整性, 但可跳

判定原则:
- 加权 high-weight 命中 + 单源 Top 3 = `[跟进]`
- 加权 mid-weight 或非加权但技术新颖 = `[关注]`
- 非 AI / 工具类 / 主题分散 = `[噪音]`

每个全列表段(PH / GH / HN)的 `[跟进]` 应当 ≤ 5 条 — 跟进太多等于没跟进。

### Step 6: 今日观察(综合段)

报告末尾写一段 150-300 字的"今日观察":

- 跨产品的共性 pattern (e.g. "今天三个 agent 框架同时上榜,都在解决 tool-use 可靠性")
- 新涌现的赛道或技术信号
- 你(AI PM)应该警觉/兴奋的点

**写法**: 3 段, 每段开头一句 punchline (粗体), 后接 2 句证据 + 1 句对用户产品的影响。**不要 4 段, 4 段开始稀释**。

**这段是整篇最有价值的部分**。宁可少写两个产品也要把这段写好。基于真实数据,不要空泛 ("AI 在加速"这种废话不要)。

**附带高权重信号**(可选子段): 加权 high-weight 命中但没进 Top 5 的、值得单提一下的, 1-2 条。例: 直撞用户竞品调研的小 PH 产品。注意命名: 这是"高价值但落选"的单提, 与全列表 `[噪音]` tag(低价值条目)是相反含义。

### Step 7: 写 Markdown 文件

写入 `D:\hrdai\daily-input\YYYY-MM-DD.md`,严格按 [输出模板](#输出模板) 结构。

### Step 8: 生成 PDF (用 gstack make-pdf)

直接调 gstack make-pdf 二进制 (skill 调用太慢, 直接 bash)。

**自检测路径** (避免硬编码用户名):

```bash
# bash / git bash on Windows
USER_HOME_WIN="$(cygpath -w "$HOME" 2>/dev/null || echo "$USERPROFILE")"
P="$HOME/.claude/skills/gstack/make-pdf/dist/pdf"
export BROWSE_BIN="${USER_HOME_WIN}\\.claude\\skills\\gstack\\browse\\dist\\browse.exe"

DATE="2026-05-06"  # replace with $TODAY
"$P" generate \
  "D:/hrdai/daily-input/${DATE}.md" \
  "D:/hrdai/daily-input/${DATE}.pdf" \
  --title "AI PM Daily Brief — ${DATE}" \
  --author "AI PM" \
  --date "${DATE}" \
  --no-confidential
```

**PowerShell 等价命令** (在 PS 跑):
```powershell
$P = "$HOME\.claude\skills\gstack\make-pdf\dist\pdf.exe"
$env:BROWSE_BIN = "$HOME\.claude\skills\gstack\browse\dist\browse.exe"
$DATE = "2026-05-06"
& $P generate `
  "D:/hrdai/daily-input/${DATE}.md" `
  "D:/hrdai/daily-input/${DATE}.pdf" `
  --title "AI PM Daily Brief — ${DATE}" `
  --author "AI PM" `
  --date $DATE `
  --no-confidential
```

**Windows 踩坑** (实测 2026-05-06):
1. **必须用 `.exe`**: `BROWSE_BIN` 指向 `browse.exe`,不能是裸 `browse`(虽然文件存在,make-pdf 的 `which browse` 找不到)。用反斜杠 Windows 路径。
2. **位置参数顺序**: `generate <input> <output> --flags`。flag 放后面,否则 `--date YYYY-MM-DD` 会吞掉位置参数。
3. **`--no-confidential` flag**: 需要 make-pdf ≥ 2026-05 版。老版没此 flag 会报错, 升级 gstack 或去掉 flag (会带 CONFIDENTIAL 角标)。

设置:
- `--no-confidential` 去掉默认 CONFIDENTIAL 角标(简报不需要)
- 不加 `--cover` 不加 `--watermark`(简报直入正文)
- 默认已有页码 + running header,够用

**失败处理**: 报告"PDF 生成失败,md 已就绪",**不要阻塞 md 交付**。常见失败:
- `browse binary not found` → 检查 BROWSE_BIN env 是否设了 `.exe` 后缀
- `input file not found` (但其实是 output 路径) → flag 顺序错,改成 `generate input output --flags`
- `unknown flag --no-confidential` → make-pdf 版本太老, `gstack-upgrade` 或去掉该 flag

### Step 9: 汇报

```
END_TS=$(date +%s)
ELAPSED=$(( END_TS - START_TS ))
```

给用户终端汇报:
- md 路径 + pdf 路径(若成功)
- 三源抓取数 + 抓取耗时(如"PH 20 / GH 25 / HN 30, 共 2m 18s")
- 入选条目数
- TL;DR 三句的浓缩(从 Step 4.5 直接复制 🚀/⚠️/🌊 三行)
- 若覆盖了同日旧版: 提示"(覆盖了 N 分钟前的同日报告)"

## 输出模板

严格按此结构。Header 用 emoji 帮助快速扫读。

```markdown
# AI PM Daily Brief — YYYY-MM-DD

> 三源抓取: ProductHunt N · GitHub Trending N · Hacker News N
> 入选: AI 核心 N · 高热非AI N
> 用户画像加权: theme1, theme2, theme3

## ⚡ TL;DR (60秒)

- **🚀 机会**: [一句话, 跟用户做的事最相关的最强信号]
- **⚠️ 风险**: [一句话, 该警觉的对手/技术/政策]
- **🌊 趋势**: [一句话, 跨源浮出的最强 pattern]

## 🌟 Top 3-5 深读

### 1. [产品名](官网) (PH ▲527, 当日榜首)
- **是什么**: ...
- **差异化**: ...
- **技术/打法**: ...
- **PM takeaway**: ...
- **🎯 今天具体动作**: ... (写不出就省略此行)
- **延伸** (可选): ...

### 2. ...
(同上)

## 🤖 AI 核心(全列表)

### ProductHunt
- `[跟进]` [产品名](官网) — tagline (▲upvotes, tags) — 一句话 PM 视角点评
- `[关注]` ...
- `[噪音]` ...

### GitHub Trending
- `[跟进]` [owner/repo](链接) — 描述 (语言, ⭐today) — 一句话点评
- ...

### Hacker News
- `[跟进]` [标题](原文链接) — 域名 (分数/评论, [HN 讨论](hn-url)) — 一句话点评
- ...

## 🌶️ 高热度非 AI

(同上格式 + tag,只列 Top 3-5 条)

## 🔭 今日观察

**[Pattern 1 punchline]**。证据 1。证据 2。对用户产品的影响。

**[Pattern 2 punchline]**。证据 1。证据 2。对用户产品的影响。

**[Pattern 3 punchline]**。证据 1。证据 2。对用户产品的影响。

### 附带高权重信号 (可选)

- [产品名](链接) — 一句话, 为什么单提 (例: 直撞 [vertical-X] 竞品调研)

---
*生成时间: YYYY-MM-DD HH:mm | 抓取耗时: Xm Ys | 抓取源: PH/GH/HN | skill: pm-daily-brief v3*
```

## 写作风格

- **中文为主**,产品名/技术术语保留英文
- 一句话点评要**信息密度高** — 不是"这是一个不错的工具",而是"用 MCP 让 Claude 直接操作 Figma,解决设计稿到代码的 last-mile"
- PM 视角 ≠ 开发者视角: 关心"用户问题/市场切口/可防御性",不只关心"用了什么算法"
- 不堆砌形容词。事实 + 判断,不写"令人激动""革命性"
- 链接全用 markdown `[text](url)` 格式,方便点
- **TL;DR 三行各≤30字**, 长了就不是 TL;DR 是 lengthy DR
- **PM takeaway 优先用条件式** ("如果做 X 类产品, 则 Y") 而非断言, 减少过度乐观

## 常见失败模式与避免

- **抓不到数据就编造** — 绝不。失败就标注。
- **AI 判定太严**,漏掉边缘 AI 产品 — 倾向纳入。
- **Top 5 都从 ProductHunt 选** — 三源各看,GH 和 HN 经常有更深的技术信号。
- **"今日观察"写成总结性废话** — 必须基于当天真实条目讲出 pattern。
- **报告太长** — TL;DR + Top 3-5 + 全列表 + 观察, 总字数 1500-3000 字为佳。超过就剪全列表的点评长度。
- **凑齐 5 条 Top 深读** — 硬凑会出泛话 takeaway, 见 Step 4 硬约束, 宁可只写 3 条。
- **"🎯 今天具体动作"凑不出就硬写"持续关注"** — 不写比写废话好。空着等于明确告诉 PM "这条没立即动作可做"。
- **TL;DR 风险项写成机会** — ⚠️ 必须是威胁视角(对手在做什么、什么会颠覆你)而非"机会"。
- **加权完全主导选品** — Step 2.5 加权是 tie-breaker, 不是 filter。突破性的非加权信号(新模型发布、政策事件)必须保留。
- **跨午夜调用日期错乱** — Step 1 已加: CLAUDE.md vs 系统时间差超 12 小时显式问。
- **用户名硬编码** — Step 8 命令必须用 `$HOME` / `cygpath` / `$USERPROFILE` 自检测, 不写死 `C:\Users\xxx`。

## 测试与迭代

skill 输出偏主观,无固定正确答案。用户跑后给反馈,迭代调整: 深度、长度、过滤严格度、PM takeaway 角度、triage tag 阈值。

## 版本历史

- **v1**: `/browse` 主路径(被 PH Cloudflare 挡 + GH 噪音), 无加权, Top 5 硬凑。
- **v2**: WebFetch 主路径, 加 Step 2.5 用户上下文加权(单一最大改进), Top 3-5 + 不补位硬约束, PDF 生成。
- **v3** (当前): 加 TL;DR 60秒摘要(机会/风险/趋势), `🎯 今天具体动作` 字段, 全列表 `[跟进/关注/噪音]` triage tag, header 显式 surface 加权列表, 多 URL fallback 替换无效 /browse fallback, 路径自检测(去硬编码用户名), 双 shell PDF 命令, 抓取耗时 footer。
