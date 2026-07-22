# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.1

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-07-22 | 初始架构设计 |
| V2.0 | 2026-07-23 | 拆分认证/数据库/RBAC/Quota 详细设计 |
| V2.1 | 2026-07-23 | 架构评审修订：Token 安全、MailDelivery 状态、Batch Lease、邮件 Snapshot、健康检测、SPF、DKIM 等 |

---

# 1. 项目目标

(同 V1.0，略)

---

# 2. 总体架构

## 2.1 架构图

```
                         Oracle VPS (Python/FastAPI)
                     ─────────────────────────────
                         Control Plane
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   Auth/Users            Mail Scheduler         Runner Manager
        │                     │                     │
        │                     │               ┌─────┴─────┐
        │                     │               │           │
        │                     │          Proxy Runner  Mail Runner
        │                     │          (frpc/frps)   (Go Agent)
        │                     │               │           │
        │                     │               │      SMTP Direct
        │                     │               │           │
        │                     │               │           ▼
        │                     │               │      Internet MX
        │                     └───── Task / Batch / Delivery
        │
        └───── Quota / RBAC / Audit
```

## 2.2 核心设计原则

1. **Oracle 是 Control Plane，Mail Runner 是 Data Plane**
2. **SMTP 流量不经过 Oracle**
3. **每种凭证拥有最小权限，严格隔离**
4. **User Token ≠ Runner Token ≠ Admin Token ≠ GitHub PAT ≠ Cloudflare Token**
5. **Cloudflare / GitHub 凭证只存在 Oracle，不下发**
6. **status 和 health_status 独立判断**
7. **Runner 健康基于窗口数据 + 区分收件人失败和 SMTP 拒绝**
8. **Quota 使用 RESERVE → CONSUME/RELEASE 事务模型**
9. **每个 Task+Recipient 唯一约束，避免重复投递**
10. **已 SMTP 尝试的地址有明确状态机，不自动重试**
11. **Batch 使用 Lease 机制实现超时回收**
12. **邮件内容在任务创建时 Snapshot，不可变**
13. **所有投递结果持久化，支持导出**

## 2.3 代码架构：Bounded Context

Proxy Manager 和 Mail Relay 共用同一 FastAPI Control Plane，但逻辑上分为两个 Bounded Context：

```
/opt/proxy-manager/
├── main.py              # FastAPI 入口，公共组件
├── config.yaml          # 非敏感配置
├── .env                 # 敏感配置（权限 chmod 600）
│
├── proxy/               # Proxy Context
│   ├── api.py
│   ├── models.py
│   ├── scheduler/
│   └── templates/
│
├── mail/                # Mail Relay Context
│   ├── api/
│   ├── models/
│   ├── scheduler/
│   ├── cloudflare/
│   └── websocket/
│
├── auth/                # 共享认证
├── database/
└── templates/
```

---

# 3. 技术栈

(同 V2.0)

---

# 4. 认证与安全设计（修订）

## 4.1 凭证隔离原则

```
普通用户 Session
        ≠
管理员 Session
        ≠
用户 API Token
        ≠
Mail Runner Token
        ≠
Bootstrap Token（一次性）
        ≠
GitHub PAT
        ≠
Cloudflare API Token
```

**每一种凭证只拥有完成自己工作的最小权限。**

## 4.2 Runner Token 安全下发（V2.1 关键修订）

### ❌ 错误方案（V2.0）

```
Oracle → GitHub API → Workflow inputs → Agent（长期 Token 暴露在多个环节）
```

### ✅ 正确方案（V2.1）

```
Oracle
  │
  │ 触发 Workflow，只携带一次性 Bootstrap Token
  ▼
GitHub Actions
  │
  │ 环境变量 BOOTSTRAP_TOKEN
  ▼
Mail Runner Agent
  │
  │ POST /api/mail-runners/bootstrap
  │ { "bootstrap_token": "...", "runner_id": "...", "public_ip": "..." }
  ▼
Oracle
  │
  │ 验证 Bootstrap Token → 生成长期 Runner Token
  │ Bootstrap Token 立即失效
  ▼
Agent 获得 Runner Token
```

### BootstrapToken 模型

```python
class RunnerBootstrapToken(Base):
    __tablename__ = "runner_bootstrap_tokens"

    id: UUID
    token_hash: str
    runner_id: UUID
    expires_at: datetime
    used_at: datetime | None
    created_at: datetime
```

