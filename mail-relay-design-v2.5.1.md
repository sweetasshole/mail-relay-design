# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.5.1（实现前最终修订版本）

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-07-22 | 初始架构设计 |
| V2.0 | 2026-07-23 | 拆分认证/数据库/RBAC/Quota 详细设计 |
| V2.1 | 2026-07-23 | 架构评审：Token 安全、MailDelivery 状态、Batch Lease、Snapshot、Health、SPF、DKIM |
| V2.2 | 2026-07-23 | MailDeliveryAttempt、Cancel 修正、Lease 分离、原子事务、PostgreSQL、GitHub Hosted Runner |
| V2.3 | 2026-07-23 | DKIM 远程签名 API、Quota 防重复扣费、SPF Active IP Set |
| V2.4 | 2026-07-23 | 实现级细化：DKIM RFC 6376、权限模型、Quota 原子事务、状态机完整定义、SPF 简化 |
| V2.5 | 2026-07-23 | **架构冻结**：Billing Boundary 修正、幂等约束、状态层级修正、SPF 原子更新、核心业务不变量 |
| V2.5.1 | 2026-07-24 | **实现前最终修订**：STARTED+Quota 同一事务、PRE_ATTEMPT_FAILURE 状态、RECOVERING 补充、Attempt Number 行锁绑定 |

---

## 核心业务不变量

> 一个 MailDelivery 可以拥有多个 MailDeliveryAttempt；
> 每一个实际产生 SMTP 投递尝试的 Attempt 最多只能对应一次 Quota CONSUME；
> 每一次 Retry 必须创建新的 Attempt 和新的 QuotaTransaction；
> 同一个 Attempt 无论 Runner 重复上报多少次，都不得重复扣除额度。

---

# 1. 项目目标

构建一个邮件代发平台，允许用户：

- 登录 Web 管理后台
- 上传 .eml 邮件模板或直接编辑邮件标题和正文
- 填写收件人列表
- 支持群发单显
- 使用平台域名作为发件地址
- 使用临时 Mail Runner 作为 SMTP 出口
- 支持多个 Mail Runner 并行发送
- 每个 Mail Runner 使用独立公网出口 IP
- 实时查看发送进度
- 用户可以随时取消任务
- 查看每个收件人的最终投递状态
- 查看失败原因，例如邮箱不存在
- 下载失败收件人列表

系统管理员可以：

- 查看所有用户
- 查看用户发送量
- 设置用户发送额度
- 停用用户
- 查看所有发送任务
- 查看所有 Mail Runner
- 查看 Runner 出口 IP
- 手动控制 Runner 状态
- 调整 Runner 进度上报策略

系统保留现有：

- Proxy Runner (frpc + frps + SOCKS5)

新增：

- Mail Runner (Go Agent)
- Mail Runner Agent (Go)
- Mail Scheduler (Python)
- Mail Task / Batch / Delivery (Python)
- SMTP Direct Egress (Go Agent)

---

# 2. 总体架构

## 2.1 架构图

```
                         Oracle VPS (Python/FastAPI)
                     -----------------------------
                         Control Plane
                              |
        +---------------------+---------------------+
        |                     |                     |
        v                     v                     v
   Auth/Users          Mail Scheduler         Runner Manager
        |                     |                     |
        |                     |               ------+------
        |                     |               |           |
        |                     |          Proxy Runner  Mail Runner
        |                     |          (frpc/frps)   (Go Agent)
        |                     |               |           |
        |                     |               |      SMTP Direct
        |                     |               |           |
        |                     |               |           v
        |                     |               |      Internet MX
        |                     +--- Task / Batch / Delivery
        |
        +--- Quota / RBAC / Audit
```

## 2.2 核心设计原则

1. **Oracle 是 Control Plane，Mail Runner 是 Data Plane**
2. **SMTP 流量不经过 Oracle**
3. **每种凭证拥有最小权限，严格隔离**
4. **User Token != Runner Token != Admin Token != GitHub PAT != Cloudflare Token**
5. **Cloudflare / GitHub 凭证只存在 Oracle，不下发**
6. **DKIM 私钥永久只保存在 Oracle，不下发到 Mail Runner**
7. **status 和 health_status 独立判断**
8. **Runner 健康基于窗口数据 + 区分收件人失败和 SMTP 拒绝**
9. **Quota 使用 RESERVE + CONSUME + RELEASE 事务模型，原子更新**
10. **每个 Task+Recipient 唯一约束，避免重复投递**
11. **每次 SMTP Attempt 有独立生命周期记录**
12. **Batch 使用 Lease 机制实现超时回收，Heartbeat 与 Lease 分离**
13. **邮件内容在任务创建时 Snapshot，不可变，大内容走对象存储**
14. **所有投递结果持久化，支持导出**
15. **开发环境 SQLite，生产环境 PostgreSQL**
16. **系统保证 At-Least-Once Attempt Semantics，不保证 Exactly-Once SMTP Delivery**

## 2.3 代码架构：Bounded Context

Proxy Manager 和 Mail Relay 共用同一 FastAPI Control Plane，但逻辑上分为两个 Bounded Context：

```
/opt/proxy-manager/
|-- main.py              # FastAPI 入口
|-- config.yaml          # 非敏感配置
|-- .env                 # 敏感配置 (chmod 600)
|
|-- proxy/               # Proxy Context
|   |-- api.py
|   |-- models.py
|   |-- scheduler/
|   +-- templates/
|
|-- mail/                # Mail Relay Context
|   |-- api/
|   |-- models/
|   |-- scheduler/
|   |-- cloudflare/
|   |-- signing/         # DKIM 签名服务
|   +-- websocket/
|
|-- auth/                # 共享认证
|-- database/
+-- templates/
```

---

# 3. 技术栈

## 3.1 Oracle Control Plane

