# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.5.1（实现前最终修订）

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0~V2.5 | 2026-07-22~23 | 从初始架构到架构冻结（详见 V2.5 修订记录） |
| V2.5.1 | 2026-07-24 | **实现前最终修订**。不改变架构，只修正 4 个事务和状态机细节 |

---

## 核心业务不变量（重申）

> 一个 MailDelivery 可以拥有多个 MailDeliveryAttempt；
> 每一个实际产生 SMTP 投递尝试的 Attempt 最多只能对应一次 Quota CONSUME；
> 每一次 Retry 必须创建新的 Attempt 和新的 QuotaTransaction；
> 同一个 Attempt 无论 Runner 重复上报多少次，都不得重复扣除额度。

---

# 🔴 P0-1：STARTED + Quota CONSUME + QuotaTransaction 必须为同一数据库事务

## 问题

V2.5 中以下三个操作不在同一事务内：

```
TCP connect 成功
        ↓
STARTED                     ← 步骤 1
        ↓
UPDATE QuotaAccount         ← 步骤 2
        ↓
INSERT QuotaTransaction     ← 步骤 3
        ↓
执行 SMTP
```

如果步骤 2 和步骤 3 之间 Oracle 进程崩溃，会导致：
- QuotaAccount 已扣额度
- QuotaTransaction 记录不存在
- Runner 重试时系统无法判断额度是否已扣

## V2.5.1 修正

**必须将 STARTED + QuotaAccount CONSUME + QuotaTransaction INSERT 放到同一个数据库事务中：**

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

# 🔴 P0-2：新增 PRE_ATTEMPT_FAILURE 状态

## 问题

V2.5 的 Attempt 状态枚举为：

```
CREATED / STARTED / DELIVERED / PERMANENT_FAILURE / TEMPORARY_FAILURE / UNKNOWN
```

但业务流程中写：

```
DNS lookup 失败 -> CREATED -> FAILED
TCP connect 失败 -> CREATED -> FAILED
```

`FAILED` 不在状态枚举中，存在文档级状态冲突。

## V2.5.1 修正

新增状态：`PRE_ATTEMPT_FAILURE`

```python
class MailDeliveryAttemptStatus:
    CREATED                 # 记录已创建，尚未开始 SMTP（不计费）
    STARTED                 # TCP connect 成功，开始 SMTP（计费点）
    DELIVERED               # 250 OK
    PERMANENT_FAILURE       # 5xx 永久失败
    TEMPORARY_FAILURE       # 4xx 临时失败（可重试）
    UNKNOWN                 # Agent 崩溃
    PRE_ATTEMPT_FAILURE     # V2.5.1 新增：Billing Boundary 前的失败（DNS/TCP 失败，不计费）
```

### 完整状态转换

```
CREATED
    |
    +-- DNS lookup 失败  -> PRE_ATTEMPT_FAILURE（不计费）
    +-- TCP connect 失败 -> PRE_ATTEMPT_FAILURE（不计费）
    |
    v
STARTED（同一事务：CONSUME 1 + INSERT QuotaTransaction）
    |
    +-- DELIVERED           <- 250 OK
    +-- PERMANENT_FAILURE   <- 5xx
    +-- TEMPORARY_FAILURE   <- 4xx（可重试）
    +-- UNKNOWN             <- Agent 崩溃
```

### Billing Boundary 精确定义

> **Billing Boundary 是 Runner 成功建立到目标 SMTP 服务端的 TCP 连接后。无论后续 SMTP、TLS、认证、Envelope 或 DATA 阶段成功还是失败，均计费一次。**

不计费场景：
- DNS lookup 失败
- TCP connect 失败
- 以上两种状态最终为 `PRE_ATTEMPT_FAILURE`

---

# 🟠 P1-3：RECOVERING 增加 TEMPORARY_FAILURE 处理

## 问题

V2.5 的 RECOVERING 逻辑：

```
+-- NOT_ATTEMPTED -> QUEUED
+-- ATTEMPTING -> 查最新 Attempt：
|     +-- CREATED  -> QUEUED
|     +-- STARTED  -> UNKNOWN
|     +-- 其他     -> 不处理
+-- DELIVERED/FAILED -> 不处理
```

缺少对 `TEMPORARY_FAILURE` 的处理。如果 Agent 崩溃时最新 Attempt 是 TEMPORARY_FAILURE，根据重试策略应该重新调度。

## V2.5.1 修正

```
RECOVERING
    |
    v
分析每个 MailDelivery：
    |
    +-- NOT_ATTEMPTED -> QUEUED
    +-- ATTEMPTING -> 查最新 Attempt：
    |     +-- CREATED               -> QUEUED
    |     +-- STARTED               -> UNKNOWN
    |     +-- TEMPORARY_FAILURE     -> 如果 retry_count < 3 则 QUEUED（继续重试）
    |     +-- 其他                  -> 不处理
    +-- DELIVERED/FAILED -> 不处理
    |
    v
全部处理完 -> COMPLETED；有剩余 -> QUEUED
```

---

# 🟠 P1-4：Attempt Number 生成与 MailDelivery 行锁绑定

