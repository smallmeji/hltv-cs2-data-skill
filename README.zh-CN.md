# hltv-cs2-data Skill 中文说明

`hltv-cs2-data` 是一个 skill，用来从 HLTV 收集和整理 CS2 数据，并输出给大模型、分析师或产品系统使用的数据包。

这个 skill 的核心定位是：只提供标准化数据，不内置策略。它负责把 HLTV 上能看到的事实数据整理成 Markdown 和 JSON，并把对模型有用的事实特征组织成 `decision_inputs`。是否预测胜率、怎么判断地图优势、怎么制定策略，由调用它的大模型或用户自己的策略决定。

## 适合什么场景

当你想让大模型分析一场 CS2 比赛，但又不想让模型凭空编数据时，可以使用这个 skill。

典型场景：

- 输入一条 HLTV 比赛链接，获取这场比赛的数据包。
- 输入两个队伍，例如 `NAVI 和 G2`，比较两队的地图池、选手状态、阵容变化和历史交手。
- 查看不同地图胜率、每张地图的警匪胜率、对位数据和选手近期状态。
- 输入赛事 rating 页面，补充选手在当前赛事里的 rating。
- 做历史回测，要求只使用某个时间点之前可见的数据。
- 把结构化 Markdown 和 JSON 交给另一个模型或策略系统继续判断。

这个 skill 可以服务预测流程，但默认不直接预测。只有当用户明确要求“请根据这些数据判断胜率 / 谁更可能赢”时，模型才可以在事实数据包之后追加 `Model Inference`，并且必须说明那是模型推理，不是 HLTV 原始事实。

具体胜率百分比必须经过数据完整度闸门。若 HLTV 被 Cloudflare 拦截，或当前年度地图池 / 选手 rating 没有加载成功，skill 应输出部分数据包和缺失字段，但不应给出具体胜率区间。

用户可读部分默认跟随用户语言。中文提问时，输出应使用中文标题、中文表格标签和中文 warnings；JSON 字段名继续保持英文，方便程序和其他模型稳定读取。

## 不做什么

这个仓库不提供：

- 私人 CS2 预测模型。
- 投注建议。
- EV、Kelly、最高买入价、仓位计算。
- 绕过访问控制的爬虫服务。
- 本地数据库或私人数据依赖。
- 在纯 HLTV 页面模式下保证完整历史快照。

默认轻量模式只读取当前会话里可访问的公开 HLTV 页面。如果配置了维护好的静态 JSON 数据包、数据 API / 数据仓，可以提供更完整、更可复现的历史数据，但这不是安装本 skill 的前提。

## 产品分层

这个 skill 可以按两层使用：

| 层级 | 适合场景 | 预期能力 |
|:--|:--|:--|
| 轻量版 | 单场、临时、公开 HLTV 查询 | 安装后即可使用。适合读取比赛基础信息、阵容、H2H、比赛页可见地图概览，以及尽力读取 stats 页面。深层 stats 页失败时必须标缺。 |
| 静态 JSON 版 | 给 Claude/GPT/用户模型共享数据包 | 推荐作为对外分发路径。模型直接读我们导出的 JSON，不需要实时访问 HLTV，能避开 CF 和页面读取失败。 |
| API 版 | 可重复分析、批量使用、正式产品 | 推荐用于完整年度数据、CT/T 警匪胜率、精确历史回测、阵容/veto/赛果快照、批量查询和稳定 freshness。 |

轻量版足够应对一次性的公开 HLTV 数据查询。静态 JSON 或 API 版更适合需要重复分析、CT/T 数据、历史回测和正式产品化使用的场景。

轻量版依赖调用模型自己的网页读取能力。Claude、ChatGPT 或其他模型如果读不到 HLTV 深层 stats 页面，这是正常限制。此时 skill 应标记 `core_data_insufficient_for_numeric_inference`，并禁止输出具体胜率百分比。

## 两种使用形态

`hltv-cs2-data` 支持两种运行方式。

