# UtilityPay 技术架构与开发规划

---

## 一、技术栈总览

### 1.1 后端核心技术栈

| 层次 | 技术选型 | 理由 |
|------|---------|------|
| **语言与框架** | **Java 17 + Spring Boot 3.x / Spring Cloud** | 企业级成熟度最高，生态丰富（Spring Security、Spring Cloud Gateway、Spring Data JPA），适合复杂计费引擎与插件化架构；多租户、事务、审计方面支持完善 |
| **API 网关** | **Spring Cloud Gateway** | 统一入口、限流、鉴权、路由、多语言请求头处理 |
| **服务注册与配置** | **Nacos / Consul** | 服务发现、动态配置（计费规则热更新等） |
| **消息队列** | **Apache Kafka** | 高吞吐、持久化，适合账单异步生成、通知推送、用量数据流式处理；支持事件溯源与审计日志 |
| **定时任务与工作流调度** | **Apache Airflow** | DAG 编排能力强，支持复杂依赖调度（账单周期批量生成、上游数据同步、碳足迹批量计算、多步骤 ETL 管道）；内置重试/告警/监控，Web UI 可视化管理；Python 生态丰富，可直接集成 AI 模型训练管道 |

### 1.2 前端技术栈

| 层次 | 技术选型 | 理由 |
|------|---------|------|
| **Web 管理端** | **React 18 + TypeScript + Ant Design Pro** | 企业级后台 UI 组件库，国际化支持完善，适合复杂表单（计费规则配置）、仪表盘、审批流 |
| **移动端** | **React Native + Expo** | 跨平台 iOS/Android，共享 Web 端类型定义，快速迭代 |
| **用户端门户** | **Next.js (SSR)** | 面向终端用户的账单查看/支付页面，SEO 友好，首屏加载快 |
| **微前端** | **qiankun / Module Federation** | 多团队协作场景下，管理端、企业端、Provider 端可独立部署 |

### 1.3 数据层

| 层次 | 技术选型 | 理由 |
|------|---------|------|
| **主数据库** | **ClickHouse** | 列式存储引擎，针对海量账单与用量数据的 OLAP 分析场景极致优化；支持高压缩比（10:1 以上）、分布式表、物化视图、实时聚合；适合时序用量查询、账单多维分析、碳足迹批量计算等场景 |
| **事务型引擎补充** | **ClickHouse Keeper + 原子性 INSERT** | ClickHouse 提供有限的事务支持（原子批量写入），支付、用户等强一致性场景通过**幂等设计 + Saga 模式**补偿，关键事务状态使用 Redis 分布式锁保障 |
| **缓存** | **Redis 7.x (Cluster)** | 计费规则缓存、用户会话、支付幂等键、分布式锁、实时用量仪表盘 |
| **搜索引擎** | **Elasticsearch + RAG 管道** | 账单历史全文检索、审计日志查询；集成 RAG（检索增强生成）能力，支持自然语言查询账单、智能客服问答、知识库检索 |
| **对象存储** | **RustFS** | 高性能 Rust 实现的对象存储，兼容 S3 API；用于发票 PDF/CSV 文件存储、账单模板、合同文件归档、Airflow DAG 脚本存储；部署轻量，资源占用低，适合私有化部署场景 |

### 1.4 AI/ML 平台

| 层次 | 技术选型 | 理由 |
|------|---------|------|
| **模型训练** | **Python + PyTorch / TensorFlow** | 时序预测模型（LSTM、Transformer） |
| **特征工程** | **Feast (Feature Store)** | 管理历史用量、天气、节假日等特征 |
| **模型服务** | **MLflow + BentoML / TorchServe** | 模型版本管理、在线推理 API |
| **离线计算** | **Apache Spark / Flink** | 大规模账单批处理、碳足迹批量计算 |
| **RAG 引擎** | **LangChain + LlamaIndex + Elasticsearch** | 文档检索增强生成，支持自然语言账单查询、智能问答、运维知识库 |

### 1.5 DevOps & 基础设施