## 问题

V2.5 要求 attempt_number 由 Oracle 生成：

```sql
INSERT INTO mail_delivery_attempts
(delivery_id, attempt_number, status)
SELECT :delivery_id, COALESCE(MAX(attempt_number), 0) + 1, :status
FROM mail_delivery_attempts
WHERE delivery_id = :delivery_id;
```

但在并发场景下（如两个 Runner 同时为重试创建 Attempt），`MAX(attempt_number)` 可能存在幻读问题，导致两个 Attempt 获得相同编号。

## V2.5.1 修正

**Attempt Number 的生成必须与 MailDelivery 行锁绑定在同一事务中：**

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

---

# 🟡 P2-5：澄清 attempted_count 语义

## 问题

V2.5 中 `attempted_count` 字段同时出现在 MailTask、MailBatch、MailDelivery 中，但语义不明确：是 recipient 数还是 Attempt 数？

## V2.5.1 修正

| 模型 | 字段 | 语义 |
|------|------|------|
| MailTask | attempted_count | 已尝试的 **recipient 数**（至少有一次 Attempt） |
| MailBatch | attempted_count | 已尝试的 **recipient 数** |
| MailDelivery | attempts | 该 recipient 的 **Attempt 次数**（实际尝试次数） |

计算公式：

```
MailTask.attempted_count =
    SELECT COUNT(*) FROM mail_deliveries
    WHERE task_id = :task_id
    AND status NOT IN ('NOT_ATTEMPTED', 'CANCELLED')

MailDelivery.attempts =
    SELECT COUNT(*) FROM mail_delivery_attempts
    WHERE delivery_id = :delivery_id
```

---

# 🟡 P2-6：收件人标准化（Recipient Canonicalization）

## 问题

当前系统直接存储用户提交的收件人地址。如果用户先后提交：

```
user@example.com
User@Example.COM
```

`UNIQUE(task_id, recipient)` 会认为这是两个不同的收件人，导致同一邮箱被发送两次。

## V2.5.1 修正

**在 INSERT 前对收件人地址做标准化：**

```python
def canonicalize_recipient(recipient: str) -> str:
    """RFC 5321 标准化：域名转为小写，本地部分保留大小写"""
    local, domain = recipient.rsplit('@', 1)
    domain = domain.lower()
    return f'{local}@{domain}'
```

标准化后写入 `mail_deliveries.recipient`，并在此基础上应用 UNIQUE 约束。

---

# 🟡 P2-7：DKIM 私钥生产环境加密存储

## 问题

V2.5 推荐私钥存储在 `.env` 文件（chmod 600），但在生产环境中仍存在文件泄露风险。

## V2.5.1 补充

### 推荐方案优先级

| 优先级 | 方案 | 适用阶段 |
|:------:|------|:--------:|
| 1 | `.env` chmod 600 + 加密磁盘 | 开发/测试 |
| 2 | systemd `LoadCredential=` + 独立 root-owned 文件 | 生产初期 |
| 3 | HashiCorp Vault / AWS Secrets Manager | 生产成熟期 |
| 4 | 硬件安全模块（HSM） | 高合规场景 |

### systemd LoadCredential 示例

```
# /etc/systemd/system/proxy-manager.service
[Service]
LoadCredential=dkim-private-key:/etc/mail-relay/secrets/dkim.key
User=proxy-manager
Group=proxy-manager
```

应用代码通过 `$CREDENTIALS_DIRECTORY` 路径读取：

```python
import os
cred_dir = os.environ.get('CREDENTIALS_DIRECTORY')
if cred_dir:
    key_path = os.path.join(cred_dir, 'dkim-private-key')
    with open(key_path) as f:
        dkim_private_key = f.read()
```

### 私钥路径

```
# 开发环境
DKIM_PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY-----

# 生产环境
# 文件：/etc/mail-relay/secrets/dkim.key
# 权限：root:root 400
```

---

# V2.5.1 变更摘要

| # | 修订 | 优先级 | 类型 |
|---|------|:------:|------|
| 1 | STARTED + Quota CONSUME + QuotaTransaction 同一数据库事务 | 🔴 P0 | 事务 |
| 2 | 新增 `PRE_ATTEMPT_FAILURE` 状态 | 🔴 P0 | 状态机 |
| 3 | RECOVERING 增加 TEMPORARY_FAILURE 处理 | 🟠 P1 | 状态机 |
| 4 | Attempt Number 生成与 MailDelivery 行锁绑定 | 🟠 P1 | 并发 |
| 5 | 澄清 attempted_count 语义（recipient 数 vs Attempt 数） | 🟡 P2 | 文档 |
| 6 | 收件人标准化（Recipient Canonicalization） | 🟡 P2 | 功能 |
| 7 | DKIM 私钥生产环境加密存储 | 🟡 P2 | 安全 |

---

## 最终架构冻结声明

**V2.5.1 是实现前最终修订版本。此后不再进行任何架构/模型/状态机级别的修改，直接进入代码实现阶段。**

*文档版本 V2.5.1 — 2026-07-24*
