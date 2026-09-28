# AGENTS.md

大模型综合使用成本对比页（国内）。由 `llm-prices.json` 驱动、`index.html` 单文件渲染，**无构建、无依赖、无外部请求**（只 fetch 自己的 JSON）。

维护边界：改价 = 改 `llm-prices.json`；改结构/算法 = 改 `index.html` 内联脚本。约定见下。

## 项目文件

| 文件 | 作用 |
| --- | --- |
| `llm-prices.json` | 唯一数据源。价格、权重、订阅、中转站、文案备注全在这里 |
| `index.html` | 单文件页面。内联全部 CSS / JS（约 2170 行，脚本自第 650 行起） |
| `README.md` | 对外说明 + 线上地址 |
| `screenshot.png` | README 里的页面截图，改版后需同步替换 |

- 本地预览：`python -m http.server` 后访问 `http://localhost:8000`（`file://` 打开会因 fetch 失败显示错误提示）。
- 部署：GitHub Pages，`main` 分支根目录，线上地址 <https://xiaojhcc.github.io/llm-api-and-plan-pricing-in-china/>。推送到 `main` 即发布。

## 页面结构与数据映射

| 区块 | 容器 | 数据来源 | 渲染 |
| --- | --- | --- | --- |
| 顶栏汇率 | `#fxChip` | `usd_to_cny` | `main()` |
| 首屏统计 | `#statModels` / `#statChans` / `#statDate` | 有权重的模型数 / `resellers` + `subscriptions` 条目数 / `updated` | `overview()` |
| 价格全景（原价 / 渠道价） | `#chartRef` / `#chartBest` | 主表行 + 当前筛选 | `overview()` |
| 详细价格主表 | `#compareBox` | `providers` + `resellers` + `subscriptions` | `compare()` |
| 筛选器 | `#fltDet` / `#fltBox` | `filter_defaults` + 主表 DOM | `filter()` |
| 订阅方案卡片 | `#plansBox` | `subscriptions` | `plans()` |
| 中转站卡片 | `#resellersBox` | `resellers` | `plans()` 尾部 |
| 原始价格（来源求证） | `#rawDet` / `#rawBox` | `providers` 全量，不做权重加工 | `raw()` |

脚本执行顺序：`fetch` → 展平渠道查找表 `CHANNELS` → `compare()` → `filter()` → `overview()` → `plans()` → `raw()` → tooltip 引擎。

几个影响改动的机制：

- **浮窗是索引池**：`TIPS[]` 收 HTML，元素挂 `data-tip="<下标>"`（`regTip()`）。新增浮窗必须走 `regTip()`，不能自己写 `data-tip` 文本。
- **主表左右是两个 `<table>`**：左表（模型/条件/参考/最优）与右表（渠道列）同序，靠 `_pair` 互相配对；行显隐、闪烁、表尾横线都同步两侧。
- **渠道查找表在运行时拼装**：`channels` + `resellers.*.groups` 展平进同一个 `CHANNELS`，中转站分组继承所属 reseller 的币种/来源，并带 `reseller` 标记（`isReseller()` 据此判断）。
- **`normalizeConditions()`**：某模型若只有中转站渠道的条件，会被并入其他条件行（中转站不区分条件），避免出现只有中转站价的重复行。
- **前端推导、不落盘**：等效月额度（周额度 × 52/12）、积分额度取整、美元折人民币、OpenCode 的 5h/周/月额度（按 1 : 2.5 : 5 从月额度推）、汇率换算，都在渲染时算。**不要在 JSON 里预先算好这些值。**
- **结构用 `<template>`**：表头、厂商卡、浮窗骨架写在模板里，脚本只填槽位（`textContent` / `setAttribute`），没有字符串拼结构。改布局优先改模板。
- **限期值在加载时先展开**：JSON 里的候选数组（常态价 / 限期价并列）由 `materialize()` 在 fetch 之后立刻按北京日期解析成普通值，`from` / `until` 被抹掉，渲染器完全看不到；页面不重载就不会跨过切换点。规则见「限期值（候选数组）」。

