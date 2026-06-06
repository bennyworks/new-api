# New API 运维手册

> 项目路径: `/root/opt/token-api-platform`
> 最后更新: 2026-06-06

---

## 一、运行环境概览

### 1.1 容器列表

| 容器 | 镜像 | 用途 | 端口 |
|------|------|------|------|
| `nginx-proxy` | `nginxproxy/nginx-proxy:latest` | 反向代理，处理 HTTP/HTTPS | 80, 443 (宿主机) |
| `acme-companion` | `nginxproxy/acme-companion:latest` | Let's Encrypt SSL 自动签发/续期 | — |
| `new-api` | `bennyworks/new-api:latest`（自定义构建） | AI API 网关主程序（含定制前端） | 3000 (内部，随机映射宿主机) |
| `postgres` | `postgres:15` | 主数据库 | 5432 (内部) |
| `redis` | `redis:latest` | 缓存 | 6379 (内部) |

### 1.2 数据卷 (Docker Volumes)

| 卷名 | 挂载路径 | 用途 |
|------|----------|------|
| `token-api-platform_pg_data` | `/var/lib/postgresql/data` | PostgreSQL 数据 |
| `token-api-platform_nginx_certs` | `/etc/nginx/certs` | SSL 证书 |
| `token-api-platform_nginx_vhost` | `/etc/nginx/vhost.d` | Nginx 虚拟主机配置 |
| `token-api-platform_nginx_html` | `/usr/share/nginx/html` | ACME 验证文件 |

### 1.3 本地挂载

| 宿主机路径 | 容器路径 | 用途 |
|-----------|---------|------|
| `./data` | `/data` | 应用数据 |
| `./logs` | `/app/logs` | 应用日志 |

### 1.4 网络

所有容器共享 `token-api-platform_new-api-network` (bridge) 网络。

### 1.5 镜像版本与大小

| 镜像 | 大小 |
|------|------|
| `bennyworks/new-api:latest` | ~184 MB |
| `postgres:15` | 445 MB |
| `redis:latest` | 143 MB |
| `nginxproxy/nginx-proxy:latest` | 181 MB |
| `nginxproxy/acme-companion:latest` | 49 MB |

---

## 二、日常操作

### 2.1 启动/停止

```bash
cd /root/opt/token-api-platform

# 启动全部服务
docker-compose up -d

# 停止全部服务
docker-compose down

# 重启全部服务
docker-compose down && docker-compose up -d

# 重启单个服务
docker-compose restart new-api

# 仅重启 new-api (不影响数据库和缓存)
docker-compose up -d --no-deps new-api
```

### 2.2 查看状态

```bash
# 查看所有容器运行状态
docker-compose ps

# 查看 new-api 日志 (实时)
docker-compose logs -f new-api

# 查看最近 50 行日志
docker-compose logs --tail=50 new-api

# 查看所有服务日志
docker-compose logs --tail=20

# 健康检查
curl -s http://localhost/api/status | python3 -m json.tool

# 查看资源占用
docker stats --no-stream
```

### 2.3 进入容器

```bash
# 进入 new-api 容器
docker exec -it new-api sh

# 进入 PostgreSQL
docker exec -it postgres psql -U root -d new-api
```

---

## 三、配置文件

### 3.1 核心配置文件

- `docker-compose.yml` — 服务编排、环境变量、端口、密码

### 3.2 关键环境变量 (new-api 服务)

| 变量 | 当前值 | 说明 |
|------|--------|------|
| `SQL_DSN` | `postgresql://root:123456@postgres:5432/new-api` | 数据库连接 |
| `REDIS_CONN_STRING` | `redis://:123456@redis:6379` | Redis 连接 |
| `TZ` | `Asia/Shanghai` | 时区 |
| `ERROR_LOG_ENABLED` | `true` | 错误日志 |
| `BATCH_UPDATE_ENABLED` | `true` | 批量更新 |
| `NODE_NAME` | `new-api-node-1` | 节点名称 |
| `VIRTUAL_HOST` | `your-domain.com` | **待配置** — 绑定域名 |
| `LETSENCRYPT_HOST` | `your-domain.com` | **待配置** — SSL 域名 |
| `FRONTEND_BASE_URL` | `https://your-domain.com` | **待配置** — 前端地址 |

