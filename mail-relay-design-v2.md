# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.0

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-07-22 | 初始架构设计 |
| V2.0 | 2026-07-23 | 拆分认证/数据库/RBAC/Quota 详细设计 |

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

核心设计原则：

- **Oracle 是 Control Plane**
- **Mail Runner Agent 是 Data Plane（Go 实现）**
- **Proxy Runner 保持现有 frpc/frps 不变**
- **SMTP 流量不经过 Oracle**
- **Agent 公网 IP 由 Agent 上报，Oracle 管理 SPF**

---

# 3. 技术栈

## 3.1 Oracle Control Plane

| 组件 | 技术 | 说明 |
|------|------|------|
| Web 框架 | Python FastAPI | 现有技术栈延续 |
| ORM | SQLAlchemy | 现有技术栈延续 |
| 数据库 | PostgreSQL（推荐）/ SQLite | 生产用 PG，开发可用 SQLite |
| Session | Server-side Session (Redis/Database) | HttpOnly Secure Cookie |
| 实时通信 | WebSocket | Runner 控制通道 |
| SPF | Cloudflare API | 引用计数管理 |
| Runner 触发 | GitHub API | 触发 workflow |

## 3.2 Mail Runner Agent

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Go | goroutine + channel + context |
| SMTP | net/smtp 或第三方库 | 并发发送 |
| 通信 | HTTPS + WebSocket | Oracle 控制通道 |
| 部署 | GitHub Runner | 临时实例 |

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
        ≠
管理员 Session
        ≠
用户 API Token
        ≠
Mail Runner Token
        ≠
GitHub PAT
        ≠
Cloudflare API Token
```

**每一种凭证只拥有完成自己工作的最小权限。**

- Mail Runner Token 只能访问自身的注册、心跳、任务领取、进度和结果接口
- 不能读取其他 Runner 的任务
- 不能访问用户数据
- 不能访问管理员 API
- 不能操作 SPF
- Cloudflare Token 和 GitHub PAT 只保存在 Oracle，绝不下发到 Runner

## 4.2 认证架构

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

## 4.3 认证方式

| 接口 | 认证方式 | 说明 |
|------|---------|------|
| Web UI（用户） | Session Cookie | HttpOnly, Secure, SameSite=Lax |
| Web UI（管理员） | Session Cookie | 独立 Session |
| Runner API | Runner Token (Bearer) | 预签发，可吊销 |
| 第三方 API（未来） | API Token (sk_live_xxx) | 用户自行生成 |

## 4.4 Web 认证方案

不推荐纯 JWT，因为 JWT 一旦签发无法撤销。用户被管理员停用后 JWT 可能仍然有效。

采用 **Server-side Session**：

```
Cookie(session_id) → Redis/Database → User Identity
```

管理员停用用户 → `User.status = DISABLED` → 立即拒绝所有 Session。

## 4.5 Runner Credential

```python
class RunnerCredential:
    id: UUID
    runner_id: UUID
    token_hash: str          # 存储哈希，不存明文
    issued_at: datetime
    expires_at: datetime | None
    revoked_at: datetime | None
    last_used_at: datetime | None
```

Runner 启动流程：

```
Oracle 签发 Runner Credential
    → 作为 workflow 参数下发
    → Agent 启动后持有 Token
    → Authorization: Bearer RUNNER_TOKEN
    → Oracle 验证 token → runner_id → runner 状态
```

---

# 5. 数据库模型设计

## 5.1 模型总览

7 个核心领域：

| 领域 | 模型 | 说明 |
|------|------|------|
| User | User, UserApiToken | 用户与 API Token |
| Admin | Role, Permission | RBAC 权限 |
| Quota | QuotaAccount, QuotaTransaction | 额度管理 |
| MailTask | MailTask | 邮件任务 |
| MailBatch | MailBatch | 批次 |
| MailDelivery | MailDelivery | 投递结果 |
| MailRunner | MailRunner, RunnerCredential, RunnerHealth | Runner 管理 |

辅助模型：

| 模型 | 说明 |
|------|------|
| AuditLog | 审计日志 |
| Session | Web Session 存储 |

## 5.2 User 模型

```python
class User(Base):
    __tablename__ = "users"

    id: UUID              # primary key
    username: str          # unique
    email: str             # unique
    password_hash: str

    status: UserStatus     # ACTIVE / DISABLED / SUSPENDED
    role: UserRole         # USER / ADMIN

    created_at: datetime
    updated_at: datetime
    last_login_at: datetime | None
