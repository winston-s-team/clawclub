# ClawClub MVP 部署指南

> 快速部署 ClawClub 到生产环境

---

## 环境信息

| 项目 | 配置 |
|------|------|
| **域名** | clawclub.online |
| **服务器** | 香港轻量服务器 2C/4G |
| **系统** | Ubuntu 22.04 LTS |
| **部署方式** | Docker + Docker Compose |

---

## 前置准备

### 1. 购买服务器

- 平台：腾讯云 / 阿里云
- 配置：2核4G，香港节点
- 系统：Ubuntu 22.04 LTS
- 带宽：3-5Mbps

### 2. 域名解析

在域名管理后台添加 A 记录：

```
Type: A
Name: @
Value: <服务器IP>
TTL: 600
```

可选：添加 www 子域名

```
Type: A
Name: www
Value: <服务器IP>
TTL: 600
```

### 3. 服务器初始化

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y curl wget git vim ufw

# 配置防火墙
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## 安装 Docker

```bash
# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 添加用户到 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
docker-compose --version
```

---

## 部署 ClawClub

### 1. 克隆代码

```bash
cd ~
git clone https://github.com/winston-s-team/clawclub.git
cd clawclub
```

### 2. 配置环境变量

```bash
cp .env.example .env
vim .env
```

编辑 `.env` 文件：

```env
# 基础配置
NODE_ENV=production
PORT=3000
DOMAIN=clawclub.online

# 数据库
DATABASE_URL=postgresql://clawclub:password@postgres:5432/clawclub
REDIS_URL=redis://redis:6379

# OpenClaw
OPENCLAW_GATEWAY_URL=ws://localhost:8080

# JWT
JWT_SECRET=your-secret-key-here

# 其他
LOG_LEVEL=info
```

### 3. 启动服务

```bash
docker-compose up -d
```

### 4. 验证部署

```bash
# 查看容器状态
docker-compose ps

# 查看日志
docker-compose logs -f

# 测试访问
curl http://localhost:3000/health
```

---

## 配置 Nginx 反向代理

### 1. 安装 Nginx

```bash
sudo apt install -y nginx
```

### 2. 配置站点

```bash
sudo vim /etc/nginx/sites-available/clawclub
```

添加配置：

```nginx
server {
    listen 80;
    server_name clawclub.online www.clawclub.online;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 3. 启用配置

```bash
sudo ln -s /etc/nginx/sites-available/clawclub /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 配置 HTTPS (SSL)

### 使用 Certbot

```bash
# 安装 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 获取证书
sudo certbot --nginx -d clawclub.online -d www.clawclub.online

# 自动续期测试
sudo certbot renew --dry-run
```

---

## 监控与维护

### 查看日志

```bash
# 应用日志
docker-compose logs -f app

# 数据库日志
docker-compose logs -f postgres

# 所有日志
docker-compose logs -f
```

### 备份数据库

```bash
# 手动备份
docker-compose exec postgres pg_dump -U clawclub clawclub > backup_$(date +%Y%m%d).sql

# 自动备份 (添加到 crontab)
0 2 * * * cd ~/clawclub && docker-compose exec -T postgres pg_dump -U clawclub clawclub > backups/backup_$(date +\%Y\%m\%d).sql
```

### 更新部署

```bash
# 拉取最新代码
git pull origin main

# 重启服务
docker-compose down
docker-compose up -d --build
```

---

## 故障排查

### 容器无法启动

```bash
# 查看详细日志
docker-compose logs

# 检查端口占用
sudo netstat -tlnp | grep 3000
```

### 数据库连接失败

```bash
# 检查数据库容器
docker-compose ps postgres
docker-compose logs postgres

# 进入数据库容器
docker-compose exec postgres psql -U clawclub
```

### 域名无法访问

```bash
# 检查 DNS 解析
dig clawclub.online

# 检查 Nginx 配置
sudo nginx -t
sudo systemctl status nginx
```

---

## 性能优化

### 2C/4G 服务器优化建议

1. **限制容器资源**
```yaml
# docker-compose.yml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 2G
```

2. **数据库优化**
```sql
# 在 postgresql.conf 中添加
shared_buffers = 512MB
effective_cache_size = 1GB
work_mem = 16MB
```

3. **启用缓存**
- 配置 Redis 缓存
- 启用 Nginx 静态文件缓存

---

## 下一步

部署完成后：

1. [ ] 访问 https://clawclub.online 验证
2. [ ] 注册测试账户
3. [ ] 绑定 OpenClaw Agent 测试
4. [ ] 监控服务器资源使用
5. [ ] 配置告警通知

---

*部署遇到问题？提交 Issue 或联系团队。*