## 字段结构（llm-prices.json）

### 顶层

| 字段 | 作用 |
| --- | --- |
| `components` | 分量 key → 中文名（`input` / `cache_write` / `cache_write_5min` / `cache_write_30min` / `cache_write_1h` / `cache_hit` / `output`）。全站表头与浮窗文案的唯一来源 |
| `weights_baseline` | 基准权重 `1 : 30 : 0.2`。**不参与计算**，只用于权重丸「高于基准黄 / 低于绿」的对比 |
| `channels` | 官方与平台渠道元数据（见下） |
| `api_refs` | 外部聚合价目（models.dev / OpenRouter），只做变化信号，见「数据来源」 |
| `usd_to_cny` | 汇率。顶栏 chip 与所有美元折人民币 |
| `updated` | 形如 `"2026.9.23"`，首屏「数据更新」 |
| `filter_defaults` | 首屏默认勾选的模型名与渠道 key（`reseller:Packy` / `sub:Kimi 订阅`）。为空或缺失则全选 |
| `billing_types` | 计费类型字典（`label` + `desc`），只供胶囊取用文案与配色 |
| `subscriptions` | 订阅组 |
| `resellers` | 中转站 |
| `providers` | 厂商 → 模型 → 条件 → 渠道 → 分量价 |

### channels 条目

```jsonc
"ollama": { "label": "Ollama", "kind": "usd",           // usd | cny | point
            "source": "https://ollama.com/pricing", "api": "https://ollama.com/v1/models",
            "note": "换算口径说明，可选，显示在原始价格表表头 ⓘ" }
```

`source` 是该渠道的定价依据页；`api`（可选）是渠道**自家发布**的机器可读端点（目前只有 Ollama / Packy / Command Code 有），命名即语义，见「数据来源」。注意**自建端点不一定带价格**：Ollama `/v1/models`、Command Code `api.commandcode.ai/provider/v1/models` 实测都只返回模型 id，能不能直接定价以实测为准。

### providers 条目

```jsonc
"深度求索": {
  "weights": { "input": 0.25, "cache_hit": 50, "output": 0.2 },  // 参与计算，见「综合价与权重算法」
  "weights_note": "来自个人使用 DeepSeek Harness 实测",           // 缓存命中率浮窗顶部的一句来源说明
  "weights_notes": { "cache_hit": { "kind": "warn", "text": "…" } }, // 分量 key → 偏离基准的原因
  "hit_rate": { "estimated": false, "stats": { "cache_write":…, "cache_hit":…, "output":… } },
  "models": { "模型名": { "条件名": { "渠道key": { "分量key": 单价 } } } },
  "sources": { "cny": "https://…" },                            // 渠道 key → 该厂商的定价页，覆盖渠道级 source
  "conditionNotes": { "梁文谷": "空闲时段说明…" },                // 条件名 → 说明，hover ⓘ
  "channelWeights": { "packy_grok_sale": { "input": 31, "output": 0.2 } } // 可选：按渠道整体替换 weights
}
```

- 价格单位一律 **每 1M tokens**；积分渠道同理折算为每 1M tokens 积分消耗。
- 条件名 `default` = 无条件，行内不显示标签；其他条件名原样显示在「条件」列，并用 `conditionNotes` 解释（如 `标准价格` / `梁文谷` / `≤200K` / `>272K`）。
- 条件内渠道 key 可以是 `channels` 的 key、`resellers.*.groups` 的 key，或订阅的 `points_channel` / `price_channel`（这样订阅才有标价可折算）。
- `hit_rate.stats` 只给实测过的厂商填；`estimated: true` 的厂商留空，表头显示「估算」徽章。
- `weights_note`（一句来源）与 `weights_notes`（分量原因）**是两个字段**，别混。

