# hltv-cs2-data Skill 中文说明

`hltv-cs2-data` 是一个 skill，用来读取 HLTV 衍生的结构化数据库 / 静态 JSON 数据，并输出给大模型、分析师或产品系统使用的数据包。

这个 skill 的核心定位是：只提供标准化数据，不内置策略，也不输出预测。它负责把 HLTV 衍生的事实数据整理成普通 Markdown 报告，并把对模型有用的事实特征组织成 `decision_inputs`。只有用户明确要求机器可读、下游模型、debug/audit 或 JSON 时，才输出 JSON。是否预测胜率、怎么判断地图优势、怎么制定策略，由调用它的大模型或用户自己的策略决定，不由本 skill 输出。

当前公开数据源版本：`static-raw-2026-05-06`。

必须读取的默认 manifest：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

这是唯一默认公开静态数据库入口。不要从记忆里推导、替换或尝试其他 host。来自其他 host 的 404 不能说明数据库不可用。

如果外部模型报告的是 GitHub Pages / 平台站点地址 404，而不是上面的 raw GitHub manifest，说明它使用了旧指令、旧 skill 安装包或缓存上下文。请重新安装最新版 `.skill`，并先要求它复述 `当前公开数据源版本` 和 `默认 manifest URL`。

## 适合什么场景

当你想让大模型分析一场 CS2 比赛，但又不想让模型凭空编数据时，可以使用这个 skill。

典型场景：

- 输入一条 HLTV 比赛链接，获取这场比赛的数据包。
- 输入两个队伍，例如 `NAVI 和 G2`，比较两队的地图池、选手状态、阵容变化和历史交手。
- 输入一个队伍，例如 `Aurora`，输出单队地图池、选手 rating 和逐图详细数据。
- 假设一场不存在或未公布的比赛，例如 `假设 NAVI 和 G2 打 S 级 LAN BO3`，输出双队结构化数据包和假设条件。
- 查看不同地图胜率、每张地图的警匪胜率、对位数据和选手近期状态。
- 输入赛事 rating 页面，补充选手在当前赛事里的 rating。
- 做历史回测，要求只使用某个时间点之前可见的数据。
- 把结构化 Markdown 或按需输出的 JSON 交给另一个模型或策略系统继续判断。

这个 skill 可以服务预测流程，但本身不直接预测。即使用户问“谁胜率高 / 谁更可能赢”，skill 也只输出事实数据包、逐图数据和 `decision_inputs`，最后提示：`本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`

若 HLTV 被 Cloudflare 拦截，或当前年度地图池 / 选手 rating 没有加载成功，skill 应输出部分数据包和缺失字段，不应给出具体胜率区间或方向性结论。

用户可读部分默认跟随用户语言。中文提问时，输出应使用中文标题、中文表格标签和中文 warnings；JSON 字段名继续保持英文，方便程序和其他模型稳定读取。

## 关键合规规则

### 第一动作：HLTV 只定位身份，分析数据必须查结构化数据库

模型不能只读 HLTV 页面就开始写分析正文。HLTV 的作用是定位 match ID、team ID、event ID、首发阵容、赛制、比赛阶段、已公开 Veto/比分等可见事实；地图池、选手 rating、CT/T、手枪局、首杀首死、Pick/Ban、近期记录等分析字段必须从结构化数据库/API/静态 JSON 中读取。

外部公开来源可以用于补充比赛背景，但不能替代结构化数据。允许的背景字段包括：首发阵容、替补/教练、赛制、比赛阶段、赛程时间、LAN/online、场馆/国家、淘汰赛/小组赛位置、公开阵容变动说明。若这些信息不是来自 HLTV 或数据库，必须标记为 `external_context`。这些来源不能用于地图胜率、样本、CT/T、手枪局、首杀首死、Pick/Ban、选手 rating、H2H、近期地图记录、Veto、比分或赛果。

每次比赛 / 双队对比查询都必须先完成：

