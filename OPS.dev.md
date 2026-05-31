# New API 本地开发运维手册

> 最后更新: 2026-05-31

---

## 一、运行环境概览

### 1.1 容器列表

| 容器 | 镜像 | 用途 | 端口 |
|------|------|------|------|
| `new-api` | `new-api-local:latest`（本地源码构建） | AI API 网关主程序 | 3000 (宿主机) |
| `lc_postgres` | `postgres:15` | 主数据库（复用现有） | 5432 (宿主机) |
| `lc_redis` | `redis:latest` | 缓存（复用现有） | 6379 (宿主机) |

### 1.2 连接方式

new-api 通过 `host.docker.internal` 连接宿主机上的 PostgreSQL 和 Redis：

| 服务 | 连接字符串 |
|------|-----------|
| PostgreSQL | `postgresql://postgres:123456@host.docker.internal:5432/new-api` |
| Redis | `redis://host.docker.internal:6379` |

### 1.3 本地挂载

| 宿主机路径 | 容器路径 | 用途 |
|-----------|---------|------|
| `./logs` | `/app/logs` | 应用日志 |

---

## 二、启动/停止

使用 `docker-compose.local.yml`（从本地源码构建后端，使用 Dockerfile.dev.local）：

```bash
cd /Users/chenjianbin/Documents/git_workspace/new-api

# 启动（首次需先创建数据库，见第五节）
docker compose -f docker-compose.local.yml up -d

# Go 代码修改后重建后端
docker compose -f docker-compose.local.yml up -d --build

# 停止
docker compose -f docker-compose.local.yml down

# 重启
docker compose -f docker-compose.local.yml restart
```

### 前端开发

后端运行在 `:3000`，前端 dev server 运行在 `:3001`，API 自动代理到后端：

```bash
cd web/default
bun install
bun run dev     # → http://localhost:3001
```

修改前端代码后热更新即时生效。

---

## 三、查看状态

```bash
# 容器状态
docker ps --filter "name=new-api"

# 实时日志
docker logs -f new-api

# 最近日志
docker logs --tail=50 new-api

# 健康检查
curl -s http://localhost:3000/api/status | python3 -m json.tool
```

---

## 四、升级

```bash
cd /Users/chenjianbin/Documents/git_workspace/new-api

# 拉取最新镜像并重建
docker compose -f docker-compose.local.yml pull
docker compose -f docker-compose.local.yml up -d
```

---

## 五、数据库管理

### 5.1 创建数据库（仅首次）

```bash
docker exec lc_postgres psql -U postgres -c "CREATE DATABASE \"new-api\";"
```

### 5.2 进入数据库

```bash
docker exec -it lc_postgres psql -U postgres -d new-api
```

### 5.3 备份

```bash
docker exec lc_postgres pg_dump -U postgres new-api > backup_$(date +%Y%m%d).sql
```

### 5.4 恢复

```bash
cat backup_20260531.sql | docker exec -i lc_postgres psql -U postgres -d new-api
```

### 5.5 重置数据库

```bash
docker exec lc_postgres psql -U postgres -c "DROP DATABASE \"new-api\";"
docker exec lc_postgres psql -U postgres -c "CREATE DATABASE \"new-api\";"
docker restart new-api
```

---

## 六、故障排查

### 6.1 容器无法启动

```bash
# 查看日志
docker logs --tail=50 new-api

# 常见问题: logs 目录不存在
mkdir -p ./logs
```

### 6.2 数据库连接失败

```bash
# 确认 lc_postgres 在运行
docker ps --filter "name=lc_postgres"

# 测试从 new-api 容器内连接
docker exec new-api wget -q -O - host.docker.internal:5432 2>&1
```

### 6.3 完全重建

```bash
docker compose -f docker-compose.local.yml down
# 如需重置数据库
docker exec lc_postgres psql -U postgres -c "DROP DATABASE IF EXISTS \"new-api\";"
docker exec lc_postgres psql -U postgres -c "CREATE DATABASE \"new-api\";"
# 重新启动
docker compose -f docker-compose.local.yml up -d
```

---

## 七、与生产环境的关键差异

| 项目 | 开发环境 (OPS.dev) | 生产环境 (OPS.prod) |
|------|-------------------|---------------------|
| 数据库 | 复用 lc_postgres | 独立 postgres 容器 |
| 缓存 | 复用 lc_redis | 独立 redis 容器 |
| 反向代理 | 无 | nginx-proxy |
| SSL | 无 | Let's Encrypt |
| 域名 | localhost | mh.api.d1xf.cn |
| 启动方式 | docker-compose.local.yml | docker-compose.yml |
| Go 环境 | 不需要 | 不需要 |
