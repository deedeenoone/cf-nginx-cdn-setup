# Cloudflare + Nginx HTTPS CDN 一键配置脚本

通过 Cloudflare API Token + Nginx 反向代理 + acme.sh 自动续期 SSL，将内部 HTTP 服务安全暴露为 HTTPS CDN 访问。

## 功能

- 自动申请 Let's Encrypt / ZeroSSL SSL 证书（自动续期）
- 通过 Cloudflare API 自动添加/续期 DNS 记录
- Nginx 反向代理 + 安全头配置
- 防火墙仅允许 Cloudflare IP 访问（防扫描）
- 阻止直接 IP 访问 443
- **证书自动续期**（acme.sh，每 60 天自动更新）

## 前置要求

- 有一个 Cloudflare 托管的域名
- Cloudflare API Token（Zone:DNS:Edit 权限）
- 服务器公网 IP 可访问
- Ubuntu/Debian 系统

## 一键安装（curl）

```bash
curl -fsSL https://raw.githubusercontent.com/deedeenoone/cf-nginx-cdn-setup/main/setup-cf-nginx-cdn.sh | sudo bash -s -- \
  --domain api.example.com \
  --port 13579 \
  --zone-id 你的ZoneID \
  --token 你的APIToken
```

**示例：**
```bash
curl -fsSL https://raw.githubusercontent.com/deedeenoone/cf-nginx-cdn-setup/main/setup-cf-nginx-cdn.sh | sudo bash -s -- \
  --domain api.mydomain.com \
  --port 13579 \
  --zone-id abc123def456 \
  --token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## 交互模式

如果不想传参，可以直接运行：

```bash
git clone https://github.com/deedeenoone/cf-nginx-cdn-setup.git
cd cf-nginx-cdn-setup
chmod +x setup-cf-nginx-cdn.sh
sudo ./setup-cf-nginx-cdn.sh
# 依次输入：域名、服务端口、Zone ID、API Token
```

## 架构

```
用户 → Cloudflare CDN (HTTPS) → Nginx (SSL Termination) → 内部服务:13579
                                        ↑
                                   acme.sh 自动续期
```

## SSL 证书自动续期

脚本使用 [acme.sh](https://github.com/acmesh-official/acme.sh) + Cloudflare DNS API 申请 Let's Encrypt 证书，并配置**自动续期**：

- 证书有效期 60 天，acme.sh 自动在到期前续期
- 续期成功后自动 reload Nginx，**零停机**
- 日志位置：`~/.acme.sh/${DOMAIN}/`

手动检查续期状态：
```bash
~/.acme.sh/acme.sh --list
~/.acme.sh/acme.sh --renew -d api.example.com --force
```

## 卸载

### 卸载单个域名配置

```bash
curl -fsSL https://raw.githubusercontent.com/deedeenoone/cf-nginx-cdn-setup/main/setup-cf-nginx-cdn.sh | sudo bash -s -- \
  --uninstall --domain api.example.com
```

或交互模式：
```bash
sudo ./setup-cf-nginx-cdn.sh --uninstall --domain api.example.com
```

这会：
1. 删除 Nginx 配置文件
2. 删除 SSL 证书和私钥
3. 移除 acme.sh 证书记录
4. 重载 Nginx

### 手动完全卸载（清理所有配置）

```bash
# 停止 Nginx
systemctl stop nginx

# 删除所有站点配置
rm -f /etc/nginx/sites-available/*
rm -f /etc/nginx/sites-enabled/*

# 删除所有证书
rm -f /etc/ssl/certs/*.pem
rm -f /etc/ssl/private/*.key

# 删除 acme.sh
rm -rf ~/.acme.sh

# 重载 Nginx
systemctl reload nginx
```

### 重置防火墙

```bash
# 关闭 ufw
ufw disable

# 或重置规则
ufw reset
ufw default deny incoming
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
```

### 清理 Cloudflare DNS 记录

手动登录 [dash.cloudflare.com](https://dash.cloudflare.com) → 你的域名 → DNS → 删除对应的 A 记录。

## 安全特性

- 全链路 HTTPS 加密
- 仅 Cloudflare IP 可访问源站
- 阻止直接 IP 访问
- 完整安全响应头（X-Frame-Options, X-Content-Type-Options 等）
- Cloudflare DDoS 防护隐藏真实 IP
- 证书自动续期，永不过期

## Cloudflare SSL/TLS 设置

在 Cloudflare Dashboard 中设置：

1. **SSL/TLS** → **Overview** → **Full (strict)**
2. **SSL/TLS** → **Edge Certificates** → 开启 **Always Use HTTPS**
3. **WAF** → 按需开启规则

## 手动管理多域名

查看已配置的域名：
```bash
ls -1 /etc/nginx/sites-enabled/
```

查看证书状态：
```bash
~/.acme.sh/acme.sh --list
```

强制续期证书：
```bash
~/.acme.sh/acme.sh --renew -d api.example.com --force
```

## License

MIT