1. 用 HLTV 定位 match/team/event ID。
2. 读取公开静态数据库 manifest：
   `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`
   - 不要把 `.../public-data` 目录当成 API 地址直接读取。GitHub raw 目录返回 404 是正常的，不代表数据库不存在。
   - 必须读取 `/manifest.json` 或具体 JSON 文件。
3. 根据 ID 读取准确记录，例如：
   - `teams/<hltvTeamId>/summary.json`
   - `teams/<hltvTeamId>/players.json`
   - `teams/<hltvTeamId>/map-details-overall.json`
   - `teams/<hltvTeamId>/map-details-lan.json`
   - `matches/<hltvMatchId>/data-pack.json`
4. 然后才可以输出地图池、选手 rating、逐图详细分析和决策输入。

如果第 2 或第 3 步失败，必须停止，只输出：

```text
当前只能定位 HLTV 基础信息；结构化数据库/API/静态 JSON 未成功读取，不能生成完整 hltv-cs2-data 报告。
warning: structured_database_not_queried
```

下面这些输出都是不合规的：

- 只写 `数据来源：HLTV.org 官方数据`，没有 `结构化数据：已读取...`。
- 没有静态数据库记录，却输出完整地图池 / 逐图详细分析。
- 普通报告里默认输出完整 JSON。
- 用搜索摘要、新闻、wiki、盘口页面替代数据库字段。

### 404 排查规则

出现 404 时，先判断访问的到底是什么：

| 访问目标 | 解释 | 正确处理 |
|:--|:--|:--|
| raw GitHub 的 `/public-data` 目录 | 目录不是 JSON 文件，返回 404 正常 | 改读 `/public-data/manifest.json` |
| raw GitHub 的 `/manifest.json` | 这是唯一默认入口 | 若失败，标记 `structured_database_unavailable` |
| raw GitHub 的具体 JSON 文件 | 说明该记录可能未导出 | 回到 manifest / index 查可用路径 |
| GitHub Pages / 平台站点派生地址 | 旧文档或模型记忆 | 不要继续使用，重新安装最新版 skill |

如果不能成功读取 manifest 和至少一个准确 JSON 记录，不能输出完整数据包、逐图分析或胜率方向。只能输出 HLTV 基础事实和 `structured_database_not_queried`。

正确输出必须在靠前位置包含 `数据源执行记录`。

对外部模型来说，合规流程应该是：

```text
用户问题 / HLTV URL
  -> HLTV 最小定位：matchId、teamId、eventId、可见阵容/比分
  -> 读取 public-data/manifest.json
  -> 读取 matches/<matchId>/data-pack.json 或 teams/<id>/*.json
  -> 只用这些结构化记录写地图池、选手 rating、逐图详细分析和决策输入
  -> 不输出胜率判断、Veto 预测、比分预测或投注建议
```

如果模型从 HLTV 页面直接跳到完整分析，或者只说“数据来源 HLTV.org”，就是没有正确使用本 skill。

没有 HLTV match URL 不代表不能用。单队查询和假设比赛都应该走数据库记录：

- 单队查询：解析 team ID，然后读取 `teams/<id>/summary.json`、`maps-overall.json`、`maps-lan.json`、`map-details-overall.json`、`map-details-lan.json`、`players.json`。
- 假设比赛 / 双队对比：解析两个 team ID，然后读取双方同样的 team records。用户给出的 BO3/BO5、LAN/online、赛事级别、日期等只作为 `assumptions`。
- 没有真实 match ID 时，Veto、地图顺序、比分、赛事 rating 和官方首发阵容默认写 `not_applicable` 或 `未加载`，不能编。

HLTV 比赛页上的首发阵容可以是正确的，也允许作为阵容来源。但这只说明身份 / 阵容字段完成了，不代表地图、rating、CT/T、手枪局、首杀首死、Pick/Ban 数据已经加载。只要没有成功读取静态数据库/API 的准确记录，输出仍然只能算 `direct_hltv_partial`，不能生成完整逐图分析或具体胜率。

别名必须先回到 HLTV 官方身份。比如 `M8` / `Gentle Mates` 在对应比赛页里应解析为 HLTV team `13404`；不能误当成 `M80` (`12376`) 或其他字符串相近的队伍。