### 1. 轻量版 / 直接 HLTV 模式

适合只想安装 skill，然后直接让模型查看公开 HLTV 页面的人。

- 不需要 API key。
- 不需要用户自己维护数据库。
- 不需要用户打开本地浏览器、CDP、Playwright 或本地爬虫。
- 模型只用自身环境可用的网页读取 / 搜索能力收集 HLTV 数据。
- 历史回测如果没有精确快照，必须标记为 `reconstructed`。
- 缺失字段必须明确写出来，不能猜。

这是最容易上手的方式：别人拿到这个 skill 后，可以直接用比赛链接或队伍名发起查询。

### 2. 静态 JSON 模式

适合已经有维护好的静态数据导出。

示例结构：

```text
/latest/manifest.json
/latest/teams/6667/summary.json
/latest/matches/2393346/data-pack.json
/latest/events/8250/player-ratings.json
/latest/compare/g2-vs-faze.json
```

配置：

```text
HLTV_CS2_STATIC_BASE_URL=https://your-static-data.example.com/latest
```

在这个模式下，skill 应先读静态 JSON，再考虑 live HLTV 页面。

### 3. Pro / API 模式

适合接入维护好的 HLTV 数据 API。

配置：

```text
HLTV_CS2_API_BASE_URL=https://your-api.example.com
HLTV_CS2_API_KEY=your_api_key
```

在 API 模式下，skill 应该优先查询数据 API。API 负责从维护好的 collector 和数据库返回标准化 Markdown + JSON 数据包。

优势：

- 历史数据更可复现。
- 队伍和比赛查询更快。
- 可以统一维护阵容、veto、比分、rating、地图池快照。
- 可以通过 `as_of` 截止时间支持更干净的回测。

如果 API 不可用，只有在用户允许 fallback 或任务不强依赖历史快照时，才回退到直接 HLTV 模式。

## 安装 / 使用方式

这个仓库不是绑定某一个特定运行环境的。它本质上是一组结构化指令文件：

- 如果你的工具支持 skill 文件夹，就安装 `hltv-cs2-data/`。
- 如果你的工具支持 `.skill` 包，就安装 `hltv-cs2-data.skill`。
- 如果你的工具不支持 skill，也可以直接把 `hltv-cs2-data/SKILL.md` 当成主指令文件，再按任务需要加载 `hltv-cs2-data/references/` 里的对应文档。

### 方式一：安装 skill 文件夹

把 `hltv-cs2-data/` 文件夹复制到你的 skills 目录：

```bash
SKILLS_HOME=/path/to/skills
mkdir -p "$SKILLS_HOME"
cp -R hltv-cs2-data "$SKILLS_HOME/"
```

如果你的环境需要重新加载 skill，重启运行环境或刷新 skills。

### 方式二：安装 `.skill` 包

如果你的环境支持安装 `.skill` 包，可以直接使用：

```text
hltv-cs2-data.skill
```

这个包里包含同一份 `hltv-cs2-data/` skill 文件夹和 references 文档。

### 方式三：当普通指令文档使用

如果你的模型或工作流没有原生 skill 支持：

1. 把 `hltv-cs2-data/SKILL.md` 作为主 system/project instruction。
2. 根据任务只加载需要的 reference：
   - 比赛 / 队伍查询：`references/query-workflow.md`
   - 输出结构：`references/data-pack-contract.md`
   - 直接 HLTV 模式：`references/standalone-mode.md`
   - 历史回测：`references/backtest-mode.md`
3. 要求模型遵守 skill 里定义的“事实数据”和“模型推理”分离边界。

## 快速使用

### 1. 根据比赛链接生成数据包

