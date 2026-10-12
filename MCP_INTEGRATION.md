# 麦当劳 MCP 接入说明（MCP_INTEGRATION.md）

> 本文件说明 `mcd-points-guardian` 如何接入并调用麦当劳官方 MCP（`https://mcp.mcd.cn`），是连接官方能力与 Skill 实现的桥梁文档。
> 所有结论均来自真实调用（见 `evidence/`），非文档转述。

---

## 一、连接方式

| 项 | 值 |
|---|---|
| 协议 | Streamable HTTP（`type: "streamable-http"`） |
| 端点 | `https://mcp.mcd.cn` |
| 鉴权 | `Authorization: Bearer <TOKEN>` |
| Token 申请 | 浏览器打开 `https://open.mcd.cn/mcp/login` → 手机号 + 短信验证码登录 → 【控制台】→【激活】→ 复制 Token |
| 限流 | 600 次/分钟/Token；`draw-lottery` 单独 1 次/秒 |
| 状态 | 无状态（每次请求独立） |

**WorkBuddy 自定义连接器配置**（`config/mcp-config.example.json` 仅含占位符，无真实凭证）：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamable-http",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer ${MCD_MCP_TOKEN}",
        "Accept": "application/json, text/event-stream"
      }
    }
  }
}
```

> ⚠️ 配置后必须**完全退出 WorkBuddy 再重开**——自定义 MCP 在启动时加载，不支持热加载。重开后在【MCP 服务管理】里启用 `mcd-mcp`。

---

## 二、本 Skill 使用的工具（7 个，全部只读）

| 工具 | 用途 | 必填参数 | 返回格式 | 调用能力 |
|---|---|---|---|---|
| `now-time-info` | 取服务器当前时间（时效判定基准） | 无 | JSON | 能力 1 / 2 |
| `query-my-account` | 取积分账户（可用/累计/过期/冻结） | 无 | JSON | 能力 1 / 2 |
| `mall-points-products` | 取积分可兑商品券列表 | `catRuleIds`（可选） | JSON | 能力 1（交叉引用） / 2（主数据） |
| `mall-product-detail` | 取某 SPU 的 SKU 规格与兑换条件 | `spuId` | JSON | 能力 2（用户选定后） |
| `query-lottery-info` | 取抽奖活动规则与单次消耗（**不执行**） | 无 | JSON | 能力 3 |
| `query-my-prizes` | 取我的抽奖记录 | 无 | JSON | 能力 3（空态） |
| `query-my-coupons` | 取我的券（空态引导，返回 **Markdown**） | 无 | Markdown | 能力 1 / 空态 |

**核心 5 个**（`now-time-info` / `query-my-account` / `mall-points-products` / `mall-product-detail` / `query-lottery-info`）覆盖三条能力的判定与计算；`query-my-prizes` 与 `query-my-coupons` 用于空态引导与降级，不进入主计算链路。

**写操作一律不调用**：`mall-create-order`（兑换下单）、`draw-lottery`（抽奖）、`create-order`（点餐）、`party-order-create`（派对订场）等。涉及消耗积分的动作只做成本核算与建议，是否执行由用户自行决定。

---

## 三、调用流程（三条能力）

```
能力 1 到期积分体检：
  now-time-info ──▶ query-my-account ──▶ mall-points-products(取最低 point 档做交叉引用)
  输出：本月将清零量（前置）+ 下月预警 + 可兑换物

能力 2 积分兑换比价：
  now-time-info ──▶ mall-points-products ──▶ [用户选定 SPU] ──▶ mall-product-detail
  输出：折算率降序表 + 最优一档推荐

能力 3 抽奖成本核算：
  query-lottery-info（仅查询）
  输出：单次成本(24 分) + 资格(drawDecision) + 与直接兑换的对比建议
```

**降级顺序**：若 `mall-points-products` 无数据 → 能力 1 退化为纯数字提示，能力 2 进入空态，能力 3 不受影响（不依赖商城）。

---

## 四、业务价值（为什么需要只读封装）

麦当劳积分有两条商城页面不直说的规则，本 Skill 把它们算清楚：

1. **会静默过期**：`query-my-account` 的 `currentMouthExpirePoint`（本月将清零）字段存在但页面不提示。
2. **不同商品积分价值差 2～5 倍**：实测 50 分换 6.9 元派（0.138 元/分）比 200 分换 10.9 元小食（0.0545 元/分）划算 **2.5 倍**，商城不标注折算率。

此外，积分商城列表里混着大量 `status=2` 但 `downTime` 已过期的失效券（实测 24 条中 15 条已过期），用户肉眼无法分辨，本 Skill 用 `upTime <= now <= downTime` 过滤掉。

---

## 五、字段陷阱速查（实现必读，详见 `references/points-tools.md`）

| # | 陷阱 | 正确处理 |
|---|---|---|
| 1 | `price` 恒为 `"0"` | 折算率从 `selling` 文案解析 `(\d+(\.\d+)?)元`，不能读 `price` |
| 2 | `status=2` ≠ 可兑换 | 必须同时满足 `status==2` **且** `upTime <= now <= downTime` |
| 3 | 过期积分字段拼 `Mouth`（非 `Month`） | 照抄 `currentMouthExpirePoint` / `lastMouthExpirePoint` / `nextMouthExpirePoint` |
| 4 | 抽奖资格看 `drawDecision` | 只看 `drawDecision.resourceEligible` + `reason`，不自行推断 |
| 5 | 返回格式不统一 | `query-my-coupons` 是 Markdown、`list-nutrition-foods` 是 TOON，不能 `JSON.parse` |
| 6 | 积分类字段均为 string | 算术前先转数值 |

---

## 六、安全边界（用户可见）

- 全程只读；仅调用上面 7 个工具。
- 不代下单、不代抽奖。`draw-lottery` 消耗真实积分且有 1 次/秒限频，本 Skill 绝不自动触发。
- 所有配置文件只使用 `${MCD_MCP_TOKEN}` 环境变量占位符，不含真实 Token / 手机号 / 账号凭证（见 `CONTEST_DECLARATION.md` 信息安全声明）。
- 真实调用证据存于 `evidence/`（含 `traceId`，未手改），供评审核对。