---

## 四、域名与 SSL 配置

### 4.1 前提条件

- DNS 已将域名解析到服务器 IP
- 防火墙已开放 **80** 和 **443** 端口

### 4.2 配置步骤

编辑 `docker-compose.yml`，找到 new-api 服务的环境变量，将 `your-domain.com` 改为实际域名：

```yaml
environment:
  - VIRTUAL_HOST=api.example.com        # 改为你的域名
  - LETSENCRYPT_HOST=api.example.com    # 改为你的域名
  - FRONTEND_BASE_URL=https://api.example.com  # 改为你的域名
```

同时将 acme-companion 中的邮箱改为你的邮箱：

```yaml
acme-companion:
  environment:
    - DEFAULT_EMAIL=admin@example.com   # 改为你的邮箱
```

重启生效：

```bash
docker-compose down && docker-compose up -d
```

SSL 证书将通过 Let's Encrypt 自动申请，60 天后自动续期。

### 4.3 验证 SSL

```bash
# HTTP 跳转 HTTPS
curl -I http://api.example.com

# HTTPS 健康检查
curl https://api.example.com/api/status
```

---

## 五、数据库管理

### 5.1 直连数据库

```bash
docker exec -it postgres psql -U root -d new-api
```

### 5.2 备份数据库

```bash
# 导出 SQL 文件
docker exec postgres pg_dump -U root new-api > backup_$(date +%Y%m%d).sql

# 压缩备份
docker exec postgres pg_dump -U root new-api | gzip > backup_$(date +%Y%m%d).sql.gz
```

### 5.3 恢复数据库

```bash
# 从 SQL 文件恢复
cat backup_20260530.sql | docker exec -i postgres psql -U root -d new-api

# 从压缩备份恢复
gunzip < backup_20260530.sql.gz | docker exec -i postgres psql -U root -d new-api
```

### 5.4 定时备份 (crontab)

```bash
# 编辑 crontab
crontab -e

# 添加每日凌晨 3 点备份，保留 7 天
0 3 * * * docker exec postgres pg_dump -U root new-api | gzip > /root/opt/token-api-platform/backups/backup_$(date +\%Y\%m\%d).sql.gz && find /root/opt/token-api-platform/backups/ -name '*.sql.gz' -mtime +7 -delete
```

别忘了创建备份目录：
```bash
mkdir -p /root/opt/token-api-platform/backups
```

---
## 六、构建自定义镜像

本项目对前端首页有定制化修改。官方 `calciumion/new-api` 镜像不包含这些修改，因此需自行构建并推送镜像。

### 6.1 版本管理策略

```
upstream/main ──→ origin/main（纯上游镜像）──→ dev（自定义提交）
                                                   │
                                                   └── 前端首页定制
```

- `main`：紧跟 `upstream/main`，不做任何自定义修改
- `dev`：工作分支，承载所有自定义提交（前端改动 + 运维配置）

日常同步上游：

```bash
git checkout main && git pull upstream main && git push origin main
git checkout dev && git merge main
```

### 6.2 构建镜像

项目根目录 `Dockerfile` 会构建前端（`web/default/`）并嵌入 Go 二进制，产出包含自定义前端的完整镜像。

```bash
cd /Users/chenjianbin/Documents/git_workspace/new-api

# 构建镜像
docker build -t bennyworks/new-api:latest .

# 带版本号构建
docker build -t bennyworks/new-api:latest -t bennyworks/new-api:v$(cat VERSION) .
```

### 6.3 推送镜像

```bash
# 登录 Docker Hub
docker login

# 推送
docker push bennyworks/new-api:latest
```

### 6.4 生产环境拉取

在生产服务器上，`docker-compose.yml` 已配置为使用 `bennyworks/new-api:latest`：

```bash
cd /root/opt/token-api-platform
docker-compose pull new-api
docker-compose up -d --no-deps new-api
```

---

## 七、升级 new-api

### 7.1 升级步骤

```bash
cd /root/opt/token-api-platform

# 拉取最新自定义镜像
docker-compose pull new-api

# 重启服务 (数据库和缓存不受影响)
docker-compose up -d --no-deps new-api

# 查看日志确认启动正常
docker-compose logs -f new-api
```

### 7.2 回滚