提示词示例：

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Output Markdown and JSON.
```

预期行为：

- 识别 HLTV match ID。
- 解析两队、赛事、赛制、时间、状态。
- 尽量收集地图池、阵容、选手 rating、veto、比分、近期比赛数据。
- 对缺失数据明确标注。
- 输出事实数据包和 JSON。
- 如果年度地图池或 player rating 被 CF/页面读取失败阻断，不输出具体胜率百分比，只输出数据缺口和 API/数据仓建议。

### 2. 比较两个队伍

提示词示例：

```text
Use hltv-cs2-data to compare NAVI and G2.
Focus on map pool, player form, roster changes, and head-to-head data.
```

预期行为：

- 解析两个队伍的 HLTV 身份。
- 收集公开 HLTV 数据。
- 将事实因素整理进 `decision_inputs`。
- 默认不直接宣布谁赢，除非用户明确要求模型推理。

### 3. 数据 + 模型判断

提示词示例：

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

预期行为：

1. 先输出 HLTV 事实数据包。
2. 再输出 `Decision Inputs`。
3. 最后追加清晰标注的 `Model Inference`。
4. 明确说明胜率和判断是模型推理，不是 HLTV 事实数据。

### 4. 历史回测

提示词示例：

```text
Use hltv-cs2-data to backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai.
Only include data visible before that time.
```

预期行为：

- 严格使用 cutoff 时间。
- 如果当时比赛还没结束，就不能把赛后比分、赛后 rating、赛后地图记录混进去。
- 如果没有精确历史快照，要标注为 `reconstructed` 或 `unavailable`。

## 示例教程：用一场比赛生成轻量版数据包

下面是一条最常见的使用路径。用户只需要给出 HLTV 比赛链接，不需要知道 API 参数。

### 第一步：直接提问

```text
请使用 hltv-cs2-data 帮我整理这场比赛的数据：
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1

请输出中文 Markdown 和 JSON。
只整理事实数据，不要直接预测胜率。
如果有字段读取失败，请写清楚失败原因和对应 HLTV URL。
```

### 第二步：skill 应该做什么

轻量版会按这个顺序尝试读取：

1. 比赛页：match ID、队伍、赛事、赛制、时间、状态、可见阵容、veto、比分。
2. 队伍身份：队伍 HLTV ID、slug、可见排名和阵容信息。
3. Event ID：从比赛页赛事链接解析 event ID。
4. 赛事 rating：尝试读取 `https://www.hltv.org/stats/players?event=<eventId>`。
5. 年度选手 rating：尝试读取 `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`。
6. 年度地图 summary：尝试读取 `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`。
7. 如果深层 stats 页读取失败，保留 URL，并在对应字段写 `missing` 和 warning。

默认只取当前年份数据，例如 2026 年比赛就使用 `2026-01-01` 到 `2026-12-31`。如果用户要求其他时间窗口，再按用户指定的时间窗口读取。

### 第三步：推荐输出结构

中文 Markdown 建议包含这些段落：

| 段落 | 内容 |
|:--|:--|
| 数据状态 | 来源模式、读取时间、完整度、重要缺失字段 |
| 比赛信息 | match ID、赛事、时间、BO 几、状态 |
| 队伍与阵容 | 两队 HLTV ID、可见首发、教练、替补、阵容 warning |
| 选手数据 | 年度 rating、赛事 rating、缺失 rating |
| 地图池 | 2026 地图 summary、样本、W/D/L、胜率、Pick%、Ban% |
| 近期记录 / H2H | 能读取到的近期比赛和直接交手 |
| Veto / 比分 | 如果页面已显示，则输出 veto、地图顺序、比分 |
| 给模型的决策输入 | 只放事实特征，不放胜率结论 |
| 数据缺口 | 哪些字段没有拿到，为什么没有拿到 |
| JSON | 稳定英文 key，方便程序或另一个模型继续使用 |

### 第四步：如果用户要求模型判断

如果用户说“根据这些数据判断谁胜率高”，输出应该变成：

1. 先给 HLTV 事实数据包。
2. 再给 `decision_inputs`。
3. 最后追加 `Model Inference`。

`Model Inference` 可以写胜率判断，但必须明确：这是调用模型基于数据做出的推理，不是 HLTV 原始数据。

