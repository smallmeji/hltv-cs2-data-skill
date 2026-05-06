# hltv-cs2-data Skill 中文说明

[English README](README.md)

`hltv-cs2-data` 是一个只做数据层的 CS2 skill。它把 HLTV 衍生的结构化数据库 / 静态 JSON 记录整理成简洁的数据包，供用户、其他大模型或下游策略系统使用。

它不输出胜率预测、赢家倾向、Veto 猜测、比分预测、投注建议、EV、Kelly 或仓位。

## 当前数据源

公开静态数据版本：

```text
static-raw-2026-05-06
```

默认 manifest：

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

这是唯一默认公开静态数据库入口。不要使用 GitHub Pages 地址、平台网站地址或 raw 目录地址作为数据源。

如果模型说返回 `404`，先看它访问的是什么：

| 访问目标 | 含义 | 正确处理 |
|:--|:--|:--|
| `/public-data` 目录 | 目录不是 JSON 文件 | 改读 `/public-data/manifest.json` |
| `/manifest.json` | 必须读取的默认入口 | 失败则标记结构化数据不可用 |
| 具体 JSON 文件 | 该记录可能未导出 | 回到 manifest/index 查可用路径 |
| GitHub Pages / 平台站点 URL | 旧文档或缓存记忆 | 更新或重装 skill |

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
- 地图 summary：样本、W-D-L、胜率、Pick%、Ban%。
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
2. 读取静态数据库 manifest。
3. 根据 ID 读取准确 JSON 记录。
4. 地图池、选手 rating、CT/T、手枪局、首杀首死、Pick/Ban、H2H、近期记录、Veto、比分、赛果等字段必须来自结构化记录。
5. 如果结构化记录没读到，只输出部分数据，并标记 `structured_database_not_queried`。

常见记录路径：

```text
matches/index.json
teams/<hltvTeamId>/summary.json
teams/<hltvTeamId>/players.json
teams/<hltvTeamId>/maps-overall.json
teams/<hltvTeamId>/map-details-overall.json
teams/<hltvTeamId>/map-details-lan.json
matches/<hltvMatchId>/data-pack.json
events/<eventId>/player-ratings.json
```

如果用户只给自然语言比赛，例如 `PGL Aurora vs Heroic`，先读 `matches/index.json` 搜索赛事名和双方队名；匹配到唯一比赛后，读取该行的 `data_pack_path`。不要在找不到 HLTV 页面时直接降级成缺字段报告。

不要用搜索摘要、wiki、盘口页面、新闻片段或模型记忆补齐地图 / 选手 / 详细数据。

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

如果用户问“谁胜率高 / 谁更强 / 谁更可能赢”，本 skill 仍只输出数据。最终判断由调用它的大模型或用户自己的策略完成。

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

正确行为：模型应先提到当前数据版本和 raw GitHub manifest，然后再输出结构化地图 / 选手字段。

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
- `hltv-cs2-data/references/data-source-modes.md`：公开静态导出 / API / direct HLTV 边界。
- `hltv-cs2-data/references/data-pack-contract.md`：输出结构和字段规则。
- `hltv-cs2-data/references/product-brief.md`：产品定位。
- `hltv-cs2-data/references/collector-contract.md`：collector/API 架构。
- `hltv-cs2-data/references/api-config.md`：可选 API 模式。
- `hltv-cs2-data/references/examples.md`：输出示例。

## License

MIT