```

## 5.3 RBAC 模型

```python
class Role(Base):
    # ADMIN, USER, 后续可扩展 SUPER_ADMIN, OPERATOR, SUPPORT
    pass

class Permission(Base):
    # mail:send, mail:read, mail:cancel, runner:manage, user:manage
    pass
```

第一版只有 USER 和 ADMIN，但数据库结构提前支持 RBAC。

## 5.4 MailRunner 模型

```python
class MailRunner(Base):
    __tablename__ = "mail_runners"

    id: UUID
    runner_id: str           # 唯一标识
    name: str
    runner_type: str         # "mail"

    # 运行时状态
    status: RunnerStatus     # STARTING / REGISTERING / READY / BUSY / ...
    health_status: RunnerHealthStatus  # UNKNOWN / HEALTHY / SUSPECTED / UNHEALTHY

    # 网络
    public_ip: str
    ip_version: int

    # 环境
    operating_system: str
    architecture: str
    cpu_count: int | None
    memory_mb: int | None

    # Agent
    agent_version: str

    # GitHub
    github_runner_name: str
    workflow_run_id: str | None
    workflow_job_id: str | None

    # 生命周期
    registered_at: datetime
    heartbeat_at: datetime
    ready_at: datetime | None
    stopped_at: datetime | None

    # 当前工作
    current_batch_id: UUID | None

    # 健康指标（汇总）
    success_count: int
    failure_count: int
    attempted_count: int
    success_rate: float
    sample_count: int

    # 配置
    smtp_concurrency: int
    report_mode: str          # per_message / batch / interval
    report_batch_size: int
    report_interval_seconds: int

    # 时间戳
    created_at: datetime
    updated_at: datetime
```

### RunnerStatus

```python
class RunnerStatus(Enum):
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

### RunnerHealthStatus（独立维度）

```python
class RunnerHealthStatus(Enum):
    UNKNOWN = "unknown"
    HEALTHY = "healthy"
    SUSPECTED = "suspected"
    UNHEALTHY = "unhealthy"
```

**status 和 health_status 是独立的**，例如：

- `status = BUSY, health_status = HEALTHY` — 正常工作
- `status = BUSY, health_status = UNHEALTHY` — 立即停止接收新 Batch

## 5.5 RunnerHealth 模型

比单纯 success_rate 更有价值的是 **窗口健康数据**：

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

    temporary_failure_count: int
    permanent_failure_count: int

    timeout_count: int
    connection_error_count: int

    smtp_4xx_count: int
    smtp_5xx_count: int

    success_rate: float

    created_at: datetime
```

Oracle 可以判断：

```
5xx 比例 → 邮箱不存在 / 服务器拒绝
4xx 比例 → 临时失败 / 频率限制
Timeout 比例 → 网络问题
```

## 5.6 MailTask 模型

```python
class MailTask(Base):
    __tablename__ = "mail_tasks"

    id: UUID
    user_id: UUID

    status: str       # PENDING / PROCESSING / COMPLETED / FAILED / CANCELLED

    subject: str | None
    from_address: str

    template_type: str           # "raw_eml" / "editor"
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

## 5.7 MailBatch 模型

```python
class MailBatch(Base):
    __tablename__ = "mail_batches"

    id: UUID
    task_id: UUID
    runner_id: UUID | None

    status: str     # QUEUED / ASSIGNED / CLAIMED / RUNNING / COMPLETED / CANCELLING / CANCELLED / FAILED

    total_recipients: int

    attempted_count: int
    delivered_count: int
    failed_count: int
    not_attempted_count: int

    claimed_at: datetime | None
    started_at: datetime | None
    completed_at: datetime | None

    retry_count: int

    created_at: datetime
```

## 5.8 MailDelivery 模型

这是以后提供「哪些邮箱不存在」的核心：