### 第五步：cache miss 怎么读

`cache miss` 不是“HLTV 没有这个数据”，也不是“URL 错了”。它表示当前轻量版读取环境没有拿到可解析页面快照。

推荐写法：

```json
{
  "field": "event_player_ratings",
  "source_url": "https://www.hltv.org/stats/players?event=8250",
  "status": "missing",
  "warning": "fetch_failed_cache_miss",
  "meaning": "The URL is known, but lightweight direct mode could not retrieve a readable table snapshot."
}
```

## 输出分层

这个 skill 把输出分成三层。

### 第一层：HLTV Data Pack

这是事实层。它可以包含：

- `metadata`：查询类型、检索时间、数据截止时间、数据源策略、warnings。
- `teams`：队伍名称、别名、HLTV ID、排名快照。
- `match`：赛事、赛制、时间、LAN/online、比赛状态。
- `lineups`：首发、教练、替补、阵容缺失提示。
- `players`：年度 rating、赛事 rating、rating 缺失状态。
- `maps`：地图池数据、样本量、原始胜率、加权胜率。
- `head_to_head`：两队历史交手。
- `recent_matches`：近期比赛和地图记录。
- `veto`：veto 步骤和地图顺序。
- `scores`：地图比分和比赛结果。
- `warnings`：小样本、数据过期、字段缺失、解析限制。
- `not_included`：明确哪些字段不属于事实数据包。

### 第二层：Decision Inputs

`decision_inputs` 是给模型使用的事实特征，不是预测结论。

常见分组：

- `map_pool`
- `head_to_head`
- `player_form`
- `roster_state`
- `match_context`
- `data_quality`

这些字段的意义是：把事实整理成大模型容易使用的结构。不同用户、不同模型可以对这些输入赋予不同权重。

### 第三层：可选 Model Inference

`model_inference` 只有在用户明确要求判断、概率或策略时才出现。

它可以包含：

- 单图胜率判断。
- 全场胜率判断。
- 倾向赢家。
- 推理摘要。
- 不确定性说明。

它必须和事实字段分开，不能混入 `teams`、`maps`、`players`、`decision_inputs` 等事实对象里。

## JSON 结构示例

```json
{
  "schema_version": "hltv-cs2-data.v1",
  "metadata": {
    "query_type": "match_data_pack",
    "requested_at": "2026-05-01T12:00:00Z",
    "data_cutoff": "2026-05-01T12:00:00Z",
    "source_policy": "direct_hltv",
    "missing_fields": [],
    "warnings": []
  },
  "match": {
    "hltv_match_id": 2393346,
    "hltv_url": "https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1",
    "event_name": null,
    "format": "bo3",
    "status": "scheduled"
  },
  "teams": [],
  "lineups": [],
  "players": [],
  "maps": [],
  "head_to_head": {
    "status": "not_loaded",
    "matches": []
  },
  "recent_matches": {},
  "veto": {
    "status": "unavailable",
    "steps": []
  },
  "scores": {
    "status": "not_started",
    "maps": []
  },
  "decision_inputs": {
    "map_pool": [],
    "head_to_head": [],
    "player_form": [],
    "roster_state": [],
    "match_context": [],
    "data_quality": []
  },
  "not_included": [
    "betting_ev",
    "stake_sizing",
    "private_strategy_rules"
  ],
  "model_inference": null
}
```

## Direct HLTV 模式

Direct HLTV mode 是默认模式。

它适合只安装 skill、没有私有 API、没有本地数据库、没有爬虫服务、也不想配置本地浏览器/CDP 的用户。模型会用自身环境可用的网页读取 / 搜索能力读取公开 HLTV 页面，并尽量输出可用的数据包。

Direct mode 的限制：