| 组件 | 技术 | 说明 |
|------|------|------|
| Web 框架 | Python FastAPI | 现有技术栈延续 |
| ORM | SQLAlchemy | 现有技术栈延续 |
| 数据库 | **PostgreSQL（生产）/ SQLite（开发）** | 并发事务需要 PG 行级锁 |
| Session | Server-side Session (Database/Redis) | HttpOnly Secure Cookie |
| 实时通信 | WebSocket | Runner 控制通道 |
| SPF | Cloudflare API | 原子引用计数管理 |
| Runner 触发 | GitHub API | 触发 workflow |
| 对象存储 | 本地文件系统 / S3 兼容 | 邮件内容分发 |
| DKIM 签名 | Oracle 侧签名 API（RFC 6376） | 私钥不下发 Runner |

## 3.2 Mail Runner Agent

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Go | goroutine + channel + context |
| SMTP | net/smtp 或第三方库 | 并发发送 |
| 通信 | HTTPS + WebSocket | Oracle 控制通道 |
| 部署 | **GitHub Hosted Runner** | IP 动态，需 SPF Active Set |

## 3.3 Proxy Runner（保持不变）

| 组件 | 技术 |
|------|------|
| 隧道 | frpc + frps |
| 代理协议 | SOCKS5 |

---

# 4. 认证与安全设计

## 4.1 凭证隔离原则

```
普通用户 Session
        !=
管理员 Session
        !=
用户 API Token
        !=
Mail Runner Token
        !=
Bootstrap Token（一次性）
        !=
GitHub PAT
        !=
Cloudflare API Token
        !=
DKIM Private Key
```

**每一种凭证只拥有完成自己工作的最小权限。**

## 4.2 Runner Token 安全下发

### 流程

```
Oracle
  |
  | 1. 生成 Runner ID + Bootstrap Token（TTL 5 分钟）
  | 2. 绑定 Runner ID 到 Token
  | 3. 触发 Workflow，只携带 Bootstrap Token
  v
GitHub Actions
  |
  | 环境变量 BOOTSTRAP_TOKEN
  v
Mail Runner Agent
  |
  | POST /api/mail-runners/bootstrap
  | { "bootstrap_token": "...", "public_ip": "..." }
  v
Oracle
  |
  | 1. 验证 Bootstrap Token（未过期、未使用）
  | 2. 从 Token 解析 Runner ID（拒绝 Runner 自行提交）
  | 3. 生成长期 Runner Token（含 scope）
  | 4. Bootstrap Token 立即失效
  v
Agent 获得 Runner Token
```

### BootstrapToken 模型

```python
class RunnerBootstrapToken(Base):
    __tablename__ = "runner_bootstrap_tokens"
    id: UUID
    token_hash: str
    runner_id: UUID              # 预绑定，Agent 不能修改
    expires_at: datetime         # TTL 5 分钟
    used_at: datetime | None
    issued_at: datetime
    ip_restriction: str | None
```

### 安全规则

- Bootstrap Token TTL = 5 分钟
- Token 与 Runner ID 预绑定，Agent 不能自行指定 runner_id
- 使用后立即失效
- 即使泄露，攻击者只有 5 分钟窗口，且只能注册特定 Runner

## 4.3 Runner Token Scope（V2.4）

```
Mail Runner Token scope: ["heartbeat", "claim", "progress", "result", "dkim:sign"]

DKIM Signing API 验证 token 中的 dkim:sign scope。
不引入独立 DkimSigningToken。
```

### RunnerCredential 模型

```python
class RunnerCredential(Base):
    __tablename__ = "runner_credentials"
    id: UUID
    runner_id: UUID
    token_hash: str
    scope: str  # JSON: ["heartbeat","claim","progress","result","dkim:sign"]
    issued_at: datetime
    expires_at: datetime | None
    revoked_at: datetime | None
    last_used_at: datetime | None
```

## 4.4 认证架构

```
                         Oracle API
                             |
         --------------------+-------------------
         |                   |                   |
         v                   v                   v
     User Web            Admin Web          Mail Runner
         |                   |                   |
   Session Cookie      Session Cookie       Runner Token
         |                   |                   |
         v                   v                   v
    User Role            Admin Role        Runner Identity
         |                   |                   |
         +-------------------+-------------------+
                             |
                       Authorization
                             |
                             v
                         API Layer
```

## 4.5 Web 认证方案

采用 **Server-side Session**（非 JWT）：

```
Cookie(session_id) -> Database -> User Identity
```

管理员停用用户 -> User.status = DISABLED -> 立即拒绝所有 Session。

## 4.6 敏感凭证存储

```bash
# /opt/proxy-manager/.env（chmod 600, .gitignore）
GITHUB_TOKEN=ghp_xxx
CLOUDFLARE_TOKEN=cfut_xxx
CLOUDFLARE_ZONE_ID=xxx
DKIM_PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY-----...
```

GitHub Token 和 Cloudflare Token 使用不同 Token，各自拥有最小权限。

---

# 5. 数据库模型设计

## 5.1 模型总览

| 领域 | 模型 | 说明 |
|------|------|------|
| User | User, UserApiToken | 用户与 API Token |
| Admin | Role, Permission | RBAC 权限 |
| Quota | QuotaAccount, QuotaTransaction | 额度管理（原子 CONSUME + 幂等约束） |
| MailTask | MailTask, MailTemplateSnapshot | 邮件任务与内容快照 |
| MailBatch | MailBatch, RunnerBatchAssignment | 批次与并行分配 |
| MailDelivery | MailDelivery | 投递结果（唯一约束） |
| MailDeliveryAttempt | MailDeliveryAttempt | SMTP Attempt 独立记录（唯一约束） |
| MailRunner | MailRunner, RunnerBootstrapToken, RunnerCredential, RunnerHealth | Runner 管理 |
| Identity | SenderIdentity | 发件身份验证 |
| SPF | SpfIpRecord | SPF 引用计数（原子更新） |
| DKIM | DkimKey, DkimSigningRate | DKIM 签名管理 |

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

## 5.3 MailRunner 模型

