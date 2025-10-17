## 🚀 工作流程概述

**`day9_getting_updated_in_crypto`** 是一个基于 **n8n** 构建的自动化加密市场情报工作流，旨在为投资者或分析师提供简洁、可执行的资产洞察。工作流通过调用 CoinGecko 市场接口、执行多段规则计算并汇总结果到 webhook 响应，形成端到端的实时行情分析能力。

- **核心组件 1**: `Webhook` → `HTTP Request` 链路，接收外部调用后提取 CoinGecko Top 10 市场数据。
- **核心组件 2**: `Function` + 条件节点组合，计算波动率与市场评分并根据阈值设置交易信号与风险等级。
- **核心组件 3**: `Generate Portfolio Summary` + `Respond to Webhook`，汇总全量分析并生成 Markdown 报告返回调用方。

### 🎯 主要功能

- 获取最近的前十大加密货币行情（价格、成交量、涨跌幅）。
- 计算 24h/7d/30d 加权波动率、市场评分与市值成交量比。
- 基于价格趋势与波动率自动判定 BUY/SELL/HOLD/NEUTRAL 信号。
- 生成包含单币分析与组合概览的 Markdown 报告并通过 Webhook 返回。

---

### 🔌 所需凭证与工具

此工作流仅依赖公开的 CoinGecko API，无需配置额外凭证。若部署在受限网络环境，请确保运行节点允许访问 `api.coingecko.com`。

---

## 📌 如何使用

### 触发方式
- **触发方式**：`Webhook` 节点（路径 `https://<your-n8n-host>/webhook/31941fd9-9fd5-4fab-a1b9-9c347e26f5c9`），任何 POST 请求都会触发全量分析。
- **MockData**：工作流未内置样例数据，调试时可向 Webhook 发送如下 JSON：
  ```json
  {
    "event": "refresh",
    "note": "Trigger day9 market snapshot"
  }
  ```

### 💡 使用示例 “refresh”

以下示例帮助快速探索返回内容：

**1. 💬 常规市场概览**
  - `"GET https://<host>/webhook/31941fd9-9fd5-4fab-a1b9-9c347e26f5c9"`
  - `{"event":"refresh"}`

**2. 💬 组合风险审查**
  - `"{"event":"risk-check","portfolio":"growth"}"`
  - `"{"event":"refresh","focus":"volatility"}"`

**3. 💬 自动化报告调用**
  - `curl -X POST https://<host>/webhook/31941fd9-9fd5-4fab-a1b9-9c347e26f5c9 -H "Content-Type: application/json" -d "{\"event\":\"daily-digest\"}"`

---

## 🧠 节点结构摘要

- **核心逻辑/大脑**: `Generate Portfolio Summary` `Function` 节点负责聚合所有币种的分析结果并生成 Markdown 报告。
- **关键节点 1**: `Fetch Crypto Data` `HTTP Request` 直接请求 CoinGecko 市场接口。
- **关键节点 2**: `Calculate Market Metrics` `Function` 节点计算波动率、市场得分等核心指标。
- **工具/子流程调用**: 条件节点 (`IF` / `Switch`) 与 `Set` 节点组合设置交易信号、风险评级与建议，再通过 `Respond to Webhook` 返回调用方。

---

#### ⚠️ 注意事项

- 📎 **API 限流**：CoinGecko 对匿名请求有速率限制，建议在高频调用场景下添加节流或缓存。
- 📍 **参数一致性**：虽然当前逻辑未使用请求体字段，推荐在外部系统中统一传递 `event` 等标识，便于后续扩展筛选条件。
- 🧩 **模块独立性**：上游数据抓取、指标计算、信号打分和报告生成均解耦，修改某一阶段逻辑时注意保持输出字段兼容。
- 🚨 **错误处理**：建议在生产环境中为 `HTTP Request` 节点开启错误分支或重试策略，以应对第三方 API 波动。
