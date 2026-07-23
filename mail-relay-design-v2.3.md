# Mail Relay Platform + Multi Mail Runner - 实施设计文档 V2.3

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-07-22 | 初始架构设计 |
| V2.0 | 2026-07-23 | 拆分认证/数据库/RBAC/Quota 详细设计 |
| V2.1 | 2026-07-23 | 架构评审修订：Token 安全、MailDelivery 状态、Batch Lease、邮件 Snapshot、健康检测、SPF、DKIM |
| V2.2 | 2026-07-23 | MailDeliveryAttempt、Cancel 修正、Heartbeat/Lease 分离、原子事务、PostgreSQL、GitHub Hosted Runner |
| V2.3 | 2026-07-23 | DKIM 远程签名 API、Quota 防重复扣费、SPF Active IP Set（本版） |

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
        ------------------------+-----------------------
        |                       |                       |
        v                       v                       v
   Auth/Users            Mail Scheduler         Runner Manager
        |                       |                       |
        |                       |                 ------+------
        |                       |                 |           |
        |                       |            Proxy Runner  Mail Runner
        |                       |            (frpc/frps)   (Go Agent)
        |                       |                 |           |
        |                       |                 |      SMTP Direct
        |                       |                 |           |
        |                       |                 |           v
        |                       |                 |      Internet MX
        |                       +--- Task / Batch / Delivery
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
9. **Quota 使用 RESERVE -> CONSUME/RELEASE 事务模型**
10. **每个 Task+Recipient 唯一约束，避免重复投递**
11. **每次 SMTP Attempt 有独立生命周期记录**
12. **Batch 使用 Lease 机制实现超时回收，Heartbeat 与 Lease 分离**
13. **邮件内容在任务创建时 Snapshot，不可变，大内容走对象存储**
14. **所有投递结果持久化，支持导出**
15. **开发环境 SQLite，生产环境 PostgreSQL**

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
| 数据库 | **PostgreSQL（生产）/ SQLite（开发）** | V2.3 明确：并发事务需要 PG 行级锁 |
| Session | Server-side Session (Database/Redis) | HttpOnly Secure Cookie |
| 实时通信 | WebSocket | Runner 控制通道 |
| SPF | Cloudflare API | 引用计数管理 |
| Runner 触发 | GitHub API | 触发 workflow |
| 对象存储 | 本地文件系统 / S3 兼容 | 邮件内容分发 |
| DKIM 签名 | Oracle 侧签名 API | 私钥不下发 Runner |

## 3.2 Mail Runner Agent

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Go | goroutine + channel + context |
| SMTP | net/smtp 或第三方库 | 并发发送 |
| 通信 | HTTPS + WebSocket | Oracle 控制通道 |
| 部署 | **GitHub Hosted Runner** | V2.3 明确：IP 动态，需 SPF Active Set |

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
  | 3. 生成长期 Runner Token
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

## 4.3 认证架构

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

## 4.4 Web 认证方案

采用 **Server-side Session**（非 JWT）：

```
Cookie(session_id) -> Database -> User Identity
```

管理员停用用户 -> User.status = DISABLED -> 立即拒绝所有 Session。

## 4.5 敏感凭证存储

```bash
# /opt/proxy-manager/.env (chmod 600, .gitignore)
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
| Quota | QuotaAccount, QuotaTransaction | 额度管理 |
| MailTask | MailTask, MailTemplateSnapshot | 邮件任务与内容快照 |
| MailBatch | MailBatch, RunnerBatchAssignment | 批次与并行分配 |
| MailDelivery | MailDelivery | 投递结果（唯一约束） |
| MailDeliveryAttempt | MailDeliveryAttempt | **V2.2 新增：每次 SMTP Attempt 独立记录** |
| MailRunner | MailRunner, RunnerBootstrapToken, RunnerCredential, RunnerHealth | Runner 管理 |
| Identity | SenderIdentity | 发件身份验证 |
| SPF | SpfIpRecord | SPF 引用计数 |
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
HEALTHY   -> max_concurrent_batches = 3
SUSPECTED -> max_concurrent_batches = 1
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
    status: str
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
    runner_id: UUID | None
    recipient: str
    status: str
    attempts: int
    first_attempt_at: datetime | None
    last_attempt_at: datetime | None
    completed_at: datetime | None
    __table_args__ = (
        UniqueConstraint(task_id, recipient, name="uq_task_recipient"),
    )
```