这段记录必须写清：

- 用哪个 HLTV match / team / event 页面完成身份定位。
- 查询了哪个公开数据库 manifest 或 API base。
- 至少读取了一个准确的结构化数据路径，例如 `matches/2394116/data-pack.json`、`teams/11861/map-details-overall.json`、`events/8250/player-ratings.json`。
- 字段来源，例如 `direct_hltv`、`static_database`、`api_warehouse`、`direct_hltv_fallback`、`missing`。

如果模型只读取 HLTV、Liquipedia/wiki、新闻片段、搜索摘要或盘口页面，就不算正确使用本 skill。此时必须标记为部分数据，加入 warning `structured_database_not_queried`，并且不能输出完整数据包、逐图详细分析、Veto 预测或具体胜率百分比。

地图池也必须只使用结构化数据里的当前 active map pool。当前 2026 公开导出的七张图是：`Ancient`、`Anubis`、`Dust2`、`Inferno`、`Mirage`、`Nuke`、`Overpass`。不要把 `Vertigo`、`Cache`、`Train` 等不在结构化记录里的地图写进 `地图池总览`、`逐图详细分析` 或 `特殊 Veto 变量`。

用户问“谁胜率高 / 谁更强 / 谁更可能赢”时，仍然只输出数据，结构必须固定：

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息`
4. `队伍与选手 rating`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `给模型的决策输入`
9. `JSON`，仅用户要求机器可读 / 下游模型 / debug / 显式 JSON 时输出

`Veto 预测`、具体胜率百分比、比分猜测、倾向赢家、投注建议都不属于本 skill。事实区的 `Veto / 比分` 只能写已经观察到的 veto、地图顺序、比分或 `赛前不可见`。

这个限制只保护数据层。外部模型可以拿 `decision_inputs` 自己设计判断方式，例如地图权重、状态权重、阵容影响、Veto 假设、胜率估计等；但这些判断不由本 skill 输出。不能把模型自己算出来的指标写成 HLTV 原始事实，也不能用缺失字段编出数字。

例如 `最近30天状态`、`赛事参赛分布`、`最近对手质量`、`地图池深度` 可以做成描述性数据摘要，但前提是说明数据窗口、字段来源和计算口径；如果数据库没有对应行，就写 `未加载`。

普通报告不需要展示原始 manifest URL、GitHub raw URL、数据库 record path 或完整 JSON。只需要展示简短来源状态，例如 `结构化数据：已读取 public static database export`。只有用户明确要求 debug、审计、source details、JSON 或给下游模型使用时，才输出具体 URL、路径和 JSON。

每个地图数字必须来自精确的队伍-地图记录。如果某队某图没有记录，就写 `无数据`，不能从对手数据、其他地图、搜索摘要或整体状态里补。例如某场数据包里没有 `Heroic + Anubis` 行，就必须显示 Heroic Anubis `无数据`。

举例：如果查询 `MOUZ vs Gentle Mates`，HLTV 定位后应尝试读取：

- `matches/<matchId>/data-pack.json`
- `teams/4494/summary.json`、`teams/4494/map-details-overall.json`、`teams/4494/map-details-lan.json`、`teams/4494/players.json`
- `teams/13404/summary.json`、`teams/13404/map-details-overall.json`、`teams/13404/map-details-lan.json`、`teams/13404/players.json`

如果这些记录没有读取成功，即使首发阵容来自 HLTV 且看起来正确，也不能输出完整报告。

本 skill 不输出投注建议、赔率分析、EV、Kelly、仓位、胜率预测、Veto 预测或比分预测。

## 不做什么

这个仓库不提供：

- 私人 CS2 预测模型。
- 投注建议。
- EV、Kelly、最高买入价、仓位计算。
- 绕过访问控制的爬虫服务。
- 本地数据库或私人数据依赖。
- 在纯 HLTV 页面模式下保证完整历史快照。

普通公开用户默认不需要配置数据源。skill 会先用 HLTV 页面定位和确认比赛、队伍、赛事。拿到 match ID / team ID / event ID 后，必须查询结构化数据层。没有 API 时，默认结构化数据源就是公开静态 JSON 数据库导出：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

如果 HLTV 找到了比赛页，数据库导出用于加载地图详情、CT/T、选手 rating、导出的 match pack 等结构化字段。如果 HLTV 找不到或读不到比赛页，才用 manifest 反查已导出的比赛 / 队伍记录。API / 数据仓可以提供更完整、更可复现的历史数据，但这不是安装本 skill 的前提。

## 产品分层

这个 skill 可以按两层使用：

| 层级 | 适合场景 | 预期能力 |
|:--|:--|:--|
| 公开独立版 | HLTV 定位 + 公开静态数据库补全 | 默认公开路径。HLTV 只负责找 match/team/event ID；地图、选手、CT/T 等结构化字段必须来自公开静态 JSON 数据库或 API。 |
| 静态 JSON 数据库导出版 | 给 Claude/GPT/用户模型共享数据包 | 没有 API 时必须查询的公开结构化数据源。提供队伍、比赛、赛事、地图详情和 compare pack。 |
| API 版 | 可重复分析、批量使用、正式产品 | 推荐用于完整年度数据、CT/T 警匪胜率、精确历史回测、阵容/veto/赛果快照、批量查询和稳定 freshness。 |

Direct HLTV-only 只够做可见事实的部分兜底，不足以生成完整报告、逐图详细分析、Veto 预测或具体胜率百分比。完整报告必须读取静态 JSON 或 API。

公开独立版依赖调用模型自己的网页读取能力完成 HLTV 定位，并依赖公开静态 JSON 数据库完成结构化数据补全。Claude、ChatGPT 或其他模型如果读不到 HLTV 深层 stats 页面，这是正常限制；不能因此用搜索摘要补数据。此时 skill 应标记 `core_data_insufficient_for_numeric_inference`，并禁止输出具体胜率百分比。

## 使用形态

`hltv-cs2-data` 支持三种运行方式。

### 1. 先 HLTV 定位，再查数据库导出

这是默认公开模式。别人安装 skill 后，可以直接自然语言提问，不需要每次写数据地址。

示例：

```text
用 hltv-cs2-data 帮我看一下 FaZe 和 G2 谁胜率高
```

skill 应先去 HLTV 定位比赛页或队伍页。例如 `PGL 上 Aurora 和 Heroic 谁胜率高`，要先搜索 / 读取 HLTV 的 match、event、results、upcoming 页面，找到确切比赛页。拿到 match ID / team ID / event ID 后，必须停止 HLTV 深层统计浏览，转去读取默认数据库 manifest：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

基础路径前缀是：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data
```