### 安全优势

| 风险点 | 旧方案 | 新方案 |
|--------|--------|--------|
| GitHub 日志泄露 | 长期 Token 可能被打印 | Bootstrap Token 一次性，即使泄露也很快过期 |
| Workflow 元数据 | 长期 Token 可执行任何 Runner API | Bootstrap Token 仅能注册 |
| Runner 环境变量泄露 | Token 长期有效 | 泄露后可在 Oracle 吊销 |
| 重放攻击 | — | Bootstrap Token 用后即失效 |

## 4.3 认证架构

```
                         Oracle API
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
     User Web            Admin Web          Mail Runner
         │                   │                   │
   Session Cookie      Session Cookie       Runner Token
         │                   │                   │
         ▼                   ▼                   ▼
    User Role            Admin Role        Runner Identity
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                       Authorization
                             │
                             ▼
                         API Layer
```

## 4.4 Web 认证方案

采用 **Server-side Session**（非 JWT）：

```
Cookie(session_id) → Database → User Identity
```

管理员停用用户 → `User.status = DISABLED` → 立即拒绝所有 Session。

## 4.5 敏感凭证存储（V2.1 修订）

❌ 旧方案：GitHub PAT / Cloudflare Token 放在 `config.yaml`

✅ 新方案：使用环境变量 `/opt/proxy-manager/.env`

```bash
# .env (chmod 600)
GITHUB_TOKEN=ghp_xxx
CLOUDFLARE_TOKEN=cfut_xxx
CLOUDFLARE_ZONE_ID=xxx
```

Oracle 启动时加载 `.env`，不提交到版本控制。

GitHub Token 和 Cloudflare Token 使用不同 Token，各自拥有最小权限：

- GitHub Token → 仅允许 workflow dispatch / workflow management
- Cloudflare Token → 仅允许指定 Zone DNS 编辑

---

# 5. 数据库模型设计（修订）

## 5.1 模型总览

| 领域 | 模型 | 说明 |
|------|------|------|
| User | User, UserApiToken | 用户与 API Token |
| Admin | Role, Permission | RBAC 权限 |
| Quota | QuotaAccount, QuotaTransaction | 额度管理 |
| MailTask | MailTask, MailTemplateSnapshot | 邮件任务与内容快照 |
| MailBatch | MailBatch, RunnerBatchAssignment | 批次与并行分配 |
| MailDelivery | MailDelivery | 投递结果（唯一约束） |
| MailRunner | MailRunner, RunnerBootstrapToken, RunnerCredential, RunnerHealth | Runner 管理 |
| Identity | SenderIdentity | 发件身份验证 |

辅助模型：AuditLog, Session

## 5.2 User 模型

```python
class User(Base):
    __tablename__ = "users"

    id: UUID
    username: str              # unique
    email: str                 # unique
    password_hash: str

    status: str                # ACTIVE / DISABLED / SUSPENDED
    role: str                  # USER / ADMIN

    created_at: datetime
    updated_at: datetime
    last_login_at: datetime | None
```

## 5.3 MailRunner 模型（V2.1 修订）

```python
class MailRunner(Base):
    __tablename__ = "mail_runners"

    id: UUID
    runner_id: str             # 唯一标识
    name: str
    runner_type: str           # "mail"

    status: str                # STARTING / REGISTERING / ... / OFFLINE
    health_status: str         # UNKNOWN / HEALTHY / SUSPECTED / UNHEALTHY

    public_ip: str
    ip_version: int

    operating_system: str
    architecture: str
    cpu_count: int | None
    memory_mb: int | None

    agent_version: str

    github_runner_name: str
    workflow_run_id: str | None
    workflow_job_id: str | None

    registered_at: datetime
    heartbeat_at: datetime
    ready_at: datetime | None
    stopped_at: datetime | None

    # 注意：V2.1 不再使用 current_batch_id 表示唯一 Batch
    # 改为支持多个并行 Batch

    smtp_concurrency: int
    report_mode: str
    report_batch_size: int
    report_interval_seconds: int

    created_at: datetime
    updated_at: datetime
```

### RunnerBatchAssignment（V2.1 新增）

支持一个 Runner 同时处理多个 Batch：

