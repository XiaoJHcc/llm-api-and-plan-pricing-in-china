# AGENTS.md

国内大模型综合使用成本对比页：`llm-prices.json` 驱动、`index.html` 单文件渲染，无构建 / 无依赖 / 无外部请求（只 fetch 自己的 JSON）。**本文件只管「怎么录入数据才能算出正确的值」**；渲染与算法的实现细节一律读 `index.html`（区块都是具名 IIFE：`main` / `compare` / `filter` / `overview` / `plans` / `raw`，按函数名 grep）。

改价格数据 = 只改 `llm-prices.json`；改页面功能 = 可改 `index.html`。
预览 `python -m http.server` 或 `http-server`（`file://` 打开会 fetch 失败）

## 一、核心规则

### 按查到的事实严格记录

- 使用**官方数据来源**，第三方数值仅供快速发现异常查缺补漏，但必须回官方源验证。
- 一些**依赖推导**的数据，例如某些计费倍率 125%、token 计量数量单位不同，需要在**确保推导方式可靠的前提下，自行换算**。
- 价格录入注意**各分量性质**：输入 / 输出 / 缓存命中 / 缓存写入 等。不同厂商的分量计费方式不同，各分量比值也完全不同，**不得混用，不得随意估算**。
- **不得拟造估算数据来满足数据空位**，如果官方源确实未给出，则数据有空位属于正常现象。例如各订阅的限额周期不同，不一定都有 5h 限额。*（前端将会自动换算为等效月额度）*
- 录入时**禁止换算币种或积分**，只能按照原始定义录入，汇率换算和积分价值这种非恒定变换不得直接在入库前转换。*（前端将会实时计算汇率和积分价值）*

### 价格认定

- **无明确期限**的优惠政策，即使营销口径为“限时”，也**当作永久计入**，确保现价正确。
- **有明确期限**的优惠政策，同时**记载截止时间**。*（前端将会自动按时间判断，过期后自动隐藏）*
- 期限已截止后（例如旧模型已下线）该字段永久过期，属于正常逻辑，不设回退。
- 计费量级一律按**每 1M token 计量**，不符合的按量级换算后填入。

### 综合价权重

- **综合价 = Σ (分量单价 × 权重)**。
- 综合价**算法依据**：一次编程任务通常包含 1M 新增输入 token、30M 缓存命中 token、0.2M 输出 token，即我们 1 : 30 : 0.2 的经验比值。未实测的供应商，按此默认权重给出综合价；实测过的供应商，后续再自行修改给定权重。
- 新增输入：即编程中未被缓存命中的输入部分。根据各家计费方式不同，有可能缓存写入不额外计费，也有可能只计缓存写入而不计输入，具体取哪个分量、是否重复计费必须根据官方文档详细判断，**计算结果必须能够如实反映模型调用时的实际成本**，且**严格按照原始分量定义入库**。

### 明确排除

- **团队**订阅方案。
- **未写明额度**的订阅套餐。例如只有所谓“调用次数”的套餐。
- 订阅外的**原价按量计费**部分。
- 被**永久划掉**或**未宣称优惠截止期限**的原价。
- 比已入库模型更老的**旧版模型**。
- 同模型的**高速版本**（`highspeed` / `ultraspeed` / `X` 等后缀）。注意 `Flash` 是常用模型后缀，不是高速版本。

## 二、数据字段

顶层：`components`（分量 key → 中文名，表头与浮窗文案的唯一来源）/ `weights_baseline`（**不参与计算**，只做权重丸的基准对比）/ `channels` / `api_refs` / `usd_to_cny` / `updated` / `filter_defaults`（首屏默认勾选的模型与渠道 key，如 `reseller:Packy` / `sub:Kimi 订阅`；缺失即全选）/ `billing_types` / `subscriptions` / `resellers` / `providers`。

```jsonc
// providers[厂商]
{ "weights": { "input": 0.25, "cache_hit": 50, "output": 0.2 },        // 参与计算
  "weights_note": "一句来源说明",                                       // 与 weights_notes 是两个字段，别混
  "weights_notes": { "cache_hit": { "kind": "warn", "text": "偏离基准的原因" } },  // kind: good | warn
  "hit_rate": { "estimated": false, "stats": { "cache_write": …, "cache_hit": …, "output": … } },
                                             // stats 只给实测过的厂商填，estimated: true 的留空（表头显示「估算」）
  "models": { "模型名": { "条件名": { "渠道key": { "分量key": 单价 } } } },
  "sources": { "usd": "https://…" },                                  // 渠道 key → 该厂商定价页，覆盖渠道级 source
  "conditionNotes": { "梁文谷": "空闲时段说明…" },                      // 条件名 → 说明；`default` = 无条件，不显示标签
  "channelNotes": { "opencode": "降智说明…" },                          // 渠道 key → 该厂商此渠道的特有说明（原始数据表列头 ⓘ，与渠道级 note 合并显示）
  "channelWeights": { "packy_grok_sale": { "input": 31, "output": 0.2 } },   // 整体替换 weights
  "modelChannelWeights": { "Qwen 3.8 Flash": { "commandcode": { … } } } }    // 同上，优先级更高
```

- 价格单位一律每 1M tokens；积分渠道同理折算为每 1M tokens 积分消耗。
- 条件内的渠道 key 可以是 `channels` 的 key、`resellers.*.groups` 的 key，或订阅的 `points_channel` / `price_channel`（订阅要能折算，其标价渠道必须出现在它覆盖的模型下）。

