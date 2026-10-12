---
name: mcd-party-booking
description: 麦当劳派对订场助手。当用户问"哪里能办麦当劳生日派对""主题派对怎么预约""派对多少钱、几人成团"时使用。基于麦当劳 MCP 只读工具做三件事：派对场馆选址（城市→门店→日期→场次）、派对成本核算、派对活动日历。纯只读——不代下单、不代订场。
---

# 麦当劳派对订场助手（MCD Party Booking）

## 它解决什么问题

麦当劳在全国大量门店提供**生日派对 / 主题派对 / 亲子读书会 / 品鉴会**等线下活动，但官方 App 里这些信息分散、且"哪天有场次、几人成团、多少钱"从不集中展示。

用户侧的真实困境：想给孩子办生日派对，却不知道**所在城市哪几家店能办**、**最近可约日期**、**当天几个场次**、**最少几人成团**。反复翻 App、打电话问店员，效率低。

本 Skill 把"选址 → 选日期 → 选场次 → 算成本"四步一次跑完，输出可立即行动的清单。

## 三条能力

### 能力 1：派对场馆选址（四步链）

**触发**："哪里能办麦当劳生日派对" / "上海哪几家店有主题派对" / "我想预约派对"

**调用链**：`query-party-city`（取城市 code/经纬度）→ `query-party-store`（取门店 code/地址）→ `query-party-store-date`（取可约日期）→ `query-party-store-session`（取当天场次）

**取数规则**：

- 派对商品 `spuId` 必须先确定：来自 `mall-points-products`（类目含"派对/读书会/品鉴会/体验营"），例如"奇妙生日派对" `spuId=1779`、"一起开心鸭尊享版生日派对" `spuId=8516`、`"亲子读书会" spuId=1830`
- `query-party-city` 返回 `code` / `latitude` / `longitude`，**必须原样回传**给 `query-party-store`
- `query-party-store` 返回的门店编码字段是 **`code`**（不是 `storeCode`）——这是最常见踩坑点
- `query-party-store-date` 返回日期字段是 **`date`**（形如 `2026-10-15`）；该值即传给 `query-party-store-session` 的 `dateStr` 参数
- 输出按"城市 → 门店 → 日期 → 场次"逐级收敛，每级只展示用户当前所在/所选层级之下的内容

**输出要点**：用户选定门店后，必须完整展示该店可约日期；选定日期后，必须完整展示当天所有场次（含起止时间、最低成团人数、余量），缺一条都会导致用户无法决策。

### 能力 2：派对成本核算

**触发**："办一场派对多少钱" / "几人成团" / "这个派对套餐值不值"

**调用链**：`mall-points-products`（取派对商品 `price`）→ `query-party-store-session`（取场次 `price`、成团人数）

**计算规则**：

1. **派对商品价目取 `price` 字段即可**——与积分商城不同，派对类商品的 `price` 是真实金额（如"奇妙生日派对" `price=50`，"一起开心鸭尊享版" `price=138`），**不要**走积分商品的"从 `selling` 解析"逻辑
2. 场次价目 `query-party-store-session.price` 单位为**分**（如 `13800` = ¥138），展示时 ÷100 转元
3. 综合"商品价 + 场次价 + 成团人数区间"给出人均成本区间估算（场次价 ÷ 成团下限 = 人均上限参考）

**输出格式**（表格）：

| 派对商品 | 商品价 | 场次价（/场） | 成团人数 | 人均参考 |
|---|---|---|---|---|
| 奇妙生日派对 | ¥50 | ¥138 | 5–12 人 | ¥11.5 起/人（按 12 人摊场次） |

### 能力 3：派对活动日历

**触发**："最近有什么派对活动" / "品鉴会什么时候有"

**调用链**：`campaign-calendar` → 过滤出含"派对/品鉴会/读书会/主题"关键词的活动

**计算规则**：取当月活动列表，仅展示与线下活动相关的条目；标注进行中 / 即将开始，并提示订阅状态。

## 工具调用总览

| 顺序 | 工具 | 必要性 | 用途 |
|---|---|---|---|
| 1 | `now-time-info` | 必调 | 时效判定基准（今天几号、周几） |
| 2 | `mall-points-products` | 必调 | 取派对 `spuId` 与商品价目 |
| 3 | `query-party-city` | 必调 | 城市列表（code/经纬度） |
| 4 | `query-party-store` | 必调 | 城市下的派对门店（字段 `code`） |
| 5 | `query-party-store-date` | 必调 | 门店可约日期（字段 `date`） |
| 6 | `query-party-store-session` | 必调 | 当天场次（时间/成团/余量/价） |
| 7 | `campaign-calendar` | 能力 3 | 派对类活动日历 |

**降级顺序**：若 `query-party-store` 某城市返回空 → 提示该城市暂不支持派对，引导换城市；若 `query-party-store-session` 余量为 0 → 提示该场次已满，引导选其他日期/场次。

## 空态降级（不得抛异常）

| 场景 | 判定 | 输出 |
|---|---|---|
| 城市无派对门店 | `query-party-store` 返回空 | 说明该城市暂不支持，列出相邻有场地的城市 |
| 所选日期无场次 | `query-party-store-session` 为空 | 说明当天无场次，展示最近有场次的日期 |
| 场次余量 0 | `leftNum == 0` | 说明已满，引导其他场次 |

## 安全边界（用户可见）

- 本 Skill **全程只读**。仅调用 `now-time-info`、`mall-points-products`、`query-party-city`、`query-party-store`、`query-party-store-date`、`query-party-store-session`、`campaign-calendar`。
- **不调用** `party-order-create`（派对下单）等任何写操作。本 Skill 只做选址、选期、成本参谋；是否下单由用户自行在官方渠道完成。
- 涉及预约的动作，本 Skill 只给出可约清单与时间，是否提交预订由用户决定。

## 字段陷阱速查

完整说明见 `references/party-tools.md`，实现时必读：

1. `query-party-store` 门店编码字段是 **`code`**（不是 `storeCode`）
2. `query-party-store-date` 日期字段是 **`date`**；`dateStr` 是传给 `query-party-store-session` 的**入参**
3. `query-party-city` 的 `code`/`latitude`/`longitude` 必须**原样回传** `query-party-store`
4. 派对商品价取 `price`（真实金额）；**不要**套用积分商品的 `selling` 解析逻辑
5. 场次 `price` 单位是**分**，展示 ÷100
6. 派对 `spuId` 须从 `mall-points-products`（派对/读书会/品鉴会类目）获取，不能硬编码