```python
class MailRunner(Base):
    __tablename__ = "mail_runners"
    id: UUID
    runner_id: str
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
    smtp_concurrency: int
    max_concurrent_batches: int
    report_mode: str
    report_batch_size: int
    report_interval_seconds: int
    created_at: datetime
    updated_at: datetime
```

### RunnerBatchAssignment

```python
class RunnerBatchAssignment(Base):
    __tablename__ = "runner_batch_assignments"
    id: UUID
    runner_id: UUID
    batch_id: UUID
    assigned_at: datetime
    released_at: datetime | None
    status: str                # ACTIVE / RELEASED
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

例如：status = BUSY, health_status = UNHEALTHY -> 立即停止接收新 Batch。

## 5.5 RunnerHealth（窗口数据）

```python
class RunnerHealth(Base):
    __tablename__ = "runner_health"
    id: UUID
    runner_id: UUID
    window_start: datetime
    window_end: datetime
    attempted_count: int
    success_count: int
    # 收件人错误（不影响 Runner 健康评分）
    mailbox_not_found_count: int
    domain_not_found_count: int
    # SMTP 拒绝（降低 Runner 健康评分）
    smtp_rejected_count: int
    rate_limited_count: int
    # 网络错误（降低 Runner 健康评分）
    connection_timeout_count: int
    connection_refused_count: int
    tls_error_count: int
    # 汇总
    recipient_error_count: int
    smtp_error_count: int
    network_error_count: int
    created_at: datetime
```

### 健康评分引擎

```python
def calculate_health(runner_health):
    if runner_health.attempted_count < 20:
        return "UNKNOWN"
    smtp_errors = runner_health.smtp_rejected_count + runner_health.rate_limited_count
    network_errors = runner_health.connection_timeout_count + runner_health.connection_refused_count
    total = smtp_errors + network_errors + runner_health.success_count
    if total == 0: return "UNKNOWN"
    error_rate = (smtp_errors + network_errors) / total
    if error_rate < 0.1: return "HEALTHY"
    elif error_rate < 0.5: return "SUSPECTED"
    else: return "UNHEALTHY"
```

**收件人错误（mailbox_not_found / domain_not_found）不降低 Runner 健康评分。**

### 动态并发控制

```
HEALTHY   -> max_concurrent_batches = 3, report_batch_size = 50, report_interval = 5s
SUSPECTED -> max_concurrent_batches = 1, report_batch_size = 1, report_interval = 1s
UNHEALTHY -> max_concurrent_batches = 0（停止分配新 Batch）
```

## 5.6 MailTask 模型

```python
class MailTask(Base):
    __tablename__ = "mail_tasks"
    id: UUID
    user_id: UUID
    status: str
    idempotency_key: str | None
    subject_snapshot: str | None
    from_snapshot: str
    headers_snapshot: str | None
    body_snapshot: str | None
    content_object_key: str | None
    attachment_keys: str | None
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

### 内容分发

大邮件内容不进入数据库：

```
Object Storage
|
+-- /objects/mail-tasks/{task_id}/
    |-- mail.eml
    |-- manifest.json
    +-- attachments/
        |-- file1.pdf
        +-- file2.jpg
```

Runner 获取任务时：Oracle API -> Presigned URL -> Runner 直接下载邮件包

## 5.7 MailBatch 模型

```python
class MailBatch(Base):
    __tablename__ = "mail_batches"
    id: UUID
    task_id: UUID
    runner_id: UUID | None
    status: str                # QUEUED/ASSIGNED/CLAIMED/RUNNING/COMPLETED/CANCELLING/CANCELLED/FAILED/RECOVERING
    total_recipients: int
    attempted_count: int
    delivered_count: int
    failed_count: int
    not_attempted_count: int
    claimed_at: datetime | None
    lease_expires_at: datetime | None
    started_at: datetime | None
    completed_at: datetime | None
    retry_count: int
    created_at: datetime
```

### 统计一致性规则

```
attempted_count
= DELIVERED + PERMANENT_FAILURE + TEMPORARY_FAILURE + UNKNOWN + CANCELLED_AFTER_ATTEMPT

NOT_ATTEMPTED 不算 Attempt
```

### Lease 机制（Heartbeat 与 Lease 分离）

```
Runner 认领 Batch
    -> lease_expires_at = now + 60s
    -> 每次 Batch Progress 上报
        -> lease_expires_at = now + 60s
    -> Runner Heartbeat 不刷新 Batch Lease

Lease Reaper（每 30 秒）：
    UPDATE mail_batches
    SET status = RECOVERING
    WHERE lease_expires_at < NOW()
    AND status IN (CLAIMED, RUNNING)
```

### 原子 Claim

```sql
UPDATE mail_batches
SET status = CLAIMED, runner_id = ?, claimed_at = NOW(), lease_expires_at = NOW() + 60
WHERE id = ? AND status = QUEUED;
-- 只有 affected_rows = 1 才算成功
```

## 5.8 MailDelivery 模型

```python
class MailDelivery(Base):
    __tablename__ = "mail_deliveries"
    id: UUID
    task_id: UUID
    batch_id: UUID
    runner_id: UUID | None      # 最终 Runner（V2.4 明确语义）
    recipient: str
    status: str                 # NOT_ATTEMPTED/ATTEMPTING/DELIVERED/FAILED/UNKNOWN/CANCELLED
    failure_reason: str | None  # RETRY_EXHAUSTED 等（V2.5）
    attempts: int
    first_attempt_at: datetime | None
    last_attempt_at: datetime | None
    completed_at: datetime | None
    __table_args__ = (
        UniqueConstraint(task_id, recipient, name="uq_task_recipient"),
    )
```

### 投递状态机（业务视角）

```
NOT_ATTEMPTED
    |
    +-- 用户取消 -> CANCELLED
    |
    v
ATTEMPTING
    |
    +-- DELIVERED           <- SMTP 250
    +-- FAILED               <- 所有 Attempt 失败，重试耗尽（reason=RETRY_EXHAUSTED）
    +-- UNKNOWN              <- Agent 崩溃
    +-- （不能直接 CANCELLED）
```