```jsonc
// channels[key]
{ "label": "Ollama", "kind": "usd",          // usd | cny | point
  "source": "https://…", "api": "https://…", // source = 定价依据页；api = 可选，渠道自家机读端点
  "note": "换算口径，可选，显示在原始价格表表头 ⓘ" }
// resellers[名称]：分组 key 被 providers 引用即成为主表渠道列（列名取 label）
{ "currency": "CNY", "source": "…", "api": "…", "notes": [],
  "groups": { "packy_grok_sale": { "label": "grok-sale", "note": "可选，进表头与浮窗" } } }
// subscriptions[名称]：priceOverrides 按模型覆盖标价，代码已支持但当前无数据
{ "currency": "CNY", "type": "money_quota",  // money_quota | points —— 决定算法分支
  "billing": "per_model_points",             // 计费类型字典 key，不参与计算，只决定胶囊文案与配色
  "points_channel": "glm_coding_points",     // points 用
  "price_channel": "opencode",               // money_quota 用，缺省按币种回退美元 usd / 人民币 cny
  "plans": { "Lite": { "fee": 118, "quota_5h": 2000, "quota_weekly": 10000, "quota_monthly": …,
                       "quota_monthly_by_model": { "模型": 值 }, "excluded_models": ["模型"] } },
  "models": ["GLM-5.3"], "notes": [ { "kind": "good", "label": "…", "text": "…" } ],
  "priceOverrideNotes": { "模型": "口径说明" }, "source": "https://…" }

// notes 条目（订阅 / 中转站通用；旧的纯字符串写法等价 info）
{ "kind": "good", "label": "…", "text": "…", "link": "…", "hover": "…" }
// kind：good 绿 ✓ / info 灰 i / warn 黄 ! / crit 红 ✕（严重问题，如降智）；text 是给人看的直白评价（可吐槽），hover 只在主表浮窗出现
```

### 限期值（候选数组）

任何「值」位置都能并列多个候选，由前端按北京日期自动二选一：

```jsonc
"ark_afp": [ { "input": 250, "cache_hit": 250, "output": 250 },                       // 常态（无日期）
             { "input": 125, "cache_hit": 125, "output": 125, "until": "2026-09-28" } ]  // 限期
```

- 候选可带 `from` / `until`（**均含当日**，`YYYY-MM-DD`，北京时间）。
- 标量候选（`fee`、额度）用 `{ "value": 值, "until": … }` 包裹。
- 在官方有指定开始日期或结束日期时，按以上规范计入，后续过期不删，作为无害历史信息。

## 三、计费方式

- 价格一律纳入下面几种计费方式，确保计算结果正确无遗漏；**无法涵盖、填不进去的数据与特殊逻辑不得随意抛弃，需反馈报出**。
- 算法只看 `type`，`billing` 不参与计算（只决定胶囊文案与配色）：`money_quota` = 额度按标价金额计，成本 = 标价渠道（`price_channel`，缺省按币种取 `usd` / `cny`）综合价 × 月费 ÷ 月额度；`points` = 额度按积分计，成本 = 积分渠道（`points_channel`）综合消耗 × 月费 ÷ 月额度。
- `billing` 按折算比的显示形态选：区间用 `per_model_*`，全档单一值用 `unified_*` / `floating_*`，中转站卡固定 `paygo`。

| `billing` | 含义与现有 |
| --- | --- |
| `floating_quota` | 浮动额度。金额为标价，用量属非官方估算 —— Claude / ChatGPT 订阅 |
| `unified_quota` | 统一额度。金额准确，全模型共用同一额度 —— Kimi 订阅 |
| `unified_points` | 统一积分。积分计费，与各模型原价成固定等比 —— MiMo |
| `per_model_quota` | 区分模型额度。各模型额度不同，各模型定价也可能不同 —— OpenCode Go / Ollama Cloud / Command Code |
| `per_model_points` | 区分模型积分。积分计费，与原价不等比、各模型比例不同 —— GLM / 百炼 / 火山 Agent Plan |
| `paygo` | 按量计费。无月费无额度 —— 中转站 |

## 四、来源与维护

- **来源各就各位，同一个源不要两处都放**：`channels.<key>.source` = 定价依据页（厂商可用 `providers.<厂商>.sources[渠道key]` 覆盖；官方美元价一律挂厂商**自家**国际站 / 英文文档）；`channels.<key>.api` / `resellers.<name>.api` = 渠道**自家发布**的机读端点；`api_refs`（models.dev / OpenRouter）= 外部聚合，只做「有没有变」的信号。
- 官方页是 SPA 或登录墙时（bigmodel.cn、千问价格页、百炼控制台、Kimi 会员页、chatgpt.com 等），先找同一家的官方文档页或官方接口，对于国外源考虑走代理（常见 `https://127.0.0.1:7890`）。
- **改任何价格都同步更新顶层 `updated` 日期。**
- **自主边界**：模型新增与价格数值 → agent 自主；实测权重与一切注释 → 用户本人填；
- **先请示**：缺失的模型 / 订阅档位、新的计价维度（Batch / 优先档 / 区域加价）。
- **不增加任何解释性文字**：所有的 `notes` / `weights_note` / `weights_notes` / `conditionNotes` / `channelNotes` 将会在前端展示，**不是你的笔记本，注意 UI 卫生**，增删改一律先问用户。用户批准前**不得私自增加任何**面板可见的解释文字。
- **不变量**：展示文案不写进 HTML（厂商名 / 条件名 / 渠道名 / 备注全部来自 JSON）；不引入构建工具、依赖、CDN、外链字体；新增订阅 / 中转站渠道 key 用英文小写下划线，且必须出现在 `providers` 里。