- 有些 HLTV 页面可能无法访问。
- 赛事 player rating、年度 player stats、年度 team map summary 这类深层 stats 页可能出现 `cache miss` 或 CF challenge；这不是 URL 错，也不代表 HLTV 没有数据。
- 赛前阵容不一定可见。
- veto 通常要临近开赛或比赛开始后才出现。
- 没有中央数据仓时，`as_of` 回测无法保证精确历史快照。
- CT/T 地图侧胜率需要逐张地图详情页，轻量版不默认抓；应由 API / warehouse 或内部 collector 提供。
- CT/T 分边比分、半场比分、回合级数据属于 Phase 2，不是第一版必备字段。

当数据缺失时，skill 应该标注缺失，而不是靠直觉补齐。

## 数据可获取性

| 数据 | 轻量版是否提供 |
|:--|:--|
| 比赛信息 | ✅ |
| 队伍 ID / 阵容 | ✅ |
| Event ID | ✅ |
| event rating | 尝试，失败就标缺 |
| 年度选手 rating | 尝试，失败就标缺 |
| 2026 地图 summary | ✅，优先读年度 team map summary；失败时可退回 match page core 数据并明确标注 |
| W/D/L、Win rate、Pick%、Ban% | ✅，来自 2026 地图 summary |
| CT/T 胜率 | ❌ 只在 API / 数据库增强版提供 |
| 历史回测快照 | ❌ 只在 API / 数据库增强版提供 |

轻量版失败时使用这些 warning：

- `fetch_failed_cache_miss`
- `fetch_failed_cf_challenge`
- `stats_page_unavailable_in_direct_mode`
- `pro_api_required_for_full_coverage`

## API / Warehouse 模式

这个 skill 也定义了未来产品化的数据架构：

```text
HLTV -> collector -> central warehouse/API -> hltv-cs2-data skill -> downstream model
```

这个模式适合：

- 可重复的历史回测。
- 保存 HLTV 原始快照。
- 更强的数据新鲜度保证。
- 采集 match detail、veto、lineup、result。
- 给多个用户或多个模型共享同一套数据 API。

但是，API / warehouse 不是第一版使用本 skill 的前提。没有 API 时，Direct HLTV mode 仍然可以工作。

## References 说明

skill 使用 progressive disclosure。`SKILL.md` 保持简洁，只在需要时加载 references。

- `references/product-brief.md`：产品定位和边界。
- `references/standalone-mode.md`：直接使用 HLTV 页面的模式。
- `references/data-availability.md`：轻量版、浏览器会话、collector、API 各自能提供哪些数据。
- `references/data-pack-contract.md`：Markdown 和 JSON 输出契约。
- `references/decision-inputs.md`：模型输入特征结构。
- `references/query-workflow.md`：常见查询流程。
- `references/backtest-mode.md`：历史回测和 cutoff 规则。
- `references/api-contract.md`：可选 API 设计。
- `references/collector-contract.md`：可选 collector / warehouse 设计。

## 数据边界

事实数据包可以包含描述性历史指标，例如：

- 地图原始胜率。
- 数据源已有的地图加权胜率。
- Pick / ban 百分比。
- 样本量。
- 选手 rating。
- 近期比赛记录。

这些是历史事实或描述性派生指标，不是预测。

预测概率、胜负判断、比分预测、策略建议，只能出现在 `model_inference`，而且必须是用户明确要求后才输出。

## 安全和访问边界

- 只使用公开 HLTV 页面，或显式配置的 HLTV-derived API。
- 不绕过访问控制。
- 如果遇到 Cloudflare 或访问限制，输出 partial pack 和 warnings。
- 不编造缺失数据。
- 不暴露私有凭证、内部数据库表、隐藏策略规则。

## 开发和发布检查

发布前校验 skill：

```bash
python3 /path/to/skill-creator/scripts/quick_validate.py hltv-cs2-data
```

建议的隐私检查：

```bash
find . -name .DS_Store -print
rg -n "credential|local path|private display URL|hidden strategy name" .
```

文档可以泛化地提到私有系统边界，但不应该包含真实私有系统名称、真实 URL、凭证或本地路径。

## License

MIT
