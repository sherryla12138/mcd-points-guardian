# WorkBuddy 开发对话上下文（workbuddy.md）

> 本文件记录 mcd-points-guardian 在 WorkBuddy 中的真实开发过程，供比赛专项奖励审核。所有调用日期与工具名均为实际发生，非事后编造。

## 项目信息

- 项目名称：麦当劳积分守门员（mcd-points-guardian）
- 赛事：麦当劳程序员创意开发大赛（M-China / 金拱门中国，WorkBuddy 为官方合作伙伴）
- 开发平台：WorkBuddy 桌面端（Windows 11）
- 开发日期：**2026-10-12**
- 主力工具：Goal 技能（lark-codex-goal）做方向规划；WorkBuddy 内置麦当劳 MCP 连接器做真实调用

## 开发时间线（真实）

### 1. 赛事调研与方向锁定（10-12 上午）
- 通过 Goal 技能编译 Goal Packet，确定三个参赛方向：S1 积分守门员 / S2 派对订场助手 / S3 团餐预算官。
- 主动检索并避让同赛道已有作品：`junxchan54-maker/mcd-cn-assistant`（活动推送+比价）、「热麦卡路里」（热量估算）、外送自动化版（配送编排）。
- 核实赛事硬规则：报名截止 **2026-10-25 23:59**，排名**唯一依据为 GitHub 公开 Star**——这意味着项目目标函数是"痛点一眼看懂 + README 三秒说清价值"，而非功能最强。

### 2. 接入麦当劳 MCP（10-12 上午，关键阻塞）
- 用户通过 `https://open.mcd.cn/mcp/login`（手机号 + 短信验证码）获取 Bearer Token。排查发现：开放平台首页的"登录"与导航"商户平台"走的是**商户登录**（需商户名+密码，B 端资质），个人开发者进不去；MCP 平台与开放平台是两套独立前端。
- 配置写入 WorkBuddy 自定义连接器 `~/.workbuddy/mcp.json`，`type` 字段对照官方 connector schema 必须是 `streamable-http`（带连字符；写成 `streamablehttp` 不生效）。
- 排查确认：自定义 MCP 不在连接器搜索框（只搜官方市场）；配置在 WorkBuddy **启动时**注入会话，运行中修改须**托盘退出重开**才生效。用户重启后【MCP 服务管理】出现 `mcd-mcp`，显示 **35/35 工具已启用**。

### 3. 连通性验证（10-12 09:02 curl 直连 + 09:35 应用内）
- `initialize`：serverInfo `mcd-mcp 1.0.0`，协议 `2025-06-18`，stateless（每次 POST 带 Authorization 即可）
- `tools/list`：返回 **35** 个工具
- 应用内实测：`now-time-info` 返回服务器时间；`query-my-account` 全字段为 0（该账号为手机号登录时自动注册的麦当劳新会员号）

### 4. S1 字段实测与实现（10-12 09:36 起）
在应用内经 MCP **真实调用**（非纸面设计）以下只读工具：
- `mall-points-products`（catRuleIds=1>4）：24 条商品券
- `query-lottery-info`：抽奖活动进行中，单次 24 积分
- `query-my-prizes`：返回空数组（天然空态）
- `query-my-coupons`：2 张新人券（返回 Markdown）

实测挖掘到的关键事实（决定实现方式）：
- `price` 字段恒为 `"0"`，折算率必须从 `selling` 文案解析 —— 这一真相是实测发现，文档无法提供
- `status==2` 的 24 条中 **15 条 `downTime` 已过期**，必须比对时间才是准的可兑换判定
- 折算率实测相差 **2.5 倍**（"50 分换 6.9 元派"= 0.138 元/分 vs "200 分换 10.9 元小食"= 0.0545 元/分）
- 抽奖能否参与**只看 `drawDecision.resourceEligible`**（实测 false，reason="积分不足"）

### 5. 产出物
- `SKILL.md`：三能力 + 空态降级 + 安全边界
- `references/points-tools.md`：5 工具字段实测与陷阱，实现依据
- `README.md`：首屏价值 + 目标用户 + 差异化避让表
- `config/mcp-config.example.json`：仅占位符，无真实凭证
- `evidence/01~06`：六个工具原始返回（未手改，含 traceId）

## 真实调用证据索引

| 证据文件 | 工具 | 时间 / traceId |
|---|---|---|
| `evidence/01_now-time-info.json` | `now-time-info` | 2026-10-12 09:35:42 |
| `evidence/02_query-my-account.json` | `query-my-account` | 2026-10-12 09:35:43 |
| `evidence/03_mall-points-products.json` | `mall-points-products` | 4a7570fa3bc9b9779ef53fc020e0c4c5 |
| `evidence/04_query-lottery-info.json` | `query-lottery-info` | 157c78a6e351d7621f0617cc63a827e2 |
| `evidence/05_query-my-prizes.json` | `query-my-prizes` | 61772ec11c248bb6627723a4d0239fcd |
| `evidence/06_query-my-coupons.md` | `query-my-coupons` | 2026-10-12 09:36:13 |

> 原始返回均未手改，`accountId` 已脱敏。可随时与 `open.mcd.cn/mcp/doc` 逐一核验工具名与字段。

## 关键决策与取舍

- **坚持纯只读**：S1 定位为"资产体检 + 参谋"，不代下单、不代抽奖；所有写操作列为禁用，安全边界写入 `SKILL.md`。
- **比价算法基于实测而非文档**：`price` 字段恒为 0 是实测发现，避免了建立在废字段上的错误实现。
- **0 积分账号不影响交付**：真实调用证据齐备（满足 R1.1/R1.3），且空态本身就是验收场景之一（R1.4）；比价表依赖商品 `selling` 金额 ÷ `point`，与用户积分多少无关。
- **记录真实过程**：本文件与 `EXPERIMENT_NOTES.md` 均按实际发生日期与工具调用撰写，未做美化。
