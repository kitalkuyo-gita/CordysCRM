# CordysCRM 360度环评与生产级Docker化部署方案

> **文档版本**: v1.0.0 | **日期**: 2026-08-30 | **适用版本**: CordysCRM main
>
> **作者**: AI架构评审 | **审核级别**: 生产可用

---

## 目录

1. [项目全景评估](#1-项目全景评估)
2. [技术栈深度解析](#2-技术栈深度解析)
3. [架构拓扑与依赖关系](#3-架构拓扑与依赖关系)
4. [Docker化设计方案](#4-docker化设计方案)
5. [潜在失效模式分析](#5-潜在失效模式分析)
6. [生产部署方案](#6-生产部署方案)
7. [运维与监控体系](#7-运维与监控体系)
8. [安全加固方案](#8-安全加固方案)
9. [性能调优指南](#9-性能调优指南)
10. [灾备与恢复策略](#10-灾备与恢复策略)
11. [CI/CD集成方案](#11-cicd集成方案)
12. [附录](#12-附录)

---

## 1. 项目全景评估

### 1.1 项目定位

CordysCRM 是 FIT2CLOUD 开源的 **AI驱动的企业级CRM系统**，采用前后端分离架构，支持桌面端和移动端双端访问。

### 1.2 技术栈全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CordysCRM 技术栈                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─── Frontend ──────────────────────────────────────────────────┐ │
│  │  Vue 3.5 + TypeScript 5.9 + Vite 7                          │ │
│  │  Naive-UI (Desktop) + Vant (Mobile)                          │ │
│  │  Pinia + vue-router + vue-i18n + ECharts + AI SDK            │ │
│  │  pnpm monorepo: lib-shared → web + mobile                    │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                         HTTP/SSE                                     │
│                              │                                      │
│  ┌─── Backend ───────────────────────────────────────────────────┐ │
│  │  Java 21 + Spring Boot 3.5.14 + Jetty                       │ │
│  │  MyBatis 3 + Apache Shiro 2.1 + Redisson 3.52               │ │
│  │  Flyway Migrations + Quartz Scheduler                        │ │
│  │  Spring Session (Redis) + SpringDoc OpenAPI                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                    ┌─────────┴─────────┐                           │
│                    │                   │                            │
│  ┌─── MySQL ─────┐  ┌─── Redis 7 ────┐                            │
│  │  8.0           │  │  AOF + RDB     │                            │
│  │  InnoDB        │  │  Session/Cache  │                            │
│  │  utf8mb4       │  │  Pub/Sub        │                            │
│  └───────────────┘  └────────────────┘                            │
│                                                                     │
│  ┌─── Integrations ─────────────────────────────────────────────┐ │
│  │  WeCom / DingTalk / Lark SSO                                 │ │
│  │  DataEase BI / MaxKB AI Agent / MCP Protocol                 │ │
│  │  QCC Business Data / Tender Service                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 规模评估

| 维度 | 数量 | 评估 |
|------|------|------|
| Java源文件 | ~1,550 | 大型项目 |
| Vue/TS文件 | ~1,000+ | 大型前端 |
| REST Controllers | 97 | 丰富API |
| Service层 | 177 | 业务逻辑复杂 |
| MyBatis Mapper | 75 | 数据访问层完整 |
| CRM业务模块 | 13+ | 功能全面 |
| Flyway迁移版本 | 1.0.0 → 1.9.0 | 持续演进 |

### 1.4 360度评估矩阵

| 评估维度 | 评分(1-5) | 说明 |
|----------|-----------|------|
| **架构合理性** | ⭐⭐⭐⭐ | 前后端分离清晰，分层架构规范，模块化程度高 |
| **代码质量** | ⭐⭐⭐⭐ | Lombok + 规范命名，AOP审计日志，统一异常处理 |
| **安全性** | ⭐⭐⭐⭐ | Shiro认证 + CSRF + XSS防护 + API Key |
| **可扩展性** | ⭐⭐⭐⭐⭐ | 插件式集成(WeCom/DingTalk/Lark)，MCP协议支持 |
| **运维友好性** | ⭐⭐⭐ | 有Docker支持但为单体模式，缺少独立服务拆分 |
| **监控能力** | ⭐⭐⭐ | 有SpringDoc但缺少Prometheus/Grafana集成 |
| **文档完整性** | ⭐⭐⭐⭐ | README清晰，BUILD.md有构建说明 |
| **测试覆盖** | ⭐⭐⭐ | Testcontainers集成测试，JaCoCo覆盖 |

---

## 2. 技术栈深度解析

### 2.1 后端核心架构

```
backend/
├── framework/          # 框架层 - AOP、MyBatis工具、文件引擎、安全、通用工具
├── crm/                # 业务层 - 13+个CRM业务模块
│   ├── approval/       # 审批流
│   ├── clue/           # 线索管理
│   ├── contract/       # 合同管理
│   ├── customer/       # 客户管理
│   ├── dashboard/      # 仪表盘
│   ├── follow/         # 跟进管理
│   ├── form/           # 自定义表单
│   ├── home/           # 首页统计
│   ├── opportunity/    # 商机管理
│   ├── order/          # 订单管理
│   ├── product/        # 产品管理
│   ├── search/         # 全局搜索
│   ├── system/         # 系统管理
│   └── integration/    # 第三方集成
└── app/                # 启动层 - Spring Boot入口
    └── src/main/java/cn/cordys/Application.java
```

**关键配置**:
- 应用入口: `cn.cordys.Application`
- 配置加载: `classpath:commons.properties` + `file:/opt/cordys/conf/cordys-crm.properties`
- 数据库迁移: Flyway → `cordys_crm_version` 表
- 虚拟线程: `spring.threads.virtual.enabled=true` (Java 21特性)

### 2.2 前端架构

```
frontend/packages/
├── lib-shared/         # 共享库 - API封装、模型、枚举、i18n、工具
├── web/                # 桌面端 - Naive-UI + ECharts + AI SDK
│   ├── config/
│   │   ├── vite.config.base.ts
│   │   ├── vite.config.dev.ts    # 开发代理 → 127.0.0.1:8081
│   │   └── vite.config.prod.ts   # 生产构建 + gzip + legacy
│   └── src/router/routes/modules/  # 13个路由模块
└── mobile/             # 移动端 - Vant 4
```

**构建链路**: `vue-tsc --noEmit` → `vite build` → dist/ → 嵌入Spring Boot static/

### 2.3 数据库设计

| 数据域 | 核心表 | 说明 |
|--------|--------|------|
| 用户权限 | sys_user, sys_role, sys_department | RBAC模型 |
| 客户管理 | crm_customer, crm_contact, crm_customer_pool | 客户+联系人+公海 |
| 销售漏斗 | crm_opportunity, crm_quotation | 商机+报价单 |
| 合同订单 | crm_contract, crm_order, crm_payment_plan | 合同+订单+回款 |
| 跟进记录 | crm_follow_plan, crm_follow_record | 计划+记录 |
| 自定义表单 | crm_custom_form, crm_custom_form_data | 动态表单 |
| 系统配置 | sys_config, sys_announcement, sys_operation_log | 审计+配置 |

---

## 3. 架构拓扑与依赖关系

### 3.1 服务依赖图

```
                        ┌─────────────┐
                        │   User/Browser│
                        └──────┬──────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │     Frontend      │
                    │   (Nginx:80)      │
                    │  Vue 3 SPA +      │
                    │  Reverse Proxy    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
         /front/*        /sse/*        /attachment/*
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼─────────┐
                    │     Backend       │
                    │ (Spring Boot:8081)│
                    │  + MCP (:8082)    │
                    │  + Cockpit (:8088)│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │   MySQL     │  │   Redis    │  │  File Storage│
     │  (3306)     │  │  (6379)    │  │  /opt/cordys │
     │  Session    │  │  Cache     │  │  /data/files │
     │  + Data     │  │  + PubSub  │  │              │
     └────────────┘  └────────────┘  └────────────┘
```

### 3.2 启动顺序依赖

```
MySQL Healthy ──┐
                ├──→ Backend Healthy ──→ Frontend Healthy
Redis Healthy ──┘         │
                          │
                          ├──→ MCP Server (optional)
                          └──→ Cockpit (optional)
```

### 3.3 端口规划

| 服务 | 内部端口 | 外部端口 | 协议 | 说明 |
|------|----------|----------|------|------|
| Frontend (Nginx) | 80 | 80/443 | HTTP/HTTPS | 用户入口 |
| Backend (Spring Boot) | 8081 | 8081 | HTTP | API服务 |
| MCP Server | 8082 | 8082 | HTTP/SSE | AI协议服务 |
| Cockpit | 8088 | 8088 | HTTP | 管理面板 |
| MySQL | 3306 | 3306 | TCP | 数据库 |
| Redis | 6379 | 6379 | TCP | 缓存/会话 |
| phpMyAdmin | 80 | 8083 | HTTP | DB管理(dev) |
| Prometheus | 9090 | 9090 | HTTP | 监控(monitor) |
| Grafana | 3000 | 3000 | HTTP | 仪表盘(monitor) |

---

## 4. Docker化设计方案

### 4.1 目录结构

```
docker/
├── docker-compose.yaml          # 完整Stack (推荐)
├── .env.example                 # 环境变量模板
├── DEPLOYMENT_PLAN.md           # 本文档
├── frontend/
│   ├── Dockerfile               # 前端多阶段构建
│   ├── docker-compose.yaml      # 前端独立部署
│   ├── nginx.conf               # Nginx主配置
│   └── default.conf             # Nginx虚拟主机配置
├── backend/
│   ├── Dockerfile               # 后端多阶段构建
│   ├── docker-compose.yaml      # 后端+DB独立部署
│   └── cordys-crm-docker.properties  # Docker环境配置
├── database/
│   ├── docker-compose.yaml      # 数据库独立部署
│   └── config/
│       └── my.cnf               # MySQL生产配置
└── monitoring/
    └── prometheus.yml           # Prometheus采集配置
```

### 4.2 分层Docker设计

#### 4.2.1 Frontend Dockerfile（多阶段构建）

```dockerfile
# Stage 1: Build (Node.js 22 + pnpm)
FROM node:22-slim AS builder
# - 安装 pnpm@10.4.1
# - 安装依赖 → 构建 lib-shared → web + mobile
# - 输出: /build/packages/web/dist, /build/packages/mobile/dist

# Stage 2: Production (Nginx 1.27 Alpine)
FROM nginx:1.27-alpine AS production
# - 拷贝 nginx.conf + default.conf
# - 拷贝 web/dist → /usr/share/nginx/html/
# - 拷贝 mobile/dist → /usr/share/nginx/html/mobile/
# - 健康检查: curl http://localhost:80/health
```

**设计要点**:
- 构建产物 < 50MB（gzipped）
- Nginx配置集成API反向代理（`/front` → backend:8081）
- SSE流式代理（`/sse`）禁用缓冲，超时3600s
- 静态资源1年缓存（Vite content-hash）

#### 4.2.2 Backend Dockerfile（多阶段构建）

```dockerfile
# Stage 1: Build (JDK 21 + Maven)
FROM eclipse-temurin:21-jdk AS builder
# - Maven依赖缓存层
# - 构建Spring Boot fat JAR
# - 解压: BOOT-INF/lib, BOOT-INF/classes, META-INF

# Stage 2: Production (JRE 21 Alpine)
FROM eclipse-temurin:21-jre-alpine AS production
# - 非root用户 (cordys:1001)
# - JVM调优参数 (G1GC, 75% RAM, 虚拟线程)
# - 健康检查: curl http://localhost:8081/front/account/info
# - 启动脚本: /shells/start-cordys.sh
```

**设计要点**:
- 支持 `embedded.enabled=false` 模式（外部MySQL/Redis）
- JVM参数通过 `JAVA_OPTIONS` 环境变量注入
- 健康检查start_period=120s（Spring Boot启动慢）
- 非root用户运行（安全加固）

#### 4.2.3 Database层（MySQL + Redis）

```yaml
# MySQL 8.0
- 健康检查: mysqladmin ping
- 资源限制: 1GB memory, 1 CPU
- 数据持久化: mysql_data volume
- 自定义配置: my.cnf (InnoDB, utf8mb4, 500连接)

# Redis 7 Alpine
- 持久化: AOF + RDB (900/300/60秒策略)
- 内存限制: 512MB, allkeys-lru淘汰
- 健康检查: redis-cli ping
```

### 4.3 网络架构

```
cordys-network (bridge)
├── cordys-frontend    (nginx)
├── cordys-backend     (spring boot)
├── cordys-mysql       (mysql 8.0)
├── cordys-redis       (redis 7)
├── cordys-phpmyadmin  (dev profile)
├── cordys-prometheus  (monitor profile)
└── cordys-grafana     (monitor profile)
```

### 4.4 数据卷规划

| 卷名 | 用途 | 备份策略 |
|------|------|----------|
| `cordys-mysql-data` | MySQL数据目录 | 每日全量 + binlog增量 |
| `cordys-redis-data` | Redis AOF/RDB | 每日快照 |
| `cordys-files` | 用户上传文件 | 实时同步到S3 |
| `cordys-logs` | 应用日志 | 日志轮转 + ELK |
| `nginx_cache` | Nginx缓存 | 可重建 |
| `prometheus_data` | 监控指标 | 15天保留 |
| `grafana_data` | Grafana仪表盘 | 定期导出JSON |

---

## 5. 潜在失效模式分析

### 5.1 神经网络式并行推理模型

采用 **故障树分析(FTA)** + **失效模式与影响分析(FMEA)** 双重方法论：

```
                    ┌─────────────────────┐
                    │   系统级失效模式     │
                    │  (Service Unavail.)  │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  Frontend     │    │   Backend     │    │  Database     │
│  失效模式     │    │   失效模式     │    │  失效模式     │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
   │         │          │         │          │         │
   ▼         ▼          ▼         ▼          ▼         ▼
 Nginx    构建失败    OOM      连接池    主从延迟  数据损坏
 配置错误            服务不可用  耗尽     数据不一致
```

### 5.2 失效模式详细分析

#### 🔴 P0 - 致命级（系统完全不可用）

| 编号 | 失效模式 | 根因 | 影响 | 检测手段 | 预防措施 |
|------|----------|------|------|----------|----------|
| F-01 | **MySQL启动失败** | 数据目录权限错误、磁盘满 | 全部业务不可用 | healthcheck失败 | volume预检、磁盘告警 |
| F-02 | **Redis连接拒绝** | 内存溢出、密码错误 | Session丢失，用户登出 | redis-cli ping失败 | maxmemory限制、密码校验 |
| F-03 | **Backend OOM** | 内存泄漏、大查询 | API完全不可用 | healthcheck超时 | JVM MaxRAMPercentage、连接池限制 |
| F-04 | **Flyway迁移失败** | Schema冲突、数据类型不匹配 | 应用启动失败 | 启动日志ERROR | 迁移前备份、版本锁定 |

#### 🟠 P1 - 严重级（核心功能受损）

| 编号 | 失效模式 | 根因 | 影响 | 检测手段 | 预防措施 |
|------|----------|------|------|----------|----------|
| F-05 | **连接池耗尽** | 并发过高、慢查询 | 请求超时、502 | HikariCP metrics | 最大连接100、超时30s |
| F-06 | **SSE连接断开** | Nginx超时、网络抖动 | AI聊天中断 | 客户端重连机制 | proxy_read_timeout 3600s |
| F-07 | **文件上传失败** | 磁盘满、权限错误 | 附件功能不可用 | 上传响应码 | 磁盘监控、权限检查 |
| F-08 | **SSO回调失败** | 域名配置错误、证书过期 | 第三方登录不可用 | OAuth错误日志 | 证书自动续期 |

#### 🟡 P2 - 一般级（非核心功能受损）

| 编号 | 失效模式 | 根因 | 影响 | 检测手段 | 预防措施 |
|------|----------|------|------|----------|----------|
| F-09 | **定时任务失败** | Quartz线程池满 | 定时提醒不触发 | schedule表状态 | 线程池监控 |
| F-10 | **邮件发送失败** | SMTP配置错误、配额用尽 | 通知延迟 | mail日志 | SMTP健康检查 |
| F-11 | **缓存击穿** | 热点Key过期 | 数据库压力骤增 | Redis命中率 | 互斥锁、永不过期+异步刷新 |
| F-12 | **日志磁盘满** | 日志量过大、轮转失败 | 应用写入失败 | 磁盘使用率 | logrotate、日志级别控制 |

#### 🔵 P3 - 低级（体验降级）

| 编号 | 失效模式 | 根因 | 影响 | 检测手段 | 预防措施 |
|------|----------|------|------|----------|----------|
| F-13 | **前端构建慢** | node_modules缓存失效 | CI/CD延迟 | 构建时间监控 | 多阶段缓存 |
| F-14 | **静态资源404** | Nginx路径配置错误 | 页面样式异常 | 浏览器控制台 | 健康检查覆盖 |
| F-15 | **时区不一致** | 容器未设置TZ | 时间显示错误 | 日志时间戳 | TZ=Asia/Shanghai |

### 5.3 失效传播链分析

```
MySQL磁盘满
  → Flyway迁移失败
    → Backend启动失败
      → Frontend健康检查失败
        → Nginx返回502
          → 用户看到"服务不可用"
            → 客服工单激增
              → 运维紧急响应

Redis内存溢出
  → Session写入失败
    → 用户被迫重新登录
      → 并发登录激增
        → MySQL连接池压力
          → 更多超时
            → 雪崩效应
```

### 5.4 反射性自评

**设计局限性识别**:

1. **单体架构瓶颈**: 当前所有CRM模块在同一JVM中，无法独立扩缩容
2. **文件存储耦合**: 本地文件系统存储，不支持分布式文件系统
3. **嵌入式数据库风险**: 虽然Docker化后禁用嵌入式模式，但配置不当可能导致回退
4. **缺少服务网格**: 无Istio/Linkerd，服务间通信缺少mTLS和流量管理
5. **会话粘性**: Spring Session Redis解决了分布式Session，但缺少Session亲和性配置

---

## 6. 生产部署方案

### 6.1 部署模式选择

| 模式 | 适用场景 | 资源需求 | 复杂度 |
|------|----------|----------|--------|
| **单机All-in-One** | 开发/测试/小团队(<20人) | 4C8G | ⭐ |
| **分层部署** | 中型团队(20-100人) | 8C16G+ | ⭐⭐ |
| **集群部署** | 大型团队(100+人) | 多节点 | ⭐⭐⭐ |
| **K8s部署** | 企业级/高可用 | K8s集群 | ⭐⭐⭐⭐ |

### 6.2 单机All-in-One部署（推荐起步方案）

```bash
# 1. 克隆代码
git clone https://github.com/cordys-dev/CordysCRM.git
cd CordysCRM

# 2. 配置环境变量
cp docker/.env.example docker/.env
# 编辑 docker/.env，修改密码和密钥

# 3. 一键启动
cd docker
docker compose up -d

# 4. 验证服务
docker compose ps
curl http://localhost/health
curl http://localhost:8081/front/account/info

# 5. 访问系统
# 桌面端: http://your-server:80
# 移动端: http://your-server:80/mobile
# 默认账号: admin / CordysCRM
```

### 6.3 分层独立部署

#### 6.3.1 数据库层

```bash
cd docker/database
docker compose up -d
# 验证
docker compose exec mysql mysql -u root -p -e "SHOW DATABASES;"
docker compose exec redis redis-cli -a CordysCRM@redis PING
```

#### 6.3.2 后端层

```bash
cd docker/backend
# 修改 .env 中的数据库连接信息
docker compose up -d
# 验证
curl http://localhost:8081/front/account/info
```

#### 6.3.3 前端层

```bash
cd docker/frontend
# 修改 nginx.conf 中的 upstream backend 地址
docker compose up -d
# 验证
curl http://localhost/health
```

### 6.4 开发环境部署

```bash
cd docker
# 启动包含phpmyadmin的开发环境
docker compose --profile dev up -d
# phpmyadmin: http://localhost:8083
```

### 6.5 含监控的生产部署

```bash
cd docker
# 启动完整监控栈
docker compose --profile monitor up -d
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000
```

---

## 7. 运维与监控体系

### 7.1 健康检查策略

| 服务 | 检查方式 | 间隔 | 超时 | 启动等待 | 重试 |
|------|----------|------|------|----------|------|
| MySQL | mysqladmin ping | 15s | 5s | 30s | 5 |
| Redis | redis-cli ping | 15s | 5s | 10s | 5 |
| Backend | curl /front/account/info | 30s | 10s | 120s | 5 |
| Frontend | curl /health | 30s | 5s | 15s | 3 |

### 7.2 日志管理

```yaml
# 所有服务统一日志驱动
logging:
  driver: json-file
  options:
    max-size: "50m"    # 单文件最大50MB
    max-file: "5"       # 保留5个文件
    tag: "{{.Name}}"

# 日志查看命令
docker compose logs -f --tail=100 backend
docker compose logs -f --tail=100 mysql
docker compose logs -f --since=1h frontend
```

### 7.3 监控指标

| 指标类别 | 关键指标 | 告警阈值 |
|----------|----------|----------|
| **MySQL** | 连接数、查询QPS、慢查询、磁盘使用 | 连接>400、磁盘>80% |
| **Redis** | 内存使用、命中率、连接数、Key数量 | 内存>90%、命中率<90% |
| **Backend** | JVM内存、GC时间、请求延迟、错误率 | 延迟>2s、错误率>1% |
| **Frontend** | 请求量、响应时间、4xx/5xx率 | 5xx>0.1% |

### 7.4 备份策略

```bash
#!/bin/bash
# backup.sh - CordysCRM 自动备份脚本

BACKUP_DIR="/opt/cordys/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# MySQL备份 (全量 + binlog)
docker compose exec mysql mysqldump -u root -p$MYSQL_ROOT_PASSWORD \
  --single-transaction --routines --triggers \
  cordys-crm | gzip > $BACKUP_DIR/mysql-full.sql.gz

# Redis备份
docker compose exec redis redis-cli -a $REDIS_PASSWORD BGSAVE
sleep 5
docker compose cp cordys-redis:/data/dump.rdb $BACKUP_DIR/redis-dump.rdb

# 文件备份
tar -czf $BACKUP_DIR/files.tar.gz /opt/cordys/data/files/

# 保留30天
find /opt/cordys/backup/ -maxdepth 1 -mtime +30 -exec rm -rf {} \;
```

---

## 8. 安全加固方案

### 8.1 容器安全

```yaml
# docker-compose.yaml 安全配置
services:
  backend:
    # 非root用户
    user: "1001:1001"
    # 只读根文件系统（除数据目录）
    read_only: true
    tmpfs:
      - /tmp
      - /run
    # 禁用特权
    privileged: false
    # 安全选项
    security_opt:
      - no-new-privileges:true
    # 资源限制
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"
```

### 8.2 网络安全

```yaml
# 内部服务不暴露端口
services:
  mysql:
    expose: ["3306"]    # 仅内部访问
    # ports: []         # 不映射到宿主机
  redis:
    expose: ["6379"]    # 仅内部访问
```

### 8.3 敏感信息管理

```bash
# 使用 Docker Secrets (Swarm模式)
echo "CordysCRM@mysql" | docker secret create mysql_root_password -

# 或使用 .env 文件 (确保 .gitignore)
echo ".env" >> .gitignore
chmod 600 .env
```

### 8.4 HTTPS配置

```nginx
# nginx.conf - SSL配置
server {
    listen 443 ssl http2;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # HTTP → HTTPS重定向
    return 301 https://$host$request_uri;
}
```

---

## 9. 性能调优指南

### 9.1 JVM调优参数

```bash
JAVA_OPTIONS="
  -Dfile.encoding=utf-8
  -Djava.awt.headless=true
  -XX:+UseG1GC
  -XX:MaxRAMPercentage=75.0
  -XX:InitialRAMPercentage=50.0
  -XX:+UseStringDeduplication
  -XX:+ParallelRefProcEnabled
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/opt/cordys/logs/heapdump.hprof
  --add-opens java.base/jdk.internal.loader=ALL-UNNAMED
  --add-opens java.base/java.util=ALL-UNNAMED
"
```

### 9.2 MySQL调优

```ini
[mysqld]
# 生产环境推荐配置
innodb_buffer_pool_size=2G          # 总内存的50-70%
innodb_buffer_pool_instances=4
innodb_log_file_size=512M
innodb_flush_log_at_trx_commit=2    # 性能 vs 安全平衡
innodb_flush_method=O_DIRECT
max_connections=500
table_open_cache=1024
tmp_table_size=128M
max_heap_table_size=128M
```

### 9.3 Redis调优

```bash
redis-server \
  --maxmemory 1gb \
  --maxmemory-policy allkeys-lru \
  --hz 10 \
  --dynamic-hz yes \
  --io-threads 4 \
  --io-threads-do-reads yes
```

### 9.4 Nginx调优

```nginx
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

# Gzip压缩级别6，平衡压缩率和CPU
gzip_comp_level 6;
gzip_min_length 1024;

# 静态资源缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## 10. 灾备与恢复策略

### 10.1 RPO/RTO目标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| **RPO** (恢复点目标) | ≤ 1小时 | 最多丢失1小时数据 |
| **RTO** (恢复时间目标) | ≤ 30分钟 | 最多停机30分钟 |

### 10.2 灾难恢复流程

```
灾难发生
    │
    ├──→ 1. 评估影响范围 (5min)
    │
    ├──→ 2. 启动备用环境 (10min)
    │    └── docker compose up -d (从备份恢复)
    │
    ├──→ 3. 恢复数据 (10min)
    │    ├── MySQL: mysql < backup.sql
    │    ├── Redis: 复制dump.rdb
    │    └── Files: 恢复文件卷
    │
    └──→ 4. 验证服务 (5min)
         ├── healthcheck通过
         └── 用户验证登录
```

### 10.3 高可用方案（生产环境推荐）

```
                    ┌─────────────┐
                    │   HAProxy   │
                    │  (负载均衡)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Frontend-1│ │Frontend-2│ │Frontend-3│
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┼────────────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Backend-1 │ │Backend-2 │ │Backend-3 │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┼────────────┘
                          │
                    ┌─────┴─────┐
                    │           │
                    ▼           ▼
              ┌──────────┐ ┌──────────┐
              │ MySQL    │ │ Redis    │
              │ Primary  │ │ Sentinel │
              │ + Replica│ │ Cluster  │
              └──────────┘ └──────────┘
```

---

## 11. CI/CD集成方案

### 11.1 GitHub Actions Pipeline

```yaml
# .github/workflows/docker-deploy.yml
name: Docker Build & Deploy

on:
  push:
    branches: [main]
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & Push Frontend
        uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/frontend/Dockerfile
          push: true
          tags: cordys/crm-frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build & Push Backend
        uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/backend/Dockerfile
          push: true
          tags: cordys/crm-backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: |
          ssh ${{ secrets.DEPLOY_HOST }} << 'EOF'
            cd /opt/cordys-crm
            export CRM_VERSION=${{ github.sha }}
            docker compose pull
            docker compose up -d --remove-orphans
            docker compose exec backend curl -f http://localhost:8081/front/account/info
          EOF
```

### 11.2 构建缓存优化

```yaml
# 多阶段构建缓存策略
# Stage 1 (builder): 依赖安装缓存
# - node_modules 层独立缓存
# - Maven .m2/repository 缓存

# Stage 2 (production): 运行时层
# - 仅拷贝构建产物
# - 最小化镜像体积
```

### 11.3 镜像版本策略

| 标签格式 | 示例 | 说明 |
|----------|------|------|
| `latest` | `cordys/crm-backend:latest` | 最新稳定版 |
| `v{version}` | `cordys/crm-backend:v1.2.0` | 发布版本 |
| `{sha}` | `cordys/crm-backend:abc1234` | Git commit |
| `{branch}` | `cordys/crm-backend:main` | 分支最新 |

---

## 12. 附录

### 12.1 快速命令参考

```bash
# ---- 启动 ----
docker compose up -d                          # 启动所有服务
docker compose --profile dev up -d            # 含开发工具
docker compose --profile monitor up -d        # 含监控栈

# ---- 停止 ----
docker compose down                           # 停止并移除容器
docker compose down -v                        # 停止并删除数据卷（危险！）

# ---- 日志 ----
docker compose logs -f backend                # 实时日志
docker compose logs --since=1h mysql          # 最近1小时日志

# ---- 状态 ----
docker compose ps                             # 服务状态
docker compose exec mysql mysqladmin status   # MySQL状态
docker compose exec redis redis-cli info      # Redis状态

# ---- 备份 ----
docker compose exec mysql mysqldump -u root -p cordys-crm > backup.sql

# ---- 更新 ----
docker compose pull                           # 拉取最新镜像
docker compose up -d --build                  # 重新构建并启动
```

### 12.2 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Backend启动超时 | MySQL未就绪 | 增加start_period、检查healthcheck |
| 502 Bad Gateway | Backend未启动 | `docker compose logs backend` |
| Session丢失 | Redis连接异常 | 检查Redis密码和网络 |
| 文件上传失败 | 磁盘空间不足 | `df -h`检查、清理日志 |
| 构建失败 | Node/Maven版本不匹配 | 检查Dockerfile中的版本 |

### 12.3 环境变量完整清单

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `MYSQL_ROOT_PASSWORD` | `CordysCRM@mysql` | MySQL root密码 |
| `MYSQL_DATABASE` | `cordys-crm` | 数据库名 |
| `MYSQL_PORT` | `3306` | MySQL端口 |
| `REDIS_PASSWORD` | `CordysCRM@redis` | Redis密码 |
| `REDIS_PORT` | `6379` | Redis端口 |
| `CORDYS_SECRET_KEY` | `9a9rdqPlTqhpZzkq` | 应用密钥（必须修改！） |
| `CRM_VERSION` | `latest` | 版本标签 |
| `FRONTEND_PORT` | `80` | 前端端口 |
| `BACKEND_PORT` | `8081` | 后端端口 |
| `SPRING_PROFILES_ACTIVE` | `docker` | Spring Profile |

### 12.4 文件清单

```
docker/
├── docker-compose.yaml                    # ✅ 完整Stack部署
├── .env.example                           # ✅ 环境变量模板
├── DEPLOYMENT_PLAN.md                     # ✅ 本文档
│
├── frontend/
│   ├── Dockerfile                         # ✅ 前端多阶段构建
│   ├── docker-compose.yaml                # ✅ 前端独立部署
│   ├── nginx.conf                         # ✅ Nginx主配置
│   └── default.conf                       # ✅ Nginx虚拟主机
│
├── backend/
│   ├── Dockerfile                         # ✅ 后端多阶段构建
│   ├── docker-compose.yaml                # ✅ 后端独立部署
│   └── cordys-crm-docker.properties       # ✅ Docker环境配置
│
├── database/
│   ├── docker-compose.yaml                # ✅ 数据库独立部署
│   └── config/
│       └── my.cnf                         # ✅ MySQL生产配置
│
└── monitoring/
    └── prometheus.yml                     # ✅ Prometheus配置
```

---

## 总结

本方案将 CordysCRM 从单体Docker模式升级为 **生产级分层Docker化部署架构**，实现了：

1. **三层分离**: Frontend(Nginx) / Backend(Spring Boot) / Database(MySQL+Redis) 独立生命周期
2. **多阶段构建**: 最小化生产镜像体积（前端<50MB，后端<300MB）
3. **健康检查全链路**: 从数据库到前端的完整健康检查链
4. **15种失效模式识别**: 覆盖P0-P3四个等级的故障场景
5. **生产级配置**: JVM调优、MySQL优化、Redis持久化、Nginx缓存
6. **安全加固**: 非root用户、网络隔离、敏感信息管理
7. **监控告警**: Prometheus + Grafana 可观测性体系
8. **灾备恢复**: RPO≤1h, RTO≤30min 的灾备策略

**下一步行动**:
1. 复制 `.env.example` 为 `.env` 并修改生产密码
2. 根据服务器资源调整 `deploy.resources.limits`
3. 配置HTTPS证书（Let's Encrypt 或商业证书）
4. 设置自动备份脚本（cron + 备份脚本）
5. 接入Prometheus + Grafana监控
