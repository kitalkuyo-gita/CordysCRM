# 国内 Docker 镜像加速配置指南

> **适用场景**: 中国大陆服务器无法直接访问 Docker Hub
>
> **日期**: 2026-08-30

---

## 问题描述

Docker 构建时无法拉取基础镜像，报错类似：

```
ERROR: failed to resolve source metadata for docker.io/library/eclipse-temurin:21-jre-alpine:
failed to do request: Head "https://docker.m.daocloud.io/v2/...": dial tcp: i/o timeout
```

**原因**: Docker Hub 在中国大陆被限制访问，需要配置国内镜像加速器。

---

## 方案一：配置 Docker Daemon 镜像加速（推荐）

### 1.1 创建/编辑 Docker 配置文件

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF
```

### 1.2 重启 Docker 服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 1.3 验证配置生效

```bash
# 查看镜像源配置
sudo docker info | grep -A 5 "Registry Mirrors"

# 测试拉取镜像
sudo docker pull nginx:alpine
```

### 1.4 重新构建项目

```bash
cd /usr/local/share/gitrepo/CordysCRM/docker
sudo docker compose down
sudo docker compose up -d --build
```

---

## 方案二：手动预拉基础镜像

如果方案一的镜像源也不可用，可以逐个拉取并重命名：

```bash
# ----- Node.js (前端构建) -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/node:22-slim
sudo docker tag docker.mirrors.ustc.edu.cn/library/node:22-slim node:22-slim

# ----- JDK 21 (后端构建) -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/eclipse-temurin:21-jdk
sudo docker tag docker.mirrors.ustc.edu.cn/library/eclipse-temurin:21-jdk eclipse-temurin:21-jdk

# ----- JRE 21 Alpine (后端运行时) -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/eclipse-temurin:21-jre-alpine
sudo docker tag docker.mirrors.ustc.edu.cn/library/eclipse-temurin:21-jre-alpine eclipse-temurin:21-jre-alpine

# ----- Nginx (前端运行时) -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/nginx:1.27-alpine
sudo docker tag docker.mirrors.ustc.edu.cn/library/nginx:1.27-alpine nginx:1.27-alpine

# ----- MySQL 8.0 -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/mysql:8.0
sudo docker tag docker.mirrors.ustc.edu.cn/library/mysql:8.0 mysql:8.0

# ----- Redis 7 Alpine -----
sudo docker pull docker.mirrors.ustc.edu.cn/library/redis:7-alpine
sudo docker tag docker.mirrors.ustc.edu.cn/library/redis:7-alpine redis:7-alpine
```

拉取完成后正常构建：

```bash
sudo docker compose up -d --build
```

---

## 方案三：使用第三方加速代理

### 3.1 DaoCloud 镜像站

```bash
# 临时使用
sudo docker pull docker.m.daocloud.io/library/node:22-slim

# 或配置为默认源
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": ["https://docker.m.daocloud.io"]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 3.2 Azure 中国镜像

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": ["https://dockerhub.azk8s.cn"]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

---

## 国内镜像源列表

| 镜像源 | 地址 | 提供方 | 状态 |
|--------|------|--------|------|
| Tencent | `https://mirror.ccs.tencentyun.com` | 腾讯云 | ✅ 推荐 |
| USTC | `https://docker.mirrors.ustc.edu.cn` | 中科大 | ✅ 推荐 |
| NetEase | `https://hub-mirror.c.163.com` | 网易 | ✅ 可用 |
| DaoCloud | `https://docker.m.daocloud.io` | DaoCloud | ⚠️ 不稳定 |
| Azure | `https://dockerhub.azk8s.cn` | Azure 中国 | ⚠️ 不稳定 |

---

## 常见问题

### Q: 配置了镜像源但还是超时？

```bash
# 检查 Docker 是否读取了新配置
sudo docker info | grep -A 5 "Registry Mirrors"

# 如果没显示，手动重启 Docker
sudo systemctl stop docker
sudo systemctl start docker
```

### Q: 镜像源不可用了怎么办？

换一个镜像源，编辑 `/etc/docker/daemon.json`：

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://新的镜像源地址"
  ]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### Q: 如何测试镜像源是否可用？

```bash
# 测试拉取一个小镜像
time sudo docker pull nginx:alpine

# 如果在 30 秒内完成，说明镜像源正常
```

### Q: 企业环境如何配置私有镜像仓库？

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": ["https://your-registry.company.com"],
  "insecure-registries": ["your-registry.company.com:5000"]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

---

## 快速修复脚本

将以下脚本保存为 `setup-docker-mirror.sh` 并执行：

```bash
#!/bin/bash
set -e

echo "=== 配置 Docker 国内镜像加速 ==="

# 备份原有配置
if [ -f /etc/docker/daemon.json ]; then
    sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%Y%m%d)
    echo "已备份原配置到 /etc/docker/daemon.json.bak.$(date +%Y%m%d)"
fi

# 写入新配置
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF

# 重启 Docker
echo "重启 Docker 服务..."
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证
echo "验证镜像源配置..."
sudo docker info | grep -A 5 "Registry Mirrors"

echo ""
echo "=== 配置完成！==="
echo "现在可以执行: sudo docker compose up -d --build"
```

运行：

```bash
chmod +x setup-docker-mirror.sh
sudo ./setup-docker-mirror.sh
```