### 投递状态机

```
NOT_ATTEMPTED
    |
    +-- 用户取消 -> CANCELLED
    |
    v
ATTEMPTING
    |
    +-- DELIVERED           <- SMTP 250
    +-- PERMANENT_FAILURE   <- SMTP 5xx
    +-- TEMPORARY_FAILURE   <- SMTP 4xx
    +-- UNKNOWN             <- Agent 崩溃
    +-- (不能直接 CANCELLED)
```

**关键规则：**

- NOT_ATTEMPTED -> 用户取消 -> CANCELLED
- ATTEMPTING -> 用户取消 -> 不能直接 CANCELLED
  - Agent 应优雅结束当前 SMTP 操作
  - 拿到明确结果 -> DELIVERED / FAILURE
  - Agent 崩溃 -> UNKNOWN
- CANCELLED 只能用于从未开始 SMTP 的收件人

## 5.9 MailDeliveryAttempt（V2.2 新增 - 核心模型）

**每次 SMTP Attempt 的独立生命周期记录：**

```python
class MailDeliveryAttempt(Base):
    __tablename__ = "mail_delivery_attempts"
    id: UUID
    delivery_id: UUID
    task_id: UUID
    batch_id: UUID
    runner_id: UUID
    attempt_number: int
    status: str
    smtp_code: int | None
    smtp_enhanced_code: str | None
    smtp_response: str | None
    error_type: str | None
    started_at: datetime
    completed_at: datetime | None
```

### 与 Quota 的关系

```
MailDeliveryAttempt
    |
    +-- STARTED -> QuotaTransaction (CONSUME 1)
    |       +-- UNIQUE(attempt_id, CONSUME)
    |
    +-- DELIVERED -> MailDelivery.status = DELIVERED
    |
    +-- PERMANENT_FAILURE -> MailDelivery.status = PERMANENT_FAILURE
    |
    +-- UNKNOWN
            -> 不自动重试
            -> 管理员决定重试 -> 新 Attempt #2 -> 新 CONSUME
```

### 防重复扣费（V2.3）

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"
    id: UUID
    user_id: UUID
    task_id: UUID | None
    attempt_id: UUID | None
    transaction_type: str
    amount: int
    created_at: datetime
    __table_args__ = (
        UniqueConstraint(attempt_id, transaction_type,
                        name="uq_quota_attempt_type"),
    )
```

同一 attempt_id + CONSUME 只能存在一条，防止 Runner 重复上报导致重复扣费。

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

### QuotaTransaction

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"
    id: UUID
    user_id: UUID
    task_id: UUID | None
    attempt_id: UUID | None
    transaction_type: str
    amount: int
    created_at: datetime
    __table_args__ = (
        UniqueConstraint(attempt_id, transaction_type,
                        name="uq_quota_attempt_type"),
    )
```

### 流程

```
用户提交 1000 个收件人
    -> RESERVE +1000（available -= 1000, reserved += 1000）

每次 SMTP Attempt 开始
    -> CONSUME 1（reserved -= 1, consumed += 1）
    -> 关联 attempt_id

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

# 6. DKIM 设计（V2.3 最终方案：远程签名 API）

## 6.1 核心原则

**DKIM 私钥永久只保存在 Oracle，不下发到 Mail Runner。**

GitHub Hosted Runner 是临时实例，私钥下发存在泄露风险。

## 6.2 方案选择

| 方案 | 描述 | 选择 |
|------|------|:----:|
| 方案 A | Oracle 预签名整个 .eml -> Runner 直接 SMTP | - |
| **方案 B** | Runner 发送 Header+Body Hash -> Oracle 返回 Signature | **V2.3 选择** |

## 6.3 签名流程

```
Mail Runner Agent (Go)
    |
    | 1. 构建邮件 Header + Body
    | 2. 计算 DKIM Hash（canonicalized headers + body hash）
    | 3. POST /api/mail/dkim/sign
    |    { "domain": "yourdomain.com", "selector": "default",
    |      "headers": [...], "body_hash": "..." }
    v