```python
class RunnerBatchAssignment(Base):
    __tablename__ = "runner_batch_assignments"

    id: UUID
    runner_id: UUID
    batch_id: UUID
    assigned_at: datetime
    released_at: datetime | None
    status: str                  # ACTIVE / RELEASED
```

## 5.4 Runner 状态

```python
class RunnerStatus:
    STARTING = "starting"
    REGISTERING = "registering"
    VERIFYING = "verifying"
    READY = "ready"
    BUSY = "busy"
    DRAINING = "draining"
    SUSPENDED = "suspended"
    STOPPING = "stopping"
    OFFLINE = "offline"
```

health_status 独立：

```python
class RunnerHealthStatus:
    UNKNOWN = "unknown"
    HEALTHY = "healthy"
    SUSPECTED = "suspected"
    UNHEALTHY = "unhealthy"
```

例如：`status = BUSY, health_status = UNHEALTHY` → 立即停止接收新 Batch。

## 5.5 RunnerHealth（窗口数据，V2.1 修订）

**V2.1 重点修订：区分收件人错误和 SMTP 拒绝**

```python
class RunnerHealth(Base):
    __tablename__ = "runner_health"

    id: UUID
    runner_id: UUID

    window_start: datetime
    window_end: datetime

    attempted_count: int
    success_count: int
    failure_count: int

    # 收件人错误（不影响 Runner 健康评分）
    mailbox_not_found_count: int     # 550 5.1.1
    domain_not_found_count: int      # 550 5.4.4

    # SMTP 拒绝（降低 Runner 健康评分）
    smtp_rejected_count: int         # 550 5.7.1 策略拒绝
    rate_limited_count: int          # 450/429 频率限制

    # 网络错误（降低 Runner 健康评分）
    connection_timeout_count: int
    connection_refused_count: int
    tls_error_count: int

    # 汇总
    recipient_error_count: int       # 不影响 Runner 健康
    smtp_error_count: int            # 降低 Runner 健康
    network_error_count: int         # 严重降低 Runner 健康

    created_at: datetime
```

### 健康评分引擎（V2.1 统一规则）

```
sample < 20             → UNKNOWN

SMTP 错误率 < 10%       → HEALTHY
10% ≤ SMTP 错误率 < 50% → SUSPECTED
SMTP 错误率 ≥ 50%       → UNHEALTHY

其中 SMTP 错误 = smtp_error_count + network_error_count
收件人错误（recipient_error_count）不参与健康评分
```

## 5.6 MailTask 模型（V2.1 修订）

**增加内容快照字段，确保任务创建后邮件内容不可变：**

```python
class MailTask(Base):
    __tablename__ = "mail_tasks"

    id: UUID
    user_id: UUID

    status: str                    # PENDING / PROCESSING / COMPLETED / FAILED / CANCELLED

    idempotency_key: str | None    # 幂等键, unique(user_id, idempotency_key)

    # 邮件内容快照（不可变）
    subject_snapshot: str | None
    from_snapshot: str
    headers_snapshot: str | None   # JSON
    body_snapshot: str             # HTML 或纯文本
    attachment_keys: str | None    # JSON 数组，对象存储 Key

    # 模板引用（仅用于记录来源）
    template_type: str
    template_storage_key: str | None

    total_recipients: int

    attempted_count: int
    delivered_count: int
    failed_count: int
    not_attempted_count: int

    reserved_quota: int
    consumed_quota: int

    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    cancelled_at: datetime | None
```

## 5.7 MailBatch 模型（V2.1 修订）

**增加 lease_expires_at 实现超时回收：**

```python
class MailBatch(Base):
    __tablename__ = "mail_batches"

    id: UUID
    task_id: UUID
    runner_id: UUID | None

    status: str                    # QUEUED / ASSIGNED / CLAIMED / RUNNING /
                                   # COMPLETED / CANCELLING / CANCELLED /
                                   # FAILED / RECOVERING

    total_recipients: int

    attempted_count: int
    delivered_count: int
    failed_count: int
    not_attempted_count: int

    claimed_at: datetime | None
    lease_expires_at: datetime | None  # V2.1 新增
    started_at: datetime | None
    completed_at: datetime | None

    retry_count: int

    created_at: datetime
```

### Lease 机制

```
Runner 认领 Batch
    → lease_expires_at = now + 60s
    → 每次 Heartbeat / Progress 上报
        → lease_expires_at += 60s
    → 如果 lease 过期
        → Batch = RECOVERING
        → 重新调度
```