### resellers 条目

```jsonc
"Packy": {
  "currency": "CNY",                       // 决定分组价格的币种与折算
  "source": "https://www.packyapi.com/pricing",
  "api": "https://www.packyapi.com/api/pricing",
  "notes": [],                             // 卡片脚注，与订阅 notes 同构
  "groups": { "packy_grok_sale": { "label": "grok-sale", "note": "缓存可能不生效…" } }
}
```

分组 key 一旦被 `providers.*.models.*.*` 引用即成为主表渠道列（列名取 `label`，继承规则见上文「渠道查找表在运行时拼装」）；分组 `note` 会出现在原始价格表表头与浮窗里。

### subscriptions 条目

```jsonc
"GLM Coding Plan": {
  "currency": "CNY",
  "type": "points",                        // money_quota | points —— 决定算法分支
  "billing": "per_model_points",           // 计费类型字典 key，不参与计算（见「计费模式」）
  "points_channel": "glm_coding_points",   // points 用：积分价来源渠道
  // "price_channel": "opencode",          // money_quota 用：标价来源渠道（缺省按币种回退 usd/opencode/cny）
  "plans": { "Lite": { "fee": 118, "quota_5h": 2000, "quota_weekly": 10000 } },
  "models": ["GLM-5.3", "GLM-5.3-Flash"],
  "notes": [ { "kind": "good", "label": "积分透明", "text": "官方明确了积分用量与消耗倍率。" } ],
  "priceOverrideNotes": { "Kimi K2.8 Preview": "API 未上线，¥ 官方价为按 K2.7 Code 同价的估算值" },
  "source": "https://…"
}
```

套餐字段：

| 字段 | 说明 |
| --- | --- |
| `fee` | 月费（币种同订阅 `currency`） |
| `quota_5h` / `quota_weekly` / `quota_monthly` | 各口径额度，只填官方规定的；月额度缺失时前端按周额度推导「等效月额度」 |
| `quota_monthly_by_model` | 各模型额度不同时（OpenCode Go / Command Code）逐模型给出，优先级高于 `quota_monthly`；多档位各自规定时（Command Code）「各模型额度」表增列「套餐」，摘要计数取去重后的模型数 |
| `excluded_models` | 该档不适用该模型 → 该行该列不出值（如 Kimi 的 Go 档不含 K3） |

`priceOverrides`（按模型覆盖标价）代码已支持但当前无数据；Kimi 用的是 `priceOverrideNotes`（只在浮窗里说明「同价估算」这类口径）。

### notes 条目

```jsonc
{ "kind": "good", "label": "额度透明", "text": "官方明确了额度倍率。",
  "link": "https://…", "hover": "主表浮窗用的短句" }
```

- `kind`：`good` 绿 ✓（官方明确、透明）/ `info` 灰 i（来源说明）/ `warn` 黄 !（不准、缺货、限制、风险）。
- 兼容旧的纯字符串写法（等价 `info` + `备注`）。
- `hover` 只在主表订阅格浮窗出现，订阅卡片脚注显示完整 `text`。
- 语气是给人看的直白评价，可以带吐槽（「你买不到。」）；维护时保持这个风格。

### 限期值（候选数组）

**任何「值」位置都可以并列多个候选**，由前端按日期自动二选一：

```jsonc
"ark_afp": [
  { "input": 250, "cache_hit": 250, "output": 250 },                        // 常态（无日期）
  { "input": 125, "cache_hit": 125, "output": 125, "until": "2026-09-28" }  // 限期
]
```