**关键规则：**

- NOT_ATTEMPTED -> 用户取消 -> CANCELLED
- ATTEMPTING -> 用户取消 -> 不能直接 CANCELLED
  - Agent 应优雅结束当前 SMTP 操作
  - 拿到明确结果 -> DELIVERED / FAILURE
  - Agent 崩溃 -> UNKNOWN
- CANCELLED 只能用于从未开始 SMTP 的收件人

## 5.9 MailDeliveryAttempt 模型（V2.2 新增，V2.4 新增 CREATED，V2.5 新增唯一约束）

**每次 SMTP Attempt 的独立生命周期记录：**

```python
class MailDeliveryAttempt(Base):
    __tablename__ = "mail_delivery_attempts"
    id: UUID
    delivery_id: UUID
    task_id: UUID
    batch_id: UUID
    runner_id: UUID
    attempt_number: int          # Oracle 生成，非 Runner 提交（V2.5.1 行锁绑定）
    status: str                  # CREATED/STARTED/DELIVERED/PERMANENT_FAILURE/TEMPORARY_FAILURE/UNKNOWN/PRE_ATTEMPT_FAILURE
    smtp_code: int | None
    smtp_enhanced_code: str | None
    smtp_response: str | None
    error_type: str | None       # RECIPIENT_NOT_FOUND/IP_BLOCKED/RATE_LIMITED/...
    started_at: datetime
    completed_at: datetime | None
    __table_args__ = (
        UniqueConstraint(delivery_id, attempt_number, name="uq_delivery_attempt"),  # V2.5
    )

### Attempt Number 生成（V2.5.1 P1 行锁绑定）

Attempt Number 的生成必须与 MailDelivery 行锁绑定在同一事务中，防止并发幻读：

```sql
BEGIN;

-- 1. 锁定 MailDelivery 行
SELECT status FROM mail_deliveries
WHERE id = :delivery_id
FOR UPDATE;

-- 2. 在锁保护下生成 attempt_number
SELECT COALESCE(MAX(attempt_number), 0) + 1
INTO :attempt_number
FROM mail_delivery_attempts
WHERE delivery_id = :delivery_id;

-- 3. INSERT Attempt
INSERT INTO mail_delivery_attempts
    (id, delivery_id, attempt_number, status)
VALUES
    (gen_random_uuid(), :delivery_id, :attempt_number, 'CREATED');

COMMIT;
```
```

### 与 Quota 的关系

```
MailDeliveryAttempt
    |
    +-- STARTED -> QuotaTransaction (CONSUME 1)
    |       +-- UNIQUE(attempt_id, transaction_type)  # V2.3 防重复扣费
    |
    +-- DELIVERED -> MailDelivery.status = DELIVERED
    |
    +-- PERMANENT_FAILURE -> MailDelivery.status = FAILED
    |
    +-- TEMPORARY_FAILURE -> 可重试（最多 3 次）
    |
    +-- UNKNOWN
            -> 不自动重试
            -> 管理员决定重试 -> 新 Attempt #2 -> 新 CONSUME
```

### 状态对应 SMTP 响应码

| Attempt 状态 | SMTP 码 | 说明 |
|-------------|---------|------|
| DELIVERED | 250 | 成功 |
| PERMANENT_FAILURE | 5xx | 永久失败，不重试 |
| TEMPORARY_FAILURE | 4xx | 临时失败，最多重试 3 次 |
| UNKNOWN | N/A | Agent 崩溃/断线 |

### 重试策略

```
TEMPORARY_FAILURE
    |
    +-- 重试上限 = 3 次
    +-- 间隔: 60s, 300s, 900s
    +-- 3 次后仍失败 -> MailDelivery.status = FAILED, reason = RETRY_EXHAUSTED（V2.5）
    +-- Attempt.status 保持 TEMPORARY_FAILURE（与 SMTP 响应一致）
    +-- 每次重试 = 新 Attempt = 新 CONSUME
```

## 5.10 Quota 设计

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

### QuotaTransaction（V2.3 防重复扣费，V2.5 幂等约束）

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"
    id: UUID
    user_id: UUID
    task_id: UUID | None
    delivery_id: UUID | None
    attempt_id: UUID | None
    transaction_type: str    # RESERVE / CONSUME / RELEASE / ADJUST / REFUND
    amount: int
    created_at: datetime
    __table_args__ = (
        UniqueConstraint(attempt_id, transaction_type,
                        name="uq_quota_attempt_type"),
    )
```

### 原子 CONSUME（V2.4 + V2.5）

```sql
UPDATE quota_accounts
SET reserved = reserved - 1,
    consumed = consumed + 1
WHERE user_id = :user_id
  AND reserved >= 1;

-- affected_rows == 1 才允许继续，否则 QUOTA_INSUFFICIENT
```

### 完整流程

```
用户提交 1000 个收件人
    -> RESERVE +1000（available -= 1000, reserved += 1000）

每次 SMTP Attempt 开始（TCP connect 成功）
    -> 原子 CONSUME 1（reserved -= 1, consumed += 1）
    -> INSERT QuotaTransaction (CONSUME, attempt_id)
    -> 同一 attempt_id 不可重复 CONSUME（唯一约束）

任务完成/取消
    -> RELEASE 剩余 reserved（reserved -= N, available += N）
```

## 5.11 SenderIdentity

```python
class SenderIdentity(Base):
    __tablename__ = "sender_identities"
    id: UUID
    user_id: UUID
    email: str
    domain: str
    status: str
    verified_at: datetime | None
    created_at: datetime
```

## 5.12 UserApiToken

```python
class UserApiToken(Base):
    __tablename__ = "user_api_tokens"
    id: UUID
    user_id: UUID
    token_hash: str
    name: str
    permissions: str
    created_at: datetime
    expires_at: datetime | None
    last_used_at: datetime | None
    revoked_at: datetime | None
