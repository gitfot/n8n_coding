## 🚀 工作流程概述

**`trader-binance-spot-cloud-no-credential`** 是一个基于 **n8n** 构建的 Binance 现货交易自动化骨架，面向希望快速验证下单链路的高级用户。工作流通过组合内置的手动触发器、参数装配节点、HMAC 签名计算与 HTTP 请求，覆盖账户查询、限价/市价下单与订单管理全流程。

- **核心组件 1**: `Manual Trigger` → `Set Credentials`，在执行时注入 API Key/Secret（示例节点提醒需在 n8n Credential 中配置）。
- **核心组件 2**: 多组 `Set` + `Code` + `Crypto` 节点，用于生成请求参数、拼装 QueryString 并计算 Binance 所需的 HMAC-SHA256 签名。
- **核心组件 3**: `HTTP Request` 节点集合，分别对 Binance Account、Order、OpenOrders、CancelAll 等端点进行调用。

### 🎯 主要功能

- 模拟账户信息查询：`/api/v3/account`
- 提交限价 BUY/SELL 与市价 BUY/SELL 订单
- 查询指定交易对的未成交订单
- 批量取消某交易对全部挂单

---

### 🔌 所需凭证与工具

1. **`Binance API` 凭证**：需在 n8n 中创建，包含 `api_key` 与 `api_secret`，供所有签名节点与请求节点引用。
2. **时间同步**：工作流依赖 `timestamp` 字段生成，确保服务器时间与 Binance 要求误差 < 1s，必要时可添加 `/api/v3/time` 校准步骤。

---

## 📌 如何使用

### 触发方式
- **触发方式**：`Manual Trigger` 节点；在编辑器中点击 “Execute Workflow” 手动执行。
- **MockData**：默认无固定输入。调试可在 `Set Credentials` 节点中临时写入测试 key/secret，或通过 `Execute Node` 指定 JSON，如：
  ```json
  {
    "api_key": "demo-key",
    "api_secret": "demo-secret"
  }
  ```

### 💡 使用示例 “BTCUSDT”

**1. 💬 账户快照**
  - 保留默认流程，执行后查看 `Get Account Info` 输出。

**2. 💬 限价下单测试**
  - 修改 `LimitBuy Parmeter` 或 `LimitSale Parameter` 中的 `price`、`quantity`，执行工作流检查 `Execute Limit BUY/SELL` 结果。

**3. 💬 市价交易与订单清理**
  - 调整 `MarketBuy Parameter` 或 `MarketSell Parameter`，执行后再查看 `Cancell All Order` 输出确认批量取消效果。

---

## 🧠 节点结构摘要

- **核心逻辑/大脑**: 各类 `Code` 节点负责将参数对象转换为 Binance 要求的 query string，配合 `Crypto` 节点生成 HMAC。
- **关键节点 1**: `Set Credentials` 为所有后续节点提供 API Key/Secret 输入。
- **关键节点 2**: `Execute Limit BUY/SELL`、`Execute MarketBuy/MarketSell` 直接向 `/api/v3/order` 发起 POST 请求。
- **工具/子流程调用**: 当前无子流程；所有操作均在主流程中串行完成。

---

#### ⚠️ 注意事项

- 📎 **凭证安全**：切勿在 `Set Credentials` 节点内硬编码真实密钥，务必使用 n8n Credential 存储。
- 📍 **参数一致性**：`symbol`、`timeInForce`、`quantity` 等字段应根据实际需求调整；所有请求依赖 `timestamp`，若出现 `-1021` 错误请校准服务器时间或增加 `recvWindow`。
- 🧩 **模块扩展性**：骨架采用 DRY 结构，同一段 `Code`/`Crypto` 逻辑复用于多种订单类型，新增操作时可复制现有模式保持一致。
- 🚨 **实盘风险**：真实环境执行前请确认 API Key 权限（只读/交易）并在 Sandbox 或 Testnet 验证；如需部署生产，请添加失败重试与错误分支处理。
