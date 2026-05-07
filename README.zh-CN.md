# hltv-cs2-data Skill 中文说明

[English README](README.md)

`hltv-cs2-data` 是一个数据优先的 CS2 skill。它把 HLTV 衍生的结构化数据库 / 静态 JSON 记录整理成简洁的数据包，供用户、其他大模型或下游策略系统使用。

它只规定数据层：触发 skill 后必须先输出事实数据包。数据包之后怎么分析、判断或使用，是使用者和宿主模型自己的事情，不由本 skill 规定。

## 结构化数据源模型

本 skill 绑定的是**结构化数据能力契约**，不是某一个固定网站或固定 JSON 路径。

可用数据源优先级：

1. 已配置的 API / warehouse。
2. 用户提供的静态 JSON 导出。
3. 默认公开 raw GitHub 静态导出。

任何数据源只要能提供等价能力，就可以替换默认源：

- 队伍 / 比赛身份解析或索引。
- match data pack。
- team map summary。
- team map details。
- player ratings。
- event ratings。
- 数据缺口 / warnings。

当前默认公开静态数据的新鲜度以 `manifest.generated_at` 为准。每天更新静态数据库时，不需要同步更新 skill 文档本身。

默认公开源 manifest：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

这是当前仓库内置的首选公开入口。它不是唯一可用数据源；如果用户配置了 API 或提供了其他静态 JSON 源，应优先使用配置源。

如果宿主模型无法读取 raw GitHub，可以尝试这些兼容入口：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/latest/manifest.json
https://smallmeji.github.io/hltv-cs2-data-skill/public-data/latest/manifest.json
```

注意：`hltv-cs2-data-platform` 是旧平台仓库名，不是公开数据源。任何 `smallmeji.github.io/hltv-cs2-data-platform/public-data/latest` 的 404 都是旧路径问题，不代表数据源不可用。

明确禁止作为结构化数据源：

- 非兼容入口的 `smallmeji.github.io` URL
- 非兼容入口的 GitHub Pages URL
- 平台网站页面
- `hltv-cs2-data-platform` 的 public-data 路径
- `https://raw.githubusercontent.com/.../public-data` 目录本身
- 搜索摘要、wiki、盘口页面、新闻片段或模型记忆

如果模型说返回 `404`，先看它访问的是什么：

| 访问目标 | 含义 | 正确处理 |
|:--|:--|:--|
| `/public-data` 目录 | 目录不是 JSON 文件 | 改读 `/public-data/manifest.json` |
| `/manifest.json` | 必须读取的默认入口 | 失败则标记结构化数据不可用 |
| 具体 JSON 文件 | 该记录可能未导出 | 回到 manifest/index 查可用路径 |
| `smallmeji.github.io/hltv-cs2-data-platform/...` | 旧文档或缓存记忆 | 改读 `hltv-cs2-data-skill` 的 raw manifest 或兼容入口；仍然使用旧地址则重装 skill |

## 能做什么

可以输出这些事实数据包：

- 一条 HLTV 比赛链接的数据包。
- 两队对比，例如 `FaZe vs G2`。
- 单队画像，例如 `Aurora`。
- 假设比赛，例如 `假设 NAVI vs G2 打 S 级 LAN BO3`。
- 已知 event ID 时的赛事选手 rating。
- 数据可用时的历史回测上下文。

常见字段：

- 比赛信息：match ID、赛事、赛制、时间、状态。
- 队伍信息：HLTV team ID、队名、排名。
- 阵容上下文：已公开或已导出的首发、替补、教练。
- 当前 active map pool。
- 地图 summary：样本、W-D-L、胜率、Pick%、Ban%。当前 active map 但没有出场样本时，显示为 `0样本 / 0-0 / 0%`，不是 `missing`。
- 地图 detail：CT/T 回合胜率、手枪局、首杀后胜率、首死后胜率、总回合、赢回合。
- 选手 rating：年度 HLTV Rating 3.0、赛事 HLTV Rating 3.0。JSON 兼容字段名仍是 `rating2`，不要使用 `rating_2_0`。
- 已公开的 Veto、地图顺序、比分和赛果。
- 数据缺口与 warnings。
- `decision_inputs`：给下游模型使用的事实信号。

当前公开导出的 active maps：

```text
Ancient, Anubis, Dust2, Inferno, Mirage, Nuke, Overpass
```

除非结构化记录里存在，否则不要加入 Vertigo、Cache、Train 等非当前地图。

## 必须遵守的流程

比赛或队伍查询必须按这个顺序：

1. 先用 HLTV 定位身份事实：match ID、team ID、event ID、可见阵容、赛制、时间、状态、已公开 Veto/比分/赛果。
2. 读取所选结构化源的 capabilities / manifest。
3. 根据 ID 读取准确记录或 endpoint。
4. 地图池、选手 rating、CT/T、手枪局、首杀首死、Pick/Ban、H2H、近期记录、Veto、比分、赛果等字段必须来自结构化记录。
5. 如果结构化记录没读到，只输出部分数据，并标记 `structured_database_not_queried`。

默认公开静态源的常见记录路径示例：

```text
matches/index.json
matches/by-date/<YYYY-MM-DD>.json
matches/by-date/index.json
events/index.json
teams/aliases.json
teams/<hltvTeamId>/summary.json
teams/<hltvTeamId>/players.json
teams/<hltvTeamId>/maps-overall.json
teams/<hltvTeamId>/map-details-overall.json
teams/<hltvTeamId>/map-details-lan.json
matches/<hltvMatchId>/data-pack.json
events/<eventId>/player-ratings.json
```