```

---

# 6. Billing Boundary 定义（V2.5.1 P0 修正）

## 6.1 精确定义

> **Billing Boundary 是 Runner 成功建立到目标 SMTP 服务端的 TCP 连接后。无论后续 SMTP、TLS、认证、Envelope 或 DATA 阶段成功还是失败，均计费一次。**

不计费场景：
- DNS lookup 失败 -> PRE_ATTEMPT_FAILURE
- TCP connect 失败 -> PRE_ATTEMPT_FAILURE

## 6.2 状态转换

```
CREATED
    |
    +-- DNS lookup 失败  -> PRE_ATTEMPT_FAILURE（不计费）
    +-- TCP connect 失败 -> PRE_ATTEMPT_FAILURE（不计费）
    |
    v
*** Billing Boundary ***
    |
    v
STARTED（同一事务：CONSUME 1 + INSERT QuotaTransaction）
    |
    +-- DELIVERED           <- 250 OK
    +-- PERMANENT_FAILURE   <- 5xx
    +-- TEMPORARY_FAILURE   <- 4xx（可重试）
    +-- UNKNOWN             <- Agent 崩溃
```

## 6.3 原子事务（V2.5.1 P0 关键修正）

**STARTED + QuotaAccount CONSUME + QuotaTransaction INSERT 必须在同一数据库事务中。**

```sql
BEGIN;

-- 1. 锁定 Attempt 行
SELECT status FROM mail_delivery_attempts
WHERE id = :attempt_id
FOR UPDATE;

-- 2. 校验状态（防止重复 STARTED）
-- 必须在应用层校验 status = CREATED

-- 3. 原子扣减额度
UPDATE quota_accounts
SET reserved = reserved - 1,
    consumed = consumed + 1
WHERE user_id = :user_id
  AND reserved >= 1;

-- 校验 affected_rows == 1，否则 ROLLBACK + QUOTA_INSUFFICIENT

-- 4. 插入 QuotaTransaction（唯一约束防重复）
INSERT INTO quota_transactions
    (id, user_id, attempt_id, transaction_type, amount, created_at)
VALUES
    (gen_random_uuid(), :user_id, :attempt_id, 'CONSUME', 1, NOW());

-- 5. 更新 Attempt 状态为 STARTED
UPDATE mail_delivery_attempts
SET status = 'STARTED',
    started_at = NOW()
WHERE id = :attempt_id
  AND status = 'CREATED';  -- 条件校验，防止重复

COMMIT;
```

**事务提交成功后，Runner 才真正执行 SMTP 操作。**

这样保证：
1. STARTED + QuotaAccount + QuotaTransaction 三状态始终一致
2. 重复提交幂等（靠 UNIQUE attempt_id + FOR UPDATE 行锁）
3. 崩溃后 Runner 重试时，数据库拒绝重复 STARTED（`AND status = 'CREATED'`）

---

# 7. DKIM 设计（V2.3 远程签名 API，V2.4 RFC 6376 规范）

## 7.1 核心原则

**DKIM 私钥永久只保存在 Oracle，不下发到 Mail Runner。**

GitHub Hosted Runner 是临时实例，私钥下发存在泄露风险。

## 7.2 签名流程（RFC 6376）

```
邮件 Body
    |
    v
Canonicalization（relaxed/relaxed）
    |
    v
Body Hash（bh=）
    |
    v
构建 DKIM-Signature Header（b= 留空）
    |  包含 v=, a=, c=, d=, s=, h=, bh=
    v
对 Canonicalized Header 签名数据做 RSA-SHA256
    |
    v
填充 b=，生成完整 DKIM-Signature
```

Key 点：

- 签名输入不是 headers 数组，而是 canonicalized DKIM-Signature header 数据
- h= 必须明确列出参与签名的 header 列表
- bh= 是 canonicalized body 的 sha256 hash
- c= 指定 canonicalization 算法（relaxed/relaxed 推荐）

## 7.3 API 定义

```
POST /api/mail/dkim/sign

Request:
{
  "domain": "yourdomain.com",
  "selector": "default",
  "canonicalization": "relaxed/relaxed",
  "signed_headers": ["From","To","Subject","Message-ID","Date"],
  "headers": {
    "From": "sender@yourdomain.com",
    "To": "recipient@example.com",
    "Subject": "Test",
    "Message-ID": "<uuid>",
    "Date": "Thu, 23 Jul 2026 12:00:00 +0000"
  },
  "body_hash": "base64(sha256(canonicalized_body))"
}

Response:
{
  "dkim_signature": "DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=yourdomain.com; s=default; h=From:To:Subject:Message-ID:Date; bh=...; b=..."
}
```

## 7.4 签名后不可变性（V2.5 文档补充）

最终 MIME Message 确定 Body 和所有 Header 后 -> canonicalize -> bh= -> 请求 Oracle -> 只插入 DKIM-Signature Header -> 禁止再次修改 Body -> SMTP。

Runner Agent 必须确保 DKIM 签名后不重新序列化邮件。

## 7.5 Oracle DKIM Signer 职责

- 验证 Runner Token 中的 dkim:sign scope
- 加载 DKIM 私钥
- 根据 RFC 6376 构建 canonicalized 签名输入
- 执行 RSA-SHA256 签名
- 返回完整 DKIM-Signature header

## 7.6 模型

```python
class DkimKey(Base):
    __tablename__ = "dkim_keys"
    id: UUID
    domain: str
    selector: str
    private_key: str
    public_key: str
    active: bool
    created_at: datetime
```

## 7.7 签名限流

```yaml
dkim:
  rate_limit:
    per_runner_per_minute: 1000
    per_domain_per_minute: 5000