## 5.8 MailDelivery 模型（V2.1 关键修订）

**V2.1 最重要的修订之一：增加 ATTEMPTING 状态和唯一约束**

```python
class MailDelivery(Base):
    __tablename__ = "mail_deliveries"

    id: UUID
    task_id: UUID
    batch_id: UUID
    runner_id: UUID | None

    recipient: str

    status: str                    # NOT_ATTEMPTED / ATTEMPTING /
                                   # DELIVERED / PERMANENT_FAILURE /
                                   # TEMPORARY_FAILURE / UNKNOWN

    smtp_code: int | None
    smtp_enhanced_code: str | None
    smtp_response: str | None

    error_type: str | None         # MAILBOX_NOT_FOUND / DOMAIN_NOT_FOUND /
                                   # RECIPIENT_REJECTED / RATE_LIMITED /
                                   # CONNECTION_TIMEOUT / TLS_ERROR /
                                   # CANCELLED

    attempts: int                  # SMTP 尝试次数

    first_attempt_at: datetime | None
    last_attempt_at: datetime | None
    completed_at: datetime | None

    __table_args__ = (
        UniqueConstraint('task_id', 'recipient', name='uq_task_recipient'),
    )
```

### 投递状态机

```
NOT_ATTEMPTED
    │
    ▼
ATTEMPTING      ← Agent 开始 SMTP 投递
    │
    ├── DELIVERED           ← SMTP 250
    ├── PERMANENT_FAILURE   ← SMTP 5xx（永久失败）
    ├── TEMPORARY_FAILURE   ← SMTP 4xx（临时失败）
    ├── UNKNOWN             ← Agent 崩溃，状态不确定
    └── CANCELLED           ← 用户取消
```

**UNKNOWN 处理策略：**

Agent 在 SMTP 操作过程中崩溃 → status = ATTEMPTING → 进入 UNKNOWN。

不由系统自动重试，由管理员策略决定：

- 保守策略：UNKNOWN 不重试
- 激进策略：UNKNOWN 允许重试

### 计费定义（V2.1 明确）

**Billing Attempt = Mail Runner 已开始针对某个 recipient 执行 SMTP 投递流程**

```
Recipient
    ↓
建立 SMTP Session
    ↓
MAIL FROM
    ↓
RCPT TO
    ↓
DATA
```

只要进入 SMTP 投递流程，attempted += 1，即使 TCP 连接失败也算一次尝试。

每个 recipient 每次 attempt 消耗 1 quota。

```
Attempt 1 = 1 quota
Attempt 2 = 1 quota
Total = 2 quota（如果允许重试）
```

## 5.9 Quota 设计

### QuotaAccount

```python
class QuotaAccount(Base):
    __tablename__ = "quota_accounts"

    user_id: UUID
    quota_limit: int
    available: int
    reserved: int
    consumed: int
```

### QuotaTransaction

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"

    id: UUID
    user_id: UUID
    task_id: UUID | None
    delivery_id: UUID | None       # 关联到具体投递

    transaction_type: str          # RESERVE / CONSUME / RELEASE / ADJUST / REFUND
    amount: int

    created_at: datetime
```

### 流程

```
用户提交 1000 个收件人
    → RESERVE +1000

实际 SMTP 尝试 900 次
    → 每次 attempt 关联一条 CONSUME
    → 最终 900 CONSUME + 100 RELEASE
```

## 5.10 SenderIdentity（V2.1 新增）

```python
class SenderIdentity(Base):
    __tablename__ = "sender_identities"

    id: UUID
    user_id: UUID
    email: str
    domain: str
    status: str                    # PENDING / VERIFIED / REVOKED
    verified_at: datetime | None
    created_at: datetime
```

用户只能使用已验证的 Sender Identity 作为发件地址。

## 5.11 UserApiToken

```python
class UserApiToken(Base):
    __tablename__ = "user_api_tokens"

    id: UUID
    user_id: UUID
    token_hash: str
    name: str
    permissions: str               # JSON: ["mail:send", "mail:read"]
    created_at: datetime
    expires_at: datetime | None
    last_used_at: datetime | None
    revoked_at: datetime | None
