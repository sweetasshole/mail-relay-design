# Mail Relay Platform + Multi Mail Runner - 实施设计文档 V2.4

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-07-22 | 初始架构设计 |
| V2.0 | 2026-07-23 | 拆分认证/数据库/RBAC/Quota |
| V2.1 | 2026-07-23 | 架构评审：Token、MailDelivery、Lease、Snapshot、Health、SPF、DKIM |
| V2.2 | 2026-07-23 | MailDeliveryAttempt、Cancel修正、Lease分离、原子事务、PostgreSQL、GitHub Hosted Runner |
| V2.3 | 2026-07-23 | DKIM远程签名API、Quota防重复扣费、SPF Active IP Set |
| V2.4 | 2026-07-23 | 实现级细化：DKIM规范的签名流程、权限模型、Quota原子事务、状态机完整定义、SPF简化 |

---

# 一、DKIM Signing API 规范修正（P0）

## 1.1 问题

V2.3 中的 DKIM API 只是对 headers 数组 + body_hash 做简单签名，不符合 RFC 6376 标准。DKIM 签名必须遵循严格的 canonicalization 规则构建签名输入。

## 1.2 签名流程（RFC 6376）

```
邮件 Body
    |
    v
Canonicalization（relaxed/simple）
    |
    v
Body Hash（bh=）
    |
    v
构建 DKIM-Signature Header（b= 留空）
    |  包含 v=, a=, c=, d=, s=, h=, bh=, l= 等
    v
对 Canonicalized 后的 Header 签名数据做 RSA-SHA256
    |
    v
填充 b=，生成完整 DKIM-Signature
```

Key 点：

- 签名输入不是 headers 数组，而是 canonicalized DKIM-Signature header 数据
- h= 必须明确列出参与签名的 header 列表
- bh= 是 canonicalized body 的 sha256 hash
- c= 指定 canonicalization 算法（relaxed/relaxed 推荐）

## 1.3 API 修正

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
    "Message-ID": "<uuid@yourdomain.com>",
    "Date": "Thu, 23 Jul 2026 12:00:00 +0000"
  },
  "body_hash": "base64(sha256(canonicalized_body))"
}

Response:
{
  "dkim_signature": "DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=yourdomain.com; s=default; h=From:To:Subject:Message-ID:Date; bh=...; b=..."
}
```

## 1.4 Oracle DKIM Signer 职责

```
Oracle DKIM Signer 负责：
  1. 验证 Runner Token
  2. 加载 DKIM 私钥
  3. 根据 RFC 6376 构建 canonicalized 签名输入
  4. 执行 RSA-SHA256 签名
  5. 返回完整 DKIM-Signature header

Runner 负责：
  1. 计算 canonicalized body 的 sha256 hash
  2. 将 Oracle 返回的 DKIM-Signature 插入邮件 header
  3. SMTP 发送
```

---

# 二、DKIM Signing Token / Runner Token 权限模型（P0）

## 2.1 问题

V2.3 存在两种设计混用的问题：Token 总结中有 DKIM Signing Token，但认证架构并未定义其数据模型。

## 2.2 决策：使用 Runner Token 权限范围（Scope），不引入新 Token

```
Mail Runner Token
    |
    +-- scope: ["heartbeat", "claim", "progress", "result", "dkim:sign"]
    |
    v
DKIM Signing API 验证 token 中的 dkim:sign scope
```

## 2.3 RunnerCredential 模型更新

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

## 2.4 不再需要 DkimSigningToken

V2.4 不再增加独立的 DkimSigningToken 模型。Runner Token scope 控制一切。

---

# 三、Quota 原子事务（P0）

## 3.1 问题

V2.3 的 CONSUME 操作在并发场景下可能将 reserved 变为负数。

## 3.2 原子 CONSUME

```sql
UPDATE quota_accounts
SET reserved = reserved - 1,
    consumed = consumed + 1
WHERE user_id = :user_id
  AND reserved >= 1;
