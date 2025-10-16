## 🚀 工作流程概述

**`Trading Bot ICT 2025 Smart money consept`** 是一个基于 **n8n** 的 ICT（Inner Circle Trader）策略自动化工作流，面向专业交易团队和量化开发者。流程统一了 Telegram 信号采集、Kill Zone 会话校验、Coinbase Advanced Trading API 市场数据、AI 解读、交易执行、风控记录与多渠道通知，形成端到端的 Smart Money Concepts 决策链。

- **核心组件 1**: `ICT Telegram Signal Trigger` → `Extract ICT Signal Data`，从 Telegram 消息提取交易指令并补全 RSI/MACD/数量等字段。
- **核心组件 2**: `ICT Session Validator` → `Get Coinbase Market Data` → `ICT AI Analysis` → `Parse ICT AI Analysis`，对 Kill Zone、市场结构与 AI 结果进行强化分析。
- **核心组件 3**: `ICT Quality & Session Filter` 分支执行，满足高质量信号时进入 `Execute ICT Trade` → Notion 记录 → 通知与回调；未满足条件则记录拒绝原因并返回结构化回应。

### 🎯 主要功能

- 捕获 Telegram 手动信号并自动规范化为结构化字段。
- 基于 ICT Kill Zone/Session 逻辑验证当前市场时段可交易性。
- 拉取 Coinbase 市场数据并调用 OpenAI 生成智能 ICT 分析和风控建议。
- 自动执行市价订单（Coinbase Advanced Trading），同步写入 Notion 审计数据库。
- 生成 Markdown 风格的 Telegram 结果通知，并将完整分析推送到外部 Webhook。

---

### 🔌 所需凭证与工具

1. **`telegramApi`**：用于 `ICT Telegram Signal Trigger` 与 `Send ICT Telegram Alert` 节点，需具备接收/发送消息权限。
2. **`coinbaseAdvancedApi`**：供 `Get Coinbase Market Data` 与 `Execute ICT Trade` 访问 Coinbase Brokerage v3 接口。
3. **OpenAI API 凭证**：支持 `ICT AI Analysis` 与 `Generate ICT Notification` LangChain 节点。
4. **Notion 集成**：`Create ICT Trading Record` 与 `Log ICT Rejected Signal` 需要 `NOTION_TRADING_DB_ID`、`NOTION_REJECTED_DB_ID` 变量指向目标数据库。
5. **环境变量**：`TELEGRAM_CHAT_ID`、`WEBHOOK_URL` 等需在 n8n 变量或 Credentials 中配置。

---

## 📌 如何使用

### 触发方式
- **触发方式**：`ICT Telegram Signal Trigger` (`telegramTrigger`) 接收来自指定聊天的消息事件。
- **MockData**：未预置 Pin 数据。调试可在触发器中创建以下 JSON：
  ```json
  {
    "message": {
      "chat": {"id": 123456789},
      "text": "Buy BTC-USD at 68000, qty 5, session London"
    }
  }
  ```

### 💡 使用示例 “BTC-USD”

**1. 💬 实盘信号注入**  
  - 在 Telegram 发送：“Buy BTC-USD at 67500 qty 3 during London kill zone”

**2. 💬 校验拒绝情形**  
  - 在非 Kill Zone 时段发送：“Sell ETH-USD 2 lots”，检查 Notion 拒绝表记录。

**3. 💬 外部集成测试**  
  - 设置 `WEBHOOK_URL` 为自建服务，验证 `httpRequest` 节点推送的 `signal_data` 载荷。

---

## 🧠 节点结构摘要

- **核心逻辑/大脑**: `ICT AI Analysis` + `Parse ICT AI Analysis` 协同完成 AI 风控解读与 JSON 结构化。
- **关键节点 1**: `ICT Quality & Session Filter` 严格控制执行链，仅当信号质量、信心与会话一致时下单。
- **关键节点 2**: `Execute ICT Trade` 直接向 Coinbase Advanced Posting 市价订单，并触发 Notion/TG/HTTP 回传。
- **工具/子流程调用**: 当前流程未使用子工作流；所有操作在主流程中串行完成。

---

#### ⚠️ 注意事项

- 📎 **权限管理**：确保 Telegram Bot 在目标群组内拥有读写权限；Coinbase API Key 必须启用交易与市场数据访问。
- 📍 **时间同步**：会话校验以 UTC 时间判断 Kill Zone，n8n 运行环境需保持准确时钟。
- 🧩 **错误兜底**：`Parse ICT AI Analysis` 已提供 JSON 解析 fallback，但建议在生产中监控 `error` 字段并补充告警。
- 🚨 **交易风险**：市价 IOC 订单执行迅速，测试阶段可连接 Sandbox 或将 `Execute ICT Trade` 节点禁用仅做模拟。Notion 数据库属性需与节点配置一致，否则写入将失败。
