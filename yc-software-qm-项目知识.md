# yc-software/qm 项目知识总结

> 日期：2026-08-07
> 来源：浏览器自动化学习对话

---

## 一、项目概述

| 知识点 | 内容 |
|-------|------|
| **是什么** | 多 Agent 团队协作平台，每个人有自己的独立工作空间 |
| **仓库地址** | https://github.com/yc-software/qm |
| **核心架构** | TypeScript + Fastify 核心，PostgreSQL 持久化，Slack 插件，Web UI（Vite + Lit） |
| **支持的 Harness** | Pi、Claude Code、Codex、OpenCode 可切换 |
| **安全策略** | Strict（全审批）/ Auto（自动分类审查）/ Dangerous（无审查） |
| **核心依赖** | PostgreSQL、Fastify、Slack Bolt、AWS SDK |
| **许可证** | MIT |

---

## 二、核心功能

| 功能 | 说明 |
|------|------|
| **个人 + 共享作用域** | 每个人自定义自己的 Agent，同时能在频道中协作 |
| **Slack + Web** | 同一身份和配置在 Slack 和 Web 间无缝切换 |
| **管理员控制** | 组织级配置、安全策略、可用模型管理 |
| **Web Apps** | 快速创建内部应用并发布给指定人员 |
| **共享技能** | 技能可按作用域授权，管理员可推广到整个组织 |
| **后台任务** | Crons 和 Watches 无人值守时自动运行 |

### 能做什么

- 搜索内部笔记、邮件、文档、数据库和网络
- 从公司知识库检索信息
- 构建内部应用并发布给相关人员
- 学习你的写作风格，定时处理收件箱
- 在现有仓库中工作：跑测试、开 PR、监控 CI、查日志
- 在共享频道中跟踪项目并发布更新

---

## 三、架构

```
每个请求 → 核心 (TypeScript + Fastify)
  ├── Postgres（持久化层：用户/会话/记忆/文件/权限/审计/队列）
  ├── Slack 插件（Bolt）
  ├── Web UI（Vite + Lit）
  ├── 管理面板
  └── 公共门户
```

每个 Agent 拥有**独立的沙箱环境**（持久化计算机），安装的工具不会丢失。

---

## 四、安全策略

| 模式 | 说明 |
|------|------|
| **Strict（严格）** | 每个工具调用都需人工审批 |
| **Auto（自动，默认）** | 分类器审查外部数据后再给模型 |
| **Dangerous（危险）** | 无审查、无暂停 |

---

## 五、部署选项

### 支持的 Target

| Target | 说明 | 数据库 |
|--------|------|--------|
| `docker` | ✅ **本地可跑** | 本地 Docker PostgreSQL 或自配 DSN |
| `fly` | Fly.io 全球边缘部署 | Fly Postgres |
| `aws` | ECS Fargate（无服务器容器，非 EC2） | RDS |

**不是必须 Fly.io 或 AWS！** macOS 本地用 Docker target 就能跑。

### 各 Target 依赖

| 需求 | Docker（本地） | Fly | AWS |
|------|:-----------:|:---:|:---:|
| PostgreSQL | 本地容器或自配 DSN | Fly Postgres | RDS |
| Docker 守护进程 | ✅ 需要 | 仅构建 | 仅构建 |
| Node 24 + qm CLI | ✅ | ✅ | ✅ |
| Slack App | 可选 | 可选 | 可选 |

---

## 六、安装与部署

### 方式一：快速部署（推荐）

```bash
# 创建组织拥有的部署仓库
npm exec --yes --package=@yc-software/qm@latest -- \
  qm init . --org <组织slug> --target <docker|fly|aws>
npm install
```

该命令会：
- 生成部署配置和基础设施
- 配置 Web 登录
- 配置连接器凭证
- 可选配置 Slack 接入
- 部署和线上验证

### 方式二：本地开发

```bash
# 克隆仓库
git clone git@github.com:yc-software/qm.git
cd qm

# 启动本地开发实例
npm run dev-instance

# 或直接
node scripts/dev/cli.ts up
```

### 方式三：私有 Fork（需要完整代码自定义）

```bash
# 克隆裸仓库
git clone --bare git@github.com:yc-software/qm qm-seed.git
git -C qm-seed.git push --mirror git@github.com:<org>/qm-private
rm -rf qm-seed.git

# 检出
git clone git@github.com:<org>/qm-private
git -C qm-private remote add upstream git@github.com:yc-software/qm
```

### 部署命令生命周期

```bash
npm exec qm -- check          # 静态检查配置
npm exec qm -- doctor         # 检查外部依赖
npm exec qm -- plan           # 预览部署计划
npm exec qm -- up --yes       # 执行部署
npm exec qm -- check --live   # 验证线上状态
npm exec qm -- down           # 下线
npm exec qm -- rollback       # 回滚
```

---

## 七、运行时依赖（package.json 核心依赖）

| 类别 | 组件 | 用途 |
|------|------|------|
| **数据库** | `pg` + `pg-boss` | PostgreSQL 驱动 + 任务队列 |
| **AI Harness** | `@earendil-works/pi-coding-agent` | 核心 Agent 驱动（Pi） |
| | `@openai/codex` | Codex 支持 |
| | `@opencode-ai/sdk` | OpenCode 支持 |
| | `@anthropic-ai/claude-agent-sdk` | Claude Code 支持 |
| **消息平台** | `@slack/bolt` + `@slack/socket-mode` + `@slack/web-api` | Slack 集成 |
| **Web 框架** | `fastify` | HTTP API 核心 |
| **云服务** | `@aws-sdk/client-s3` | S3 存储 |
| | `@aws-sdk/client-secrets-manager` | 密钥管理 |
| | `@aws-sdk/client-sts` | AWS 凭证 |
| **其他** | `croner` | 定时任务 |
| | `jose` | JWT 加密 |
| | `lru-cache` | 缓存 |
| | `typebox` / `zod` | 类型校验 |

---

## 八、常见问题澄清

| 问题 | 答案 |
|------|------|
| **必须 Fly.io 或 AWS 吗？** | ❌ **不用**，支持 `target: docker` 本地跑 |
| **Fly.io 是啥？** | 基于 Firecracker microVM 的全球边缘部署平台，类似 Heroku 但更现代，自带 Fly Postgres |
| **AWS 不是卖 EC2 吗？** | qm 用的是 **ECS Fargate**（无服务器容器），不是 EC2（虚拟机） |
| **macOS 本地能跑吗？** | ✅ **可以**，有完整的 `dev up` 本地开发命令 |
| **数据库咋办？** | 本地 Docker 起一个 PostgreSQL 容器即可，或用 `dev up` 自动处理 |
| **需要自己买服务器吗？** | 本地开发不需要。生产部署用 Fly.io 或 AWS 托管服务 |

---

## 九、部署目录结构

```
qm.config.jsonc          ← 部署配置（无密钥）
package.json             ← 固定 CLI 版本
package-lock.json
deployment.md            ← 部署手册
.codex/skills/deploy-qm/ ← Agent 部署技能
.env.example             ← 环境变量模板
.env                     ← 实际值（gitignored）
slack-app-manifest.yml   ← Slack App 配置
sandbox/
  tools/<id>/tool.json   ← 工具描述
  tools/<id>/<binary>    ← 工具二进制
  skills/<id>/SKILL.md   ← 技能定义
  Dockerfile
plugins/<name>/Dockerfile
infra/                   ← AWS Terraform 模块
```

---

*本文档由学习对话总结生成，用于记录 yc-software/qm 项目相关知识点。*