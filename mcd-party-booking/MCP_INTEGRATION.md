# 麦当劳 MCP 接入说明（MCP_INTEGRATION.md）

> 本文件说明 `mcd-party-booking` 如何接入并调用麦当劳官方 MCP（`https://mcp.mcd.cn`），是连接官方能力与 Skill 实现的桥梁文档。
> 所有结论均来自真实调用（见 `evidence/`），非文档转述。

---

## 一、连接方式

| 项 | 值 |
|---|---|
| 协议 | Streamable HTTP（`type: "streamable-http"`） |
| 端点 | `https://mcp.mcd.cn` |
| 鉴权 | `Authorization: Bearer <TOKEN>` |
| Token 申请 | 浏览器打开 `https://open.mcd.cn/mcp/login` → 手机号 + 短信验证码登录 → 【控制台】→【激活】→ 复制 Token |
| 限流 | 600 次/分钟/Token |
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
| `now-time-info` | 取服务器当前时间（时效基准） | 无 | JSON | 全能力 |
| `mall-points-products` | 取派对/读书会商品（含 `spuId` 与 `price`） | 无 | JSON | 能力 1（取 spuId）/ 2（价目） |
| `query-party-city` | 取支持派对的城市（code/经纬度） | `spuId` | JSON | 能力 1 |
| `query-party-store` | 取城市下派对门店（字段 `code`） | `code`+`latitude`+`longitude`+`spuId` | JSON | 能力 1 |
| `query-party-store-date` | 取门店可约日期（字段 `date`） | `storeCode`+`spuId` | JSON | 能力 1 |
| `query-party-store-session` | 取当天场次（时间/成团/余量/价） | `spuId`+`storeCode`+`dateStr` | JSON | 能力 1 / 2 |
| `campaign-calendar` | 当月活动日历 | 无 | Markdown | 能力 3 |

**写操作一律不调用**：`party-order-create`（派对订场）等。本 Skill 只做选址、选期、成本参谋，是否下单由用户自行决定。

---

## 三、调用流程（三条能力）

```
能力 1 派对场馆选址（四步链）：
  mall-points-products ──▶ query-party-city(spuId)
       ──▶ query-party-store(code,lat,lng,spuId)   # 门店编码在字段 code
       ──▶ query-party-store-date(storeCode,spuId) # 日期在字段 date
       ──▶ query-party-store-session(spuId,storeCode,dateStr)
  输出：城市→门店→日期→场次 逐级收敛

能力 2 派对成本核算：
  mall-points-products(price) + query-party-store-session(price,partyMin,partyMax)
  输出：商品价 + 场次价(分→元) + 成团人数 + 人均参考

能力 3 派对活动日历：
  campaign-calendar ──▶ 过滤派对/品鉴会/读书会关键词
```

**降级顺序**：城市无门店 → 引导换城市；日期无场次 → 引导其他日期；场次余量 0 → 提示已满。

---

## 四、业务价值（为什么需要只读封装）

麦当劳线下派对信息在官方 App 里**分散且不含"可约日期/场次/成团人数"的集中视图**。本 Skill 用 4 个派对工具一次拉全"城市→门店→日期→场次"，把原本要翻 App + 打电话才能拼出的决策信息，压缩成一次对话输出。

实测发现的关键事实（决定实现方式）：

1. `query-party-store` 的门店编码字段是 **`code`**（不是 `storeCode`）——文档示例易误导，实测才确认。
2. `query-party-store-date` 的日期字段是 **`date`**；`dateStr` 是下一步的**入参**而非返回字段。
3. 派对商品 `price` 是**真实金额**（与积分商城 `price` 恒为 `"0"` 完全不同），直接读取即可。
4. 场次 `price` 单位为**分**（如 `13800` = ¥138），展示须 ÷100。

---

## 五、字段陷阱速查（实现必读，详见 `references/party-tools.md`）

| # | 陷阱 | 正确处理 |
|---|---|---|
| 1 | 门店编码字段名 | `query-party-store` 返回 `code`，不是 `storeCode` |
| 2 | 日期字段名 | `query-party-store-date` 返回 `date`；`dateStr` 是传给 session 的参数 |
| 3 | 城市参数回传 | `query-party-city` 的 `code`/`latitude`/`longitude` 必须原样回传 `query-party-store` |
| 4 | 派对价目 | 取 `price`（真实金额），不要套积分商品的 `selling` 解析 |
| 5 | 场次价单位 | `query-party-store-session.price` 是分，展示 ÷100 |
| 6 | spuId 来源 | 须从 `mall-points-products`（派对/读书会/品鉴会类目）取，不硬编码 |

---

## 六、安全边界（用户可见）

- 全程只读；仅调用上面 7 个工具。
- 不代订场。`party-order-create` 等写操作一律不触发；是否下单由用户自行在官方渠道完成。
- 所有配置文件只使用 `${MCD_MCP_TOKEN}` 环境变量占位符，不含真实 Token / 手机号 / 账号凭证（见 `CONTEST_DECLARATION.md` 信息安全声明）。
- 真实调用证据存于 `evidence/`（含 `traceId`，未手改），供评审核对。