```

## 7.8 Key Rotation

1. 创建 selector2 密钥
2. 发布 selector2 DNS TXT 记录
3. 等待 DNS 生效（TTL + 24h）
4. 验证 selector2 TXT 记录
5. 切换签名使用 selector2
6. 保留 selector1 至少 TTL + 24h
7. 删除 selector1

## 7.9 生产安全

- 私钥文件：/etc/opt/proxy-manager/dkim/private.pem
- 权限：root:root 600
- 未来可迁移至 HashiCorp Vault / AWS Secrets Manager

---

# 8. SPF 管理（V2.3 Active IP Set，V2.4 简化，V2.5 原子更新）

## 8.1 前提

Mail Runner 使用 GitHub Hosted Runner，IP 动态变化。

## 8.2 SPF 子域名架构

```
主域名 SPF（不频繁修改）：
v=spf1 include:_spf.yourdomain.com ~all

子域名 SPF（动态维护）：
_spf.yourdomain.com TXT "v=spf1 ip4:RunnerA ip4:RunnerB ... ~all"
```

## 8.3 SpfIpRecord（V2.4 简化）

```python
class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"
    ip: str
    active_runner_count: int
    grace_period_until: datetime | None
    created_at: datetime
    updated_at: datetime
```

## 8.4 原子更新（V2.5 P1 修正）

V2.4 的问题：两个 Runner 同时上线时，应用层 read-modify-write 可能导致计数错误。

```sql
-- Runner 上线（原子递增）
INSERT INTO spf_ip_records (ip, active_runner_count, created_at, updated_at)
VALUES (:ip, 1, NOW(), NOW())
ON CONFLICT (ip) DO UPDATE
SET active_runner_count = spf_ip_records.active_runner_count + 1,
    grace_period_until = NULL,
    updated_at = NOW();

-- Runner 下线（原子递减 + Grace Period）
UPDATE spf_ip_records
SET active_runner_count = active_runner_count - 1,
    grace_period_until = CASE
        WHEN active_runner_count - 1 <= 0
        THEN NOW() + INTERVAL '30 minutes'
        ELSE grace_period_until
    END,
    updated_at = NOW()
WHERE ip = :ip
  AND active_runner_count > 0;
```

**禁止在应用层读-改-写。**

## 8.5 逻辑

```
Runner 上线：active_runner_count += 1; grace_period_until = NULL
Runner 下线：active_runner_count -= 1
  if active_runner_count == 0: grace_period_until = now + 30min
Grace Period 到期：if active_runner_count == 0: 从 SPF 删除
```

## 8.6 Active IP 保护

```
上限 50 个 IP。
达到上限时：
  - 不删除活跃 Runner 的 IP（会造成 SPF 失败）
  - 拒绝新 Mail Runner 注册
  - 暂停启动新 Runner，直至有 IP 释放
```

## 8.7 SPF 生效验证

```
Runner IP 注册
    -> 更新 SPF
    -> 从多个公共 DNS Resolver 查询（1.1.1.1, 8.8.8.8）
    -> 确认 SPF 已可见
    -> Runner = READY
```

> 注意：SPF 生效验证只能证明 Oracle 自己查询到新记录，不能保证所有收件服务器已看到。Grace Period 机制用于缓解此问题。

## 8.8 配置

```yaml
spf:
  grace_period_minutes: 30
  verification_resolvers:
    - 1.1.1.1
    - 8.8.8.8
  active_ip_set_max_size: 50
```

---

# 9. Mail Scheduler

## 9.1 调度逻辑

```
用户创建 MailTask（10000 收件人）
    -> RESERVE 额度
    -> 拆分为 MailBatch（batch_size = 500）
    -> Batch 001~020 进入 QUEUED

Scheduler 循环：
    -> 寻找 READY / BUSY 但未达到 max_concurrent_batches 的 Runner
    -> 按健康状态限制并发数
    -> 原子分配 Batch
```

## 9.2 claim-next API（V2.4 统一）

```
POST /api/mail-runners/{id}/claim-next

Oracle 内部原子事务：
1. SELECT batch FOR UPDATE SKIP LOCKED
2. 验证 Runner 健康状态（V2.5 补充）
3. UPDATE batch.status = CLAIMED, runner_id, lease_expires_at
4. INSERT runner_batch_assignments
5. COMMIT
```

### 健康与并发控制

```
HEALTHY   -> max_concurrent_batches = 3, report_batch_size = 50, report_interval = 5s
SUSPECTED -> max_concurrent_batches = 1, report_batch_size = 1, report_interval = 1s
UNHEALTHY -> max_concurrent_batches = 0（禁止 claim）
```

## 9.3 Lease 机制（Heartbeat 与 Lease 分离）

```
Runner Heartbeat（30s）: 仅证明 Runner 活着，不刷新 Batch Lease
Batch Progress（50 封/5s）: 刷新 batch.lease_expires_at
Lease Reaper（30s）: 回收过期 Lease，进入 RECOVERING
```

## 9.4 RECOVERING 状态机（V2.5.1 P1 修正）

V2.4 混用 NOT_ATTEMPTED（Delivery 级）和 CREATED（Attempt 级），状态层级混乱。
V2.5.1 增加 TEMPORARY_FAILURE 处理。

```
RECOVERING
    |
    v
分析每个 MailDelivery：
    |
    +-- NOT_ATTEMPTED -> QUEUED（重新分配）
    +-- ATTEMPTING -> 查最新 Attempt：
    |     +-- CREATED               -> QUEUED（尚未开始）
    |     +-- STARTED               -> UNKNOWN（已开始但结果未知）
    |     +-- TEMPORARY_FAILURE     -> 如果 retry_count < 3 则 QUEUED（继续重试）
    |     +-- 其他                  -> 不处理
    +-- DELIVERED/FAILED -> 不处理
    |
    v