- 候选可带 `from` / `until`（**均含当日**），日期格式 `YYYY-MM-DD`，按**北京时间（UTC+8）** 判定。
- 判定规则：带日期的候选命中当天 → 取它并**丢弃同级的无期限候选**；没命中 → 取无期限候选；都没有 → 该项为空（该键被删掉）。
- 标量候选（`fee`、额度数字）用 `{ "value": 值, "until": … }` 包裹：`"fee": [40, { "value": 9.9, "until": "2026-11-08" }]`。
- **只有数组里出现带日期的候选时才按候选数组处理**，普通数组（`filter_defaults.models`、订阅 `models`、`notes`）不受影响。
- 解析在加载 JSON 后立即完成（`index.html` 的 `materialize()`），`from` / `until` 会被抹掉，渲染器看不到它们。切换只在页面加载时判定，跨过切换点后刷新即可。

| 场景 | 写法 | 到期后 |
| --- | --- | --- |
| 渠道限时折扣 | 渠道价写成 `[ 常态, 限期 ]` | 回到常态价 |
| 整体换价（如 Google 2026 年 → 2027 年） | 条件值写成候选数组，两个条件各带 `until` / `from` | 旧条件整条消失，新条件生效 |
| 订阅档位活动（fee / 额度） | `"fee": [标价, { "value": 活动价, "until": … }]` | 回到标价 |
| 模型下线（没有常态） | 模型值写成只含一个带 `until` 候选的数组 | 整个模型消失（各表都不再出现） |

**限时项一律用这个字段记录，不要写进 `notes`** —— note 是给读者看的评价，不是维护备忘。

**官方写明的起止日期就是事实**（含已经过去的）：活动写 `from` + `until`、限时额度同样用 `{ "value": … }` 包裹；常态值取官方给出的「活动结束后」的值（如「returns to $2 in / $6 out」就把 2 / 6 写成常态候选）。已经到期的候选不必删，留着就是这段历史的记录，前端会自己切回常态。渠道值只有一个带 `until` 的候选时表示该渠道价目在那天下线（键被删掉，参考列随之留空）。

## 计费模式

算法只看 `type`，**`billing` 不参与任何计算**——它只决定胶囊文案和配色（`unified*` 统一色 / `floating*` 浮动色 / 其余 区分模型色）。这是最容易踩的坑：改 `billing` 不会改变数值。

`billing_types` 六种：

| key | 名称 | 含义 |
| --- | --- | --- |
| `floating_quota` | 浮动额度 | 额度以标价金额计，但用量为非官方估算（Claude / ChatGPT 订阅） |
| `unified_quota` | 统一额度 | 金额为准确值，所有模型共用同一额度（Kimi / Ollama Cloud） |
| `per_model_quota` | 区分模型额度 | 金额准确，但各模型额度不同，需逐一换算（OpenCode Go） |
| `unified_points` | 统一积分 | 积分计费，与各模型原价有固定等比关系（百炼 / MiMo） |
| `per_model_points` | 区分模型积分 | 积分计费，与原价不等比且各模型比例不同，折算比只能给区间（GLM / 火山 Agent Plan） |
| `paygo` | 按量计费 | 无月费无额度；中转站卡片固定挂这个标签 |

**折算公式**（两者都是「综合价 × 月费 ÷ 月额度」，只是综合价的来源渠道不同）：

- `money_quota`：综合价 = 标价渠道（`price_channel`，缺省 USD 取 `usd`→`opencode`、CNY 取 `cny`）按权重汇总 → 成本 = 综合价 × 月费 ÷ 月额度。
- `points`：综合消耗 = 积分渠道（`points_channel`）按权重汇总（单位：积分 / 1M tokens）→ 成本 = 综合消耗 × 月费 ÷ 月额度（额度单位同为积分）。积分→人民币的口径写在渠道 `note` 里，会附在浮窗末尾。

**折算比 1 : n**（`1 : n = 官方标价 ÷ 实际成本`，n 越大越便宜；整页口径统一，订阅格与主表、中转站卡不会互相打架）：

