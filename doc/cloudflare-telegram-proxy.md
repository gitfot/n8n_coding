# Cloudflare Worker 反向代理 Telegram 操作指南

> 目的：在无法直接访问 `https://api.telegram.org` 的环境（例如 Hugging Face Space）中，通过 Cloudflare Worker 转发 Telegram Bot API 请求，避免 308 重定向或地区封禁导致的调用失败。

---

## 一、前提条件

- 已注册 Cloudflare 账户，并可使用 Workers（免费套餐即可）。
- 已创建 Telegram Bot，掌握 Bot Token。
- 了解 n8n/Tg 机器人调用流程，明确需要转发的请求类型（Webhook、sendMessage、getUpdates 等）。
- 本地或服务器具备 `curl` 等测试工具，方便验证 Worker 是否生效。

---

## 二、整体流程

1. 在 Cloudflare Dashboard 或 Wrangler CLI 中创建全新的 Worker。
2. 将 Worker 代码替换为反向代理脚本，确保转发请求到 `api.telegram.org`。
3. （可选）为 Worker 配置自定义子域，方便记忆与证书管理。
4. 部署并测试 Worker，确认能正常转发 Telegram API。
5. 在 n8n 或其他服务中，将 Telegram Base URL 改为 Worker 域名。

---

## 三、创建 Worker

### 方式 A：Dashboard

1. 登录 Cloudflare → 选择 **Workers & Pages** → 点击 **Create Worker**。
2. 输入名称（示例：`telegram-proxy`），创建后进入在线编辑器。
3. 删除模板代码，填入下方脚本并保存。

### 方式 B：Wrangler CLI

```bash
npm create cloudflare@latest telegram-proxy
cd telegram-proxy
npm install
```

随后将 `src/index.ts`（或 `index.js`）替换为下方脚本，再执行 `npm run deploy`。

---

## 四、Worker 脚本示例

```javascript
const TELEGRAM_HOST = 'api.telegram.org';

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const telegramUrl = buildTelegramUrl(request.url);
  const init = await buildRequestInit(request);

  const telegramResponse = await fetch(telegramUrl, init);
  return relayResponse(telegramResponse);
}

function buildTelegramUrl(originalUrl) {
  const url = new URL(originalUrl);
  url.protocol = 'https:';
  url.hostname = TELEGRAM_HOST;
  return url.toString();
}

async function buildRequestInit(request) {
  const init = {
    method: request.method,
    headers: new Headers(request.headers),
    redirect: 'follow',
  };

  // Cloudflare 会自动注入 Host，手动覆盖确保 Telegram 接受。
  init.headers.set('host', TELEGRAM_HOST);
  init.headers.delete('cf-connecting-ip');
  init.headers.delete('x-forwarded-for');

  if (!['GET', 'HEAD'].includes(request.method)) {
    init.body = await request.arrayBuffer();
  }

  return init;
}

function relayResponse(response) {
  const headers = new Headers(response.headers);
  headers.delete('content-security-policy');
  headers.delete('content-security-policy-report-only');

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers,
  });
}
```

> 提示  
> - Worker 默认会跟随 3xx 重定向，此脚本已经满足避免 308 卡死的需求。  
> - 如果需要记录请求，可在 `handleRequest` 中加入 `console.log`，通过 Cloudflare Logs 查看。

---

## 五、可选增强

| 场景 | 操作 |
| --- | --- |
| 限制来源 | 在 `handleRequest` 中检查 `request.headers.get('x-real-ip')` 或自定义鉴权头，仅允许白名单来源调用。 |
| 自定义上游 | 将 `TELEGRAM_HOST` 改写为 Secrets，通过 `wrangler secret put TELEGRAM_HOST` 注入。 |
| 路径映射 | 若需增加前缀，可在 `buildTelegramUrl` 中通过 `url.pathname = url.pathname.replace(/^\\/proxy/, '')` 等方式裁剪。 |

---

## 六、部署与绑定自定义域名（可选）

1. 在 Worker 控制台点击 **Deploy**（或执行 `npm run deploy`），记录生成的默认域名（例如：`https://telegram-proxy.your-subdomain.workers.dev`）。
2. 如需绑定自定义域：
   - 前往 **Triggers → Custom Domains → Add**。
   - 输入 `telegram.yourdomain.com`，按提示在 DNS 中添加 CNAME。
   - 待状态显示 Active 后即可使用自定义域。

---

## 七、功能验证

使用 `curl` 测试 Worker 是否能正确转发：

```bash
curl -v "https://telegram-proxy.your-subdomain.workers.dev/bot<YOUR_BOT_TOKEN>/getMe"
```

若返回包含 `"ok": true` 与 Bot 信息，则代理正常。

如调用 `sendMessage` 等 POST 接口：

```bash
curl -X POST \
  "https://telegram-proxy.your-subdomain.workers.dev/bot<YOUR_BOT_TOKEN>/sendMessage" \
  -H "content-type: application/json" \
  -d '{"chat_id": "<测试群或用户ID>", "text": "Cloudflare Worker 代理测试成功"}'
```

---

## 八、在 n8n 中更新 Telegram 凭据

1. 打开 n8n → Credentials → 找到 Telegram API 凭证（例如 `n8n_quant_agent`）。
2. 将 **Base URL** 修改为 Worker 域名（如 `https://telegram-proxy.your-subdomain.workers.dev`），留空 Token 字段以沿用原 Bot Token。
3. 保存后，在相关工作流中执行一次 `Webhook` 或 `sendMessage`，确认不再出现 `308` 之类的错误。

---

## 九、常见问题与排查

| 问题 | 排查要点 |
| --- | --- |
| 返回 5xx 错误 | 查看 Cloudflare Worker Logs，确认请求是否命中 Telegram、Worker 是否被速率限制。 |
| 仍然出现 3xx | 检查是否在 Worker 里再次跳转（例如自定义域额外重定向），确认 `TELEGRAM_HOST` 没写错。 |
| 消息未送达 | Telegram Bot 可能被封禁或 Token 错误，可通过直接访问 `api.telegram.org` 验证 Token 是否有效。 |
| 带文件上传失败 | Worker 默认使用 `arrayBuffer` 读取 body，适用于常见 JSON/表单。如果要转发大文件，建议使用 `request.body` + `fetch` 的 Streaming（需 Workers Paid 版）。 |

---

## 十、运维与安全建议

- **速率限制**：免费 Worker 日调用量有限，长时间高频调用请升级套餐或配合 Cloudflare Turnstile 限流。
- **日志监控**：定期在 Cloudflare 中查看 Request Logs 或接入 Workers Analytics，分析失败请求。
- **安全控制**：对 Worker 增加 IP 白名单、签名校验等逻辑，防止被外部滥用转发到 Telegram。
- **灾备方案**：保留直接访问 Telegram API 的方式（例如使用代理服务器），以便 Cloudflare Worker 异常时快速切换。

---

## 十一、下一步

1. 在测试环境先试跑，确认 Worker 正常转发后再切换生产工作流。
2. 若需在其他服务（如自建后端）复用此代理，只需更新调用域名即可。
3. 根据业务需求扩展 Worker，比如加入请求签名、指标埋点或缓存非敏感请求等。

> 至此，一个可靠的 Cloudflare Worker Telegram 反向代理就搭建完成，可有效规避地区限制与 308 重定向问题。