全部处理完 -> COMPLETED；有剩余 -> QUEUED
```

---

# 10. Mail Runner Agent（Go）

## 10.1 启动流程

```
GitHub Hosted Runner 启动
    -> 下载/启动 Mail Runner Agent
    -> Agent 获取公网 IP
    -> Bootstrap Token 注册（5 分钟 TTL）
    -> Oracle 验证并返回长期 Runner Token（含 scope）
    -> Bootstrap Token 立即失效
    -> Oracle 保存 IP，更新 SPF
    -> SPF 生效验证
    -> Heartbeat 开始（30s）
    -> Runner = READY
```

## 10.2 Agent 职责

- 获取公网 IP / Bootstrap 注册
- Heartbeat（30s）
- WebSocket 控制通道
- claim-next 获取 Batch
- 下载邮件内容（Presigned URL）
- SMTP 投递（goroutine 并发）
- 进度上报（50 封/5s）
- 批量投递结果上报
- DKIM 签名请求（Oracle 远程 Signing）
- 处理 CANCEL / PAUSE / CONFIG_UPDATE
- 断线恢复（Grace Period 30s）

## 10.3 Agent 不负责

- 用户/额度/任务管理
- Runner 生命周期/SPF/DKIM 私钥管理
- 其他 Runner 管理

## 10.4 通信方式

```
HTTPS API:
    注册 (bootstrap)
    心跳（仅证明 Runner 活着）
    任务 Claim
    Batch Progress（刷新 Lease）
    批量投递结果
    Batch 完成

WebSocket:
    CANCEL
    PAUSE
    RESUME
    CONFIG_UPDATE
    STOP
```

---

# 11. Runner 生命周期

## 11.1 状态机

```
STARTING -> REGISTERING -> VERIFYING -> READY <-> BUSY
                                          |
                                          +-> DRAINING -> STOPPING -> OFFLINE
                                          +-> SUSPENDED
                                          +-> HEARTBEAT_TIMEOUT -> OFFLINE
```

健康状态独立：

```
UNKNOWN / HEALTHY / SUSPECTED / UNHEALTHY
status = BUSY, health_status = UNHEALTHY -> 立即停止 claim
```

## 11.2 Workflow 生命周期

```
Oracle Scheduler
    |
    触发 Workflow
    |
    记录 workflow_run_id + workflow_job_id
    |
    轮询 GitHub API
    |
    Workflow completed
    |
    检查 Runner 状态：
    +-- Runner = OFFLINE -> 正常
    +-- Runner = BUSY    -> 异常
```

---

# 12. 故障转移与 Cancel

## 12.1 故障转移

```
Runner A 突然离线
    -> Oracle 检测 Heartbeat 超时（90s）
    -> Runner A = OFFLINE
    -> Lease Reaper 回收所有关联 Batch
    -> Batch = RECOVERING
    -> 根据 MailDeliveryAttempt 判断：
        - STARTED -> UNKNOWN（不确定是否已发送）
        - NOT_ATTEMPTED -> 重新 QUEUED
        - DELIVERED / FAILED -> 不处理
    -> 重新调度到 Runner B / C
```

## 12.2 取消任务

```
用户点击取消
    -> MailTask = CANCELLING
    -> WebSocket 下发 CANCEL 到所有关联 Runner
    -> Agent 处理：
        - 停止领取新收件人
        - 当前 SMTP 操作优雅结束（等待 250 OK）
        - 记录明确结果
        - 上报最终进度
    -> Oracle 处理：
        - NOT_ATTEMPTED -> CANCELLED
        - ATTEMPTING -> 等待 Agent 结果
            - 有明确 SMTP 结果 -> DELIVERED / FAILURE
            - Agent 崩溃 -> UNKNOWN
        - 释放未尝试收件人的预占额度
    -> MailTask = CANCELLED
```

**关键规则：ATTEMPTING 不能直接 CANCELLED。SMTP DATA 可能已发送完成只是没拿到回应。**

---

# 13. API 端点总览

## 13.1 用户 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /auth/register | 注册 |
| POST | /auth/login | 登录 |
| POST | /auth/logout | 登出 |
| GET  | /auth/me | 当前用户 |
| PUT  | /auth/password | 修改密码 |

## 13.2 任务 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/tasks | 创建任务（支持 Idempotency-Key） |
| GET  | /api/tasks | 任务列表 |
| GET  | /api/tasks/{id} | 任务详情 |
| POST | /api/tasks/{id}/cancel | 取消任务 |
| GET  | /api/tasks/{id}/results | 下载结果 CSV |
| GET  | /api/tasks/{id}/failures | 下载失败列表 |
| GET  | /api/tasks/{id}/attachment/{name} | 附件下载（Presigned URL） |

## 13.3 Runner API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/mail-runners/bootstrap | 使用 Bootstrap Token 注册 |
| POST | /api/mail-runners/{id}/heartbeat | 心跳（仅证明 Runner 活着） |
| GET  | /api/mail-runners/{id}/config | 获取配置 |
| POST | /api/mail-runners/{id}/claim-next | 原子领取下一个 Batch（V2.4 统一） |
| POST | /api/mail-batches/{id}/progress | 上报进度（刷新 Lease） |
| POST | /api/mail-batches/{id}/results | 批量上报投递结果 |
| POST | /api/mail-batches/{id}/complete | 完成 Batch |

## 13.4 管理员 API

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

# 14. Token 与凭证总结

| 凭证类型 | 用途 | 存储位置 | 可撤销 | 下发？ |
|----------|------|---------|:------:|:------:|
| User Session | Web 登录 | Database | Y | — |
| Admin Session | 管理后台 | Database | Y | — |
| User API Token | 第三方 API | 数据库(哈希) | Y | — |
| Bootstrap Token | Runner 初始化 | 数据库(哈希) | Y (TTL 5min) | Workflow 参数 |
| Mail Runner Token | Runner 认证 + Scope | 数据库(哈希) | Y | 注册时获得 |
| GitHub PAT | 触发 Workflow | .env (chmod 600) | Y | N |
| Cloudflare Token | SPF 管理 | .env (chmod 600) | Y | N |
| DKIM Private Key | 邮件签名 | .env (chmod 600) | Y | **N** |

---

# 15. 推荐默认配置

```yaml
mail:
  runner:
    heartbeat_interval: 30
    offline_timeout: 90
    progress:
      mode: batch
      report_batch_size: 50
      report_interval: 5
    health:
      min_sample_size: 20
      healthy_threshold: 0.9
      suspected_threshold: 0.5
    healthy:
      max_concurrent_batches: 3
      report_batch_size: 50
      report_interval: 5
    suspected:
      max_concurrent_batches: 1
      report_batch_size: 1
      report_interval: 1
    unhealthy:
      max_concurrent_batches: 0
    cancel:
      websocket_timeout: 5
      smtp_graceful_shutdown: true

  scheduler:
    batch_size: 500
    max_concurrent_runners: 10
    lease_duration: 60
    lease_reaper_interval: 30

  quota:
    billing_mode: smtp_attempt

  spf:
    grace_period_minutes: 30
    verification_resolvers:
      - 1.1.1.1
      - 8.8.8.8
    active_ip_set_max_size: 50

  dkim:
    rate_limit:
      per_runner_per_minute: 1000
      per_domain_per_minute: 5000

  database:
    development: sqlite:///proxy.db
    production: postgresql://user:pass@localhost/mailrelay