- 订阅卡：`n = 官方（参考）综合价折人民币 ÷ 订阅折算后综合价`，逐模型算后取区间显示 `lo ~ hi`。**不能只用 `月额度 ÷ 月费`** —— 那只是额度倍率，只在订阅的标价渠道（`price_channel`）价目与模型官方价一致时才等价；Command Code 对 GPT 5.6 Sol 收官方价的 1.25 倍、对 MiMo V2.5 打近一折，用额度倍率会显示 1 : 7 而实际只有 1 : 5.4。积分制的 `n = 月额度积分 × 标价综合价 ÷ 积分综合价 ÷ 月费`（即同一口径的积分版）。
- 取价规则：该模型**同时含标价渠道与官方参考价**的首个条件（峰谷/上下文档才不会串）；官方参考价 = `cny` 综合价，整家无人民币价目时用 `usd`/`opencode` 综合价 × 汇率（与主表参考列同规则）。
- 中转站卡：`n = 官方综合价（折人民币）÷ 该分组综合价`（同一口径）。
- **没有兜底口径**：一个模型都取不到（订阅不覆盖 / 无权重厂商 / 该条件没有官方参考价）时不给折算比 —— 卡片与主表格显示 `—`，不要退回「额度倍率」。单值还是区间按 `ratioNum()` 之后的显示值判定（两者显示相同即单值），不是数值阈值。
- hover 浮窗一律逐模型列出（模型 / 月额度 / 折算比），不分档、不分单值区间 —— 各模型单价不同，逐模型才是全貌。

**「各模型额度」明细表**（卡片内 `<details>`，只有 `quota_monthly_by_model` 的订阅才有）：列 = 套餐（多档各自规定时才出）/ 模型 / **订阅单价** / 5h 额度 / 周额度 / 月额度 / 折算比。行序 = 套餐 → 月额度 → 模型名。

- 订阅单价是**该订阅标价渠道下每 1M tokens 的分量价**（标签取 `components`，币种随订阅），取自与该行折算比**同一个条件**（峰谷/上下文档不串），**只列与官方参考价不同的分量**：全部分量同价显示「同官方价」。浮窗给全分量的订阅价 / 官方价对照。
- 「同价」是**严格同价**：官方参考价只取**与订阅同币种**的官方价目（美元订阅看 `usd`、人民币订阅看 `cny`），**不做汇率折算、不设容差**，逐分量 `===` 才算同价。该币种没有官方价目（如国产模型只有人民币价目、而订阅以美元计价）时不折算、不猜：只列订阅价，**不出对照列**（浮窗里没有那一列，也不要填 `—`）。
- 模型不在 `providers` 里、或该渠道取不到价时留 `—`。5h/周两列 = 月额度 × 20% / 50%：两家官方的 5h / 周窗口都是月额度的 20% / 50%（Command Code 的口径是计划级 credit 上限 14 / 35，对 $70 档恰好也是 20% / 50%），逐模型值即「只用该模型时」的等价额度，前端推导。

**计费胶囊必须与折算比的显示形态一致**：折算比只要是区间，`billing` 就必须是 `per_model_quota` / `per_model_points`（区间意味着各模型的实际单价不同，读者必须逐模型看）；只有折算比全档单一值时才用 `unified_*` / `floating_*`。Ollama Cloud 就是按这条从 `unified_quota` 改成 `per_model_quota` 的（额度统一但各模型价目与官方价的差不同，折算比因此是区间）。

**订阅的 `models` 默认包含该厂商在 `providers` 里的全部模型**（官方订阅通常覆盖自家所有在售模型）：维护时先按 `providers` 对齐，再照官方页面调整。目前的例外是 **GLM Coding Plan** —— 官方明确只支持 GLM-5.3 / GLM-5.3-Flash，调用 GLM-5.2 / 5.1 会被自动切到 5.3，所以不纳入。含他厂模型的订阅（百炼 / 火山）与第三方聚合订阅（OpenCode Go / Ollama Cloud）逐项对官方清单，官方没列的不加。

