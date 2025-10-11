# Crypto 工作流总览

本文汇总了个人项目 Crypto 相关的 n8n 工作流，包含清单、作用说明、端到端调用流程、可直接调用示例、触发方式与启用建议，并附“工具调用简报”。

## 工作流清单（Crypto 主题）
- Binance SM Indicators Webhook Tool（active）
- Binance SM 15min Indicators Tool（inactive）
- Binance SM 1hour Indicators Tool（inactive）
- Binance SM 4hour Indicators Tool（inactive）
- Binance SM 1day Indicators Tool（inactive）
- Binance SM Financial Analyst Tool（inactive）
- Binance Spot Market Quant AI Agent（inactive）
- Binance SM Price-24hrStats-OrderBook-Kline Tool（inactive）
- Binance SM News and Sentiment Analyst Webhook Tool（inactive）
- test webhook（inactive）

## 作用简介

### Binance SM Indicators Webhook Tool
- 核心指标计算后端，统一处理 15m/1h/4h/1d K 线并计算 RSI/MACD/BBANDS/SMA/EMA/ADX。
- 暴露多条 Webhook 路径，按时间粒度分流，返回清洗后的指标 JSON。
- 同时包含一个“15分钟指标智能体”子路，由其他工作流以 Execute Workflow 触发，并通过 HTTP 工具调用该后端。

### Binance SM 15min/1h/4h/1d Indicators Tool
- 各自为对应时间粒度的指标智能体，输入 `message(=symbol)`、`sessionId`。
- 通过 HTTP 工具 POST 到指标后端 Webhook 获取指标，组织为易读输出（适配 Telegram/上层代理）。

### Binance SM Financial Analyst Tool
- 分析编排层。解析用户意图与时间粒度，调用价格/结构工具与多个指标工具（15m/1h/4h/1d），合并结果为结构化市场快照。
- 输入同样是 `message` 与 `sessionId`，常由更高一层代理触发。

### Binance Spot Market Quant AI Agent
- 顶层量化报告代理。支持 Telegram 触发、用户鉴权、会话 ID 注入。
- 并行调用 Financial Analyst Tool 与 News/Sentiment 工具，合成交易报告（现货/杠杆建议、信号强度、新闻情绪）。

### Binance SM Price-24hrStats-OrderBook-Kline Tool
- 市场结构子代理：并行拉取 `getCurrentPrice`、`get24hrStats`、`getOrderBook`、`getKlines(15m/1h/4h/1d)`，返回统一快照，供上层分析工具使用。

### Binance SM News and Sentiment Analyst Webhook Tool
- 新闻情绪子代理：Webhook 收到 `{message:<symbol>}` 后，聚合多家 RSS，按关键字过滤，调用 LLM 总结情绪与头条，返回适配 Telegram 的文本。

## 调用流程（端到端）
1. Telegram → Binance Spot Market Quant AI Agent
   - 触发：`Telegram Trigger`，鉴权后设置 `sessionId`（来自 chat_id），抽取 `message`（消息文本）。
2. 代理并发两类能力：
   - 市场分析：`Binance SM Financial Analyst Tool`
     - 进一步并行：
       - 价格结构：`Binance SM Price-24hrStats-OrderBook-Kline Tool`
         - 内部并发调用 Binance 公共 API：
           - `GET /api/v3/ticker/price?symbol=...`
           - `GET /api/v3/ticker/24hr?symbol=...`
           - `GET /api/v3/depth?symbol=...&limit=100`
           - `GET /api/v3/klines?symbol=...&interval=15m|1h|4h|1d&limit=1`
       - 技术指标：15m/1h/4h/1d 指标工具
         - 各指标工具通过 HTTP 工具 POST 到“指标后端 Webhook”（即 Binance SM Indicators Webhook Tool 中的对应路径），提交 `{symbol}` 获取指标 JSON。
   - 新闻情绪：`Binance SM News and Sentiment Analyst Webhook Tool`
     - 向其 Webhook POST `{message:<symbol>}`；内部聚合 RSS，过滤关键字并由 LLM 总结情绪与头条。
3. 汇总：Quant AI Agent 合并“结构+指标+情绪”，格式化为 Telegram HTML，若超长自动切分并发送。

## 直接调用示例
> 将 `<你的 n8n 基础 URL>` 替换为你的实际 n8n 域名或实例地址。

### 指标后端（Binance SM Indicators Webhook Tool）
- 15 分钟
  - `POST <你的 n8n 基础 URL>/webhook/wan-15m-indicators`
  - Body: `{"symbol":"BTCUSDT"}`
- 1 小时
  - `POST <你的 n8n 基础 URL>/webhook/wan-1h-indicators`
  - Body: `{"symbol":"BTCUSDT"}`
- 4 小时
  - `POST <你的 n8n 基础 URL>/webhook/wan-4h-indicators`
  - Body: `{"symbol":"BTCUSDT"}`
- 1 天
  - `POST <你的 n8n 基础 URL>/webhook/wan-1d-indicators`
  - Body: `{"symbol":"BTCUSDT"}`

### 新闻情绪（Binance SM News and Sentiment Analyst Webhook Tool）
- `POST <你的 n8n 基础 URL>/webhook/a41af97d-6bd6-44d2-b79e-dca6df1e6e7e`
- Body: `{"message":"ETH"}`

## 触发方式对照
- `Binance Spot Market Quant AI Agent`：Telegram 触发 → 鉴权后自动编排全链路
- `Binance SM Financial Analyst Tool`：被上层代理以 Execute Workflow 触发
- `Binance SM Price-24hrStats-OrderBook-Kline Tool`：被 Analyst 工具以 Execute Workflow 触发
- `Binance SM Indicators Webhook Tool`：HTTP Webhook 触发（15m/1h/4h/1d）
- `Binance SM News and Sentiment Analyst Webhook Tool`：HTTP Webhook 触发（`{message:<symbol>}`）

## 建议的启用顺序
1. 激活 `Binance SM Indicators Webhook Tool`（确保 15m/1h/4h/1d 路径可达）
2. 如需 Telegram 端到端体验：配置 `Binance Spot Market Quant AI Agent` 的 Telegram 凭据和允许的用户 ID
3. 上层编排：启用 `Binance SM Financial Analyst Tool`、`Binance SM Price-24hrStats-OrderBook-Kline Tool`、`Binance SM News and Sentiment Analyst Webhook Tool`

---

## 工具调用简报
- 使用：n8n-mcp
- 调用：
  - `n8n_list_workflows(limit=100)` → 获取全部工作流列表
  - `n8n_get_workflow_details(UhRglwk6f7ME5fGR)` → Indicators Webhook Tool 详情（含 15m/1h/4h/1d Webhook）
  - `n8n_get_workflow_details(6vm2E7Wn7iArM4op)` → 15min Indicators Tool 详情
  - `n8n_get_workflow_details(0IWoziYDLkF6vBdM)` → Financial Analyst Tool 详情
  - `n8n_get_workflow_details(exS3Gm4iii4KPlmP)` → Quant AI Agent 详情（含 Telegram 触发/鉴权/分发）
  - `n8n_get_workflow_details(QnBjPQyd6bg3y6E3)` → Price/24hrStats/OrderBook/Kline 工具详情
  - `n8n_get_workflow_details(1w2jAPPpZoeoS6Zk)` → News & Sentiment Webhook 工具详情
- 结果：列出 Crypto 相关工作流，梳理作用与互相调用流程，给出可直接调用的 Webhook 路径与输入格式。