| 层次 | 技术选型 | 理由 |
|------|---------|------|
| **容器编排** | **Kubernetes (K8s) + Helm**（生产） / **Docker Compose**（开发） | 生产环境多活部署、弹性伸缩；开发环境一键启动全栈服务 |
| **CI/CD** | **GitHub Actions + ArgoCD** | GitOps 流水线，自动化构建、测试、部署、静态检查 |
| **监控** | **Prometheus + Grafana** | 服务指标、业务指标、KPI 仪表盘 |
| **日志** | **ELK (Elasticsearch + Logstash + Kibana) / Loki** | 聚合日志、审计追踪 |
| **链路追踪** | **SkyWalking / Jaeger** | 微服务调用链追踪、性能瓶颈定位 |
| **IaC** | **Terraform + Ansible** | 多云环境一致部署 |
| **静态检查** | **Checkstyle + PMD + SpotBugs**（Java）、**ESLint + Prettier**（前端）、**hadolint**（Dockerfile）、**markdownlint**（文档） | 代码质量门禁，PR 自动触发 |

---

## 二、架构设计

### 2.1 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         客户端层 (Client)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Web 门户  │  │ 移动 App  │  │ 企业 ERP  │  │ Provider │           │
│  │ (Next.js) │  │ (RN)     │  │ (SAP/金蝶)│  │ 系统对接  │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
└───────┼────────────┼──────────────┼──────────────┼─────────────────┘
        │            │              │              │
┌───────┴────────────┴──────────────┴──────────────┴─────────────────┐
│                       API 网关层 (Gateway)                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │     Spring Cloud Gateway: 鉴权/限流/路由/多语言/日志          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
        │