## 综合价与权重算法

```
综合价 = Σ (分量单价 × 权重)          # 单位：每 1M 新增输入 token 的花费
```

权重取厂商 `weights`；若该渠道在 `channelWeights` 里有条目，则**整体替换** `weights`（用于「该渠道不按缓存计费」这类情形：写成「输入 ×31 + 输出 ×0.2」等价于把 30 份缓存命中并入输入）。**当前无厂商使用**，`channelWeights` 为空即可省略。

- **权重语义是 token 配比**：基准 `1 : 30 : 0.2` 意为「每 1M 新增输入 token，附带 30M 缓存命中 token、0.2M 输出 token」。基准标定目标是让综合价接近「一次 256K 上下文的典型编码任务开销」，从而与输出单价在同一量级可比。
- **输入侧分量名因厂商而异**，`weights` 必须写自家价目里实际存在的那个 key：Anthropic 用 `cache_write_5min`、OpenAI 用 `cache_write_30min`、千问用 `cache_write`（故意按写入价计全部输入）、其余厂商用 `input`。**权重 key 找不到对应价格时按 0 计**（不报错、不回退），所以 name 写错的表现是「该分量静默消失、综合价偏低」。
- **基准对比着色**：`cache_hit` 比 `weights_baseline.cache_hit`、`output` 比 `output`、其他一切 key 都按输入侧比 `input`；高于基准黄、低于绿，与基准相等则不着色也不挂浮窗。偏离原因写 `weights_notes[分量key]`（`good` 绿 / `warn` 黄）。
- **缓存命中率胶囊** = `weights.cache_hit ÷ (weights.cache_hit + 所有 cache_write_* + input)`；浮窗里给 `weights_note` 与 `hit_rate.stats`。注意公式只认 `input` 与 `cache_write*` 前缀，新增输入侧命名时不会自动跟着变。
- **热度色**：只给中转站格 / 订阅格 / 最优格着色，取全局所有中转站与订阅格（折人民币）的对数刻度均分 5 档，`heat0` 绿（廉）→ `heat4` 红（贵）。参考列不着色。
- 浮窗固定顺序：分量明细（单价 × 权重 = 小计）→ 综合价（积分制为「综合消耗」）→ 折算后 → 汇率换算。

## 入表条件

**厂商进主表**：`providers` 中定义了 `weights` 的厂商。没有 `weights` 的厂商（当前是腾讯混元、LongCat、Meta、Omen）不进主表、不进筛选器、不进价格全景——它们**只出现在「原始价格」表**里，用来留档那些在订阅（如 OpenCode Go / Command Code）里出现但自身无可比价目的模型。订阅 `models` 里若含它们，会照常出现在「各模型额度」明细表（有额度、有订阅单价），但折算比显示 `—`。

**主表行（模型 × 条件）**：

- 数据里怎么分组就怎么出行。
- 行隐藏：该模型未勾选，或「参考列无值 **且** 它的所有产数渠道都被关掉」。
- 参考列：`¥` 优先取 `cny` 渠道综合价；该厂商整家都没有人民币价时，用美元综合价 × 汇率换算（表头显示「¥ 换算」，美元表头显示「$ OpenCode」，否则「$ 官方」）。
- 最优列：候选 = 该行全部中转站分组 + 订阅档折人民币，取最低；**随渠道筛选实时重算**（被关掉的渠道不参与最优）。

**渠道列**：

- 中转站分组列：本厂商任一模型/条件引用了该分组 key 就出列。
- 订阅列：订阅 `models` 与该厂商模型有交集、该档未 `excluded_models` 排除该模型、且该模型在该条件下能取到 `points_channel` / `price_channel` 标价，才出值。
- 空列剔除：可见行中整列无值的计划列整列隐藏（含组头 `colspan` 收缩）；渠道筛选未勾选不是隐藏，而是灰化并整组排到右侧。
- 渠道单元格一律显示换算后的人民币价，原币种与换算过程留在浮窗里。