不要把基础目录当 JSON 读取；必须先追加 `/manifest.json`。基础目录只是路径前缀，不是 JSON 文件；某些运行环境直接读取基础目录会返回 404，这不代表数据不存在。

示例结构：

```text
/manifest.json
/teams/index.json
/teams/6667/summary.json
/teams/6667/maps-overall.json
/teams/6667/map-details-overall.json
/matches/2393346/data-pack.json
/events/8250/player-ratings.json
```

如果用户想使用自己的静态数据源，可以配置：

```text
HLTV_CS2_STATIC_BASE_URL=https://your-static-data.example.com/latest
```

在这个模式下，skill 应用 live HLTV 做比赛定位和可见事实确认，再用静态 JSON 数据库导出读取结构化字段。只有数据库导出已经尝试且不可用或缺字段时，才把 HLTV 深层 stats 页作为补充 fallback。不要把“HLTV 页面能打开”当成数据已经加载完成。

不要使用 Liquipedia / Liquidpedia / wiki / 新闻片段 / 搜索摘要替代地图数据、选手 rating、CT/T、veto 或赛果字段。

如果用户直接问“谁胜率高 / 谁更强 / 哪边更占优”，skill 仍然先输出事实数据包。只要静态数据里有地图详情，输出必须包含：

- `地图池总览`：七张地图的样本、胜率、Pick%、Ban% 总表。
- `逐图详细分析`：逐张可打地图拆解样本、LAN/overall、CT/T、手枪局、首杀后、首死后、回合数、Pick/Ban。
- `特殊 Veto 变量`：一方不打、无 2026 数据、高 Ban%、极小样本的地图单独列出，不混入普通地图平均。
- `给模型的决策输入`：只放事实特征，不放胜率结论。

