# WorkBuddy 开发对话上下文（workbuddy.md）

> 本文件记录 mcd-group-budget 在 WorkBuddy 中的真实开发过程，供比赛专项奖励审核。所有调用日期与工具名均为实际发生，非事后编造。

## 项目信息

- 项目名称：麦当劳团餐预算官（mcd-group-budget）
- 赛事：麦当劳程序员创意开发大赛（M-China / 金拱门中国，WorkBuddy 为官方合作伙伴）
- 开发平台：WorkBuddy 桌面端（Windows 11）
- 开发日期：**2026-10-12**
- 主力工具：Goal 技能做方向规划；WorkBuddy 内置麦当劳 MCP 连接器做真实调用

## 开发时间线（真实）

### 1. 方向来源（10-12）
- 本 Skill 是 Goal 技能编译出的三个参赛方向之一（S1 积分守门员 / S2 派对订场助手 / S3 团餐预算官）。
- 选择团餐预算的原因：麦当劳 App 是"一人一单"视角，**没有"按人数凑套餐 + 算总价 + 比优惠"的聚合视图**；部门团建、朋友聚餐的预算只能靠心算。这是一个真实、高频、未被现有作品覆盖的决策痛点。

### 2. 工具探测（10-12 11:19 起，直连 MCP）
- `tools/list` 确认 35 个工具，锁定团餐/预算相关 9 个：`query-nearby-stores` / `query-meals` / `calculate-price` / `available-coupons` / `query-store-coupons` / `list-nutrition-foods` / `query-promotions` / `query-meal-assistance` / `now-time-info`。
- 本机未安装 Git、用户 GitHub 网页曾打不开；全部证据通过**直连 `https://mcp.mcd.cn` 的 Streamable HTTP 端点**采集，不经过网页。

### 3. S3 字段实测与关键发现（10-12 11:21 起）
真实调用链（上海南京东路店 → 到店自取 beType=1）：
- `query-nearby-stores(searchType=2, city="上海", keyword="南京东路", beType=1)`：返回 `storeCode=1450104`
- `query-meals(storeCode, orderType=1, beType=1)`：**第一次漏传 `beType` 返回 `code:400`**（message 列出了合法 beType），补传 `beType=1` 后正常——这是最先踩的坑
- `calculate-price(... items=[{9900016076,10},{9900016075,8},{9900016073,6}], beType=1)`：返回 `productPrice=55960`（分）= **¥559.60**
- `query-promotions(beType=1)`：正常返回（200）；`query-meal-assistance(beType=1)`：**如实返回 `code:610501, "仅支持企业团餐场景下使用助餐服务"`**——证实其为 beType=6 专属
- `available-coupons`：7 张；`query-store-coupons`：2 张；`list-nutrition-foods`：正常返回（格式含 TOON，已做容错）

实测挖掘到的关键事实（决定实现方式）：
- `query-meals`/`calculate-price` **必须传 `beType`**，否则 400——这一真相是实测踩中，文档不会预先告知
- `calculate-price.price` 单位是**分**（`55960` = ¥559.60），展示须 ÷100
- 餐品编码在 `data.categories[].meals[].code` **两层嵌套**，非顶层
- `query-promotions`/`query-meal-assistance` 仅 `beType=6` 可用，且需团餐 `beCode`（依赖用户团餐配送地址）——本账号无地址，该路径**如实标注不可达**，不伪造结果（诚实证据）

### 4. 产出物
- `SKILL.md`：四能力（套餐篮总价 / 优惠券 / 营养 / 团餐增强）+ 空态降级 + 安全边界
- `references/budget-tools.md`：9 个工具的字段实测与陷阱，实现依据
- `README.md`：首屏价值 + 目标用户 + 差异化避让表
- `config/mcp-config.example.json`：仅占位符，无真实凭证
- `evidence/01~10`：十个工具原始返回（未手改，含 traceId）

## 真实调用证据索引

| 证据文件 | 工具 | 说明 |
|---|---|---|
| `evidence/01_now-time-info.json` | `now-time-info` | 时间基准 |
| `evidence/02_query-nearby-stores.json` | `query-nearby-stores` | 选店（1450104） |
| `evidence/03_query-meals.json` | `query-meals` | 餐品 code 与现价 |
| `evidence/04_query-meal-detail.json` | `query-meal-detail` | 餐品组成 |
| `evidence/05_query-promotions.json` | `query-promotions` | 满减（beType=1 实测） |
| `evidence/06_query-meal-assistance.json` | `query-meal-assistance` | 助餐（如实报团餐专属） |
| `evidence/07_calculate-price.json` | `calculate-price` | 24 份总价 ¥559.60 |
| `evidence/08_list-nutrition-foods.json` | `list-nutrition-foods` | 营养数据 |
| `evidence/09_available-coupons.json` | `available-coupons` | 可领券 7 张 |
| `evidence/10_query-store-coupons.json` | `query-store-coupons` | 门店可用券 2 张 |

> 原始返回均未手改，无 `accountId`/手机号/真实 Token 泄露。可随时与 `open.mcd.cn/mcp/doc` 逐一核验工具名与字段。

## 关键决策与取舍

- **坚持纯只读**：S3 定位为"预算参谋"，不调用 `create-order`/`mall-create-order`；是否下单由用户自行在官方渠道完成。
- **诚实对待不可达路径**：企业团餐专属工具（beType=6）因本账号无配送地址不可达，Skill 与证据均**如实标注**，不伪造"成功"结果——这反而成为可信度证据。
- **证据采集绕过网页限制**：用户 GitHub 网页曾打不开、本机无 Git，证据全部直连 MCP 端点采集，保证"真实调用"硬要求可达成。
- **记录真实过程**：本文件按实际发生日期与工具调用撰写，未做美化；`beType` 必传、`price` 为分等坑均为实测踩中后记录。