```python
class MailDelivery(Base):
    __tablename__ = "mail_deliveries"

    id: UUID
    task_id: UUID
    batch_id: UUID

    recipient: str

    status: str     # DELIVERED / FAILED / NOT_ATTEMPTED / CANCELLED

    smtp_code: int | None
    smtp_enhanced_code: str | None    # 例如 "5.1.1"
    smtp_response: str | None

    error_type: str | None            # MAILBOX_NOT_FOUND / DOMAIN_NOT_FOUND / ...

    attempts: int

    first_attempt_at: datetime | None
    last_attempt_at: datetime | None
    completed_at: datetime | None
```

## 5.9 Quota 设计

不要简单在 User 表放 quota_limit / quota_used。

### QuotaAccount

```python
class QuotaAccount(Base):
    __tablename__ = "quota_accounts"

    user_id: UUID

    quota_limit: int       # 总配额
    available: int          # 可用
    reserved: int           # 已预占
    consumed: int           # 已消耗
```

### QuotaTransaction

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"

    id: UUID
    user_id: UUID
    task_id: UUID | None

    transaction_type: str    # RESERVE / CONSUME / RELEASE / ADJUST / REFUND
    amount: int

    created_at: datetime
```

### 流程示例

```
用户提交 1000 个收件人
    → RESERVE +1000（available -= 1000, reserved += 1000）

实际 SMTP 尝试 900 次
    → CONSUME 900（reserved -= 900, consumed += 900）
    → RELEASE 100（reserved -= 100, available += 100）

最终：consumed = 900, available = +100
```

审计清晰。

## 5.10 UserApiToken（未来扩展）

```python
class UserApiToken(Base):
    __tablename__ = "user_api_tokens"

    id: UUID
    user_id: UUID

    token_hash: str          # 只存哈希
    name: str                # 用户可命名

    permissions: str         # JSON: ["mail:send", "mail:read"]

    created_at: datetime
    expires_at: datetime | None
    last_used_at: datetime | None
    revoked_at: datetime | None
```

---

# 6. API 设计

## 6.1 用户 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /auth/register | 注册 |
| POST | /auth/login | 登录 |
| POST | /auth/logout | 登出 |
| GET  | /auth/me | 当前用户 |
| PUT  | /auth/password | 修改密码 |

## 6.2 任务 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/tasks | 创建任务 |
| GET  | /api/tasks | 任务列表 |
| GET  | /api/tasks/{id} | 任务详情 |
| POST | /api/tasks/{id}/cancel | 取消任务 |
| GET  | /api/tasks/{id}/results | 下载结果 CSV |
| GET  | /api/tasks/{id}/failures | 下载失败列表 |

## 6.3 Runner API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/mail-runners/register | 注册 |
| POST | /api/mail-runners/{id}/heartbeat | 心跳 |
| GET  | /api/mail-runners/{id}/config | 获取配置 |
| GET  | /api/mail-runners/{id}/jobs/next | 领取任务 |
| POST | /api/mail-batches/{id}/claim | 认领 Batch |
| POST | /api/mail-batches/{id}/progress | 上报进度 |
| POST | /api/mail-batches/{id}/results | 上报结果 |
| POST | /api/mail-batches/{id}/complete | 完成 |

## 6.4 管理员 API

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

# 7. Mail Runner Agent 设计（Go）

## 7.1 启动流程

```
GitHub Runner 启动
    → 下载/启动 Mail Runner Agent
    → Agent 获取公网 IP
    → Agent 向 Oracle 注册
    → Oracle 验证 Runner Credential
    → Oracle 保存 IP，更新 SPF
    → Runner 状态 = READY
```

## 7.2 Agent 职责

- 获取公网 IP
- 注册
- 发送 Heartbeat（每 30 秒）
- 建立 WebSocket 控制通道
- 获取任务 / 认领 Batch
- 下载任务数据（收件人列表 + 邮件模板）
- SMTP 投递（Go goroutine 并发）
- 上报进度（每 50 封或每 5 秒）
- 上报投递结果（每 100~500 条批量）
- 处理取消 / 暂停 / 配置更新
- 断线恢复（Grace Period 30s）

## 7.3 Agent 不负责

- 用户管理
- 额度管理
- 任务创建
- Runner 生命周期管理
- SPF 管理
- Cloudflare API
- 其他 Runner 管理

## 7.4 通信方式

```
HTTPS API:
    注册
    心跳
    任务 Claim
    进度
    结果

