# UtilityPay 贡献指南

感谢您对 **UtilityPay** 项目的关注！本指南帮助您了解如何参与项目贡献。

---

## 目录

- [UtilityPay 贡献指南](#utilitypay-贡献指南)
  - [目录](#目录)
  - [行为准则](#行为准则)
  - [如何贡献](#如何贡献)
    - [报告 Bug](#报告-bug)
    - [提交功能请求](#提交功能请求)
    - [提交代码](#提交代码)
    - [代码审查](#代码审查)
  - [开发环境搭建](#开发环境搭建)
    - [前置要求](#前置要求)
    - [一键启动开发环境](#一键启动开发环境)
    - [可访问的服务](#可访问的服务)
  - [项目结构](#项目结构)
  - [编码规范](#编码规范)
    - [Java 后端](#java-后端)
    - [前端](#前端)
    - [通用](#通用)
    - [命名约定](#命名约定)
  - [Git 工作流](#git-工作流)
    - [分支策略](#分支策略)
    - [开发流程](#开发流程)
  - [提交信息规范](#提交信息规范)
    - [类型 (Type)](#类型-type)
    - [范围 (Scope)](#范围-scope)
    - [示例](#示例)
  - [测试指南](#测试指南)
    - [测试策略](#测试策略)
    - [运行测试](#运行测试)
  - [文档贡献](#文档贡献)
    - [文档类型](#文档类型)
    - [文档规范](#文档规范)
  - [社区沟通](#社区沟通)
  - [许可协议](#许可协议)
    - [开发者证书 (Developer Certificate of Origin)](#开发者证书-developer-certificate-of-origin)

---

## 行为准则

本项目遵循 [Contributor Covenant 行为准则](CODE_OF_CONDUCT.md)。所有参与者都应遵守该准则，共同营造开放、友好的社区环境。

---

## 如何贡献

### 报告 Bug

在提交 Bug 报告前，请：

1. 搜索 [Issues](https://github.com/LisaHusband/UtilityPay/issues) 确认该 Bug 未被报告
2. 使用 **Bug Report** 模板创建 Issue，包含：
   - 清晰的标题
   - 复现步骤
   - 期望行为与实际行为
   - 运行环境（OS、Docker 版本、浏览器等）
   - 相关截图或日志

### 提交功能请求

1. 搜索现有 Issues 确认该功能未被请求
2. 使用 **Feature Request** 模板，说明：
   - 功能描述
   - 使用场景
   - 期望实现方式
   - 是否愿意参与实现

### 提交代码

**贡献流程：**

```
1. Fork 仓库
2. 创建特性分支 (feat/xxx 或 fix/xxx)
3. 编写代码与测试
4. 确保 CI 通过
5. 提交 Pull Request
6. 参与代码审查
```

**首次贡献（Good First Issues）：**

我们标记了 `good first issue` 标签的任务，适合新贡献者上手。可以查看 [Good First Issues](https://github.com/LisaHusband/UtilityPay/labels/good%20first%20issue) 列表。

### 代码审查

PR 提交后将由项目维护者审查，审查要点包括：

- 代码风格符合项目规范
- 测试覆盖率未下降
- 无安全漏洞
- 文档同步更新
- CI 全部通过（包括静态检查）

---

## 开发环境搭建

### 前置要求

| 工具 | 最低版本 | 用途 |
|------|---------|------|
| **Docker** | 24.x+ | 容器运行时 |
| **Docker Compose** | 2.x+ | 服务编排 |
| **Java JDK** | 17+ | 后端开发 |
| **Node.js** | 20.x LTS | 前端开发 |
| **pnpm** | 9.x+ | 前端包管理 |
| **Git** | 2.x+ | 版本控制 |

### 一键启动开发环境

```bash
# 1. 克隆项目
git clone https://github.com/LisaHusband/UtilityPay.git
cd UtilityPay

# 2. 配置环境变量
cp .env.example .env

# 3. 启动基础设施
docker compose up -d

# 4. 验证服务状态
docker compose ps

# 5. 启动后端（热重载模式）
cd backend
./gradlew bootRun --args='--spring.profiles.active=dev'

# 6. 启动前端（新终端）
cd frontend
pnpm install
pnpm dev
```

### 可访问的服务

| 服务 | 地址 | 认证 |
|------|------|------|
| Web 门户 | http://localhost:3000 | - |
| API Swagger | http://localhost:8080/swagger-ui.html | - |
| Airflow | http://localhost:8080 | airflow / airflow |
| Kafka UI | http://localhost:8088 | - |
| ClickHouse HTTP | http://localhost:8123/play | utilitypay / utilitypay_dev |
| RustFS Console | http://localhost:9001 | utilitypay_admin / utilitypay_dev123 |
| Kibana | http://localhost:5601 | - |
| Grafana | http://localhost:3000 | admin / admin |

---

## 项目结构

```
UtilityPay/
├── backend/                      # Java 后端服务
│   ├── utilitypay-common/        # 公共模块（工具类、常量、DTO）
│   ├── utilitypay-gateway/       # API 网关 (Spring Cloud Gateway)
│   ├── utilitypay-user/          # 用户服务
│   ├── utilitypay-provider/      # Provider 接入服务
│   ├── utilitypay-rating/        # 计费引擎
│   ├── utilitypay-billing/       # 账单服务
│   ├── utilitypay-payment/       # 支付服务
│   ├── utilitypay-notification/  # 通知服务
│   ├── utilitypay-invoice/       # 发票服务
│   ├── utilitypay-enterprise/    # 企业服务
│   ├── utilitypay-workflow/      # 审批工作流
│   ├── utilitypay-carbon/        # 碳追踪服务
│   ├── utilitypay-ai-prediction/ # AI 预测服务
│   ├── utilitypay-rag-search/    # RAG 搜索服务
│   ├── utilitypay-market/        # 能源市场服务
│   ├── utilitypay-microgrid/     # 微电网服务
│   ├── utilitypay-plugin-sdk/    # 插件 SDK 接口定义
│   └── build.gradle              # 根构建文件
├── frontend/                     # 前端项目
│   ├── apps/
│   │   ├── web-portal/           # 用户 Web 门户 (Next.js)
│   │   ├── admin-console/        # 管理后台 (React + Ant Design Pro)
│   │   └── mobile/               # 移动端 (React Native)
│   ├── packages/
│   │   ├── shared/               # 共享类型定义与工具库
│   │   ├── ui-components/        # 通用 UI 组件库
│   │   └── api-client/           # API 客户端 SDK
│   └── pnpm-workspace.yaml
├── infrastructure/               # 基础设施配置
│   ├── clickhouse/
│   │   ├── config/               # ClickHouse 配置
│   │   ├── migrations/           # 数据库迁移 SQL
│   │   └── init/                 # 初始化脚本
│   ├── prometheus/
│   │   └── prometheus.yml
│   └── grafana/
│       ├── dashboards/
│       └── datasources/
├── airflow/                      # Airflow DAG 与插件
│   ├── dags/                     # DAG 定义
│   └── plugins/                  # Airflow 插件
├── plugins/                      # 可插拔 Provider / Payment 插件
│   ├── provider-china-power/
│   ├── provider-eu-water/
│   └── payment-alipay/
├── docs/                         # 项目文档
│   ├── deep-research-report.md
│   └── technical-architecture.md
├── .github/                      # GitHub 配置
│   ├── workflows/                # CI/CD 流水线
│   └── ISSUE_TEMPLATE/
├── docker-compose.yml            # 开发环境编排
├── Dockerfile                    # 生产镜像构建
├── CONTRIBUTING.md               # 本文件
├── CODE_OF_CONDUCT.md            # 行为准则
├── LICENSE                       # Apache 2.0
└── README.md
```

---

## 编码规范

### Java 后端

| 规范 | 工具 | 配置 |
|------|------|------|
| 代码风格 | **Checkstyle** (Google Java Style) | `config/checkstyle/google_checks.xml` |
| 静态分析 | **PMD** + **SpotBugs** | `config/pmd/rules.xml` |
| 代码格式化 | **Spotless** (自动格式化) | `build.gradle` 中配置 |
| 导入顺序 | Google Style | 自动排序 |

```bash
# 运行静态检查
./gradlew checkstyleMain pmdMain spotbugsMain

# 自动格式化代码
./gradlew spotlessApply
```

### 前端

| 规范 | 工具 | 配置 |
|------|------|------|
| JavaScript/TypeScript | **ESLint** (Airbnb + TypeScript) | `frontend/.eslintrc.js` |
| 代码格式化 | **Prettier** | `frontend/.prettierrc` |
| 样式 | **Stylelint** | `frontend/.stylelintrc` |
| 提交前检查 | **lint-staged** + **husky** | `frontend/package.json` |

```bash
# 运行检查
pnpm lint

# 自动修复
pnpm lint:fix
pnpm format
```

### 通用

| 规范 | 工具 | 适用范围 |
|------|------|---------|
| Dockerfile | **hadolint** | 所有 `Dockerfile` |
| Markdown | **markdownlint** | 所有 `.md` 文件 |
| YAML | **yamllint** | 所有 `.yml` / `.yaml` 文件 |
| Commit 信息 | **commitlint** | Git 提交 |

### 命名约定

| 类型 | 规范 | 示例 |
|------|------|------|
| Java 包名 | 小写，点分隔 | `com.utilitypay.billing.service` |
| Java 类名 | 大驼峰 (PascalCase) | `BillingService`, `RateCalculator` |
| Java 方法 | 小驼峰 (camelCase) | `calculateBill()`, `syncUsageData()` |
| REST API | 复数名词，小写 | `/api/v1/bills`, `/api/v1/users` |
| SQL 表名 | 小写蛇形 (snake_case) | `user_account`, `bill_item` |
| SQL 列名 | 小写蛇形 | `billing_period_start`, `total_amount` |
| Git 分支 | 前缀/描述 | `feat/rating-engine`, `fix/bill-overdue` |

---

## Git 工作流

### 分支策略

```
main                    # 生产就绪代码
├── develop             # 开发主分支
│   ├── feat/xxx        # 新功能
│   ├── fix/xxx         # Bug 修复
│   ├── docs/xxx        # 文档更新
│   ├── refactor/xxx    # 代码重构
│   └── chore/xxx       # 构建/工具变更
└── release/x.y.z       # 发布分支
```

### 开发流程

```bash
# 1. 从 develop 创建特性分支
git checkout develop
git pull origin develop
git checkout -b feat/my-feature

# 2. 开发并提交（遵循提交信息规范）
git add .
git commit -m "feat(rating): add tiered pricing calculation"

# 3. 推送并创建 PR
git push origin feat/my-feature

# 4. 在 GitHub 创建 Pull Request 到 develop 分支
#    - 填写 PR 模板
#    - 关联相关 Issue
#    - 等待 CI 通过和审查
```

---

## 提交信息规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/zh-hans/v1.0.0/) 规范：

```
<类型>(<范围>): <描述>

[可选的正文]

[可选的脚注]
```

### 类型 (Type)

| 类型 | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档更新 |
| `style` | 代码格式（不影响功能） |
| `refactor` | 代码重构 |
| `test` | 测试相关 |
| `chore` | 构建/工具/依赖变更 |
| `perf` | 性能优化 |
| `ci` | CI/CD 配置变更 |

### 范围 (Scope)

建议使用受影响模块名：`rating`、`billing`、`payment`、`user`、`gateway`、`frontend`、`docs`、`infra` 等。

### 示例

```
feat(rating): add seasonal pricing support

Implemented seasonal pricing calculation with configurable
winter/summer rate differentials.

Closes #42
```

---

## 测试指南

### 测试策略

| 测试类型 | 覆盖目标 | 工具 |
|---------|---------|------|
| **单元测试** | ≥80% | JUnit 5 + Mockito |
| **集成测试** | API 端点全覆盖 | Spring Boot Test + Testcontainers |
| **契约测试** | 微服务间接口 | Spring Cloud Contract / Pact |
| **端到端测试** | 关键业务流程 | Playwright / Cypress |
| **性能测试** | 计费引擎吞吐量 | JMeter / k6 |

### 运行测试

```bash
# 后端单元测试
cd backend && ./gradlew test

# 后端集成测试（需要 Docker）
cd backend && ./gradlew integrationTest

# 前端测试
cd frontend && pnpm test

# 前端 E2E 测试
cd frontend && pnpm test:e2e

# 全部测试 + 覆盖率报告
cd backend && ./gradlew test jacocoTestReport
```

---

## 文档贡献

### 文档类型

| 文档 | 位置 | 备注 |
|------|------|------|
| 架构设计 | `docs/technical-architecture.md` | ADR 记录含在此文件 |
| 产品需求 | `docs/deep-research-report.md` | 功能需求来源 |
| API 文档 | Swagger 自动生成 | 代码注解驱动 |
| 插件开发指南 | `docs/plugin-development.md` | 如何开发 Provider/Payment 插件 |
| 部署手册 | `docs/deployment.md` | 生产环境部署步骤 |

### 文档规范

- 使用 Markdown 格式
- 中文优先，关键术语保留英文
- 代码块标注语言类型
- 流程图使用 Mermaid 语法

---

## 社区沟通

| 渠道 | 用途 |
|------|------|
| [GitHub Issues](https://github.com/LisaHusband/UtilityPay/issues) | Bug 报告、功能请求 |
| [GitHub Discussions](https://github.com/LisaHusband/UtilityPay/discussions) | 技术讨论、问答 |
| [Discord]() | 即时沟通（建设中） |
| [邮件列表]() | 版本发布通知（建设中） |

---

## 许可协议

UtilityPay 基于 **Apache License 2.0** 发布。

通过提交 Pull Request，您同意您贡献的代码将在 Apache 2.0 许可下发布。如果您贡献的代码包含第三方依赖，请确保其许可证与 Apache 2.0 兼容。

### 开发者证书 (Developer Certificate of Origin)

通过提交代码，您确认：

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the Apache 2.0 license; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified it.
```

---

**再次感谢您的贡献！** 🎉

如有任何疑问，请通过 [GitHub Discussions](https://github.com/LisaHusband/UtilityPay/discussions) 联系我们。