```

---

# 16. 文件结构

```
/opt/proxy-manager/
|-- main.py
|-- config.yaml
|-- .env
|-- requirements.txt
|
|-- database/
|   |-- __init__.py
|   |-- database.py
|   +-- models.py
|
|-- auth/
|   |-- __init__.py
|   |-- session.py
|   |-- runner_token.py
|   +-- deps.py
|
|-- proxy/
|   |-- __init__.py
|   |-- api.py
|   |-- scheduler/
|   +-- templates/
|
|-- mail/
|   |-- __init__.py
|   |-- api/
|   |-- models/
|   |-- scheduler/
|   |-- cloudflare/
|   |-- signing/
|   +-- websocket/
|
+-- templates/
```

---

# 17. 实施阶段划分

| Phase | 内容 |
|-------|------|
| 1 | 基础设施 + Auth（15+ 张表，Session + Bootstrap Token + Runner Token） |
| 2 | Runner 生命周期 + Health + SPF 生效验证 |
| 3 | MailTask + MailBatch + Quota（Snapshot + Idempotency） |
| 4 | Scheduler + Lease + 多 Batch |
| 5 | Go Agent + SMTP（含 MailDeliveryAttempt 记录） |
| 6 | 故障转移 + Cancel + Results CSV |
| 7 | SPF 子域名 + DKIM 签名 + Sender Identity |
| 8 | Admin + Audit + Observability |

---

# 18. 架构边界

| 负责 | Oracle Control Plane | Mail Runner Agent (Go) |
|------|----------------------|------------------------|
| 用户管理 | Y | N |
| 额度管理（原子 CONSUME + 幂等约束） | Y | N |
| 任务创建/拆分 | Y | N |
| Batch 调度（claim-next） | Y | N |
| Runner 生命周期 | Y | N |
| Runner 健康评定 | Y | N |
| DKIM 签名（RFC 6376） | Y | N |
| SPF 管理（原子计数） | Y | N |
| SMTP 投递（TCP connect+） | N | Y |
| 进度上报 | N | Y |
| 投递结果上报 | N | Y |

---

# 19. V2.5.1 变更日志

| # | 修订 | 优先级 | 类型 |
|---|------|:------:|------|
| 1 | STARTED + Quota CONSUME + QuotaTransaction 同一数据库事务 | P0 | 事务 |
| 2 | 新增 PRE_ATTEMPT_FAILURE 状态 | P0 | 状态机 |
| 3 | RECOVERING 增加 TEMPORARY_FAILURE 处理 | P1 | 状态机 |
| 4 | Attempt Number 生成与 MailDelivery 行锁绑定 | P1 | 并发 |
| 5 | 澄清 attempted_count 语义（recipient 数 vs Attempt 数） | P2 | 文档 |
| 6 | 收件人标准化（Recipient Canonicalization） | P2 | 功能 |
| 7 | DKIM 私钥生产环境加密存储（systemd LoadCredential） | P2 | 安全 |

---

# 20. V1.0 到 V2.5 全版本修订总览

| 版本 | 日期 | 主要变更数 | 核心贡献 |
|------|------|:----------:|---------|
| V1.0 | 07-22 | — | 初始架构：Oracle Control Plane + Runner Data Plane |
| V2.0 | 07-23 | 12+ | 认证隔离、数据库模型、Quota 事务模型 |
| V2.1 | 07-23 | 15 | Bootstrap Token、MailDeliveryAttempt、Lease、Snapshot、健康评分、SPF Grace Period |
| V2.2 | 07-23 | 8 | MailDeliveryAttempt 模型、Cancel 修正、Heartbeat/Lease 分离、原子事务、PostgreSQL、DKIM 不下发 |
| V2.3 | 07-23 | 4 | DKIM 远程签名 API（方案B）、Quota 防重复扣费、SPF Active IP Set |
| V2.4 | 07-23 | 5 | DKIM RFC 6376 规范、Runner Token Scope、Quota 原子 CONSUME、状态机完整定义、SPF 简化 |
| V2.5 | 07-23 | 6 | Billing Boundary 修正、幂等约束、RECOVERING 层级修正、attempt_number 唯一约束、RETRY_EXHAUSTED、SPF 原子更新 |
| V2.5.1 | 07-24 | 7 | 原子事务修正、PRE_ATTEMPT_FAILURE 状态、RECOVERING 补充、行锁绑定、attempted_count 语义、收件人标准化、DKIM 加密存储 |

---

## 架构冻结声明

**V2.5.1 是实现前最终修订版本。此后不再进行任何架构/模型/状态机级别的修改，直接进入代码实现阶段。**

*文档版本 V2.5.1（实现前最终修订版本）— 2026-07-24*
