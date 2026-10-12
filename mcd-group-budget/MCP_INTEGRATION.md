# 麦当劳 MCP 接入说明（MCP_INTEGRATION.md）

> 本文件说明 `mcd-group-budget` 如何接入并调用麦当劳官方 MCP（`https://mcp.mcd.cn`），是连接官方能力与 Skill 实现的桥梁文档。
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

## 二、本 Skill 使用的工具（9 个，全部只读）

| 工具 | 用途 | 必填参数 | 返回格式 | 调用能力 |
|---|---|---|---|---|
| `now-time-info` | 时间基准 | 无 | JSON | 全能力 |
| `query-nearby-stores` | 选店（到店自取） | `searchType`+`city`+`keyword`+`beType` | JSON | 能力 1 |
| `query-meals` | 餐品列表（含 `code`/现价） | `storeCode`+`orderType`+**`beType`** | JSON | 能力 1 |
| `calculate-price` | 批量总价（分） | `storeCode`+`orderType`+`items`+`beType` | JSON | 能力 1 |
| `available-coupons` | 我的可领券 | 无 | JSON | 能力 2 |
| `query-store-coupons` | 门店可用券 | `storeCode`+`orderType`+`beType` | JSON | 能力 2 |
| `list-nutrition-foods` | 营养成分 | 无 | TOON/JSON | 能力 3 |
| `query-promotions` | 团餐满减（beType=6） | `storeCode`+`beCode`+`orderType`+`beType` | JSON | 能力 4 |
| `query-meal-assistance` | 团餐助餐（beType=6） | `storeCode`+`beCode`+`orderType`+`beType` | JSON | 能力 4 |

**写操作一律不调用**：`create-order`（点餐）、`mall-create-order`（兑换）、`party-order-create`（派对订场）等。本 Skill 只做预算参谋，是否下单由用户自行决定。

---

## 三、调用流程（四条能力）

```
能力 1 多人套餐篮与总价：
  query-nearby-stores(beType=1) ──▶ query-meals(storeCode,beType=1)  # 取 categories[].meals[].code
       ──▶ calculate-price(storeCode, items=[{productCode,quantity}], beType=1)  # price 单位分
  输出：套餐篮表 + 总价(分→元) + 人均

能力 2 优惠券核销：
  available-coupons ──▶ query-store-coupons(storeCode, orderType, beType)
  输出：可领券 + 门店可用券

能力 3 营养平衡：
  list-nutrition-foods ──▶ 容错解析(TOON/JSON) ──▶ 套餐篮营养提示

能力 4 企业团餐增强（说明型）：
  query-promotions(beType=6) / query-meal-assistance(beType=6)  # 需团餐 beCode，本账号无地址则不可达
```

**降级顺序**：`query-meals` 空 → 提示门店无餐品；`calculate-price` 失败 → 退化为单价×数量估算；`list-nutrition-foods` 解析异常 → 跳过营养项。

---

## 四、业务价值（为什么需要只读封装）

麦当劳 App 是"一人一单"视角，**没有"按人数凑套餐 + 算总价 + 比优惠"的聚合视图**。本 Skill 用 `query-meals` + `calculate-price`（带 `quantity`）一次算出多人总价（实测 24 份 ¥559.60），把团建/聚餐的预算从心算变成一次对话。

实测发现的关键事实（决定实现方式）：

1. `query-meals`/`query-meal-detail`/`calculate-price` **必须传 `beType`**，否则返回 `code:400`（实测第一次漏传即 400）
2. `calculate-price.price` 单位是**分**（如 `55960` = ¥559.60），展示须 ÷100
3. 餐品编码在多层嵌套 `data.categories[].meals[].code`，不是顶层字段
4. `query-promotions`/`query-meal-assistance` 仅 `beType=6` 企业团餐可用，且需 `delivery-query-stores` 的 `beCode`——本账号无团餐配送地址，该路径如实标注为"不可达"而非伪造

---

## 五、字段陷阱速查（实现必读，详见 `references/budget-tools.md`）

| # | 陷阱 | 正确处理 |
|---|---|---|
| 1 | `query-meals` 等必须 `beType` | 不传返回 `code:400`；到店 `1` / 团餐 `6` |
| 2 | `calculate-price.price` 单位 | 分，展示 ÷100 |
| 3 | 餐品编码位置 | `data.categories[].meals[].code`（嵌套） |
| 4 | 团餐专属工具 | `query-promotions`/`query-meal-assistance` 仅 `beType=6` 且需 `beCode` |
| 5 | 营养返回格式 | `list-nutrition-foods` 可能 TOON，不能整体 `JSON.parse` |
| 6 | 到店门店无 beCode | `query-nearby-stores` beType=1 返回门店无 `beCode` |

---

## 六、安全边界（用户可见）

- 全程只读；仅调用上面 9 个工具。
- 不代下单。`create-order` 等写操作一律不触发；是否下单由用户自行在官方渠道完成。
- 所有配置文件只使用 `${MCD_MCP_TOKEN}` 环境变量占位符，不含真实 Token / 手机号 / 账号凭证（见 `CONTEST_DECLARATION.md` 信息安全声明）。
- 真实调用证据存于 `evidence/`（含 `traceId`，未手改），供评审核对。