**卡片区**：「订阅方案」渲染全部 `subscriptions`；「中转站」渲染全部 `resellers`，无 `resellers` 时整个区块隐藏。中转站卡的折算比口径同主表。

**不收的东西**（看到也不用请示，直接跳过）：

- 太老的世代：远代旧版、已退役或已下线的同族模型（Gemini 2.5 系、GLM-4.7、claude-opus-4.x、grok-4.20 这类）。
- 同模型的高速 / 加速变体：`highspeed`、`ultraspeed`、`FlashX`、`-x` 等后缀（GLM-5.3-FlashX、kimi-k2.7-code-highspeed、MiniMax-M2.7-highspeed、mimo-v2.6-pro-ultraspeed 都不收）。
- 团队 / 企业档订阅：团队席、Enterprise、共享用量包（Ollama Team、百炼团队版、GLM 团队版等）。
- 新发现的、官方未写明额度或积分数值的订阅套餐 —— 写不出 `quota_*` 就无法折算。
- **现实没给的就是没有**：官方没公布的分量、额度、换算系数一律留空，不要用第三方或估算补齐（实测权重与备注是另一回事，那些由用户本人填）。

无权重的小厂商（Meta / LongCat / Omen）只挂在 `opencode` 渠道、只进原始价格表，属无害数据，不必给它们补 `source`。

## 数据来源

| 用途 | 位置 |
| --- | --- |
| 定价依据（人读） | `channels.<key>.source` 官方页面；厂商可用 `providers.<厂商>.sources[渠道key]` 覆盖（原始价格表表头链接取后者优先） |
| 定价依据（机读） | `channels.<key>.api` / `resellers.<name>.api`：渠道**自家发布**的端点。Packy 的 `api` 带完整价目，可直接定价；Ollama 的 `/v1/models` 与 Command Code 的 `/provider/v1/models` 实测只返回模型 id，价格仍要看官方页 |
| 订阅档位与逐模型额度 | 文档站的计划页（Command Code：`/docs/plans/{go,goat,pro}` 的 per-model credits 表；页内嵌的模型表同时给 `availability` 与各档额度，可用来定 `models` / `excluded_models`；限时增量按 `through <日期>` 记成候选数组） |
| 比对用（非官方） | `api_refs`（models.dev、OpenRouter）：外部聚合，只做「有没有变」的信号 |
| 额度反推 / 备注来源 | `notes[].link`、`notes[].hover`、`weights_note`、`conditionNotes` |

**字段名即语义**：`api` = 定价方自己发的端点（可信、可定价），`api_refs` = 外部聚合（只比对）。新来源按「谁发布的」归位，同一个源不要两处都放；`api_refs` 目前只比对美元渠道（人民币渠道无公开 JSON）。

官方没公开的部分（订阅额度、缓存命中率、积分换算）来自个人实测或社区反推，必须用 `info` / `warn` 备注写清来源与可靠性，不能默默填数；所有价格与权重都要能追溯到某个 `source` 或某条 `notes`。

## 维护约定（不变量）

- 改任何价格都同步更新顶层 `updated`。
- 新增分量：先加 `components`，再在 `weights_baseline` 里想清楚它归输入侧还是输出侧（`weightBase()` 只看名字是不是 `cache_hit` / `output`），最后检查各厂商 `weights` 是否要跟着加。
- 新增订阅/中转站渠道：key 用英文小写下划线；确保它作为渠道 key 出现在 `providers` 里（否则主表无标价可折）。
- 限时项（活动价 / 限时额度 / 模型下线）一律写成候选数组 + `until`（或 `from`），常态值并列在同一个数组里；不要用注释或 `notes` 记到期时间。
- 不要把展示文案写进 HTML：厂商名、条件名、渠道名、备注全部来自 JSON。
- 不要引入构建工具、依赖、CDN、外链字体——单文件直开 + 一个 JSON 是这个项目的底线。
- 改版或改数据后，`screenshot.png` 与 README 里的说明如已过时一并更新。