如果用户只给自然语言比赛，例如 `5月10号 PGL Aurora vs Heroic`，先用结构化源的日期索引和 match search/index 搜索日期、赛事名和双方队名；匹配到唯一比赛后，读取对应 match data pack。默认公开源中，这一步优先对应 `matches/by-date/<YYYY-MM-DD>.json`，再回退到 `matches/index.json` 和该行的 `data_pack_path`。不要要求用户先提供 event ID 或 match ID，也不要在找不到 HLTV 页面时直接降级成缺字段报告。

赛事 rating 的 ID 也应自动解析：先从 match data pack 或 match index 的 `event_id` 读取，再尝试 `events/<eventId>/player-ratings.json`。如果新赛事第一天还没有赛事 rating，输出 `赛事 Rating：尚未采集/尚不可用`，继续保留年度 Rating 3.0 和地图数据。

不要用搜索摘要、wiki、盘口页面、新闻片段或模型记忆补齐地图 / 选手 / 详细数据。

中文输出应使用中文字段名或中文+标准英文缩写，例如 `范围`、`队伍`、`地图`、`样本`、`W-L`、`地图胜率`、`Pick%`、`Ban%`、`CT/T`、`手枪局`、`首杀后`、`首死后`、`回合`。除非用户要求 raw JSON 或 schema，不要把 `Scope`、`sample`、`wr`、`pick_pct`、`ban_pct` 这类内部字段名直接放进报告表头。

默认公开 raw GitHub 数据源的最小合法流程：

```text
manifest.json -> matches/by-date/<YYYY-MM-DD>.json（有日期时）-> matches/index.json -> matches/<matchId>/data-pack.json -> events/<eventId>/player-ratings.json（有 event ID 时）
```

如果换成其他 API / 静态源，使用等价的 capabilities/search/data-pack 流程即可，不要求路径名一样。如果解析到的数据包里有 `markdown` 字段，应优先用它作为事实骨架。不要只用 HLTV 页面重新写一份缺字段报告。

## 两阶段使用方式

本 skill 只负责第一阶段：

1. 从结构化记录输出事实数据包。
2. 明确结束 skill 边界。

如果用户问“谁胜率高”“哪边更强”“地图怎么倾向”，宿主大模型可以在数据包之后继续判断，但必须清楚标注那是 skill 外的模型判断。判断必须基于数据包，不能改写缺失字段、地图样本、rating 或数据源状态。

推荐边界：

```text
--- hltv-cs2-data 数据包结束 ---
以下为非本 skill 的模型判断：
```

## 输出边界

普通用户报告应该简洁。除非用户明确要求 debug、审计、source details、JSON 或给下游模型使用，否则不要输出 raw URL、record path 或完整 JSON。

推荐中文报告结构：

1. 数据源执行记录
2. 数据状态 / 数据缺口
3. 比赛信息
4. 队伍与选手 Rating 3.0
5. 地图池总览
6. 逐图详细数据
7. 特殊 Veto 变量
8. 给模型的决策输入

普通报告的事实数据包里不要出现这些内容：

- Veto 预测
- 可能的地图序列
- 胜率最高 / 具体胜率百分比
- Model Inference / 模型推理
- 下游建议或结论

数据包之后如何分析，不属于本 skill 的约束范围。

## 安装

安装仓库里的 skill 包：

```text
hltv-cs2-data.skill
```

安装后用这条测试：

```text
用 hltv-cs2-data 获取这场比赛的数据包：
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
```

正确行为：安装 / debug 测试时，模型可以提到当前数据版本和 raw GitHub manifest；普通用户报告应只显示简洁数据源状态，不显示 raw URL。

自然语言 smoke test：

```text
用 hltv-cs2-data 看 PGL Astana 2026 Aurora vs Heroic 的数据包
```

正确行为：

- 内部读取 `manifest.json`
- 内部读取 `matches/index.json`
- 匹配 `matches/2394116/data-pack.json`
- 输出里有 Overall/LAN 逐图详情、CT/T、手枪局、首杀后、首死后、Rating 3.0
- 不显示 raw URL / JSON，除非用户要求 debug
- 不在事实数据包里输出 Veto 预测或胜率判断

如果模型仍然说 `smallmeji.github.io/hltv-cs2-data-platform` 返回 404，说明它加载的是旧 skill 或旧上下文，需要重装 skill 并新开对话。

## 示例提示词

```text
用 hltv-cs2-data 获取这场比赛的事实数据包：
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
```

```text
用 hltv-cs2-data 对比 NAVI 和 G2 的地图池、选手 rating、阵容上下文和逐图详细数据。
```

```text
用 hltv-cs2-data 输出 Aurora 的单队数据画像。
```

```text
用 hltv-cs2-data 做一个 Aurora vs Heroic 的假设 S 级 LAN BO3 数据包。
```

## Reference 文件

详细规则在 skill 包内：

- `hltv-cs2-data/SKILL.md`：核心运行规则。
- `hltv-cs2-data/references/standalone-mode.md`：公开静态导出 / direct HLTV 边界和 smoke test。
- `hltv-cs2-data/references/data-availability.md`：不同数据源模式能提供什么、不能提供什么。
- `hltv-cs2-data/references/data-pack-contract.md`：输出结构和字段规则。
- `hltv-cs2-data/references/product-brief.md`：产品定位。
- `hltv-cs2-data/references/collector-contract.md`：collector/API 架构。
- `hltv-cs2-data/references/api-config.md`：可选 API 模式。
- `hltv-cs2-data/references/examples.md`：输出示例。

## License

MIT