```

affected_rows == 1 才允许继续 Consume，否则返回 QUOTA_INSUFFICIENT。

## 3.3 完整流程

```
1. INSERT MailDeliveryAttempt (status=CREATED)
2. 原子 UPDATE quota_accounts (reserved >= 1)
3. 如果 affected_rows == 0 -> QUOTA_INSUFFICIENT
4. 如果 affected_rows == 1 -> INSERT QuotaTransaction (CONSUME 1)
5. 启动 SMTP
```

步骤 2 和 3 在同一数据库事务中，保证一致性。

---

# 四、MailDeliveryAttempt 完整状态机（P0）

## 4.1 状态定义

```
CREATED               <- 记录已创建，待开始（不计费）
    |
    v
STARTED               <- Agent 开始 SMTP 投递（计费点）
    |
    +-- DELIVERED           <- 250 OK
    +-- PERMANENT_FAILURE   <- 5xx（不重试）
    +-- TEMPORARY_FAILURE   <- 4xx（可重试）
    +-- UNKNOWN             <- Agent 崩溃（不自动重试）
```

## 4.2 状态对应 SMTP 响应码

| 状态 | SMTP 码范围 | 说明 |
|------|-------------|------|
| DELIVERED | 250 | 成功 |
| PERMANENT_FAILURE | 5xx | 永久失败，不重试 |
| TEMPORARY_FAILURE | 4xx | 临时失败，可能重试 |
| UNKNOWN | N/A | Agent 崩溃/断线 |

## 4.3 重试策略

```
TEMPORARY_FAILURE
    |
    +-- 重试上限 = 3 次
    +-- 间隔: 60s, 300s, 900s
    +-- 3 次后仍失败 -> PERMANENT_FAILURE
    +-- 每次重试 = 新 Attempt = 新 CONSUME
```

---

# 五、SMTP Attempt 计费边界定义（P0）

## 5.1 Billing Attempt 定义

Billing Attempt = Agent 已开始对指定 recipient 执行 SMTP 投递操作。

产生 Attempt：
  - TCP connect 成功（建立 SMTP 连接）
  - SMTP EHLO/HELO 成功
  - MAIL FROM 执行（无论成败）
  - RCPT TO 执行（无论成败）
  - DATA 执行（无论成败）

不产生 Attempt：
  - DNS lookup 失败（未连接）
  - TCP connect 失败（未连接）
  - Runner 崩溃后无法确定状态

---

# 六、SPF Active IP Set 简化（P1）

## 6.1 简化方案

```python
class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"
    ip: str
    active_runner_count: int
    grace_period_until: datetime | None
    created_at: datetime
    updated_at: datetime
```

## 6.2 逻辑

```
Runner 上线：active_runner_count += 1; grace_period_until = None
Runner 下线：active_runner_count -= 1
  if active_runner_count == 0: grace_period_until = now + 30min
Grace Period 到期：if active_runner_count == 0: 从 SPF 删除
```

## 6.3 Active IP 保护

当 Active IP 达到上限（50）：
  - 不删除活跃 Runner 的 IP
  - 拒绝新 Mail Runner 注册
  - 暂停启动新 Runner

---

# 七、MailBatch.runner_id 与 RunnerBatchAssignment 关系（P1）

MailBatch.runner_id = 当前分配的 Runner（实时状态）
RunnerBatchAssignment = 分配历史记录（审计）

两者不应冲突：
  - MailBatch.runner_id 始终等于最后一个 ACTIVE 状态的 Assignment
  - 认领时同时设置 MailBatch.runner_id 和创建 Assignment
  - 故障转移时更新 MailBatch.runner_id = 新 Runner
  - 旧 Assignment 变为 RELEASED

---

# 八、MailDelivery.runner_id 语义（P1）

MailDelivery.runner_id = 最后一次投递时的 Runner

投递历史查询：MailDeliveryAttempt.runner_id

如果重试发生在不同 Runner 上：
  - Attempt #1: Runner A
  - Attempt #2: Runner B
  - MailDelivery.runner_id = B（最终 Runner）

---

# 九、At-Least-Once 语义声明（P1）

系统保证 At-Least-Once Attempt Semantics，不保证 Exactly-Once SMTP Delivery。

这是 SMTP 分布式系统的固有特性。Agent 在 SMTP DATA 后崩溃可能导致：
  - SMTP 服务器已接受邮件（250 OK）
  - Agent 尚未上报结果
  - Oracle 标记为 UNKNOWN
  - 重新调度导致重复投递

UNKNOWN 处理策略：
  - 保守策略（默认）：UNKNOWN 不重试
  - 激进策略：UNKNOWN 允许重试
  - 由管理员在系统配置中设置

---

# 十、progress.report_batch_size 语义修正（P2）

```yaml
runner:
  progress:
    mode: batch
    report_batch_size: 50
    report_interval: 5