### 2. 轻量版 / 直接 HLTV 模式

适合默认静态源缺数据，或者用户只想让模型查看公开 HLTV 页面的时候。

- 不需要 API key。
- 不需要用户自己维护数据库。
- 不需要用户打开本地浏览器、CDP、Playwright 或本地爬虫。
- 模型只用自身环境可用的网页读取 / 搜索能力收集 HLTV 数据。
- 历史回测如果没有精确快照，必须标记为 `reconstructed`。
- 缺失字段必须明确写出来，不能猜。

### 3. Pro / API 模式

适合接入维护好的 HLTV 数据 API。

配置：

```text
HLTV_CS2_API_BASE_URL=https://your-api.example.com
HLTV_CS2_API_KEY=your_api_key
```

在 API 模式下，skill 应该优先查询数据 API。API 负责从维护好的 collector 和数据库返回标准化数据包。普通报告默认输出 Markdown；只有用户明确要求机器可读、下游模型、debug/audit 或 JSON 时才输出 JSON。

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

### 安装后冒烟测试

安装或更新 skill 后，先用这句测试，不要直接跑完整报告：

```text
使用 hltv-cs2-data。先复述 skill 里的当前公开数据源版本和默认 manifest URL，然后读取 manifest。不要使用 GitHub Pages 或平台站点派生地址。
```

预期结果：

- 数据源版本是 `static-raw-2026-05-06`。
- manifest URL 是 `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`。
- 模型不应该提到 GitHub Pages / 平台站点的 public-data 地址。

如果模型使用了其他 host，说明它仍在使用旧 skill、旧安装包或旧会话缓存。需要重新安装 `.skill`，并开新对话测试。

### 方式三：当普通指令文档使用

如果你的模型或工作流没有原生 skill 支持：

1. 把 `hltv-cs2-data/SKILL.md` 作为主 system/project instruction。
2. 根据任务只加载需要的 reference：
   - 比赛 / 队伍查询：`references/query-workflow.md`
   - 输出结构：`references/data-pack-contract.md`
   - 直接 HLTV 模式：`references/standalone-mode.md`
   - 历史回测：`references/backtest-mode.md`
3. 要求模型遵守 skill 里定义的“只输出数据层、不输出预测层”边界。

## 快速使用

### 1. 根据比赛链接生成数据包