WebSocket:
    CANCEL
    PAUSE
    RESUME
    CONFIG_UPDATE
    STOP
```

## 7.5 心跳

```
POST /api/mail-runners/{runner_id}/heartbeat
{
  "runner_id": "runner-001",
  "status": "READY",
  "public_ip": "1.2.3.4",
  "current_job": null
}
```

- Heartbeat：30 秒
- Offline 判定：90 秒无心跳

---

# 8. Mail Scheduler

## 8.1 调度逻辑

```
用户创建 MailTask（10000 收件人）
    → 预占额度
    → 拆分为 MailBatch（batch_size = 500）
    → Batch 001~020 进入 QUEUED

Scheduler 循环：
    → 寻找 READY 状态的 Mail Runner
    → 分配 QUEUED 的 Batch 给 Runner
    → Runner 认领后进入 CLAIMED
```

## 8.2 多 Runner 并行

```
5 个 READY Runner → 5 个 Batch 并行发送
         ↓
任意 Runner 完成 → 领取下一个 QUEUED Batch
```

## 8.3 进度上报模式

| 模式 | 说明 |
|------|------|
| PER_MESSAGE | 每发送一封即上报 |
| BATCH | 每 N 封上报一次 |
| INTERVAL | 每 N 秒上报一次 |

支持组合规则：达到 50 封 **或** 超过 5 秒 → 立即上报

## 8.4 动态进度策略

Oracle 可动态修改上报配置：

```
正常: batch_size = 50, interval = 5s
异常: batch_size = 1, interval = 1s（逐封上报）
```

---

# 9. Runner 生命周期

## 9.1 状态机

```
STARTING → REGISTERING → VERIFYING → READY ↔ BUSY
                                          │
                                          ├→ DRAINING → STOPPING → OFFLINE
                                          ├→ SUSPENDED
                                          └→ HEARTBEAT_TIMEOUT → OFFLINE
```

## 9.2 健康检测

Oracle 根据窗口数据判断健康状态：

```
最近 100 次：
  成功 ≥ 95  → HEALTHY
  成功 50~95 → SUSPECTED
  成功 < 50  → UNHEALTHY
```

UNHEALTHY + BUSY → 立即停止接收新 Batch

---

# 10. SPF 管理

## 10.1 流程

```
Runner 注册
    → Oracle 获取 IP
    → Cloudflare API 更新 SPF
    → DNS 验证
    → Runner = READY
```

## 10.2 引用计数

```python
class SpfIpRecord(Base):
    ip: str
    runner_count: int   # 引用计数
    active: bool
```

只有 `runner_count = 0` 时才从 SPF 删除该 IP。

**Cloudflare Token 和 GitHub PAT 只保存在 Oracle，绝不下发到 Runner。**

---

# 11. Runner 故障转移

```
Runner A 突然离线
    → Oracle 检测 Heartbeat 超时
    → Runner A = OFFLINE
    → Batch 001 已尝试 180 封，剩余 320 封
    → 重新拆分为 Batch 001-Retry-A / B
    →分配给 Runner B / C
    → 已 SMTP 尝试的地址不重复发送
```

---

# 12. 用户取消任务

```
用户点击取消
    → MailTask = CANCELLING
    → WebSocket 下发 CANCEL
    → Agent 停止 SMTP 操作
    → 上报最终进度
    → MailTask = CANCELLED
    → 释放未尝试收件人的预占额度
```

---

# 13. 管理员功能

```
用户管理
├── 用户列表
├── 用户详情
├── 发送额度
├── 已用额度
├── 停用账户
└── 恢复账户

任务管理
├── 全部任务
├── 任务状态
├── 取消任务
└── 投递结果

Runner 管理
├── Runner 列表
├── Runner IP
├── Runner 状态
├── 当前任务
├── 健康状态
├── 暂停
├── 恢复
└── 停止