## 更新流程速查

### 三类例行更新

| 情形 | 怎么做 |
| --- | --- |
| **新模型发布** | 查阅官方页面 → 在该厂商 `models` 下按既有结构补价格 → 在 `filter_defaults.models` 里**用同档新版替换旧版**（例：上了 Opus 5.5 就把 Opus 5 撤出默认勾选） |
| **中转站更新**（如 Packy） | 查 Packy 的 `api` 逐分组比对价格 → 看有没有新模型被覆盖 → **排除无关渠道**：不要顺手引入本页不跟踪的渠道与模型（Packy 有 22 个分组，本页只用 8 个） |
| **订阅更新** | 档位调价改 `plans.*.fee`；额度变更改 `quota_5h` / `quota_weekly` / `quota_monthly`（逐模型额度另见 `quota_monthly_by_model`、按档排除模型见 `excluded_models`），按官方口径填，官方没规定的就留空（前端会推导等效月额度）；`models` 按「官方订阅默认含自家全部模型」与该厂商在 `providers` 里的模型对齐（例外见「计费模式」）；活动价 / 活动额度带期限的一律写成候选数组（见「限期值」） |

**连不上的源先走代理**：常见代理地址 `https://127.0.0.1:7890`，访问国外源（OpenAI、Google、Packy 等）时先走代理确保可连接。

### 定期普查

例行更新是事件驱动（看到新模型才去查），会漏；普查走另一条路径，按仓库登记的来源做全量核对：

1. 枚举来源：`channels.*.source` / `providers.*.sources` / `resellers.*.api` / `subscriptions.*.source` / `api_refs`，逐源核对（各源独立，可并行）。
2. 官方页常常是 SPA 或登录墙（bigmodel.cn、千问价格页、百炼控制台、Kimi 会员页、chatgpt.com 都碰得到）——**先找同一家的官方文档页或官方接口**，再走代理重试（见上），最后才考虑放弃；不要因此改用非官方聚合定价。
3. 每家都要看三件事：价格有没有变、有没有本地缺的新模型、官方页上有没有已经消失或宣布下线的模型。
4. 顺带检查**已登记的限时项**（`until`）是否临近或已经过期——过期后前端会自动切回常态，但数据里的常态值也需要是当前正确的。

### 先请示用户，不要自作主张

| 发现 | 为什么不自己决定 |
| --- | --- |
| 缺失模型 | 可能是冷门模型被有意舍弃，不入表 |
| 缺失订阅档位 | 可能是团队订阅、按量增量付费等本页不关心的计费模式 |
| 特殊条件 | 可能是「所谓」限时优惠：**未定期限的当永久看待**；**有期限的用候选数组 + `until` 记录**（见「限期值（候选数组）」），到期由前端自动切回常态，不需要人工在到期日改数据 |
| 官方页已消失 / 宣布下线的模型 | 本地可能仍在跟踪别家的同一模型；保留、加 `until`、还是删除，由用户定 |
| 新的计价维度（Batch / 优先档 / 区域加价等） | 可能是本页有意不做的一层，别顺手加条件行 |

### 自主边界

- **agent 可自主**：模型新增、价格数值。
- **由用户本人填**：实测权重（未实测时填默认 `1 : 30 : 0.2` —— 输入侧分量 `1`、`cache_hit` `30`、`output` `0.2`，输入侧分量名按该厂商价目实际存在的 key 选）、所有注释。
- **note 一律先请示**：`notes` / `weights_note` / `weights_notes` / `conditionNotes` 是给读者看的结论与评价，**不是 agent 的任务笔记**；任何 note 的增删改都必须先问用户。到期时间属于数据字段，写在候选数组的 `until` 里，不要写进 note。