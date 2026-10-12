# 积分域工具字段实测（references/points-tools.md）

> 实测环境：麦当劳 MCP `https://mcp.mcd.cn`（Streamable HTTP，Bearer Token）
> 实测时间：2026-10-12 09:36（GMT+08:00），服务器时间取自 `now-time-info`
> 原始返回存档：`evidence/01`~`06`，**未手改**。traceId 见各文件。

本文件是 S1 积分守门员的实现依据。所有结论均来自真实调用，非文档转述。

---

## 一、工具清单（本 Skill 只用 5 个，全部只读）

| 工具 | 用途 | 必填参数 | 返回格式 |
|---|---|---|---|
| `now-time-info` | 取服务器当前时间，用于判断兑换券时效 | 无 | JSON |
| `query-my-account` | 取积分账户（可用/累计/过期/冻结） | 无 | JSON |
| `mall-points-products` | 取积分可兑商品券列表 | `catRuleIds`（可选） | JSON |
| `mall-product-detail` | 取某 SPU 的 SKU 规格与兑换条件 | `spuId` | JSON |
| `query-lottery-info` | 取抽奖活动规则与单次消耗，**不执行抽奖** | 无 | JSON |

辅助（本 Skill 用于"空态引导"，非核心）：`query-my-coupons`（返回 **Markdown**，非 JSON）。

**写操作（`mall-create-order`、`draw-lottery`）本 Skill 一律不调用。**

---

## 二、字段陷阱（实测发现，直接影响实现）

### 陷阱 1：`price` 字段恒为 `"0"`，折算率不能读它

`mall-points-products` 的文档把 `price` 描述为"所需金额"，但实测 24 条商品 **全部返回 `"0"`**。

因此"积分值不值钱"**必须从 `selling` 文案里解析金额**，而不是读 `price`。

- 实际含义：这些是**纯积分兑换券**（`price=0` 表示兑换时不额外付现金），券面价值写在 `selling` 里。
- 解析规则：`selling` 形如 `"18.8元麦辣鸡腿汉堡两件套"`、`"10.9元指定小食任选1份"`、`"3.9元圆筒冰淇淋1份"`，取开头的 `(\d+(\.\d+)?)元` 作为券面金额。

**折算率 = 券面金额 ÷ point（元/积分）**，越大越划算。

实测样例（`catRuleIds=1>4`，仅列当前有效的 9 条）：

| spuId | 商品 | point | 券面金额 | 折算率（元/积分） |
|---|---|---|---|---|
| 17074 | 6.9元香芋派/菠萝派任选 | 50 | 6.9 | **0.1380** |
| 17073 | 6.9元可乐麦炫酷 | 50 | 6.9 | **0.1380** |
| 17075 | 3.9元圆筒冰淇淋 | 100 | 3.9 | 0.0390 |
| 17076 | 10.9元指定小食任选 | 100 | 10.9 | 0.1090 |
| 17077 | 10.9元指定小食任选 | 200 | 10.9 | 0.0545 |
| 17078 | 16.9元2份麦辣鸡翅 | 200 | 16.9 | 0.0845 |
| 17079 | 16.9元麦辣可乐组合 | 500 | 16.9 | 0.0338 |
| 17080 | 17.9元板烧可乐组合 | 500 | 17.9 | 0.0358 |
| 17081 | 21.9元巨无霸可乐组合 | 800 | 21.9 | 0.0274 |

> 结论：同为"小食/派"，**50 积分档（0.138）比 200 积分档（0.0545）划算 2.5 倍**。这种差异用户自己算不出来，是 S1 的核心输出。

### 陷阱 2：`status=2` 不代表当前可兑换，必须比对 `downTime`

实测 24 条商品 **全部** `status=2`（"上架"），但其中 **15 条 `downTime` 已过期**（如 `2026-03-21`、`2026-06-24`，而当前时间是 `2026-10-12`）。

**可兑换判定必须同时满足**：`status == 2` **且** `upTime <= now <= downTime`。

这条过滤是 S1 的真实价值点——商城原始列表里混着大量已过期券，用户肉眼分辨不出，兑换时才发现失效。

`status` 取值：1-仓库中 / 2-上架 / 3-售罄 / 4-下架 / 5-预热。