提示词示例：

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Output Markdown only.
```

预期行为：

- 识别 HLTV match ID。
- 解析两队、赛事、赛制、时间、状态。
- 在解析身份后读取公开静态数据库 manifest 或配置好的 API。
- 输出 `数据源执行记录`，证明已经读取结构化数据库/API/静态 JSON。普通报告不显示原始 URL 和准确路径。
- 尽量收集地图池、阵容、选手 rating、veto、比分、近期比赛数据。
- 对缺失数据明确标注。
- 输出事实 Markdown 报告；JSON 只在用户要求机器可读、下游模型或 debug/audit 时输出。
- 如果年度地图池或 player rating 被 CF/页面读取失败阻断，不输出具体胜率百分比，只输出数据缺口和 API/数据仓建议。

### 2. 比较两个队伍

提示词示例：

```text
Use hltv-cs2-data to compare NAVI and G2.
Focus on map pool, player form, roster changes, and head-to-head data.
```

预期行为：

- 解析两个队伍的 HLTV 身份。
- 拿到 team ID 后，查询公开静态 JSON 数据库导出或配置好的 API，读取队伍、地图、选手数据。
- 输出 manifest/API 状态和实际读取的数据库记录路径。
- 将事实因素整理进 `decision_inputs`。
- 默认不直接宣布谁赢；即使用户要求判断，也只输出数据和决策输入。

### 2.1 单队数据包

提示词示例：

```text
用 hltv-cs2-data 看一下 Aurora 这个队伍的数据。
```

预期行为：

- 解析 Aurora 的 HLTV team ID。
- 读取公开静态数据库里的单队记录。
- 输出队伍信息、选手 rating、地图池总览、逐图详细数据、近期地图记录和数据缺口。
- 不要求 match ID；没有对手、Veto、比分、赛事 rating 时写 `not_applicable`。

### 2.2 假设比赛 / 双队对比

提示词示例：

```text
假设 NAVI 和 G2 打一场 S 级 LAN BO3，用 hltv-cs2-data 给我数据包。
```

预期行为：

- 解析两队 team ID。
- 读取双方 team records。
- 把 `S 级`、`LAN`、`BO3` 作为 `假设条件`，不是 HLTV 已观察事实。
- 输出双方地图池、逐图详细数据、选手 rating、特殊 Veto 变量和给模型的决策输入。
- 不输出胜率结论、Veto 预测或比分预测。

### 3. 比较两个队伍并准备给外部模型判断

推荐提示词：

```text
用 hltv-cs2-data 帮我看一下 PGL Aurora 和 Heroic 谁胜率高。
只输出事实数据和给模型的决策输入，不要给预测。
```

预期行为：

1. 先用 HLTV 定位相关比赛 / 队伍身份，再查询 API 或公开静态 JSON 数据库导出，读取结构化地图、选手、警匪数据。
2. 输出 `地图池总览`，展示七张图的年度 overall / LAN 样本、W-D-L、胜率、Pick%、Ban%。
3. 输出 `逐图详细分析`，每张可打地图都要拆：
   - 样本可信度。
   - overall / LAN 表现。
   - CT/T 胜率。
   - 手枪局胜率。
   - 首杀后 / 首死后回合胜率。
   - Pick/Ban 倾向。
   - 这张图的数据对位结论。
4. 输出 `特殊 Veto 变量`，例如一方固定 ban、无 2026 数据、样本只有 1 张图的地图。
5. 最后给 `给模型的决策输入`，不输出谁更占优。

注意：这不是预测模型。skill 只是组织数据；最终判断由调用模型或用户策略完成。

### 4. 数据-only 输出

提示词示例：

```text
Use hltv-cs2-data to collect G2 vs FaZe data. Return factual data only.
```

预期行为：

1. 先输出 HLTV 事实数据包。
2. 再输出 `Decision Inputs`。
3. 如果有地图详情数据，必须先输出 `逐图详细分析` 和 `特殊 Veto 变量`。
4. 不追加 `Model Inference`。
5. 如果用户问预测，补一句：`本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`

### 5. 历史回测

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

请输出中文 Markdown。
只整理事实数据，不要直接预测胜率。
如果有字段读取失败，请写清楚失败原因和 warning code。普通报告不要显示原始 URL 和数据库路径。
```

### 第二步：skill 应该做什么

公开独立版会按这个顺序尝试读取：

1. 比赛页：match ID、队伍、赛事、赛制、时间、状态、可见阵容、veto、比分。
2. 队伍身份：队伍 HLTV ID、slug、可见排名和阵容信息。
3. 公开静态数据库 manifest：`public-data/manifest.json`。
4. 准确静态记录：match data-pack、team summary、map-details、players、event ratings。
5. 只有静态数据库缺字段时，才把 HLTV 年度 stats 页作为补充 fallback。
6. 如果结构化数据完全没读到，必须停在部分事实，输出 warning `structured_database_not_queried`。

默认只取当前年份数据，例如 2026 年比赛就使用 `2026-01-01` 到 `2026-12-31`。如果用户要求其他时间窗口，再按用户指定的时间窗口读取。

### 第三步：推荐输出结构

中文 Markdown 建议包含这些段落：