┌───────┴────────────────────────────────────────────────────────────┐
│                       业务服务层 (Microservices)                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ 用户服务  │ │ 账单服务  │ │ 计费引擎  │ │ 支付服务  │ │通知服务  │ │
│  │ (User)   │ │ (Billing)│ │ (Rating) │ │(Payment) │ │(Notify) │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ 企业服务  │ │Provider  │ │ 发票服务  │ │ 碳追踪   │ │能源市场  │ │
│  │(Enterprise│ │ 接入服务  │ │ (Invoice)│ │ (Carbon) │ │(Market) │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│  │ AI预测   │ │ 审批工作流 │ │ RAG 搜索  │                          │
│  │ (ML)    │ │(Workflow)│ │ (RAG)    │                          │
│  └──────────┘ └──────────┘ └──────────┘                          │
└────────────────────────────────────────────────────────────────────┘
        │
┌───────┴────────────────────────────────────────────────────────────┐
│                    调度与数据管道层 (Orchestration)                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Apache Airflow: DAG 编排 / 账单批量生成 / 数据同步 / ETL    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
        │
┌───────┴────────────────────────────────────────────────────────────┐
│                       基础设施层 (Infrastructure)                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ClickHouse│ │  Redis   │ │  Kafka   │ │  RustFS  │ │ Elastic │ │
│  │(OLAP)   │ │ Cluster  │ │          │ │  (S3 API)│ │ Search  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 服务拆分与职责

| 微服务 | 核心职责 | 关键模块 |
|--------|---------|---------|
| **user-service** | 用户注册、认证、角色权限、多租户隔离 | 支持 OAuth2/OIDC、SAML SSO、MFA、数据脱敏 |
| **provider-service** | 上游 Provider 管理、数据同步、接入适配 | 插件化 Provider Adapter（API/文件/数据库） |
| **rating-service** | 计费规则引擎（核心） | 阶梯价、峰谷、季节、动态价、PPA 合同价、DSL 脚本引擎 |
| **billing-service** | 账单生成、查询、状态管理、异常检测 | 批量账单生成、逾期/罚金计算、月结 |
| **payment-service** | 支付聚合、渠道管理、退款、分账、钱包 | 统一支付接口，插件化支付渠道（Stripe/Alipay/WeChat 等） |
| **notification-service** | 多渠道消息推送、模板管理、用户偏好 | 邮件/SMS/Push/微信/WhatsApp/Slack 等 |
| **invoice-service** | 发票生成、模板管理、税务合规 | 各地区发票模板（增值税/VAT/GST） |
| **enterprise-service** | 企业组织管理、成本中心、多账号 | 部门-项目分摊、审批流集成 |
| **workflow-service** | 可配置审批流引擎 | 基于 Camunda/Flowable，可视化流程设计 |
| **carbon-service** | 碳排放计算、绿证管理、碳报告 | 实时碳强度、REC/GEC 核销、ISO 14064/14067 报告 |
| **ai-prediction-service** | 能耗预测、费用预测 | ML 模型服务、特征工程管道 |
| **rag-search-service** | RAG 智能搜索与问答 | LangChain + LlamaIndex + Elasticsearch，自然语言账单查询、智能客服 |
| **market-service** | 能源交易市场、PPA 合同管理 | 撮合、合约生命周期、结算 |
| **microgrid-service** | 微电网/储能管理、调度优化 | 多能源切换、充放电策略 |

### 2.3 核心数据流

**账单生成主流程：**
```
Provider 同步用量数据 → Kafka (raw.usage) → billing-service
    → 调用 rating-service 计费计算
    → 写入 ClickHouse (Bill 记录)
    → Kafka (bill.generated)
    → notification-service 消费 → 推送用户
    → invoice-service 消费 → 生成发票 → RustFS 存储
    → carbon-service 消费 → 计算碳排放
```

**Airflow 批量调度流程：**
```
Airflow DAG: monthly_billing_pipeline
  1. check_provider_data_availability  → 检查上游数据就绪
  2. sync_usage_data                  → 同步用量数据到 ClickHouse
  3. batch_rating_calculation         → 批量计费计算
  4. generate_bills                   → 生成账单
  5. trigger_notifications            → 触发通知
  6. generate_carbon_reports           → 生成碳报告
  7. archive_to_rustfs               → 归档到 RustFS
```

### 2.4 插件化架构设计

```
┌──────────────────────────────────────┐
│         Plugin SDK / SPI             │
│  ┌────────────────────────────────┐  │
│  │  ProviderPlugin (Interface)     │  │
│  │  + syncUsers()                 │  │
│  │  + syncBills()                 │  │
│  │  + queryUsage()                │  │
│  │  + payBill()                   │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │  PaymentPlugin (Interface)      │  │
│  │  + createPayment()             │  │
│  │  + queryStatus()               │  │
│  │  + refund()                    │  │
│  │  + webhook()                   │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │  NotificationPlugin (Interface) │  │
│  │  + send()                      │  │
│  │  + validateTemplate()          │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

插件通过 **Java SPI 机制** 或 **Spring Bean 动态注册** 实现热加载，每个插件独立 JAR 包，存放在 `/plugins` 目录，支持运行时加载/卸载。

---

## 三、ClickHouse 数据库设计

### 3.1 设计原则

ClickHouse 作为列式 OLAP 数据库，设计上需遵循以下原则：

| 原则 | 说明 |
|------|------|
| **宽表优先** | 尽量将关联数据扁平化为宽表，避免 JOIN（ClickHouse 对 JOIN 支持有限） |
| **分区键选择** | 按时间分区（`toYYYYMM(billing_period_start)`），账单数据按月分区 |
| **排序键优化** | 按高频查询字段设置 ORDER BY，利用主键索引加速 |
| **物化视图** | 复杂聚合查询使用物化视图预计算，如月度汇总、Provider 统计 |
| **ReplacingMergeTree** | 使用 ReplacingMergeTree 引擎支持幂等写入（去重） |
| **分布式表** | 生产环境使用 Distributed + ReplicatedMergeTree 实现高可用 |

### 3.2 核心表结构

```sql
-- ==================== 租户与用户 ====================

CREATE TABLE tenant (
    id              String,
    name            String,
    country         LowCardinality(String),
    locale          LowCardinality(String),
    currency        LowCardinality(String),
    timezone        String,
    subscription_plan String,
    status          LowCardinality(String),
    created_at      DateTime64(3),
    updated_at      DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY id;

CREATE TABLE user_account (
    id              String,
    tenant_id       String,
    username        String,
    email           String,
    phone           String,
    password_hash   String,
    mfa_enabled     UInt8,
    locale          LowCardinality(String),
    status          LowCardinality(String),
    created_at      DateTime64(3),
    updated_at      DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, id);

-- ==================== Provider 与计量 ====================

CREATE TABLE provider (
    id              String,
    tenant_id       String,
    name            String,
    country         LowCardinality(String),
    type            LowCardinality(String),  -- ELECTRICITY/WATER/GAS/HEAT
    config          String,                   -- JSON: API URL, auth, format
    plugin_class    String,
    status          LowCardinality(String),
    created_at      DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, id);

CREATE TABLE meter (
    id              String,
    provider_id     String,
    tenant_id       String,
    meter_number    String,
    location        String,
    type            LowCardinality(String),
    associated_user_id String,
    install_date    Date,
    last_reading    Decimal(18,4)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, provider_id, id);

-- ==================== 用量记录（核心时序表）====================

CREATE TABLE consumption (
    time            DateTime64(3),
    meter_id        String,
    tenant_id       String,
    provider_id     String,
    consumption_value Decimal(18,4),
    unit            LowCardinality(String),  -- kWh/m³
    reading_type    LowCardinality(String),  -- AUTO/MANUAL/ESTIMATED
    date            Date DEFAULT toDate(time)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (tenant_id, meter_id, time)
TTL date + INTERVAL 3 YEAR DELETE
SETTINGS index_granularity = 8192;

-- 用量月度物化视图
CREATE MATERIALIZED VIEW consumption_monthly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(month)
ORDER BY (tenant_id, meter_id, month)
AS SELECT
    tenant_id,
    meter_id,
    toStartOfMonth(date) AS month,
    sum(consumption_value) AS total_consumption,
    max(consumption_value) AS peak_consumption,
    avg(consumption_value) AS avg_consumption
FROM consumption
GROUP BY tenant_id, meter_id, month;

-- ==================== 计费规则 ====================

CREATE TABLE rate_plan (
    id              String,
    provider_id     String,
    tenant_id       String,
    name            String,
    type            LowCardinality(String),  -- TIERED/TOU/SEASONAL/DYNAMIC/PPA/CUSTOM
    effective_from  Date,
    effective_to    Date,
    priority        Int32,
    status          LowCardinality(String),
    created_at      DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, provider_id, id);

CREATE TABLE rate_rule (
    id              String,
    rate_plan_id    String,
    tenant_id       String,
    condition       String,                   -- JSON: 条件表达式
    price           Decimal(18,6),
    unit            String,
    tax_included    UInt8,
    script          String,                   -- DSL 脚本 (CUSTOM 类型)
    sort_order      Int32
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, rate_plan_id, id);

-- ==================== 账单（宽表设计）====================

CREATE TABLE bill (
    id                      String,
    tenant_id               String,
    user_id                 String,
    provider_id             String,
    meter_id                String,
    billing_period_start    Date,
    billing_period_end      Date,
    consumption             Decimal(18,4),
    unit                    LowCardinality(String),
    subtotal_amount         Decimal(18,4),
    tax_amount              Decimal(18,4),
    total_amount            Decimal(18,4),
    currency                LowCardinality(String),
    status                  LowCardinality(String),  -- GENERATED/UNPAID/PAID/OVERDUE/CANCELLED
    due_date                Date,
    created_at              DateTime64(3),
    updated_at              DateTime64(3)
) ENGINE = ReplacingMergeTree(updated_at)
PARTITION BY toYYYYMM(billing_period_start)
ORDER BY (tenant_id, user_id, billing_period_start, id)
SETTINGS index_granularity = 8192;

-- 账单细项
CREATE TABLE bill_item (
    id              String,
    bill_id         String,
    tenant_id       String,
    rate_rule_id    String,
    description     String,
    consumption_portion Decimal(18,4),
    unit_price      Decimal(18,6),
    amount          Decimal(18,4)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, bill_id, id);

-- 账单汇总物化视图（按租户/月份）
CREATE MATERIALIZED VIEW bill_summary_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(month)
ORDER BY (tenant_id, provider_id, month)
AS SELECT
    tenant_id,
    provider_id,
    toStartOfMonth(billing_period_start) AS month,
    currency,
    count() AS bill_count,
    sum(total_amount) AS total_revenue,
    sum(consumption) AS total_consumption,
    countIf(status = 'OVERDUE') AS overdue_count
FROM bill
GROUP BY tenant_id, provider_id, month, currency;

-- ==================== 支付 ====================

CREATE TABLE payment (
    id                  String,
    tenant_id           String,
    bill_id             String,
    user_id             String,
    amount              Decimal(18,4),
    currency            LowCardinality(String),
    method              LowCardinality(String),  -- ALIPAY/WECHAT/STRIPE/...
    transaction_id      String,
    status              LowCardinality(String),
    idempotency_key     String,
    paid_at             DateTime64(3),
    refunded_at         Nullable(DateTime64(3)),
    created_at          DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, idempotency_key, id);

-- ==================== 发票 ====================

CREATE TABLE invoice (
    id              String,
    bill_id         String,
    tenant_id       String,
    invoice_number  String,
    invoice_type    LowCardinality(String),  -- VAT_SPECIAL/VAT_NORMAL/GST/TAX_RECEIPT
    amount          Decimal(18,4),
    tax_amount      Decimal(18,4),
    template_data   String,                   -- JSON
    file_url        String,                   -- RustFS 存储路径
    issued_at       DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, bill_id, id);

-- ==================== 企业与审批 ====================

CREATE TABLE organization (
    id              String,
    tenant_id       String,
    parent_id       String,
    name            String,
    type            LowCardinality(String),  -- COMPANY/DEPT/PROJECT
    cost_center_code String,
    path            String,                   -- 物化路径
    level           UInt16
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, id);

CREATE TABLE approval_workflow (
    id              String,
    tenant_id       String,
    bill_id         String,
    workflow_def_id String,
    current_step    String,
    status          LowCardinality(String),
    initiator_id    String,
    created_at      DateTime64(3),
    updated_at      DateTime64(3)
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (tenant_id, id);

CREATE TABLE approval_record (
    id              String,
    workflow_id     String,
    tenant_id       String,
    step_name       String,
    approver_id     String,
    action          LowCardinality(String),  -- APPROVE/REJECT/DELEGATE
    comment         String,
    timestamp       DateTime64(3)
) ENGINE = MergeTree()
ORDER BY (tenant_id, workflow_id, timestamp);

-- ==================== PPA 与能源市场 ====================

CREATE TABLE ppa_contract (
    id                  String,
    buyer_id            String,
    seller_id           String,
    tenant_id           String,
    power_amount_mw     Decimal(18,4),
    price_per_kwh       Decimal(18,6),
    currency            LowCardinality(String),
    start_date          Date,
    end_date            Date,
    minimum_take_or_pay Decimal(18,4),
    status              LowCardinality(String),  -- DRAFT/ACTIVE/EXPIRED/TERMINATED
    created_at          DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, id);

CREATE TABLE energy_order (
    id              String,
    type            LowCardinality(String),  -- BUY/SELL
    party_id        String,
    tenant_id       String,
    amount_mw       Decimal(18,4),
    price_per_kwh   Decimal(18,6),
    location        String,
    duration_months UInt16,
    status          LowCardinality(String),  -- OPEN/MATCHED/CLOSED
    created_at      DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, status, id);

-- ==================== 碳管理 ====================

CREATE TABLE carbon_record (
    id                      String,
    tenant_id               String,
    bill_id                 String,
    consumption_kwh         Decimal(18,4),
    grid_emission_factor    Decimal(18,6),
    total_kgco2e            Decimal(18,4),
    rec_coverage_kwh        Decimal(18,4),
    net_kgco2e              Decimal(18,4),
    calculation_method      LowCardinality(String),  -- MARKET_BASED/LOCATION_BASED
    period_start            Date,
    period_end              Date,
    created_at              DateTime64(3)
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(period_start)
ORDER BY (tenant_id, period_start, id);

CREATE TABLE rec_certificate (
    id                  String,
    tenant_id           String,
    certificate_number  String,
    source_project      String,
    mwh_amount          Decimal(18,4),
    issue_date          Date,
    expiry_date         Date,
    status              LowCardinality(String),  -- ACTIVE/RETIRED
    created_at          DateTime64(3)
) ENGINE = ReplacingMergeTree()
ORDER BY (tenant_id, id);

-- ==================== 审计日志 ====================

CREATE TABLE audit_log (
    id              String,
    tenant_id       String,
    user_id         String,
    action          String,
    resource_type   LowCardinality(String),
    resource_id     String,
    old_value       String,                   -- JSON
    new_value       String,                   -- JSON
    ip_address      String,
    user_agent      String,
    timestamp       DateTime64(3),
    date            Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (tenant_id, timestamp, resource_type)
TTL date + INTERVAL 5 YEAR DELETE;
```

### 3.3 ClickHouse 使用注意事项

| 场景 | 方案 |
|------|------|
| **高频更新/删除** | ClickHouse 不适合高频 UPDATE/DELETE。账单状态变更使用 ReplacingMergeTree + 版本字段，定期 OPTIMIZE 去重；或另建轻量状态表 |
| **事务一致性** | 支付场景使用 Redis 分布式锁 + 幂等键 + Kafka 事件溯源保障最终一致性 |
| **点查询优化** | 利用 ORDER BY 主键索引 + `query_id` 缓存，单行查询延迟 <10ms |
| **批量写入** | 每 1-5 秒合并写入一次（Buffer 表），单次插入 5000-10000 行 |
| **JOIN 限制** | 尽量避免大表 JOIN，优先使用字典表（Dictionary）或物化视图预 JOIN |

---

## 四、RustFS 对象存储集成

### 4.1 选型理由

RustFS 是 Rust 语言实现的高性能分布式对象存储，兼容 Amazon S3 API，具有以下优势：

- **轻量级**：单二进制部署，资源占用低，适合私有化部署
- **高性能**：Rust 实现，内存安全且接近 C 的性能
- **S3 兼容**：可直接使用 S3 SDK（Java AWS SDK / MinIO Client），无需修改代码
- **数据冗余**：支持纠删码（Erasure Coding），存储效率高

### 4.2 集成方式

```
Spring Boot 应用
    └── Spring Cloud AWS / MinIO Client
          └── S3 API 兼容层
                └── RustFS 集群
                      ├── 发票 PDF/CSV
                      ├── 账单导出文件
                      ├── 合同模板
                      ├── Airflow DAG 备份
                      └── 日志归档
```

### 4.3 配置示例

```yaml
# application.yml
rustfs:
  endpoint: http://rustfs-cluster:9000
  access-key: ${RUSTFS_ACCESS_KEY}
  secret-key: ${RUSTFS_SECRET_KEY}
  bucket-invoice: utilitypay-invoices
  bucket-contract: utilitypay-contracts
  bucket-archive: utilitypay-archive
```

---

## 五、RAG 智能搜索架构

### 5.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        RAG 搜索服务                               │
│                                                                   │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐     │
│  │  用户查询     │───▶│  Embedding   │───▶│ Elasticsearch    │     │
│  │ "我上月电费"  │    │  Model       │    │ 向量检索 (kNN)   │     │
│  └─────────────┘    │(text2vec)    │    └────────┬─────────┘     │
│                      └──────────────┘             │               │
│                                                   ▼               │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐     │
│  │  最终回答    │◀───│  LLM 生成    │◀───│  检索上下文       │     │
│  │ "上月电费   │    │ (ChatGLM/    │    │  + 原始问题       │     │
│  │  350元..."  │    │  Qwen)       │    │                  │     │
│  └─────────────┘    └──────────────┘    └──────────────────┘     │
│                                                                   │
│  知识库:                                                          │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  • 用户账单历史  • 产品文档  • FAQ  • 运维手册  • API 文档 │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 技术实现

| 组件 | 选型 | 职责 |
|------|------|------|
| **文档解析** | Unstructured / Apache Tika | 解析 PDF 账单、CSV、Markdown 文档 |
| **文本分割** | LangChain Text Splitters | 将文档按语义分割为 Chunk |
| **向量化模型** | text2vec-large-chinese / BGE-M3 | 中英文文本向量化 |
| **向量存储** | Elasticsearch 8.x (dense_vector) | 存储向量 + 原始文本，支持混合检索 |
| **LLM** | ChatGLM3-6B / Qwen2.5 (本地部署) 或 OpenAI API | 基于检索上下文生成回答 |
| **编排** | LangChain / LlamaIndex | RAG 管道编排、Prompt 模板管理 |

### 5.3 搜索场景

| 场景 | 示例查询 | RAG 输出 |
|------|---------|---------|
| **账单查询** | "我上个月电费多少钱？" | 检索用户账单记录，生成摘要回答 |
| **用量趋势** | "今年用电量比去年多多少？" | 聚合历史数据，生成趋势对比 |
| **智能客服** | "如何设置峰谷电价提醒？" | 检索产品文档，给出步骤指引 |
| **运维诊断** | "为什么支付失败？" | 检索支付日志 + 故障手册，定位原因 |

---

## 六、开发环境 (Docker Compose)

### 6.1 开发环境拓扑

```
┌──────────────────────────────────────────────────────────────┐
│                    Docker Compose 开发环境                      │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ user-svc │  │bill-svc  │  │rating-svc│  │payment   │     │
│  │  :8081   │  │  :8082   │  │  :8083   │  │  :8084   │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ClickHouse│  │  Redis   │  │  Kafka   │  │  RustFS  │     │
│  │  :8123   │  │  :6379   │  │  :9092   │  │  :9000   │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Elastic   │  │ Airflow  │  │Prometheus│  │ Grafana  │     │
│  │  :9200   │  │  :8080   │  │  :9090   │  │  :3000   │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 热重载机制

| 层面 | 方案 | 工具 |
|------|------|------|
| **Java 后端** | Spring Boot DevTools + 自动重启 | `spring-boot-devtools`，修改代码后自动重启（<5s） |
| **前端 React** | Vite/Webpack HMR | 修改代码后浏览器即时刷新，无需手动刷新 |
| **Next.js** | Fast Refresh | 保留组件状态的增量热更新 |
| **Docker Compose** | Volume 挂载 + Watch 模式 | 源代码目录挂载到容器，`docker compose watch` 自动同步 |
| **配置热更新** | Spring Cloud Config + Nacos | 配置中心修改后自动推送到服务实例 |

### 6.3 开发环境启动流程

```bash
# 1. 克隆项目
git clone https://github.com/LisaHusband/UtilityPay.git
cd UtilityPay

# 2. 复制环境配置
cp .env.example .env

# 3. 一键启动基础设施（数据库、缓存、消息队列、存储）
docker compose up -d clickhouse redis kafka rustfs elasticsearch airflow

# 4. 初始化数据库
docker compose run --rm db-migrator

# 5. 启动后端服务（热重载模式）
docker compose up -d --watch

# 6. 启动前端开发服务器
cd frontend && pnpm dev

# 7. 访问
# - Web 门户: http://localhost:3000
# - 管理后台: http://localhost:3001
# - API 文档 (Swagger): http://localhost:8080/swagger-ui.html
# - Airflow: http://localhost:8080 (airflow/airflow)
# - Grafana: http://localhost:3000 (admin/admin)
# - RustFS Console: http://localhost:9001
# - ClickHouse HTTP: http://localhost:8123/play
```

---

## 七、安全与合规架构

| 层面 | 方案 |
|------|------|
| **传输安全** | TLS 1.3 全链路加密，mTLS 服务间通信 |
| **认证** | OAuth2 + OIDC，支持 SAML 企业 SSO、LDAP |
| **授权** | RBAC + ABAC 细粒度权限（Spring Security） |
| **数据脱敏** | 敏感字段 AES-256 加密存储至 ClickHouse String 列，日志自动脱敏 |
| **API 安全** | 速率限制、IP 白名单、Webhook 签名验证 |
| **PCI-DSS** | 支付数据不落库，Token 化处理 |
| **GDPR/PIPL** | 数据留存策略自动执行（ClickHouse TTL）、用户数据导出/删除 API |
| **密钥管理** | HashiCorp Vault / K8s Secrets |

---

## 八、开发路线规划

### 阶段 1：基础核心（2026 下半年，约 6 个月）

| 里程碑 | 内容 | 产出 |
|--------|------|------|
| **M1.1 项目基础设施** | 搭建 Monorepo、CI/CD（含静态检查）、Docker Compose 开发环境、数据库初始化 | 可部署的空平台骨架 |
| **M1.2 用户系统** | 注册/登录/OAuth2/MFA、多租户基础、RBAC 权限 | user-service + 前端登录页 |
| **M1.3 Provider 管理** | Provider CRUD、插件接口定义、首个 Provider 适配器（模拟） | provider-service + Plugin SDK v1 |
| **M1.4 计费引擎** | 规则配置 API、5 种计费类型实现、DSL 脚本引擎 | rating-service 核心 |
| **M1.5 账单系统** | 账单生成、查询、状态流转、PDF 导出 | billing-service |
| **M1.6 通知与支付** | 通知中心（邮件/SMS）、支付聚合（Stripe + 模拟渠道） | notification-service + payment-service |
| **M1.7 Airflow 集成** | DAG 管道开发、账单月结调度、数据同步任务 | Airflow DAG 集合 |

### 阶段 2：企业功能（2027 年，约 6 个月）

| 里程碑 | 内容 | 产出 |
|--------|------|------|
| **M2.1 企业管理** | 组织架构、成本中心、多账号管理 | enterprise-service |
| **M2.2 审批工作流** | 可配置审批流引擎、审批记录审计 | workflow-service |
| **M2.3 发票系统** | 多地区发票模板、自动开票、ERP 对接、RustFS 存储 | invoice-service |
| **M2.4 AI 预测** | 数据管道、模型训练、预测 API | ai-prediction-service |
| **M2.5 PPA & 专线** | 合同管理、专线监控、峰谷调度 | PPA 模块 |
| **M2.6 RAG 搜索** | 知识库构建、向量化、LLM 集成、自然语言查询 | rag-search-service |

### 阶段 3：全球扩展（2028 年，约 9 个月）

| 里程碑 | 内容 | 产出 |
|--------|------|------|
| **M3.1 多地区 Provider** | 中国/欧洲/美洲/澳洲 Provider 适配器 | 10+ Provider 插件 |
| **M3.2 碳管理** | 排放计算、绿证管理、ISO 报告 | carbon-service |
| **M3.3 能源市场** | 交易撮合、合约管理、结算 | market-service |
| **M3.4 微电网** | 储能管理、调度优化 | microgrid-service |
| **M3.5 生态建设** | 文档完善、社区运营、插件市场 | 开发者门户 |

---

## 九、关键架构决策记录 (ADR)

| # | 决策 | 理由 | 权衡 |
|---|------|------|------|
| ADR-1 | 后端语言选 Java 而非 Go/Node.js | 企业级生态成熟，Spring 生态覆盖安全、数据、集成全场景；计费引擎需要强类型和复杂事务 | 开发速度略慢于 Node.js，资源占用高于 Go |
| ADR-2 | 微服务而非单体 | 各模块独立迭代、独立部署、独立扩展；支付/计费需要高可用隔离 | 运维复杂度增加，分布式事务需 Saga 模式 |
| ADR-3 | 主数据库选 ClickHouse 而非 PostgreSQL | 账单/用量数据天然 OLAP 场景，列式存储压缩比高，百亿级数据查询秒级响应；时序分析能力远超传统行式数据库 | 不支持高频 UPDATE/DELETE，需额外设计状态变更机制；事务支持弱，需幂等+Saga 补偿 |
| ADR-4 | 对象存储选 RustFS 而非 MinIO | 性能更高（Rust 实现），资源占用更低，Apache 2.0 许可；S3 兼容 API 生态完整 | 社区规模小于 MinIO，中文文档较少 |
| ADR-5 | 定时任务选 Apache Airflow 而非 Quartz | DAG 可视化编排，任务依赖清晰；内置重试/告警/监控；Python 生态可联动 AI 训练管道；Web UI 管理方便 | 部署运维成本高于 Quartz 嵌入式方案，需独立 Airflow 集群 |
| ADR-6 | 搜索集成 RAG 管道 | 自然语言查询体验远超传统全文搜索；LLM 理解用户意图，直接返回答案而非链接列表；可与运维知识库结合 | LLM 推理延迟（1-3s），需 GPU 资源；RAG 检索质量依赖知识库构建 |
| ADR-7 | 插件化通过 Java SPI 而非 OSGi | 轻量、标准 JDK 机制、Spring 原生支持 | 不支持运行时完整热部署（需重启加载新插件） |
| ADR-8 | Kafka 作为消息骨干 | 高吞吐、持久化、重放能力，适合账单生成和审计 | 运维成本高于 RabbitMQ |

---

## 十、下一步行动建议

1. **搭建项目骨架**：创建 Monorepo 结构，配置 CI/CD 流水线（含静态检查）
2. **搭建开发环境**：Docker Compose 一键启动 ClickHouse + Redis + Kafka + RustFS + Airflow
3. **核心服务原型**：优先实现 user-service 和 provider-service 的 MVP
4. **Plugin SDK 定义**：产出 Provider 和 Payment 插件的接口规范
5. **计费引擎 POC**：验证 DSL 计费规则引擎的可行性与性能
6. **RAG 知识库搭建**：整理产品文档、FAQ，构建向量索引

---

*本文档基于 UtilityPay PRD（deep-research-report.md）编写，技术选型在开发过程中可根据实际情况调整。*