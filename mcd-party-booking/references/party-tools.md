# 派对工具字段实测与陷阱（references/party-tools.md）

> 本文件记录 `mcd-party-booking` 调用麦当劳 MCP 派对相关工具的**真实返回结构**与实现陷阱。所有内容来自 `evidence/` 原始返回，非文档转述。
> 适用场景：实现或维护本 Skill 时逐条核对。

---

## 1. `mall-points-products`（取派对 spuId 与价目）

返回 `data[]`，关键字段：

- `spuName`：商品名（如"麦当劳奇妙生日派对"）
- `spuId`：**派对链必须**，从类目含"派对/读书会/品鉴会/体验营"的条目取。实测样例：`1779`(奇妙生日派对)、`8516`(一起开心鸭尊享版)、`1830`(亲子读书会)、`1065`(迪士尼探索)、`5247`(动物城尊享)、`5224`(动物城普通)
- `price`：**真实金额**（如 `50`/`138`/`45`），与积分商城 `price` 恒为 `"0"` 不同——派对场景直接读 `price`
- `catName`：类目（"生日类派对"/"读书会"等）
- `status`：`2`=上架；`downTime` 为下架时间，派对商品多到 2026-12-31 / 2027-12-31
- `selling`：卖点文案

**陷阱**：不可把积分商城的"`price` 恒 0、折算率从 `selling` 解析"逻辑套到派对商品上。

## 2. `query-party-city`（城市列表）

入参：`spuId`
返回 `data[]`：

- `code`：城市编码（**必须原样回传** `query-party-store`）
- `latitude` / `longitude`：经纬度（**必须原样回传**）
- `name`：城市名
- `initial` / `sortNo`：首字母 / 排序

**陷阱**：`code`/`latitude`/`longitude` 三者需成组回传，缺一 `query-party-store` 不返回数据。

## 3. `query-party-store`（城市下的派对门店）⚠️ 重点

入参：`code` + `latitude` + `longitude` + `spuId`
返回 `data[]`：

- `code`：**门店编码**（如 `1450713`）——**注意是 `code`，不是 `storeCode`**
- `name` / `shortName`：门店名
- `address`：地址
- `cityCode` / `cityName`：城市
- `businessStatus`：`1`=营业中
- `distance` / `distanceText`：距离

**陷阱 1（最易踩）**：门店编码字段名是 `code`，但后续 `query-party-store-date` / `query-party-store-session` 的**入参名**却是 `storeCode`。即"返回叫 `code`，传入叫 `storeCode`"，必须做一次字段映射。
**陷阱 2**：若 `code`/`lat`/`lng` 任一不符，返回空数组（不是报错），需回退到 `query-party-city` 重新取。

## 4. `query-party-store-date`（可约日期）

入参：`storeCode` + `spuId`
返回 `data[]`：

- `date`：**可约日期**（如 `2026-10-15`）——**注意字段名是 `date`，不是 `dateStr`**
- `storeCode`：门店编码

**陷阱**：`dateStr` 是**下一步入参**（传给 `query-party-store-session`），不是本工具返回字段。实现时拿 `date` 的值当作 `dateStr` 传入即可。

## 5. `query-party-store-session`（当天场次）⚠️ 重点

入参：`spuId` + `storeCode` + `dateStr`
返回 `data[]`：

- `timeStart` / `timeEnd`：场次起止（如 `12:00`/`13:30`）
- `price`：**场次价，单位分**（如 `13800` = ¥138）——展示 ÷100
- `partyMin` / `partyMax`：最小/最大成团人数（如 `5`/`12`）
- `leftNum`：剩余场次（如 `12`）
- `cookEmpNo` / `opEmpNo` / `rgmEmpNo`：大厨/操作人/店长编码（一般不需展示）

**陷阱**：`price` 是分。成本核算时 `人均 = price/100 ÷ partyMin`（按成团下限得人均上限）。

## 6. `campaign-calendar`（活动日历）

入参：无
返回：Markdown 文本，含当月进行中/往期/未来活动（标题、介绍、图片）。按"派对/品鉴会/读书会/主题"关键词过滤即可。

## 7. `now-time-info`（时间基准）

入参：无
返回：`data` 含 `date`(`yyyy-MM-dd`)、`dayOfWeek`、`formatted`、`timestamp`(毫秒)。用于"今天几号、周几"的时效提示。

---

## 空态与降级

| 场景 | 判定 | 处理 |
|---|---|---|
| 城市无派对门店 | `query-party-store` 返回 `[]` | 提示该城市暂不支持，引导换城市 |
| 日期无场次 | `query-party-store-session` 返回 `[]` | 提示当天无场次，展示最近有场次日期 |
| 场次余量 0 | `leftNum == 0` | 提示已满，引导其他场次 |

---

## 证据索引（真实调用，未手改）

| 证据文件 | 工具 | 说明 |
|---|---|---|
| `evidence/01_now-time-info.json` | `now-time-info` | 时间基准 |
| `evidence/02_mall-points-products.json` | `mall-points-products` | 派对 spuId 与价目 |
| `evidence/04_query-party-store.json` | `query-party-store` | 上海派对门店（字段 `code`） |
| `evidence/05_query-party-store-date.json` | `query-party-store-date` | 可约日期（字段 `date`） |
| `evidence/06_query-party-store-session.json` | `query-party-store-session` | 当天场次（价/成团/余量） |
| `evidence/07_campaign-calendar.json` | `campaign-calendar` | 派对活动日历 |