Oracle DKIM Signer
    |
    | 1. 验证 Runner Token
    | 2. 加载 DKIM 私钥
    | 3. 对传入数据签名
    | 4. 返回 DKIM-Signature header
    v
Mail Runner
    |
    | 4. 将 DKIM-Signature 插入邮件 header
    | 5. SMTP 发送（完整邮件含 DKIM 签名）
    v
Internet MX
```

## 6.4 API 定义

```
POST /api/mail/dkim/sign

Request:
{
  "domain": "yourdomain.com",
  "selector": "default",
  "headers": [
    "from: sender@yourdomain.com",
    "to: recipient@example.com",
    "subject: Test"
  ],
  "body_hash": "base64(sha256(canonicalized_body))"
}

Response:
{
  "dkim_signature": "v=1; a=rsa-sha256; c=relaxed/relaxed; d=yourdomain.com; ..."
}
```

## 6.5 模型

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

## 6.6 签名限流

```yaml
dkim:
  rate_limit:
    per_runner_per_minute: 1000
    per_domain_per_minute: 5000
```

## 6.7 安全保证

- DKIM 私钥永远不离开 Oracle
- Runner Token 验证后才可调用签名 API
- 签名内容不包含私钥信息
- 私钥在 .env 中，chmod 600

---

# 7. SPF 管理（V2.3 修订：GitHub Hosted Runner + Active IP Set）

## 7.1 重要前提：Mail Runner 使用 GitHub Hosted Runner

GitHub Hosted Runner 的 IP 池非常大且动态变化：
- Runner 每次启动的 IP 可能不同
- IP 生命周期 = Runner 生命周期（通常几分钟到几小时）
- SPF 动态更新存在 DNS 传播延迟（TTL 300 秒）

## 7.2 SPF 子域名架构（推荐长期方案）

```
主域名 SPF（不频繁修改）：
v=spf1 include:_spf.yourdomain.com ~all

子域名 SPF（动态维护）：
_spf.yourdomain.com TXT "v=spf1 ip4:RunnerA ip4:RunnerB ... ~all"
```

## 7.3 Active IP Set + Grace Period（V2.3）

Runner 下线后 IP 不立即删除，保留 30 分钟（Grace Period）：

```python
class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"
    ip: str
    runner_count: int
    retained_count: int        # V2.3 新增：Grace Period 保留计数
    active: bool
    first_seen_at: datetime
    last_seen_at: datetime
    remove_after: datetime | None
```

### 流程

```
Runner A（IP 1.2.3.4）上线
    -> runner_count += 1
    -> 加入 SPF

Runner A 下线
    -> runner_count -= 1
    -> 如果 runner_count = 0
        -> retained_count = 1
        -> 保留在 SPF（Grace Period）
        -> remove_after = now + 30min
        -> 期间有新 Runner 使用该 IP -> 取消 Grace Period

Grace Period 到期
    -> 如果 runner_count + retained_count = 0
        -> 从 SPF 删除
```

### 最大 IP 限制

```
SPF Active IP Set 最大 50 个 IP
如果超出，优先删除 Grace Period 中最老的 IP
```

## 7.4 SPF 生效验证

```
Runner IP 注册
    -> 更新 SPF
    -> 从多个公共 DNS Resolver 查询（1.1.1.1, 8.8.8.8）
    -> 确认 SPF 已可见
    -> Runner = READY
```

> 注意：SPF 生效验证只能证明 Oracle 自己查询到新记录，不能保证所有收件服务器已看到。Grace Period 机制用于缓解此问题。

## 7.5 配置

```yaml
spf:
  grace_period_minutes: 30
  verification_resolvers:
    - 1.1.1.1
    - 8.8.8.8
  active_ip_set_max_size: 50
