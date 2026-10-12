---
name: mcd-group-budget
description: 麦当劳团餐预算官。当用户问"我们 10 个人吃麦当劳要多少钱""团建点餐怎么划算""多人聚餐预算多少"时使用。基于麦当劳 MCP 只读工具做四件事：多人套餐篮与总价测算、优惠券核销建议、营养平衡建议、企业团餐增强路径说明。纯只读——不代下单。
---

# 麦当劳团餐预算官（MCD Group Budget）

## 它解决什么问题

部门团建、朋友聚餐、班级活动——凡"一群人吃麦当劳"，核心问题都是**预算**：点哪些、点多少、总价多少、能用什么券、营养搭得平不平。

官方 App 的痛点：点餐页是"一人一单"视角，**没有"按人数凑套餐 + 算总价 + 比优惠"的聚合视图**；营养信息散在单品页；企业团餐（beType=6）的满减满折规则更是藏在团餐专属入口里，普通用户看不见。

本 Skill 把"人数 → 套餐篮 → 总价 → 优惠 → 营养"一次算完。

## 四条能力

### 能力 1：多人套餐篮与总价测算

**触发**："10 个人吃麦当劳要多少" / "团建点餐怎么划算" / "帮我配个 20 人份"

**调用链**：`query-nearby-stores`（选店，`beType=1` 到店自取）→ `query-meals`（取餐品 `code` 与现价）→ `calculate-price`（批量 `quantity` 算总价）

**计算规则**：

1. 选店：`query-nearby-stores` 需 `searchType=2` + `city` + `keyword` + `beType`；到店自取 `beType=1` 返回门店 `storeCode`（无 `beCode`）
2. 取餐品：`query-meals` **必须传 `beType`**（实测不传返回 `code:400`）——`beType=1` 到店取餐 / `beType=6` 企业团餐
3. 餐品编码取 `data.categories[].meals[].code`（如 `9900016076`）
4. 批量测算：`calculate-price` 的 `items` 数组每项 `{productCode, quantity}`，`quantity>=1`；返回 `price` 单位为**分**（如 `55960` = ¥559.60），展示 ÷100
5. 按人数给建议：人均 = 总价 ÷ 人数；并给出"汉堡+薯条+饮料"等搭配档位（参考 `query-meals` 的团餐搭配规则：20 元以下小食 / 20–30 汉堡+小食 / …）

**输出格式**（表格）：

| 餐品 | 单价 | 数量 | 小计 |
|---|---|---|---|
| "我就喜欢"鸡薯双全盒 | ¥15.90 | 10 | ¥159.00 |
| 龙焰芝士棒鸡腿堡三件套 | ¥29.90 | 8 | ¥239.20 |
| 龙焰鸡腿堡三件套 | ¥26.90 | 6 | ¥161.40 |
| **合计** | | **24 份** | **¥559.60** |

### 能力 2：优惠券核销建议

**触发**："有什么券能用" / "这家店能用什么券"

**调用链**：`available-coupons`（我的可领券）→ `query-store-coupons`（指定门店可用券）

**计算规则**：

- `available-coupons` 返回用户可领取的"麦麦省"券列表（名称/图片/状态/促销标签）
- `query-store-coupons` 需 `storeCode` + `orderType`（到店 `1` / 外送 `2`），返回该门店可叠加的券
- 输出时标注哪些券"下单前可领""该门店可用"，并提示叠加顺序（先领后叠加）

### 能力 3：营养平衡建议

**触发**："这顿健康吗" / "热量多少"

**调用链**：`list-nutrition-foods`（能量/蛋白/脂肪/碳水/钠/钙）

**计算规则**：按套餐篮里的餐品，匹配营养成分，给出总热量与"蛋白/蔬菜是否够"的提示，避免全油炸组合。营养数据返回格式不一（TOON/JSON），需容错解析，不能直接 `JSON.parse` 整个响应。

### 能力 4：企业团餐增强路径（说明型）

**触发**："企业团餐有什么满减" / "团餐怎么送"

**说明**：`query-promotions`（满减/满折）、`query-meal-assistance`（助餐服务）**仅 `beType=6` 企业团餐场景可用**，且需先经 `delivery-query-stores` 拿到 `storeCode`+`beCode`（依赖用户已配置的团餐配送地址）。本账号无团餐地址时该路径不可达——Skill 应如实告知用户"企业团餐专属优惠需先在麦当劳 App 配置团餐配送地址"，并给出手动路径，而非伪造结果。

## 工具调用总览

| 顺序 | 工具 | 必要性 | 用途 |
|---|---|---|---|
| 1 | `now-time-info` | 必调 | 时效基准 |
| 2 | `query-nearby-stores` | 必调 | 选店（`beType=1`） |
| 3 | `query-meals` | 必调 | 餐品 `code` 与现价（**须 `beType`**） |
| 4 | `calculate-price` | 必调 | 批量总价（分→元） |
| 5 | `available-coupons` | 能力 2 | 可领券 |
| 6 | `query-store-coupons` | 能力 2 | 门店可用券 |
| 7 | `list-nutrition-foods` | 能力 3 | 营养数据 |
| 8 | `query-promotions` | 能力 4 | 团餐满减（beType=6） |
| 9 | `query-meal-assistance` | 能力 4 | 团餐助餐（beType=6） |

**降级顺序**：`query-meals` 无数据 → 提示门店暂无可点餐品；`calculate-price` 失败 → 退化为"单价×数量"估算；`list-nutrition-foods` 解析异常 → 跳过营养项，不报错。

## 空态降级（不得抛异常）

| 场景 | 判定 | 输出 |
|---|---|---|
| 无可选门店 | `query-nearby-stores` 空 | 提示换城市/关键词重试 |
| 无餐品 | `query-meals` `data` 空 | 提示该门店暂未上架 |
| 无可用券 | `query-store-coupons` 空 | 说明该门店当前无可用券 |
| 无团餐地址 | `query-promotions` 返回"仅支持企业团餐" | 如实说明需配置团餐配送地址 |

## 安全边界（用户可见）

- 本 Skill **全程只读**。仅调用上述 9 个工具。
- **不调用**任何写操作：`create-order`（点餐下单）、`mall-create-order`（兑换）、`party-order-create`（派对订场）。本 Skill 只做预算参谋，是否下单由用户自行在官方渠道完成。
- 涉及消耗的动作只做测算与建议，是否执行由用户决定。

## 字段陷阱速查

完整说明见 `references/budget-tools.md`，实现时必读：

1. `query-meals` / `query-meal-detail` / `calculate-price` **必须传 `beType`**，否则 `code:400`
2. `calculate-price.price` 单位是**分**，展示 ÷100
3. 餐品编码字段是 `data.categories[].meals[].code`（多层嵌套）
4. `query-promotions` / `query-meal-assistance` 仅 `beType=6` 可用，且需团餐 `beCode`
5. `list-nutrition-foods` 返回格式不一（TOON/JSON），**不能**整体 `JSON.parse`
6. `query-nearby-stores` 到店自取（`beType=1`）返回门店**无 `beCode`**
