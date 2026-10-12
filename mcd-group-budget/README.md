# 麦当劳团餐预算官 · mcd-group-budget

> 把"一群人吃麦当劳"的**预算**算清楚：套餐篮 + 总价 + 优惠 + 营养，一次完成。纯只读——不代下单。

一个 WorkBuddy Skill，接麦当劳官方 MCP（`https://mcp.mcd.cn`），帮团建 / 聚餐 / 班级活动快速算出"点哪些、点多少、花多少、能用什么券"。

## 它解决什么问题

官方 App 是"一人一单"视角，**没有"按人数凑套餐 + 算总价 + 比优惠"的聚合视图**；营养信息散在单品页；企业团餐的满减满折藏在专属入口。部门团建、朋友聚餐时，预算只能靠人肉心算。

实测：上海南京东路店，3 款套餐各 10/8/6 份（共 24 份）`calculate-price` 返回总价 **¥559.60**——这类"多人总价"App 不会直接给。

## 目标用户

- 负责部门团建 / 班级活动预算的行政、班委
- 想给多人聚餐做"人均控制"的普通用户
- 关注团餐营养均衡（别全油炸）的健康意识用户

## 四条能力

| 能力 | 用户问什么 | 核心输出 |
|---|---|---|
| 多人套餐篮与总价 | "10 个人吃多少" | 套餐篮表 + 批量总价（分→元） |
| 优惠券核销建议 | "有什么券能用" | 可领券 + 门店可用券 |
| 营养平衡建议 | "这顿健康吗" | 总热量 + 搭配提示 |
| 企业团餐增强路径 | "团餐满减多少" | beType=6 专属优惠说明（需团餐地址） |

## 使用示例（真实输出样本，来自 `evidence/` 实测）

### 能力 1 · 多人套餐篮与总价（实测：上海南京东路店，storeCode=1450104）

| 餐品 | 单价 | 数量 | 小计 |
|---|---|---|---|
| "我就喜欢"鸡薯双全盒 | ¥15.90 | 10 | ¥159.00 |
| 龙焰芝士棒鸡腿堡三件套 | ¥29.90 | 8 | ¥239.20 |
| 龙焰鸡腿堡三件套 | ¥26.90 | 6 | ¥161.40 |
| **合计** | | **24 份** | **¥559.60** |

> 数据来自 `calculate-price`：`productPrice=55960`（分）→ ¥559.60。人均约 ¥23.3（按 24 人）。

### 能力 2 · 优惠券核销（实测）

```
🎟️ 优惠券建议
• 我的可领券（available-coupons）：7 张（麦麦省系列）
• 该门店可用券（query-store-coupons）：2 张
建议：下单前先领可领券，再于该门店叠加适用券，顺序以结算页为准。
```

### 能力 3 · 营养平衡（实测）

```
🥗 营养提示（list-nutrition-foods）
套餐篮含 3 款炸鸡/薯条类，蛋白充足但蔬菜偏少。
建议：加 1 份玉米杯 / 苹果片平衡膳食纤维与钠摄入。
```

### 能力 4 · 企业团餐增强（如实说明）

```
🏢 企业团餐专属（beType=6）
query-promotions / query-meal-assistance 仅企业团餐可用，且需先在麦当劳 App 配置团餐配送地址。
本账号未配置团餐地址，该路径暂不可达——请于 App 内开通后重试。
```

> 注：上述为真实运行样本；如需界面截图，在 WorkBuddy 启用 `mcd-mcp` 后新建对话触发即可。

## 安装

1. 申请麦当劳 MCP Token：浏览器打开 `https://open.mcd.cn/mcp/login` → 手机号 + 短信验证码登录 → 右上角【控制台】→【激活】→ 复制 Token
   （注意：`open.mcd.cn` 首页的【登录】与导航【商户平台】是**商户入口**，个人开发者进不去）
2. 在 WorkBuddy 的【连接器】→【自定义连接器】按 `config/mcp-config.example.json` 填入配置，把 `${MCD_MCP_TOKEN}` 换成你的 Token
3. **完全退出 WorkBuddy 再重开**（自定义 MCP 在启动时加载），重开后在【MCP 服务管理】里启用 `mcd-mcp`
4. 复制本 Skill 到技能目录，新建对话即可触发

## 安全边界

本 Skill **全程只读**，仅调用：`now-time-info`、`query-nearby-stores`、`query-meals`、`calculate-price`、`available-coupons`、`query-store-coupons`、`list-nutrition-foods`、`query-promotions`、`query-meal-assistance`。

**不调用** `create-order`、`mall-create-order`、`party-order-create`。本 Skill 只做预算参谋；是否下单由用户自行在官方渠道完成。

## 差异化说明

已主动避让同赛道已有作品：

| 已有作品 | 方向 | 本 Skill 的差异 |
|---|---|---|
| `junxchan54-maker/mcd-cn-assistant` | 活动推送 + 比价 | 本 Skill 聚焦**多人预算聚合**（套餐篮+总价+人均） |
| 「热麦卡路里」 | 单品热量估算 | 本 Skill 做**套餐篮级营养平衡**，非单品 |
| 外送链路编排类 | 配送流程自动化 | 本 Skill **不涉及下单**，纯只读预算 |

## 目录结构

```
mcd-group-budget/
├── SKILL.md                      # Skill 定义（四能力 + 空态 + 安全边界）
├── README.md                     # 本文件（介绍/安装/使用示例/目标用户）
├── MCP_INTEGRATION.md            # 麦当劳 MCP 接入说明（Server/Tool/调用流/业务价值）
├── workbuddy.md                  # WorkBuddy 专项奖励：真实开发记录
├── CONTEST_DECLARATION.md        # 参赛声明（官方原文件，不可改动）
├── references/budget-tools.md    # 工具字段实测与陷阱（实现依据）
├── config/mcp-config.example.json# 配置示例（仅占位符，无真实凭证）
└── evidence/                     # 真实调用原始返回（未手改，含 traceId）
```
