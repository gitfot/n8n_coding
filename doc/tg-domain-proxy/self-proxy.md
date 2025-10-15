## 方案二：自建服务器 Nginx 反代 Telegram

若拥有额外的云服务器/VPS（建议位于可访问 Telegram 的地区），可通过 Nginx 自建反向代理来完全掌控访问链路：

### 1. 环境准备

- 操作系统：常见 Linux 发行版（Ubuntu/Debian/CentOS 等）
- 拥有 sudo 或 root 权限
- 服务器需开放 80/443 端口，并能顺利访问 `https://api.telegram.org`

### 2. 安装 Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

CentOS/RHEL 可使用 `sudo yum install nginx`。

### 3. 申请 HTTPS 证书（推荐）

使用 Certbot 自动申请：

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d telegram.yourdomain.com
```

> 暂不使用 HTTPS 时，可先保留 HTTP，待测试完成后再切换到 TLS。

### 4. 配置反向代理

创建 `/etc/nginx/sites-available/telegram-proxy`（CentOS 使用 `/etc/nginx/conf.d/telegram-proxy.conf`）：

```nginx
server {
    listen 80;
    server_name telegram.yourdomain.com;

    location / {
        proxy_pass https://api.telegram.org$request_uri;
        proxy_set_header Host api.telegram.org;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 90;
        proxy_read_timeout 90;
        proxy_redirect off;
    }
}
```

若已启用 HTTPS，将 `listen 80;` 改为 `listen 443 ssl http2;` 并添加：

```nginx
    ssl_certificate /etc/letsencrypt/live/telegram.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/telegram.yourdomain.com/privkey.pem;
```

启用并重载：

```bash
sudo ln -s /etc/nginx/sites-available/telegram-proxy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 5. 功能测试

```bash
curl -v "https://telegram.yourdomain.com/bot<YOUR_BOT_TOKEN>/getMe"
```

若返回 `{"ok":true,...}` 说明代理正常。也可测试发送消息：

```bash
curl -X POST \
  "https://telegram.yourdomain.com/bot<YOUR_BOT_TOKEN>/sendMessage" \
  -H "content-type: application/json" \
  -d '{"chat_id":"<CHAT_ID>","text":"自建反代测试"}'
```

### 6. 在 n8n 中使用

1. 打开 Telegram Credentials → Base URL 改为 `https://telegram.yourdomain.com`
2. 保存后执行 `sendMessage` 或触发工作流，确认不再出现 308/403 等错误

> 可在 Nginx 中继续加入 IP 白名单、Basic Auth 或 Fail2Ban 等安全措施，防止代理被滥用。

---