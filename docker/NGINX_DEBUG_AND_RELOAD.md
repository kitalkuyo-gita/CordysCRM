# Nginx 配置调试与热加载

## 问题描述

执行 `docker cp` 和 `nginx -s reload` 后，`/get-key` 和 `/is-login` 接口仍然返回前端 HTML，说明 Nginx 配置没有生效。

## 调试步骤

### 1. 检查容器内配置是否已更新

```bash
docker exec cordys-frontend cat /etc/nginx/conf.d/default.conf | grep "get-key"
```

- **有输出**：配置已复制，问题在 Nginx 重载
- **无输出**：`docker cp` 没有成功，配置未更新

### 2. 手动复制配置文件

```bash
docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf cordys-frontend:/etc/nginx/conf.d/default.conf
```

### 3. 验证配置语法

```bash
docker exec cordys-frontend nginx -t
```

- 应输出 `syntax is ok` 和 `test is successful`
- 如果报错，检查配置文件语法

### 4. 强制重载 Nginx

```bash
docker exec cordys-frontend nginx -s reload
```

### 5. 测试接口

```bash
curl -s http://115.159.88.112/get-key | head -5
```

应返回 Base64 编码的 RSA 公钥字符串，而不是 HTML。

## 常见问题

### docker cp 没有生效

- 检查容器名是否正确：`docker ps | grep frontend`
- 检查目标路径是否正确：`docker exec cordys-frontend ls -la /etc/nginx/conf.d/`
- 尝试先停止再启动容器：`docker restart cordys-frontend`

### nginx -s reload 报错

- 检查 Nginx 进程：`docker exec cordys-frontend ps aux | grep nginx`
- 检查错误日志：`docker exec cordys-frontend cat /var/log/nginx/error.log`
- 尝试重启容器：`docker restart cordys-frontend`

### 配置生效但接口仍返回 HTML

- 检查 Nginx 配置中的 upstream 是否正确：`docker exec cordys-frontend cat /etc/nginx/conf.d/default.conf | grep upstream`
- 检查后端容器是否运行：`docker ps | grep backend`
- 测试后端直接访问：`curl -s http://localhost:8081/get-key`

## 完整修复流程

```bash
# 1. 确认容器运行状态
docker ps | grep -E "frontend|backend"

# 2. 复制配置
docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf cordys-frontend:/etc/nginx/conf.d/default.conf

# 3. 验证配置
docker exec cordys-frontend nginx -t

# 4. 重载 Nginx
docker exec cordys-frontend nginx -s reload

# 5. 验证配置已更新
docker exec cordys-frontend cat /etc/nginx/conf.d/default.conf | grep "get-key"

# 6. 测试接口
curl -s http://115.159.88.112/get-key
curl -s http://115.159.88.112/is-login
```
