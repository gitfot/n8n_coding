# Binance SM 15min Indicators Tool 工作流说明

## 基本信息
- 工作流 ID：`6vm2E7Wn7iArM4op`
- 当前状态：已归档（inactive），默认不主动运行
- 触发方式：`When Executed by Another Workflow` 节点，需由上游工作流通过 `Execute Workflow` 方式调用
- 典型上游：`Binance SM Financial Analyst Tool`、`Binance Spot Market Quant AI Agent`
- 核心目标：基于 15 分钟 K 线数据计算关键技术指标，并生成适配 Telegram/代理层的结构化行情快照

## 节点总览
| 节点名称 | 节点类型 | 核心配置 | 主要职责 |
| --- | --- | --- | --- |
| When Executed by Another Workflow | `n8n-nodes-base.executeWorkflowTrigger` | 输入参数 `message`（交易对）、`sessionId` | 接收上游工作流传入的交易对与会话上下文 |
| Binance SM 15min Indicators Agent | `@n8n/n8n-nodes-langchain.agent` | Prompt 详述 15m 指标用途与调用约束；引用记忆、模型、工具三类资源 | 解析输入、维护会话上下文、决定是否调用指标工具、整合最终回复 |
| Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | 缓存 `sessionId`、最近查询的交易对 | 支撑多轮会话与跨工具一致性 |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | 模型：`gpt-5`；凭证：`new api` | 将原始指标数值转写为可读分析与信号标签 |
| HTTP Request 15m Indicators Tool | `@n8n/n8n-nodes-langchain.toolHttpRequest` | `POST https://wanwanlu-n8n.hf.space/webhook/wan-15m-indicators`；Body：`{ "symbol": "<交易对>" }` | 调用 Treasurium 指标后端，返回 RSI/MACD/BBANDS/SMA/EMA/ADX 等 15 分钟指标 |
| Sticky Note 系列 | `n8n-nodes-base.stickyNote` | 仅用于设计时文档，无运行时影响 | 记录工作流说明、节点职责、示例输出等 |

## 执行流程
1. **上游触发**  
   上游工作流使用 `Execute Workflow` 调用此工具，传入 `message`（如 `BTCUSDT`）与 `sessionId`（常为 Telegram `chat_id`）。触发节点将负载传给主代理。

2. **代理接管**  
   `Binance SM 15min Indicators Agent` 根据系统提示词确认任务为“15 分钟技术指标分析”，并在需要时调用记忆、模型和外部工具。

3. **外部指标查询**  
   代理调用 `HTTP Request 15m Indicators Tool`，向 `https://wanwanlu-n8n.hf.space/webhook/wan-15m-indicators` 发送 JSON：  
   ```json
   { "symbol": "BTCUSDT" }
   ```  
   后端会抓取最近 100 根 15m K 线，计算 RSI(14)、MACD(12/26/9)、布林带(20/2)、SMA/EMA(20)、ADX(14) 等指标，并返回干净的 JSON 结果。

4. **信号解读与回复**  
   代理将指标结果交给 `OpenAI Chat Model (gpt-5)`，生成结构化文本（常见为 Telegram Markdown/HTML），解释超买/超卖、趋势强度、金叉/死叉等信号，并结合记忆中的上下文给出最终回复。

5. **响应返回**  
   整理后的文本直接返回给上游调用者，由上游决定推送目标（如 Telegram 用户）。

## 输入与输出
- **输入要求**  
  ```json
  {
    "message": "BTCUSDT",
    "sessionId": "123456789"
  }
  ```  
  - `message`：必填，需为有效 Binance 现货交易对（大写）。  
  - `sessionId`：必填，用于维护多轮会话或跨工具一致性。

- **输出形态**  
  - 文本化的 15 分钟技术分析摘要，示例：  
    ```
    📉 BTCUSDT 15 分钟技术概览
    • RSI: 72 → 超买
    • MACD: 金叉向上（看涨）
    • BB: 价格贴近上轨，波动扩张
    • EMA20 > SMA20 → 动能偏强
    • ADX: 34 → 趋势强度中高
    ```
  - 上游工作流可直接将结果推送至 Telegram 或作为综合分析的一部分继续处理。

## 对外依赖
- **指标服务 Webhook**：`https://wanwanlu-n8n.hf.space/webhook/wan-15m-indicators`  
  - 需保持可访问。若部署迁移，请同步更新节点配置。  
  - 服务返回的 JSON 约定含有各指标当前值、方向标签（如 `overbought`、`bullish_cross`）。
- **OpenAI API**：使用凭证 `new api`，模型配置为 `gpt-5`。若需降本，可改用其他模型，但需验证输出质量。

## 与其他工作流的关系
- **Binance SM Financial Analyst Tool**：调用此 15m 工具获取短周期动量，叠加 1h/4h/1d 指标与价格结构做多尺度分析。
- **Binance Spot Market Quant AI Agent**：在生成综合行情快照时并行调用价格结构、新闻情绪与 15m 技术指标模块。
- **指标后端（Indicators Webhook Tool）**：15m 工具依赖该后端的路由 `/wan-15m-indicators`，后端也服务于 1h/4h/1d 等工具。

## 使用场景与最佳实践
- **日内波动监控**：识别短周期的超买/超卖与趋势强弱，用于判断是否存在快速回调或突破机会。
- **多周期确认**：与 1 小时、4 小时、1 天指标交叉验证，辅助做出方向性决策。
- **自动化告警**：与 Telegram 机器人配合，在指标达到阈值时推送即时提醒。
- **多轮会话分析**：记忆节点可保持同一 `sessionId` 的上下文，支持追加问答或比较不同交易对。

## 运维与扩展建议
1. **恢复上线**：如需重新激活此工作流，请确认：
   - 指标 Webhook 可用；
   - OpenAI 凭证仍有效；
   - 上游调用者（Financial Analyst/Quant Agent）已更新为新的工作流 ID（若有变更）。
2. **监控执行质量**：历史统计显示共执行 10 次，其中成功 7 次、失败 3 次。若重新启用，建议在上游添加错误捕获节点，记录失败原因（常见为外部 Webhook 超时）。
3. **模型替换**：如需切换至 `gpt-4.1-mini` 等模型，请同步更新代理系统提示，确保输出格式保持一致。
4. **指标扩展**：若要增加自定义指标（例如 Ichimoku、VWAP），可在后端 Webhook 增强计算逻辑，并在代理提示中增加解释规则。

## 验证清单
- [ ] 指标 Webhook 返回含 RSI/MACD/BBANDS/SMA/EMA/ADX 的 JSON  
- [ ] `message` 参数为合法交易对且大写  
- [ ] OpenAI API 响应成功并生成结构化文本  
- [ ] 上游调用链正确处理返回文本（如 Telegram 模式需启用 HTML/Markdown）  
- [ ] 错误分支记录在上游日志中，便于溯源

---
若需扩展更多时间粒度，可复制此工作流并替换 Webhook 目标及系统提示内容，以复用整体编排架构。