```

---

# 6. SPF 管理（V2.1 修订）

## 6.1 引用计数 + Grace Period

```python
class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"

    ip: str
    runner_count: int              # 引用计数
    active: bool
    first_seen_at: datetime
    last_seen_at: datetime
    remove_after: datetime | None  # Grace Period 到期时间
```

### 删除策略

```
Runner OFFLINE
    → runner_count -= 1
    → 如果 runner_count = 0
        → 进入 Grace Period（30~60 分钟）
        → 期间如果有新 Runner 使用该 IP → 取消删除
        → Grace Period 到期仍然无人使用 → 从 SPF 删除
```

### DNS TTL 保护

由于 DNS 缓存，删除 SPF 后外部解析器可能仍缓存旧记录。

Grace Period 缓解此问题。

## 6.2 长期架构建议

如果 Runner 数量较多，SPF 可能超出 DNS TXT 长度和 lookup 限制：

```
主域名 SPF:
v=spf1 include:spf.yourdomain.com ~all

子域名 SPF (不频繁修改):
spf.yourdomain.com TXT "v=spf1 ip4:RunnerA ip4:RunnerB ... ~all"
```

---

# 7. Mail Runner Agent 设计（Go）

## 7.1 启动流程（V2.1 修订）

```
GitHub Runner 启动
    → 下载/启动 Mail Runner Agent
    → Agent 获取公网 IP
    → Agent 使用 BOOTSTRAP_TOKEN 向 Oracle 注册
    → Oracle 验证 Bootstrap Token
    → Oracle 返回长期 Runner Token
    → Bootstrap Token 立即失效
    → Oracle 保存 IP，更新 SPF
    → Agent 开始发送 Heartbeat
    → Runner 状态 = READY
```

## 7.2 Agent 职责

- 获取公网 IP
- 使用 Bootstrap Token 注册
- 发送 Heartbeat（每 30 秒）
- 建立 WebSocket 控制通道
- 获取任务 / 认领 Batch
- 下载邮件内容快照（Oracle Presigned URL）
- SMTP 投递（Go goroutine 并发）
- 上报进度（每 50 封或每 5 秒）
- 批量上报投递结果
- 处理 CANCEL / PAUSE / CONFIG_UPDATE
- 断线恢复（Grace Period 30s）

## 7.3 Agent 不负责

- 用户管理
- 额度管理
- 任务创建
- Runner 生命周期管理
- SPF 管理 / Cloudflare API
- 其他 Runner 管理

## 7.4 通信方式

```
HTTPS API:
    注册 (bootstrap)
    心跳
    任务 Claim
    进度
    结果
    完成

WebSocket:
    CANCEL
    PAUSE
    RESUME
    CONFIG_UPDATE
    STOP
```

## 7.5 心跳与 Lease

```
POST /api/mail-runners/{runner_id}/heartbeat
{
  "runner_id": "runner-001",
  "status": "BUSY",
  "public_ip": "1.2.3.4",
  "current_batches": ["batch-001", "batch-002"]
}
```

- Heartbeat：30 秒
- Offline 判定：90 秒无心跳
- Heartbeat 同时刷新所有关联 Batch 的 lease_expires_at

---

# 8. Mail Scheduler

## 8.1 调度逻辑

```
用户创建 MailTask（10000 收件人）
    → 预占额度
    → 拆分为 MailBatch（batch_size = 500）
    → Batch 001~020 进入 QUEUED

Scheduler 循环：
    → 寻找 READY / BUSY 但 smtp_concurrency 未满的 Runner
    → 分配 QUEUED 的 Batch
    → Runner 认领后进入 CLAIMED
    → lease_expires_at = now + 60s
```

## 8.2 Lease 回收

```
每 30 秒扫描：
    → lease_expires_at < now
    → Batch.status = RECOVERING
    → 重新调度
```

## 8.3 进度上报

| 模式 | 说明 |
|------|------|
| PER_MESSAGE | 每发送一封即上报 |
| BATCH | 每 N 封上报一次 |
| INTERVAL | 每 N 秒上报一次 |

支持组合规则：达到 50 封 **或** 超过 5 秒 → 立即上报。

## 8.4 动态进度策略

Oracle 可动态修改：

```
正常: batch_size = 50, interval = 5s
异常: batch_size = 1, interval = 1s（逐封上报）
```

---

# 9. Mail Task 与幂等性（V2.1 新增）

## 9.1 Idempotency-Key

用户发送请求时携带：

```
Idempotency-Key: uuid-v4
```

数据库约束：

```
UNIQUE(user_id, idempotency_key)
```

相同 Key 重复请求 → 返回已有 Task，不创建新 Task。

## 9.2 邮件内容快照

任务创建时，邮件模板立即创建不可变快照：

```
Task Created
    ↓
