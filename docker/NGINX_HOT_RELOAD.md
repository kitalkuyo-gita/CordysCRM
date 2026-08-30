# Nginx 热更新指南（无需重建镜像）

> **场景**: 修改了 Nginx 配置后不想重新构建 Docker 镜像（构建耗时较长）
>
> **日期**: 2026-08-30

---

## 快速命令

```bash
# 1. 将本地修改的配置拷贝到容器内
sudo docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf cordys-frontend:/etc/nginx/conf.d/default.conf

# 2. 重载 Nginx 配置（秒级生效，不中断服务）
sudo docker exec cordys-frontend nginx -s reload
```

---

## 完整操作流程

### 场景一：修改 default.conf（虚拟主机配置）

```bash
# 本地编辑配置
vim /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf

# 拷贝到容器
sudo docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/default.conf cordys-frontend:/etc/nginx/conf.d/default.conf

# 验证配置语法
sudo docker exec cordys-frontend nginx -t

# 重载配置
sudo docker exec cordys-frontend nginx -s reload
```

### 场景二：修改 nginx.conf（主配置）

```bash
# 本地编辑配置
vim /usr/local/share/gitrepo/CordysCRM/docker/frontend/nginx.conf

# 拷贝到容器
sudo docker cp /usr/local/share/gitrepo/CordysCRM/docker/frontend/nginx.conf cordys-frontend:/etc/nginx/nginx.conf

# 验证并重载
sudo docker exec cordys-frontend nginx -t && sudo docker exec cordys-frontend nginx -s reload
```

### 场景三：添加新的配置文件

```bash
# 创建新配置文件
vim /tmp/my-new-location.conf

# 拷贝到容器
sudo docker cp /tmp/my-new-location.conf cordys-frontend:/etc/nginx/conf.d/my-new-location.conf

# 重载
sudo docker exec cordys-frontend nginx -s reload
```

---

## 注意事项

| 操作 | 是否需要重建镜像 | 说明 |
|------|:---:|------|
| 修改 default.conf | ❌ 不需要 | docker cp + nginx -s reload |
| 修改 nginx.conf | ❌ 不需要 | docker cp + nginx -s reload |
| 添加/删除 .conf 文件 | ❌ 不需要 | docker cp + nginx -s reload |
| 修改前端代码 | ✅ 需要 | 需要重新构建镜像 |
| 修改 Nginx 版本 | ✅ 需要 | 需要修改 Dockerfile |

---

## 常用 Nginx 管理命令

```bash
# 查看 Nginx 进程
sudo docker exec cordys-frontend ps aux | grep nginx

# 查看 Nginx 错误日志
sudo docker exec cordys-frontend tail -f /var/log/nginx/error.log

# 查看 Nginx 访问日志
sudo docker exec cordys-frontend tail -f /var/log/nginx/access.log

# 测试配置文件语法
sudo docker exec cordys-frontend nginx -t

# 重载配置（不中断服务）
sudo docker exec cordys-frontend nginx -s reload

# 重启 Nginx（会短暂中断）
sudo docker exec cordys-frontend nginx -s reopen

# 查看当前生效的配置
sudo docker exec cordys-frontend nginx -T
```

---

## 为什么要用 reload 而不是 restart？

| 方式 | 命令 | 影响 |
|------|------|------|
| **reload** | `nginx -s reload` | 平滑重载，不中断现有连接 |
| **restart** | `docker restart cordys-frontend` | 停止再启动，中断所有连接 |

**推荐使用 reload**，除非修改了主配置文件（nginx.conf）中 `worker_processes` 等需要重启才能生效的参数。
