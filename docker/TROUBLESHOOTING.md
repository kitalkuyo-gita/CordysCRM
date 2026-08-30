# CordysCRM Docker 部署故障排查指南

> **文档版本**: v1.0.0 | **日期**: 2026-08-30

---

## 目录

1. [常见问题速查表](#1-常见问题速查表)
2. [MySQL 连接失败排查](#2-mysql-连接失败排查)
3. [容器健康检查失败](#3-容器健康检查失败)
4. [前端无法访问](#4-前端无法访问)
5. [Docker 网络问题](#5-docker-网络问题)
6. [日志查看命令](#6-日志查看命令)
7. [完全重置部署](#7-完全重置部署)

---

## 1. 常见问题速查表

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| `bash: not found` | Alpine 镜像无 bash | Dockerfile 已安装 bash |
| `Connection refused` 到 MySQL | MySQL 未启动/网络不通 | 见 [第2节](#2-mysql-连接失败排查) |
| `MySQL not ready, retrying` | 后端无法连接 MySQL | 见 [第2节](#2-mysql-连接失败排查) |
| 后端 healthcheck 失败 | curl 不可用/应用未启动 | 检查后端日志 |
| 前端 502 Bad Gateway | 后端未启动 | 先启动后端，再启动前端 |
| `permission denied` | 文件权限错误 | 检查 volume 挂载权限 |

---

## 2. MySQL 连接失败排查

### 2.1 确认 MySQL 容器状态

```bash
# 查看容器运行状态
sudo docker ps -a | grep mysql

# 预期输出: Up X minutes (healthy)
# 如果显示 Restarting 或 Exit，说明 MySQL 启动失败
```

### 2.2 查看 MySQL 启动日志

```bash
sudo docker logs cordys-mysql 2>&1 | tail -30

# 正常应看到:
# [System] /usr/sbin/mysqld: ready for connections. Version: '8.0.x'
# 如果看到 ERROR 或 Warning，根据具体错误排查
```

### 2.3 测试 Docker 网络连通性

```bash
# 从后端容器内部测试 DNS 解析
sudo docker exec cordys-backend nslookup mysql

# 从后端容器内部测试 TCP 连接
sudo docker exec cordys-backend nc -z mysql 3306

# 如果 nslookup 失败 → DNS 问题
# 如果 nc 失败但 nslookup 成功 → 网络/防火墙问题
```

### 2.4 检查容器网络配置

```bash
# 查看后端容器的 IP 和网络
sudo docker inspect cordys-backend --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}{{"\n"}}{{end}}'

# 查看 MySQL 容器的 IP 和网络
sudo docker inspect cordys-mysql --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}{{"\n"}}{{end}}'

# 两个容器必须在同一网络中才能通信
```

### 2.5 手动测试数据库连接

```bash
# 从后端容器内用 mysql 客户端测试 (需安装 mysql-client)
sudo docker exec -it cordys-backend sh -c "apk add --no-cache mysql-client && mysql -h mysql -u root -p'CordysCRM@mysql' -e 'SELECT 1'"

# 或者从宿主机测试
mysql -h 127.0.0.1 -u root -p'CordysCRM@mysql' -e "SHOW DATABASES;"
```

### 2.6 常见 MySQL 启动失败原因

| 错误信息 | 原因 | 解决 |
|----------|------|------|
| `Table 'mysql.user' doesn't exist` | 数据目录损坏 | 删除 volume 重建: `docker volume rm cordys-mysql-data` |
| `InnoDB: Unable to lock ./ibdata1` | 数据目录被占用 | 停止所有 MySQL 容器后重试 |
| `Access denied for user 'root'` | 密码不匹配 | 检查 `.env` 中的 `MYSQL_ROOT_PASSWORD` |
| `Too many connections` | 连接数超限 | 调整 `my.cnf` 中 `max_connections` |

---

## 3. 容器健康检查失败

### 3.1 查看容器健康状态

```bash
# 查看所有容器的健康状态
sudo docker ps --format "table {{.Names}}\t{{.Status}}"

# 查看特定容器的健康检查日志
sudo docker inspect cordys-backend --format '{{json .State.Health}}' | jq .
```

### 3.2 后端健康检查

```bash
# 后端健康检查命令
curl -f http://localhost:8081/front/account/info

# 如果返回 200 → 正常
# 如果超时或连接拒绝 → 后端未启动或端口不对
```

### 3.3 前端健康检查

```bash
# 前端健康检查命令
curl -f http://localhost/health

# 如果返回 200 → Nginx 正常
# 如果 502 → 后端未启动
# 如果 404 → Nginx 配置错误
```

---

## 4. 前端无法访问

### 4.1 检查端口映射

```bash
# 查看端口映射
sudo docker port cordys-frontend

# 预期输出: 80/tcp -> 0.0.0.0:80
```

### 4.2 检查 Nginx 状态

```bash
# 查看前端日志
sudo docker logs cordys-frontend 2>&1 | tail -20

# 进入容器检查 Nginx
sudo docker exec -it cordys-frontend sh -c "nginx -t"
```

### 4.3 检查防火墙

```bash
# Ubuntu/Debian
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# CentOS/RHEL
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload
```

### 4.4 检查云服务器安全组

如果是云服务器（阿里云、腾讯云等），需要在安全组中放行端口：
- **80** (HTTP)
- **443** (HTTPS)
- **8081** (后端 API，可选)

---

## 5. Docker 网络问题

### 5.1 查看 Docker 网络

```bash
# 查看所有 Docker 网络
sudo docker network ls

# 查看 cordys-network 详情
sudo docker network inspect cordys-network
```

### 5.2 检查容器是否在同一网络

```bash
# 查看连接到 cordys-network 的容器
sudo docker network inspect cordys-network --format '{{range .Containers}}{{.Name}} {{end}}'
```

### 5.3 重建网络

```bash
# 停止所有容器
sudo docker compose down

# 删除旧网络
sudo docker network rm cordys-network

# 重新启动
sudo docker compose up -d
```

---

## 6. 日志查看命令

```bash
# 查看所有服务日志
sudo docker compose logs -f

# 查看特定服务日志
sudo docker compose logs -f backend
sudo docker compose logs -f mysql
sudo docker compose logs -f redis
sudo docker compose logs -f frontend

# 查看最近 N 行日志
sudo docker compose logs --tail=100 backend

# 查看最近 N 小时的日志
sudo docker compose logs --since=1h backend

# 查看容器状态
sudo docker compose ps

# 查看容器资源使用
sudo docker stats
```

---

## 7. 完全重置部署

### 7.1 保留数据重置

```bash
# 停止容器（保留数据卷）
sudo docker compose down

# 清理构建缓存
sudo docker compose down --rmi local

# 重新构建并启动
sudo docker compose up -d --build
```

### 7.2 完全重置（清除所有数据）

```bash
# ⚠️ 警告: 这会删除所有数据！

# 停止并删除所有容器和数据卷
sudo docker compose down -v

# 删除构建缓存
sudo docker system prune -a

# 重新构建并启动
sudo docker compose up -d --build
```

### 7.3 重置后重新初始化

```bash
# 启动后等待所有服务就绪
sudo docker compose up -d

# 查看启动进度
sudo docker compose logs -f

# 等待看到后端日志:
# [INFO] Starting CordysCRM Backend (version: latest)...
# [INFO] Started Application in XX seconds

# 访问系统
# 桌面端: http://你的服务器IP
# 移动端: http://你的服务器IP/mobile
# 默认账号: admin / CordysCRM
```

---

## 8. 环境变量配置

确保 `.env` 文件配置正确：

```bash
# 复制模板
cp .env.example .env

# 编辑配置（必须修改以下项）
vim .env
```

```env
# 必须修改!
MYSQL_ROOT_PASSWORD=你的MySQL密码
REDIS_PASSWORD=你的Redis密码
CORDYS_SECRET_KEY=你的应用密钥

# 可选修改
FRONTEND_PORT=80
BACKEND_PORT=8081
```

---

## 9. 性能监控

### 9.1 实时资源监控

```bash
# 查看所有容器资源使用
sudo docker stats

# 查看特定容器
sudo docker stats cordys-backend cordys-mysql cordys-redis
```

### 9.2 启用监控栈

```bash
# 启动 Prometheus + Grafana
sudo docker compose --profile monitor up -d

# 访问:
# Prometheus: http://你的服务器IP:9090
# Grafana: http://你的服务器IP:3000 (admin/cordys-crm)
```

---

## 10. 联系支持

如果以上步骤都无法解决问题，请提供以下信息：

```bash
# 收集诊断信息
echo "=== Docker Version ===" && docker version
echo "=== Docker Compose Version ===" && docker compose version
echo "=== Container Status ===" && docker compose ps
echo "=== Backend Logs ===" && docker compose logs --tail=50 backend
echo "=== MySQL Logs ===" && docker compose logs --tail=50 mysql
echo "=== Redis Logs ===" && docker compose logs --tail=50 redis
echo "=== Network Info ===" && docker network inspect cordys-network
echo "=== Disk Usage ===" && df -h
echo "=== Memory Usage ===" && free -h
```