health:
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
```

---

# 十一、claim-next API 统一（P2）

统一为：POST /api/mail-runners/{id}/claim-next

Oracle 内部：
  BEGIN;
  SELECT id FROM mail_batches WHERE status = QUEUED
  ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED;
  UPDATE mail_batches SET status = CLAIMED, runner_id = :id;
  INSERT runner_batch_assignments;
  COMMIT;

返回 Batch ID 或 204 No Content。

---

# 十二、RECOVERING 完整状态机（P2）

```
RECOVERING
    |
    v
Oracle 分析每个 recipient：
    |
    +-- NOT_ATTEMPTED -> CREATED（重新入队）
    +-- ATTEMPTING -> UNKNOWN（不确定）
    +-- DELIVERED -> 不处理
    +-- FAILED -> 不处理
    |
    v
如果所有 recipient 已处理 -> COMPLETED
如果还有待发送 -> QUEUED（Scheduler 重新分配）
```

---

# 十三、DKIM Key Rotation 与生产安全（P2）

## 13.1 Key Rotation

支持多 selector，实现无缝轮换：

阶段 1：使用 selector1
阶段 2：生成 selector2 密钥，上传公钥至 DNS
阶段 3：切换签名使用 selector2，旧邮件仍可通过 selector1 验证
阶段 4：删除 selector1 DNS 记录

## 13.2 生产安全

V2.4 推荐：
  - 私钥文件：/etc/opt/proxy-manager/dkim/private.pem
  - 权限：root:root 600
  - 未来可迁移至 HashiCorp Vault / AWS Secrets Manager

---

# 十四、V2.4 小结

| 编号 | 修订 | 优先级 |
|------|------|:------:|
| 1 | DKIM Signing API RFC 6376 规范修正 | P0 |
| 2 | Runner Token scope 统一权限模型 | P0 |
| 3 | Quota 原子 CONSUME 事务 | P0 |
| 4 | MailDeliveryAttempt 完整状态机（+CREATED） | P0 |
| 5 | SMTP Attempt 计费边界定义 | P0 |
| 6 | SPF Active IP 简化（active_runner_count + grace_period_until） | P1 |
| 7 | MailBatch.runner_id 与 Assignment 关系 | P1 |
| 8 | MailDelivery.runner_id 语义 | P1 |
| 9 | At-Least-Once 语义声明 | P1 |
| 10 | progress.report_batch_size 语义修正 | P2 |
| 11 | claim-next API 统一 | P2 |
| 12 | RECOVERING 完整状态机 | P2 |
| 13 | DKIM Key Rotation + 生产安全 | P2 |

---

# 十五、V2.4 变更的数据库模型

```python
class MailDeliveryAttempt(Base):
    __tablename__ = "mail_delivery_attempts"
    id: UUID
    delivery_id: UUID
    task_id: UUID
    batch_id: UUID
    runner_id: UUID
    attempt_number: int
    status: str  # CREATED/STARTED/DELIVERED/PERMANENT_FAILURE/TEMPORARY_FAILURE/UNKNOWN
    smtp_code: int | None
    smtp_enhanced_code: str | None
    smtp_response: str | None
    error_type: str | None
    started_at: datetime
    completed_at: datetime | None

class RunnerCredential(Base):
    __tablename__ = "runner_credentials"
    id: UUID
    runner_id: UUID
    token_hash: str
    scope: str  # JSON array
    issued_at: datetime
    expires_at: datetime | None
    revoked_at: datetime | None
    last_used_at: datetime | None

class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"
    ip: str
    active_runner_count: int
    grace_period_until: datetime | None
    created_at: datetime
    updated_at: datetime
```

---

*文档版本 V2.4 - 2026-07-23*