```

---

# 8. Mail Runner Agent 设计（Go）

## 8.1 启动流程

```
GitHub Hosted Runner 启动
    -> 下载/启动 Mail Runner Agent
    -> Agent 获取公网 IP
    -> Agent 使用 BOOTSTRAP_TOKEN 向 Oracle 注册
    -> Oracle 验证 Bootstrap Token（5 分钟 TTL）
    -> Oracle 返回长期 Runner Token
    -> Bootstrap Token 立即失效
    -> Oracle 保存 IP，更新 SPF
    -> SPF 生效验证
    -> Agent 开始发送 Heartbeat
    -> Runner 状态 = READY
```

## 8.2 Agent 职责

- 获取公网 IP
- 使用 Bootstrap Token 注册
- 发送 Heartbeat（每 30 秒）
- 建立 WebSocket 控制通道
- 获取任务 / 认领 Batch
- 下载邮件内容（Oracle Presigned URL）
- SMTP 投递（Go goroutine 并发）
- 上报进度（每 50 封或每 5 秒）
- 批量上报投递结果
- 处理 CANCEL / PAUSE / CONFIG_UPDATE
- 断线恢复（Grace Period 30s）

## 8.3 Agent 不负责

- 用户管理
- 额度管理
- 任务创建
- Runner 生命周期管理
- SPF 管理 / Cloudflare API
- DKIM 签名
- 其他 Runner 管理

## 8.4 通信方式

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

## 8.5 Heartbeat 与 Lease 分离（V2.2 关键设计）

```
Runner Heartbeat（每 30 秒）：
    -> 仅证明 Runner 活着
    -> 不刷新任何 Batch Lease

Batch Progress（每 50 封或每 5 秒）：
    -> 刷新该 Batch 的 lease_expires_at
    -> 证明该 Batch 正在处理

Lease Reaper（每 30 秒）：
    -> 扫描过期 Lease
    -> 原子 UPDATE 回收
```

---

# 9. Mail Scheduler

## 9.1 调度逻辑

```
用户创建 MailTask（10000 收件人）
    -> 预占额度
    -> 拆分为 MailBatch（batch_size = 500）
    -> Batch 001~020 进入 QUEUED

Scheduler 循环：
    -> 寻找 READY / BUSY 但未达到 max_concurrent_batches 的 Runner
    -> 分配 QUEUED 的 Batch
    -> 原子 UPDATE ... WHERE status = QUEUED
    -> 只有 affected_rows = 1 才算成功
```

## 9.2 原子 Claim

```sql
UPDATE mail_batches
SET status = CLAIMED, runner_id = :runner_id,
    claimed_at = NOW(), lease_expires_at = NOW() + INTERVAL 60 seconds
WHERE id = :batch_id AND status = QUEUED;
```

## 9.3 Lease Reaper

```sql
UPDATE mail_batches
SET status = RECOVERING
WHERE lease_expires_at < NOW()
AND status IN (CLAIMED, RUNNING);
```

## 9.4 动态并发控制

```
HEALTHY   -> max_concurrent_batches = 3, batch_size = 50, interval = 5s
SUSPECTED -> max_concurrent_batches = 1, batch_size = 1, interval = 1s
UNHEALTHY -> max_concurrent_batches = 0（停止分配）
```

---

# 10. Runner 生命周期

## 10.1 状态机

```
STARTING -> REGISTERING -> VERIFYING -> READY <-> BUSY
                                          |
                                          +-> DRAINING -> STOPPING -> OFFLINE
                                          +-> SUSPENDED
                                          +-> HEARTBEAT_TIMEOUT -> OFFLINE