Snapshot subject / from / headers / body / attachments
    ↓
存储在 MailTask 字段中
    ↓
Runner 始终读取 Snapshot，而非原始模板
```

## 9.3 附件存储

使用对象存储：

```
/objects/mail-tasks/{task_id}/
    attachments/
        file1.pdf
        file2.jpg
```

Oracle 返回 Presigned URL：

```
GET /api/tasks/{task_id}/attachment/{name}
→ 302 Redirect to Presigned URL
```

Runner：

```
Oracle API → Presigned URL → 直接下载附件
```

邮件内容不经过 Oracle API，避免网络瓶颈。

---

# 10. Runner 生命周期

## 10.1 状态机

```
STARTING → REGISTERING → VERIFYING → READY ↔ BUSY
                                          │
                                          ├→ DRAINING → STOPPING → OFFLINE
                                          ├→ SUSPENDED
                                          └→ HEARTBEAT_TIMEOUT → OFFLINE
```

## 10.2 健康检测（V2.1 统一规则）

```python
def calculate_health(runner_health):
    if runner_health.attempted_count < 20:
        return "UNKNOWN"

    # 只计算 SMTP 错误和网络错误
    smtp_errors = runner_health.smtp_rejected_count + runner_health.rate_limited_count
    network_errors = runner_health.connection_timeout_count + runner_health.connection_refused_count
    total_smtp_events = smtp_errors + network_errors + runner_health.success_count

    if total_smtp_events == 0:
        return "UNKNOWN"

    error_rate = (smtp_errors + network_errors) / total_smtp_events

    if error_rate < 0.1:
        return "HEALTHY"
    elif error_rate < 0.5:
        return "SUSPECTED"
    else:
        return "UNHEALTHY"
```

**收件人错误（mailbox_not_found / domain_not_found）不降低 Runner 健康评分。**

## 10.3 Workflow 生命周期（V2.1 新增）

```
Oracle Scheduler
    ↓
触发 Workflow
    ↓
记录 workflow_run_id + workflow_job_id
    ↓
轮询 GitHub API
    ↓
Workflow completed
    ↓
检查 Runner 状态：
    ├── Runner = OFFLINE → 正常
    └── Runner = BUSY    → 异常
```

---

# 11. API 设计

## 11.1 用户 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /auth/register | 注册 |
| POST | /auth/login | 登录 |
| POST | /auth/logout | 登出 |
| GET  | /auth/me | 当前用户 |
| PUT  | /auth/password | 修改密码 |

## 11.2 任务 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/tasks | 创建任务（支持 Idempotency-Key） |
| GET  | /api/tasks | 任务列表 |
| GET  | /api/tasks/{id} | 任务详情 |
| POST | /api/tasks/{id}/cancel | 取消任务 |
| GET  | /api/tasks/{id}/results | 下载结果 CSV |
| GET  | /api/tasks/{id}/failures | 下载失败列表 |
| GET  | /api/tasks/{id}/attachment/{name} | 附件下载（Presigned URL） |

## 11.3 Runner API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/mail-runners/bootstrap | 使用 Bootstrap Token 注册 |
| POST | /api/mail-runners/{id}/heartbeat | 心跳（刷新 Lease） |
| GET  | /api/mail-runners/{id}/config | 获取配置 |
| GET  | /api/mail-runners/{id}/jobs/next | 领取下一个 Batch |
| POST | /api/mail-batches/{id}/claim | 认领 Batch |
| POST | /api/mail-batches/{id}/progress | 上报进度 |
| POST | /api/mail-batches/{id}/results | 批量上报投递结果 |
| POST | /api/mail-batches/{id}/complete | 完成 Batch |

## 11.4 管理员 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET  | /admin/users | 用户列表 |
| PUT  | /admin/users/{id}/quota | 设置额度 |
| PUT  | /admin/users/{id}/status | 停用/恢复 |
| GET  | /admin/runners | Runner 列表 |
| POST | /admin/runners/{id}/pause | 暂停 |
| POST | /admin/runners/{id}/resume | 恢复 |
| POST | /admin/runners/{id}/stop | 停止 |
| GET  | /admin/tasks | 所有任务 |
| POST | /admin/tasks/{id}/cancel | 取消任务 |

---

# 12. 故障转移

```
Runner A 突然离线
    → Oracle 检测 Heartbeat 超时（90s）
    → Runner A = OFFLINE
    → 所有关联 Batch 的 lease 过期
    → Batch = RECOVERING
    → 根据已上报的 delivery 状态判断：
        - ATTEMPTING → UNKNOWN（不确定是否已发送）
        - NOT_ATTEMPTED → 重新进入 QUEUED
        - DELIVERED / FAILED → 不处理
    → 重新调度到 Runner B / C
