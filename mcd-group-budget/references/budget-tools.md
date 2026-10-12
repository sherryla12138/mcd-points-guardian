# 团餐工具字段实测与陷阱（references/budget-tools.md）

> 本文件记录 `mcd-group-budget` 调用麦当劳 MCP 团餐/预算相关工具的**真实返回结构**与实现陷阱。所有内容来自 `evidence/` 原始返回，非文档转述。
> 适用场景：实现或维护本 Skill 时逐条核对。

---

## 1. `query-nearby-stores`（选店，到店自取）⚠️ 重点

入参：`searchType`(2=按位置)、`city`、`keyword`、`beType`(1=到店自取 / 5=得来速)
返回 `data[]`：

- `storeCode`：门店编码（如 `1450104`）——到店自取场景**无 `beCode`**
- `storeName` / `address` / `distance`
- `reservation`：是否支持预约；`reservationTimeOptions` 需完整展示

**陷阱**：默认 `searchType=1`（收藏餐厅），新账号收藏为空会报错"收藏餐厅列表为空，请您提供城市+关键词进行搜索"——必须显式 `searchType=2` + `city` + `keyword`。

## 2. `query-meals`（餐品列表）⚠️ 重点

入参：`storeCode` + `orderType`(1 到店 / 2 外送) + **`beType`**（必填！）
返回 `data`：

- `data.categories[].meals[].code`：**餐品唯一编码**（如 `9900016076`）——**多层嵌套**，不是顶层字段
- `data.categories[].meals[].name` / `tags`
- `data.meals.{code}.currentPrice`：现价（字符串）
- `data.meals.{code}.discountType`：优惠类型（早餐卡/促销/麦金卡…）

**陷阱 1（最易踩）**：`query-meals` **必须传 `beType`**，否则返回 `code:400, message:"beType1-到店取餐，2-麦乐送到家，5-得来速(DT)，6-企业团餐"`。第一次未传即踩中。
**陷阱 2**：餐品编码在 `categories[].meals[].code` 嵌套里，遍历时需下钻两层。

## 3. `calculate-price`（批量总价）⚠️ 重点

入参：`storeCode` + `orderType` + `beType` + `items:[{productCode, quantity, couponId?, modification?}]`
返回 `data`：

- `price`：**总价，单位分**（如 `55960` = ¥559.60）——展示 ÷100
- `productOriginalPrice` / `originalPrice` / `discount`：原价/折扣
- `productList[]`：每项 `productName`/`quantity`/`subtotal`（分）
- `takeWayList[]`：堂食/外带等取餐方式

**陷阱**：`price` 是分。实测 10×鸡薯双全盒 + 8×龙焰芝士棒鸡腿堡三件套 + 6×龙焰鸡腿堡三件套 = `productPrice 55960` 分 = ¥559.60。

## 4. `available-coupons`（我的可领券）

入参：无
返回 `data[]`：可领取的"麦麦省"券（名称/图片/状态/促销标签）。实测返回 7 张。

## 5. `query-store-coupons`（门店可用券）

入参：`storeCode` + `orderType`(1 到店 / 2 外送) + `beType`
返回 `data[]`：该门店可叠加券。实测返回 2 张。

## 6. `list-nutrition-foods`（营养）

入参：无
返回：**格式不统一**（TOON / JSON 混合），含能量/蛋白质/脂肪/碳水/钠/钙。

**陷阱**：**不能**对整段响应 `JSON.parse`；需容错提取可用字段，解析失败则跳过营养项（不报错）。

## 7. `query-promotions`（团餐满减）⚠️ 团餐专属

入参：`storeCode` + `beCode` + `orderType=2` + **`beType=6`**（企业团餐）
返回：满减/满折规则（`promotionType` 31=满额减 / 33=订单折扣；`ruleCategory` 30=满折 / 40=满减）。

**陷阱**：仅 `beType=6` 可用；且 `beCode` 须从 `delivery-query-stores` 获取，而后者需用户已配置的团餐配送地址（`delivery-query-addresses` 为空则不可达）。本账号无团餐地址 → 该路径如实标注不可达，不伪造。

## 8. `query-meal-assistance`（团餐助餐）⚠️ 团餐专属

入参：同 `query-promotions`（需 `beType=6` + `beCode`）
返回：助餐服务列表（`gmServiceCode` 传给 `calculate-price`）。
**陷阱**：实测 `beType=1` 时返回 `code:610501, message:"仅支持企业团餐场景下使用助餐服务"`——证实为团餐专属。

## 9. `now-time-info`（时间基准）

入参：无，返回 `data.date` / `dayOfWeek` / `timestamp`(毫秒)。

---

## 空态与降级

| 场景 | 判定 | 处理 |
|---|---|---|
| 无可选门店 | `query-nearby-stores` 空 | 提示换城市/关键词 |
| 无餐品 | `query-meals.data` 空 | 提示门店未上架 |
| 无可用券 | `query-store-coupons` 空 | 说明该门店当前无券 |
| 无团餐地址 | `query-promotions` 报"仅企业团餐" | 如实说明需配置团餐配送地址 |
| 营养解析异常 | TOON 解析失败 | 跳过营养项，不报错 |

---

## 证据索引（真实调用，未手改）

| 证据文件 | 工具 | 说明 |
|---|---|---|
| `evidence/01_now-time-info.json` | `now-time-info` | 时间基准 |
| `evidence/02_query-nearby-stores.json` | `query-nearby-stores` | 选店（storeCode=1450104） |
| `evidence/03_query-meals.json` | `query-meals` | 餐品 code 与现价 |
| `evidence/04_query-meal-detail.json` | `query-meal-detail` | 餐品组成 |
| `evidence/05_query-promotions.json` | `query-promotions` | 满减规则（beType=1 实测） |
| `evidence/06_query-meal-assistance.json` | `query-meal-assistance` | 助餐（beType=1 如实报团餐专属） |
| `evidence/07_calculate-price.json` | `calculate-price` | 24 份总价 ¥559.60 |
| `evidence/08_list-nutrition-foods.json` | `list-nutrition-foods` | 营养数据 |
| `evidence/09_available-coupons.json` | `available-coupons` | 可领券 7 张 |
| `evidence/10_query-store-coupons.json` | `query-store-coupons` | 门店可用券 2 张 |