| 段落 | 内容 |
|:--|:--|
| 数据状态 | 来源模式、读取时间、完整度、重要缺失字段 |
| 比赛信息 | match ID、赛事、时间、BO 几、状态 |
| 队伍与阵容 | 两队 HLTV ID、可见首发、教练、替补、阵容 warning |
| 选手数据 | 年度 rating、赛事 rating、缺失 rating |
| 地图池总览 | 2026 地图 summary、样本、W/D/L、胜率、Pick%、Ban% |
| 逐图详细分析 | 每张可打地图的 overall/LAN、CT/T、手枪、首杀后、首死后、回合数、Pick/Ban、数据对位 |
| 特殊 Veto 变量 | 一方不打、无年度数据、高 Ban%、极小样本的地图，单独列出 |
| 近期记录 / H2H | 能读取到的近期比赛和直接交手 |
| Veto / 比分 | 如果页面已显示，则输出 veto、地图顺序、比分 |
| 给模型的决策输入 | 只放事实特征，不放胜率结论 |
| 数据缺口 | 哪些字段没有拿到，为什么没有拿到 |
| JSON | 仅用户明确要求时输出；稳定英文 key，方便程序或另一个模型继续使用 |

### 第四步：如果用户要求模型判断

如果用户说“根据这些数据判断谁胜率高”，本 skill 仍然只输出：

1. 先给 HLTV 事实数据包。
2. 再给 `decision_inputs`。
3. 最后追加一句边界说明：`本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`

### 第五步：cache miss 怎么读

`cache miss` 不是“HLTV 没有这个数据”，也不是“URL 错了”。它表示当前轻量版读取环境没有拿到可解析页面快照。

推荐写法：

```json
{
  "field": "event_player_ratings",
  "source_url": "https://www.hltv.org/stats/players?event=8250",
  "status": "missing",
  "warning": "fetch_failed_cache_miss",
  "meaning": "The URL is known, but direct fallback could not retrieve a readable table snapshot."
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
- `maps`：地图池数据、样本量、原始胜率、加权胜率、Pick/Ban。
- `map_details`：逐图详情，例如总回合、赢回合、CT/T、手枪局、首杀后、首死后。
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

### 第三层：不包含预测层

本 skill 不输出 `model_inference`。如果用户或其他模型需要预测，可以把第一层事实数据包和第二层 `decision_inputs` 交给自己的模型处理。

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
    "private_strategy_rules",
    "model_inference"
  ]
}
```

## Direct HLTV 部分兜底模式

Direct HLTV-only 不是完整分析的默认路径，只是结构化静态数据库/API 读取失败时的部分兜底。

它适合模型只能读取公开 HLTV 页面、但无法读取静态 JSON 数据库或 API 的情况。此时只能输出可见比赛事实和数据缺口，不能输出完整报告、逐图详细分析、Veto 预测或具体胜率百分比。

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

| 数据 | 公开独立版是否提供 |
|:--|:--|
| 比赛信息 | ✅，来自 HLTV 定位或导出的 match pack |
| 队伍 ID / 阵容 | ✅，可见或已导出时提供 |
| Event ID | ✅，可解析或已导出时提供 |
| event rating | 静态 JSON/API 已导出时提供；Direct fallback 可尝试，失败就标缺 |
| 年度选手 rating | 静态 JSON/API 已导出时提供；Direct fallback 可尝试，失败就标缺 |
| 2026 地图 summary | 静态 JSON/API 已导出时提供；Direct fallback 可尝试并明确标注 |
| W/D/L、Win rate、Pick%、Ban% | 来自静态 JSON/API 精确行 |
| CT/T 胜率 | 静态 JSON/API 已导出时提供 |
| 历史回测快照 | ❌ 只在 API / 数据库增强版提供 |

Direct fallback 失败时使用这些 warning：

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

但是，API / warehouse 不是第一版使用本 skill 的前提。没有 API 时，应使用公开静态 JSON 数据库作为结构化数据源；Direct HLTV-only 只能作为部分兜底。

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

预测概率、胜负判断、比分预测、策略建议不属于本 skill。需要判断时，把事实数据包和 `decision_inputs` 交给外部模型或用户策略处理。

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
