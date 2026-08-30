# 后端直连测试指南（绕过 Nginx）

> **场景**: 前端登录报错时，需要判断问题是出在 Nginx 代理层还是后端本身
>
> **日期**: 2026-08-30

---

## 问题排查思路

```
前端请求 → Nginx (80) → 后端 (8081)
                      ↑
                 问题可能在这两层中的任一层
```

**排查方法**: 直接请求后端（绕过 Nginx），对比结果。

---

## 测试命令

### 1. 登录接口测试

```bash
curl -v -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"CordysCRM","platform":"PC"}'
```

### 2. 通过 Nginx 测试（对比）

```bash
curl -v -X POST http://localhost/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"CordysCRM","platform":"PC"}'
```

---

## 结果判断

| 直连后端 (8081) | 通过 Nginx (80) | 问题所在 |
|:---:|:---:|:---|
| ✅ 成功 | ✅ 成功 | 无问题 |
| ✅ 成功 | ❌ 失败 | **Nginx 代理配置问题** |
| ❌ 失败 | ❌ 失败 | **后端本身问题** |

---

## 常见后端问题及解决方案

### 问题：返回 400 + `{password_is_null} {user_name_is_null}`

**原因**: 请求体未正确传递到后端

**排查**:
```bash
# 检查 Content-Type 是否正确
curl -v -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"CordysCRM","platform":"PC"}' 2>&1 | grep -i content-type
```

### 问题：返回 403 Forbidden

**原因**: Shiro 认证拦截

**排查**:
```bash
# 检查后端日志
sudo docker logs cordys-backend 2>&1 | grep -i "shiro\|forbidden\|unauthorized" | tail -10
```

### 问题：返回 500 Internal Server Error

**原因**: 后端代码异常

**排查**:
```bash
# 查看后端错误日志
sudo docker logs cordys-backend 2>&1 | grep -i "exception\|error" | tail -20
```

### 问题：连接拒绝 (Connection refused)

**原因**: 后端未启动或端口不对

**排查**:
```bash
# 检查后端是否运行
sudo docker ps | grep backend

# 检查端口监听
sudo docker exec cordys-backend netstat -tlnp 2>/dev/null || ss -tlnp
```

---

## 常用 API 测试命令

```bash
# 健康检查
curl http://localhost:8081/front/account/info

# 登录
curl -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"CordysCRM","platform":"PC"}'

# 登出
curl http://localhost:8081/logout

# 获取用户信息（需要 Cookie）
curl -b cookies.txt http://localhost:8081/front/account/info

# Swagger API 文档
curl http://localhost:8081/swagger-ui.html
```

---

## Nginx 代理问题排查

```bash
# 测试 Nginx 是否能连接到后端
sudo docker exec cordys-frontend curl -s http://cordys-backend:8081/front/account/info

# 查看 Nginx 错误日志
sudo docker exec cordys-frontend tail -20 /var/log/nginx/error.log

# 查看 Nginx 访问日志
sudo docker exec cordys-frontend tail -20 /var/log/nginx/access.log

# 测试 Nginx 配置语法
sudo docker exec cordys-frontend nginx -t
```