### 陷阱 3：过期积分字段拼写是 `Mouth`，不是 `Month`

`query-my-account` 返回的三个字段官方拼写为：

- `currentMouthExpirePoint`（本月将过期）
- `lastMouthExpirePoint`（上月已过期）
- `nextMouthExpirePoint`（下月将过期）

**照抄，不要"修正"成 Month**，否则取不到值。实测已确认。

其余字段：`availablePoint`（可用）、`accumulativePoint`（累计）、`expiredPoint`（已过期）、`frozenPoint`（冻结）、`usedPoint`（已使用）、`currency`（币种，实测值 `"麦享会积分"`）、`accountId`（账户号）。
**所有积分类字段均为 string 类型**，不是 number，做算术前必须转数值。

### 陷阱 4：抽奖能否参与，只能看 `drawDecision`，不能自己推断

`query-lottery-info` 的 `drawDecision` 是**服务端计算的资源决策**：

- `resourceEligible`（bool）：当前积分/次数是否满足本次抽奖
- `reason`：不足原因（满足时为 `null`），实测返回 `"积分不足"`
- `nextConsumption`：本次实际消耗，`type` 为 `NONE`/`POINTS`/`CHANCES`

**严禁**根据 `availableTimes` 为 0/null 或 `drawTypeText` 自行推断能否抽奖。混合规则下 `availableTimes=0` 且 `nextConsumption.type=POINTS` 仍可用积分抽。

实测值：`activityCode=dtx2`，活动名"麦麦积分抽奖"，`beginTime=2025-01-26`，`endTime=2030-12-31`，`drawPoint="24"`（24 积分/次），`drawTypeText="消耗积分抽奖"`，奖品 10 项（含"30积分"积分奖）。

### 陷阱 5：返回格式不统一

同一批工具里返回格式三种并存：

| 工具 | 格式 | 处理方式 |
|---|---|---|
| `query-my-account`、`mall-points-products`、`query-lottery-info`、`query-my-prizes` | JSON | 直接解析 |
| `query-my-coupons` | **Markdown** | 正则提取，不可 `JSON.parse` |
| `list-nutrition-foods`（本 Skill 不用） | **TOON** | 同上 |

把 `query-my-coupons` 的返回当 JSON 解析会直接抛错。

---

## 三、实测原始返回摘要（完整见 evidence/）

**`query-my-account`（testing 账户为手机号自动注册的新号）**

```
availablePoint=0  accumulativePoint=0  currentMouthExpirePoint=0
expiredPoint=0  frozenPoint=0  lastMouthExpirePoint=0
nextMouthExpirePoint=0  usedPoint=0  currency=麦享会积分
```

**`mall-points-products`（catRuleIds=1>4）**：24 条；当前有效 9 条（17073~17081），已过期 15 条。

**`query-lottery-info`**：`activityStatusText=进行中`；`drawPoint=24`；`drawDecision.resourceEligible=false`，`reason=积分不足`。

**`query-my-prizes`**：`prizes=[]`，`status.code=SUCCESS`，`hasMore=false` —— **这就是天然空态样例（R1.4 证据）**。

**`query-my-coupons`**：2 张新人券（中杯美式7折 ¥13.3、中杯奶铁7折 ¥14.7），有效期 2026-10-12 ~ 2026-11-10。

---

## 四、空态与降级（三种真实场景）

| 场景 | 触发条件 | 期望输出 |
|---|---|---|
| 零积分 | `availablePoint == 0` | 不报错；提示"当前 0 积分"，并引导：① 抽奖需 24 分/次但暂不可参与 ② 商城最低 50 分可兑，可先下单攒分 ③ 账户内若有券，提示先用券 |
| 无可兑商品 | 时效过滤后列表为空 | 提示"当前时段暂无可兑换商品券"，列出下次开兑时间（取 `upTime` 最早的预热项） |
| 无抽奖记录 | `query-my-prizes` 返回 `prizes=[]` | 提示"暂无抽奖记录"，并展示当前抽奖单次成本与首个奖品，供用户判断是否尝试 |

三种空态**均不得抛出未捕获异常**，且必须给出下一步可执行建议。