如需回滚到指定版本，编辑 `docker-compose.yml` 中 new-api 的 `image` 字段：

```yaml
image: bennyworks/new-api:v1.0.0-rc.9
```

然后重启：
```bash
docker-compose up -d --no-deps new-api
```

---

## 八、安全加固

### 8.1 必须修改的默认密码

`docker-compose.yml` 中以下密码应尽快修改：

| 位置 | 默认值 | 行号 |
|------|--------|------|
| `SQL_DSN` 密码 | `123456` | 第 29 行 |
| `REDIS_CONN_STRING` 密码 | `123456` | 第 31 行 |
| `POSTGRES_PASSWORD` | `123456` | 第 72 行 |
| Redis `requirepass` | `123456` | 第 63 行 |

> ⚠️ 修改密码后需执行 `docker-compose down && docker-compose up -d` 完全重启。

### 8.2 生产环境建议

1. 取消注释 `SESSION_SECRET` 并设置为随机字符串：
   ```yaml
   - SESSION_SECRET=<随机生成的64位字符串>
   ```
2. 如有多节点部署，每个节点设置不同的 `NODE_NAME`
3. 考虑设置 `STREAMING_TIMEOUT` 防止长连接耗尽

---

## 九、日志查看

### 9.1 应用日志

```bash
# 实时日志
docker-compose logs -f new-api

# 按时间过滤
docker-compose logs --since="2026-05-30T10:00:00" new-api

# 日志文件保存在宿主机
ls ./logs/
tail -f ./logs/new-api.log
```

### 9.2 Nginx 访问日志

```bash
docker exec nginx-proxy cat /var/log/nginx/access.log
```

### 9.3 PostgreSQL 日志

```bash
docker-compose logs postgres
```

---

## 十、故障排查

### 10.1 服务无法访问

```bash
# 1. 检查容器状态
docker-compose ps

# 2. 检查 new-api 日志
docker-compose logs --tail=50 new-api

# 3. 测试内部健康检查
docker exec new-api wget -q -O - http://localhost:3000/api/status

# 4. 测试 nginx 代理
curl -H "Host: <你的域名>" http://localhost/api/status
```

### 10.2 数据库连接问题

```bash
# 检查 PostgreSQL 是否运行
docker-compose ps postgres

# 测试数据库连接
docker exec new-api sh -c "wget -q -O - postgres:5432 2>&1 || echo 'Connection failed'"

# 进入数据库检查
docker exec -it postgres psql -U root -d new-api
```

### 10.3 SSL 证书问题

```bash
# 查看 acme-companion 日志
docker-compose logs acme-companion

# 手动触发证书申请
docker exec acme-companion /app/signal_le_service

# 检查证书文件
docker exec nginx-proxy ls -la /etc/nginx/certs/
```

### 10.4 磁盘空间不足

```bash
# 查看磁盘使用
df -h
du -sh /root/opt/token-api-platform/data/
du -sh /root/opt/token-api-platform/logs/

# 清理旧的 Docker 镜像
docker image prune -a

# 清理 Docker 构建缓存
docker builder prune
```

### 10.5 完全重置

```bash
# ⚠️ 危险操作 — 删除所有数据
docker-compose down -v  # 删除容器 + 数据卷
rm -rf ./data/* ./logs/*
# 重新启动
docker-compose up -d
```

---

## 十一、常用命令速查

```bash
# 在项目目录执行
cd /root/opt/token-api-platform

# --- 服务管理 ---
docker-compose up -d                    # 启动
docker-compose down                     # 停止
docker-compose restart new-api          # 重启网关
docker-compose ps                       # 状态

# --- 日志 ---
docker-compose logs -f new-api          # 实时日志
docker-compose logs --tail=100 new-api  # 最近 100 行

# --- 数据库 ---
docker exec -it postgres psql -U root -d new-api              # 进入数据库
docker exec postgres pg_dump -U root new-api > backup.sql     # 备份

# --- 健康检查 ---
curl http://localhost/api/status        # API 状态
curl -s http://localhost/api/status | python3 -m json.tool     # 格式化输出

# --- 升级 ---
docker-compose pull new-api             # 拉取新镜像
docker-compose up -d --no-deps new-api  # 滚动更新

# --- 清理 ---
docker system prune -f                  # 清理无用资源
```