```

---

# 13. 用户取消任务

```
用户点击取消
    → MailTask = CANCELLING
    → WebSocket 下发 CANCEL 到所有关联 Runner
    → Agent 停止 SMTP 操作
    → 更新所有 ATTEMPTING → CANCELLED
    → 更新所有 NOT_ATTEMPTED → CANCELLED
    → 上报最终进度
    → MailTask = CANCELLED
    → 释放未尝试收件人的预占额度
```

---

# 14. Token 与凭证总结（V2.1 修订）

| 凭证类型 | 用途 | 存储位置 | 可否撤销 | 备注 |
|----------|------|---------|:--------:|------|
| User Session | Web 登录 | Database | ✅ | HttpOnly Secure Cookie |
| Admin Session | 管理后台 | Database | ✅ | 独立 Session |
| User API Token | 第三方 API | 数据库（哈希） | ✅ | 用户自行生成 |
| Bootstrap Token | Runner 初始化 | 数据库（哈希） | ✅ | 一次性，用后即失效 |
| Mail Runner Token | Runner 认证 | 数据库（哈希） | ✅ | 长期，可吊销 |
| GitHub PAT | 触发 Workflow | .env（chmod 600） | ✅ | 不下发到 Runner |
| Cloudflare Token | SPF 管理 | .env（chmod 600） | ✅ | 不下发到 Runner |

---

# 15. 推荐默认配置

```yaml
mail:
  runner:
    heartbeat_interval: 30
    offline_timeout: 90
    max_concurrent_batches: 3     # 每个 Runner 最多同时处理 Batch 数
    progress:
      mode: batch
      batch_size: 50
      max_interval: 5
    health:
      min_sample_size: 20
      healthy_threshold: 0.9       # 成功率 ≥ 90%
      suspected_threshold: 0.5     # 成功率 ≥ 50%
    cancel:
      websocket_timeout: 5
      smtp_graceful_shutdown: true

  scheduler:
    batch_size: 500
    max_concurrent_runners: 10
    lease_duration: 60            # 秒
    lease_refresh: 30             # 秒
    recovery_delay: 30            # 故障转移延迟

  quota:
    billing_mode: smtp_attempt

  spf:
    grace_period_minutes: 30      # SPF 删除前等待时间
```

---

# 16. 文件结构（V2.1 Bounded Context）

```
/opt/proxy-manager/
├── main.py                    # FastAPI 入口
├── config.yaml                 # 非敏感配置
├── .env                        # 敏感配置 (chmod 600, .gitignore)
├── requirements.txt
│
├── database/
│   ├── __init__.py
│   ├── database.py             # 数据库连接
│   └── models.py               # 所有模型
│
├── auth/                       # 共享认证模块
│   ├── __init__.py
│   ├── session.py
│   ├── runner_token.py
│   └── deps.py                 # FastAPI dependencies
│
├── proxy/                      # Proxy Context
│   ├── __init__.py
│   ├── api.py
│   ├── scheduler/
│   │   ├── __init__.py
│   │   ├── allocator.py
│   │   ├── monitor.py
│   │   └── cleanup.py
│   └── templates/
│       └── workflow.yml.j2
│
├── mail/                       # Mail Relay Context
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── tasks.py
│   │   ├── runners.py
│   │   └── admin.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── task.py
│   │   ├── batch.py
│   │   ├── delivery.py
│   │   ├── runner.py
│   │   ├── quota.py
│   │   └── identity.py
│   ├── scheduler/
│   │   ├── __init__.py
│   │   ├── mail_scheduler.py
│   │   └── lease_reaper.py
│   ├── cloudflare/
│   │   ├── __init__.py
│   │   └── spf_manager.py
│   └── websocket/
│       ├── __init__.py
│       └── mail_ws.py
│
├── proxy.db                    # SQLite 数据库（开发环境）
└── templates/
    ├── workflow.yml.j2         # Proxy workflow
    └── mail-runner.yml.j2      # Mail Runner workflow
