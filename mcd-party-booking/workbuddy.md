# WorkBuddy 开发对话上下文（workbuddy.md）

> 本文件记录 mcd-party-booking 在 WorkBuddy 中的真实开发过程，供比赛专项奖励审核。所有调用日期与工具名均为实际发生，非事后编造。

## 项目信息

- 项目名称：麦当劳派对订场助手（mcd-party-booking）
- 赛事：麦当劳程序员创意开发大赛（M-China / 金拱门中国，WorkBuddy 为官方合作伙伴）
- 开发平台：WorkBuddy 桌面端（Windows 11）
- 开发日期：**2026-10-12**
- 主力工具：Goal 技能做方向规划；WorkBuddy 内置麦当劳 MCP 连接器做真实调用

## 开发时间线（真实）

### 1. 方向来源（10-12）
- 本 Skill 是 Goal 技能编译出的三个参赛方向之一（S1 积分守门员 / S2 派对订场助手 / S3 团餐预算官）。
- 选择派对场景的原因：麦当劳线下派对（生日/主题/读书会）信息在官方 App 分散，且"可约日期/场次/成团人数"从不集中展示——这是一个真实、未被现有作品覆盖的决策痛点。

### 2. 工具探测（10-12 11:19 起，直连 MCP）
- 通过 `tools/list` 确认 35 个工具，锁定派对相关 6 个：`query-party-city` / `query-party-store` / `query-party-store-date` / `query-party-store-session` / `mall-points-products` / `campaign-calendar`，另加 `now-time-info` 做时效基准。
- 本机未安装 Git，且用户 GitHub 网页曾打不开；最终全部证据通过**直连 `https://mcp.mcd.cn` 的 Streamable HTTP 端点**（用 `mcp.json` 中的 Token）采集，不经过网页。

### 3. S2 字段实测与关键发现（10-12 11:22 起）
真实调用链（spuId=8516 尊享版生日派对 → 上海）：
- `query-party-city(8516)`：返回城市 `code`/`latitude`/`longitude`
- `query-party-store(310100, 31.2304, 121.4737, 8516)`：返回上海派对门店，**门店编码字段是 `code`（不是 `storeCode`）**——这是最先踩的坑，第一次因误以为是 `storeCode` 导致链中断
- `query-party-store-date(1450713, 8516)`：返回可约日期 `date`（如 `2026-10-15`），**字段名是 `date` 不是 `dateStr`**
- `query-party-store-session(8516, 1450713, "2026-10-15")`：返回场次 `12:00-13:30`/`18:30-20:00`，成团 5–12 人，余量 12，场次价 `13800` 分

实测挖掘到的关键事实（决定实现方式）：
- `query-party-store` 返回门店编码字段名是 `code`，但下游入参名是 `storeCode`——必须做一次字段映射
- 派对商品 `price` 是真实金额（`50`/`138`/`45`），与积分商城 `price` 恒为 `"0"` 完全不同
- 场次 `price` 单位为分（`13800` = ¥138），展示须 ÷100

### 4. 产出物
- `SKILL.md`：三能力（选址四步链 / 成本核算 / 活动日历）+ 空态降级 + 安全边界
- `references/party-tools.md`：7 个工具的字段实测与陷阱，实现依据
- `README.md`：首屏价值 + 目标用户 + 差异化避让表
- `config/mcp-config.example.json`：仅占位符，无真实凭证
- `evidence/01~07`：七个工具原始返回（未手改，含 traceId）

## 真实调用证据索引

| 证据文件 | 工具 | 说明 |
|---|---|---|
| `evidence/01_now-time-info.json` | `now-time-info` | 时间基准 |
| `evidence/02_mall-points-products.json` | `mall-points-products` | 派对 spuId 与价目 |
| `evidence/04_query-party-store.json` | `query-party-store` | 上海派对门店（字段 `code`） |
| `evidence/05_query-party-store-date.json` | `query-party-store-date` | 可约日期（字段 `date`） |
| `evidence/06_query-party-store-session.json` | `query-party-store-session` | 当天场次 |
| `evidence/07_campaign-calendar.json` | `campaign-calendar` | 派对活动日历 |

> 原始返回均未手改，无 `accountId`/手机号/真实 Token 泄露。可随时与 `open.mcd.cn/mcp/doc` 逐一核验工具名与字段。

## 关键决策与取舍

- **坚持纯只读**：S2 定位为"派对选址 + 选期 + 成本参谋"，不调用 `party-order-create`；是否订场由用户自行在官方渠道完成。
- **证据采集绕过网页限制**：用户 GitHub 网页曾打不开、本机无 Git，故证据全部通过直连 MCP 端点采集，保证"真实调用"硬要求可达成。
- **记录真实过程**：本文件按实际发生日期与工具调用撰写，未做美化；字段名 `code` vs `storeCode`、`date` vs `dateStr` 的坑均为实测踩中后记录。