系统配置
├── Batch Size
├── 最大 Runner 数
├── Progress Report Mode
├── Report Batch Size
└── Report Interval
```

---

# 14. Token 与凭证总结

| 凭证类型 | 用途 | 存储位置 | 可否撤销 |
|----------|------|---------|:--------:|
| User Session | Web 登录 | Redis/Database | ✅ |
| Admin Session | 管理后台 | Redis/Database | ✅ |
| User API Token | 第三方 API | 数据库（哈希） | ✅ |
| Mail Runner Token | Runner 认证 | 数据库（哈希） | ✅ |
| GitHub PAT | 触发 Workflow | config.yaml | ✅ |
| Cloudflare Token | SPF 管理 | config.yaml | ✅ |

---

# 15. 推荐默认配置

```yaml
mail:
  runner:
    heartbeat_interval: 30
    offline_timeout: 90
    progress:
      mode: batch
      batch_size: 50
      max_interval: 5
    suspicious:
      failure_rate_threshold: 0.5
      min_sample_size: 20
    cancel:
      websocket_timeout: 5
      smtp_graceful_shutdown: true

  scheduler:
    batch_size: 500
    max_concurrent_runners: 10

  quota:
    billing_mode: smtp_attempt
```

---

# 16. 文件结构（Oracle Backend）

```
/opt/proxy-manager/
├── main.py                    # FastAPI 入口
├── config.yaml                 # 配置文件
├── requirements.txt
│
├── database/
│   ├── __init__.py
│   ├── database.py             # 数据库连接
│   └── models.py               # 所有模型
│
├── api/
│   ├── __init__.py
│   ├── proxy.py                # 代理 API（现有）
│   ├── auth.py                 # 认证 API
│   ├── mail.py                 # 邮件任务 API
│   ├── admin.py                # 管理员 API
│   └── mail_runners.py         # Runner API
│
├── scheduler/
│   ├── __init__.py
│   ├── allocator.py            # 端口分配（现有）
│   ├── monitor.py              # Proxy 监控（现有）
│   ├── cleanup.py              # 清理（现有）
│   └── mail_scheduler.py       # Mail 调度器
│
├── cloudflare/
│   ├── __init__.py
│   └── spf_manager.py          # SPF 管理
│
├── websocket/
│   ├── __init__.py
│   └── mail_ws.py              # Runner 控制通道
│
├── auth/
│   ├── __init__.py
│   ├── session.py              # Session 管理
│   └── runner_token.py         # Runner Token 管理
│
└── templates/
    ├── workflow.yml.j2         # Proxy workflow（现有）
    └── mail-runner.yml.j2      # Mail Runner workflow
```

---

# 17. 实施阶段划分

## Phase 1：基础设施

- 数据库模型定义（所有 10+ 张表）
- 数据库迁移
- 配置文件扩展
- 认证模块（Session + Runner Token）
- 基础 API 路由框架

## Phase 2：Mail Runner 生命周期

- Runner 注册/心跳/状态管理
- Mail Scheduler 调度器
- Runner 健康检测
- SPF Manager
- Mail Runner API

## Phase 3：任务管理

- MailTask / MailBatch 创建
- Batch 拆分
- 额度预占
- Web 管理界面

## Phase 4：SMTP 发送 + 结果

- Mail Runner Agent（Go）
- 进度上报
- 投递结果存储
- 取消任务流程
- 用户报告/CSV 导出

## Phase 5：增强功能

- WebSocket 控制通道
- 动态进度策略
- Runner 故障转移
- 管理员面板完善
- 审计日志

---

# 18. 设计原则总结

1. **Oracle 是 Control Plane，Mail Runner 是 Data Plane**
2. **SMTP 流量不经过 Oracle**
3. **每种凭证拥有最小权限**
4. **User Token ≠ Runner Token ≠ Admin Token**
5. **Cloudflare / GitHub 凭证只存在 Oracle，不下发**
6. **status 和 health_status 独立判断**
7. **Runner 健康基于窗口数据而非简单 success_rate**
8. **Quota 使用 RESERVE → CONSUME/RELEASE 事务模型**
9. **已 SMTP 尝试的地址不重复发送**
10. **所有投递结果持久化，支持导出**

---

*文档版本 V2.0 — 2026-07-23*