```

---

# 17. 实施阶段划分（V2.1 修订）

## Phase 1：基础设施

- 数据库模型定义（12+ 张表）
- 数据库迁移脚本
- 配置文件 + .env
- 认证模块（Session + Bootstrap Token + Runner Token）
- 基础 API 路由框架
- 代码结构重构（proxy/ + mail/ Bounded Context）

## Phase 2：Mail Runner 生命周期

- Runner Bootstrap / 注册 / 心跳
- Runner 状态机
- Runner Health 引擎
- SPF Manager（引用计数 + Grace Period）
- Runner API（register / heartbeat / config）

## Phase 3：任务管理

- MailTask 创建（含 Snapshot + Idempotency-Key）
- Batch 拆分
- 额度预占（RESERVE）
- Web 管理界面
- Sender Identity 验证

## Phase 4：SMTP 发送 + 结果

- Mail Runner Agent（Go）
- Batch 认领 / Lease 机制
- 进度上报
- 投递结果存储（MailDelivery + Unique 约束）
- 取消任务（WebSocket）
- 用户报告 / CSV 导出

## Phase 5：增强功能

- 多个并行 Batch 支持（RunnerBatchAssignment）
- 动态进度策略
- 故障转移（Lease Reaper）
- 管理员面板
- 审计日志
- 附件 Presigned URL
- DKIM 签名

---

# 18. V2.1 架构评审变更摘要

## 🔴 必须修改

| # | 变更 | 说明 |
|---|------|------|
| 1 | Runner Bootstrap Token | 一次性 Token 换长期 Token，不用 Workflow 参数直接下发 |
| 2 | MailDelivery ATTEMPTING 状态 | 增加 ATTEMPTING + UNKNOWN，避免 Agent 崩溃后自动重试导致重复投递 |
| 3 | UNIQUE(task_id, recipient) | 唯一约束，严格避免重复投递 |
| 4 | SMTP Attempt 计费定义 | 明确进入 SMTP 投递流程即算一次 Attempt，每次消耗 1 quota |
| 5 | 健康评分规则统一 | 区分收件人错误和 SMTP 拒绝，只有 SMTP 错误参与健康评分 |
| 6 | 邮件内容 Snapshot | 任务创建时冻结邮件内容，Runner 始终读取 Snapshot |
| 7 | Batch Lease | lease_expires_at 机制，不依赖 Runner 正常上报即可回收 |
| 8 | .env 替代 config.yaml | GitHub / Cloudflare Token 移出 config.yaml 到 .env |
| 9 | DKIM / Sender Identity | 明确发件地址验证模型 |

## 🟡 强烈建议

| # | 变更 | 说明 |
|---|------|------|
| 1 | SPF Grace Period | runner_count=0 后等待 30~60 分钟再删除，缓解 DNS 缓存问题 |
| 2 | 多 Batch 并行 | RunnerBatchAssignment 模型，支持一个 Runner 处理多个 Batch |
| 3 | 附件 Presigned URL | 邮件附件通过对象存储分发，不经过 Oracle API |
| 4 | Workflow 生命周期监控 | 轮询 GitHub API，检查 Workflow 完成时 Runner 状态是否正常 |
| 5 | Idempotency-Key | 避免用户重复点击创建多个 Task |
| 6 | 代码 Bounded Context | proxy/ 和 mail/ 分离，避免单体膨胀 |

## 🟢 后续增强

| # | 变更 | 说明 |
|---|------|------|
| 1 | Redis Session | 生产环境用 Redis 替代 Database Session |
| 2 | 2FA | 管理员二次认证 |
| 3 | Secret Manager | 生产环境用 Secret Manager 替代 .env |
| 4 | DKIM 自动签名 | Go Agent 自动签名邮件 |
| 5 | 邮件模板管理 | Web 界面管理模板 |

---

*文档版本 V2.1 — 2026-07-23*