```

## 10.2 Workflow 生命周期

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

# 11. 故障转移

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

---

# 12. 用户取消任务

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

# 13. API 设计

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
| GET  | /api/mail-runners/{id}/jobs/next | 领取下一个 Batch |
| POST | /api/mail-batches/{id}/claim | 认领 Batch |
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

| 凭证类型 | 用途 | 存储位置 | 可否撤销 | 备注 |
|----------|------|---------|:--------:|------|
| User Session | Web 登录 | Database | Y | HttpOnly Secure Cookie |
| Admin Session | 管理后台 | Database | Y | 独立 Session |
| User API Token | 第三方 API | 数据库（哈希） | Y | 用户自行生成 |
| Bootstrap Token | Runner 初始化 | 数据库（哈希） | Y | TTL 5 分钟，一次性 |
| Mail Runner Token | Runner 认证 | 数据库（哈希） | Y | 长期，可吊销 |
| DKIM Signing Token | Runner 调签名 API | 数据库（哈希） | Y | 注册时获得 |
| GitHub PAT | 触发 Workflow | .env（chmod 600） | Y | 不下发到 Runner |
| Cloudflare Token | SPF 管理 | .env（chmod 600） | Y | 不下发到 Runner |
| DKIM Private Key | 邮件签名 | .env（chmod 600） | Y | **不下发到 Runner** |

---

# 15. 推荐默认配置

```yaml
mail:
  runner:
    heartbeat_interval: 30
    offline_timeout: 90
    max_concurrent_batches: 3
    progress:
      mode: batch
      batch_size: 50
      max_interval: 5
    health:
      min_sample_size: 20
      healthy_threshold: 0.9
      suspected_threshold: 0.5
    cancel:
      websocket_timeout: 5
      smtp_graceful_shutdown: true

  scheduler:
    batch_size: 500
    max_concurrent_runners: 10
    lease_duration: 60
    lease_reaper_interval: 30
    recovery_delay: 30

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

# 18. V2.3 架构评审变更摘要

## 从 V2.1 到 V2.3 的所有修订

| # | 修订 | 级别 | 版本 |
|---|------|:----:|:----:|
| 1 | Runner Bootstrap Token 安全下发 | 必须 | V2.1 |
| 2 | MailDelivery ATTEMPTING 状态 + UNIQUE 约束 | 必须 | V2.1 |
| 3 | SMTP Attempt 计费定义 | 必须 | V2.1 |
| 4 | 健康评分规则统一（区分收件人错误和 SMTP 拒绝） | 必须 | V2.1 |
| 5 | 邮件内容 Snapshot | 必须 | V2.1 |
| 6 | Batch Lease 机制 | 必须 | V2.1 |
| 7 | .env 替代 config.yaml 存敏感凭证 | 必须 | V2.1 |
| 8 | SenderIdentity + DKIM 模型 | 必须 | V2.1 |
| 9 | **MailDeliveryAttempt 表** | 必须 | V2.2 |
| 10 | **Cancel 状态机修正**（ATTEMPTING 不能直接 CANCELLED） | 必须 | V2.2 |
| 11 | **Heartbeat 与 Lease 分离** | 必须 | V2.2 |
| 12 | **原子事务**（Batch Claim / Lease Reaper） | 必须 | V2.2 |
| 13 | **统计一致性公式** | 必须 | V2.2 |
| 14 | **DKIM 私钥不下发 Runner** | 必须 | V2.2 |
| 15 | **GitHub Hosted Runner SPF** | 必须 | V2.2 |
| 16 | **PostgreSQL 生产环境** | 必须 | V2.2 |
| 17 | **DKIM 远程签名 API（方案 B）** | 必须 | V2.3 |
| 18 | **Quota 防重复扣费** UNIQUE(attempt_id, transaction_type) | 必须 | V2.3 |
| 19 | **SPF Active IP Set**（retained_count + Grace Period） | 必须 | V2.3 |
| 20 | DKIM 签名限流 | 建议 | V2.3 |
| 21 | 内容对象存储 | 建议 | V2.2 |
| 22 | 动态并发控制 | 建议 | V2.2 |
| 23 | Phase 顺序优化 | 建议 | V2.2 |

---

*文档版本 V2.3 - 2026-07-23*
