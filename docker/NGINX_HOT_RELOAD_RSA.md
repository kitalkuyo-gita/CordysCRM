# Nginx 热加载 RSA 公钥接口配置

## 问题描述

前端登录需要先获取 RSA 公钥（`GET /get-key`），但 Nginx 缺少该接口的代理规则，导致请求被 catch-all 规则返回前端 HTML。

## 修复内容

在 `docker/frontend/default.conf` 中新增以下代理规则：

```nginx
# RSA 公钥接口
location /get-key {
    proxy_pass http://backend_api;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_connect_timeout 10s;
    proxy_read_timeout 30s;
}

# 登录状态检查接口
location /is-login {
    proxy_pass http://backend_api;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_connect_timeout 10s;
    proxy_read_timeout 30s;
}

# 用户信息接口
location /get-auth {
    proxy_pass http://backend_api;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_connect_timeout 10s;
    proxy_read_timeout 30s;
}
```

## 热加载命令

无需重建前端容器，直接热加载 Nginx 配置：

```bash
# 1. 复制新配置到容器
docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf cordys-frontend:/etc/nginx/conf.d/default.conf

# 2. 验证配置语法
docker exec cordys-frontend nginx -t

# 3. 热加载 Nginx（不中断服务）
docker exec cordys-frontend nginx -s reload
```

## 验证

```bash
# 测试 RSA 公钥接口（应返回 Base64 编码的公钥字符串）
curl -s http://115.159.88.112/get-key

# 测试登录状态接口（未登录应返回 401 或 null）
curl -s http://115.159.88.112/is-login
```

## 后端接口说明

| 接口 | 方法 | 说明 | 是否需要登录 |
|------|------|------|-------------|
| `/get-key` | GET | 获取 RSA 公钥 | 否 |
| `/login` | POST | 用户登录（RSA 加密） | 否 |
| `/logout` | GET/POST | 退出登录 | 否 |
| `/is-login` | GET | 检查登录状态 | 否 |
| `/get-auth` | GET | 获取当前用户信息 | 是 |
| `/front/*` | GET/POST | 业务接口（去除 /front 前缀） | 是 |
