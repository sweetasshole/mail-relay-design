# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.3

> **版本:** V2.3 (2026-07-23)
> **基准:** [V2.2 完整文档](mail-relay-design-v2.2.md)
> **定位:** V2.2 的补充修订。新增 DKIM 远程签名 API、Quota 防重复扣费、SPF Active IP Set。

## 修订驱动

V2.3 解决 V2.2 中的 4 个遗留设计问题：

1. DKIM 签名方案选择 — 远程 Signing API vs Oracle 预签名
2. Quota 防重复扣费 — 同一 Attempt 不能被重复计费
3. SPF DNS 缓存不一致 — GitHub Hosted Runner IP 快速变更导致
4. SPF Active IP Set — 保留 Grace Period 内旧 Runner IP

---

## 🔴 修订 #1 — DKIM 方案 B：远程 DKIM Signing API

### 方案对比

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **方案 A** | Oracle 预签名整个 .eml → Runner 直接 SMTP | Runner 完全无感知 | 邮件内容需完整传给 Oracle 再回传 |
| **方案 B** (V2.3 选择) | Runner 发送 Header+Body Hash → Oracle 返回 Signature | Oracle 不接触完整邮件 | Oracle 仍参与签名流程 |

### 方案 B 架构

```
Mail Runner Agent (Go)
    │
    │ 1. 构建邮件 Header + Body
    │ 2. 计算 DKIM Hash（canonicalized headers + body hash）
    │ 3. POST /api/mail/dkim/sign
    │    { "headers": "...", "body_hash": "...", "selector": "default" }
    ▼
Oracle DKIM Signer
    │
    │ 1. 验证 Runner Token
    │ 2. 加载 DKIM 私钥
    │ 3. 对传入数据签名
    │ 4. 返回 DKIM-Signature header
    ▼
Mail Runner
    │
    │ 4. 将 DKIM-Signature 插入邮件 header
    │ 5. SMTP 发送（完整邮件含 DKIM 签名）
    ▼
Internet MX
```

### 安全保证

- **DKIM 私钥永远不离开 Oracle**
- Runner Token 验证后才可调用签名 API
- 签名内容（Headers + Body Hash）不包含私钥信息

### API 新增

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

### 模型变更

V2.2 中的 `DkimKey` 模型保持不变：

```python
class DkimKey(Base):
    __tablename__ = "dkim_keys"

    id: UUID
    domain: str
    selector: str
    private_key: str               # 加密存储，仅 Oracle 可读
    public_key: str
    active: bool
    created_at: datetime
```

---

## 🔴 修订 #2 — Quota 防重复扣费

### 问题

同一 Attempt 可能因 Runner 重试/网络重试被重复上报，导致：

```
1 SMTP Attempt → 3 CONSUME
```

### 解决

`QuotaTransaction` 增加唯一约束：

```python
class QuotaTransaction(Base):
    __tablename__ = "quota_transactions"

    id: UUID
    user_id: UUID
    task_id: UUID | None
    attempt_id: UUID | None        # 关联到具体 Attempt

    transaction_type: str          # RESERVE / CONSUME / RELEASE / ADJUST / REFUND
    amount: int

    created_at: datetime

    __table_args__ = (
        UniqueConstraint('attempt_id', 'transaction_type',
                        name='uq_quota_attempt_type'),
    )
```

### 约束说明

```
同一 attempt_id + 'CONSUME' → 只能存在一条
同一 attempt_id + 'REFUND'  → 可以存在（退费，独立交易）
```

### 流程

```
MailDeliveryAttempt
    │
    ├── STARTED
    │       → INSERT QuotaTransaction (attempt_id, CONSUME, 1)
    │       → UNIQUE(attempt_id, 'CONSUME') 防止重复
    │
    └── UNKNOWN
            → 不产生 REFUND
            → 下次重试 = 新 Attempt #2
            → 产生新的 QuotaTransaction (attempt_id=#2, CONSUME, 1)
```

**即使 Runner 重复上报，数据库层面保证 1 Attempt = 1 CONSUME。**

---

## 🔴 修订 #3 — SPF Active IP Set

### 问题

GitHub Hosted Runner 的 IP 生命周期很短：

```
Runner A: IP 1.2.3.4
    → 加入 SPF
    → 结束
    → 立即删除 1.2.3.4

Runner B: IP 5.6.7.8
    → 加入 SPF
```

但 DNS 缓存 300 秒内，外部邮件服务器可能查到旧 SPF（不含 5.6.7.8），导致 SPF Fail。

### 解决：Active IP Set

```
SPF 记录不立即删除旧 IP
    ↓
保留在 Active IP Set 中
    ↓
直到 Grace Period 结束
```

### 模型变更

```python
class SpfIpRecord(Base):
    __tablename__ = "spf_ip_records"

    ip: str
    runner_count: int              # 活跃引用计数
    retained_count: int            # Grace Period 保留计数
    active: bool                   # 是否在 SPF 中
    first_seen_at: datetime
    last_seen_at: datetime
    remove_after: datetime | None  # Grace Period 到期时间
```

### 流程

```
Runner A（IP 1.2.3.4）上线
    → runner_count += 1
    → 加入 SPF

Runner A 下线
    → runner_count -= 1
    → 如果 runner_count = 0
        → retained_count = 1
        → 保留在 SPF（Grace Period）
        → remove_after = now + 30min

Grace Period 内 Runner B（IP 1.2.3.4）上线
    → runner_count += 1
    → retained_count -= 1
    → 取消 Grace Period

Grace Period 到期
    → retained_count - 1
    → 如果 runner_count + retained_count = 0
        → 从 SPF 删除
```

### SPF 记录示例

```
现在 SPF 中包含：
├── Runner A IP (活跃)
├── Runner B IP (活跃)
├── Runner C IP (Grace Period 内保留，即将删除)
└── Runner D IP (Grace Period 内保留，即将删除)
```

### 配置

```yaml
spf:
  grace_period_minutes: 30       # 下线后保留时间
  verification_resolvers:
    - 1.1.1.1
    - 8.8.8.8
  active_ip_set_max_size: 50     # SPF 最大 IP 数量（防止无限增长）
```

### 限制

- **SPF Active IP Set 最大 50 个 IP**
- 如果超出，优先删除 Grace Period 中最老的 IP
- 这确保 SPF TXT 记录不超过 512 字节限制

---

## 🟡 修订 #4 — DKIM 签名限流

### 新增

DKIM 签名 API 有速率限制，防止滥用：

```yaml
dkim:
  rate_limit:
    per_runner_per_minute: 1000   # 每个 Runner 每分钟最多签名 1000 封
    per_domain_per_minute: 5000   # 每个域名每分钟最多签名 5000 封
```

### 实现

```python
class DkimSigningRate(Base):
    __tablename__ = "dkim_signing_rates"

    runner_id: UUID
    domain: str
    window_start: datetime
    count: int
```

---

## 修订总结

| # | 修订 | 级别 | V2.2 → V2.3 变更 |
|---|------|:----:|-------------------|
| 1 | DKIM 远程签名 API | 🔴 | 方案 A → 方案 B（Oracle 提供 Signing API） |
| 2 | Quota 防重复扣费 | 🔴 | 新增 UNIQUE(attempt_id, transaction_type) |
| 3 | SPF Active IP Set | 🔴 | 新增 retained_count + remove_after + 最大 50 IP 限制 |
| 4 | DKIM 签名限流 | 🟡 | 新增 DkimSigningRate 模型 |

## 凭证总结（V2.3 最终版）

| 凭证 | 用途 | 存储 | 可撤销 | 下发？ |
|------|------|------|:------:|:------:|
| User Session | Web 登录 | Database | ✅ | — |
| Admin Session | 管理后台 | Database | ✅ | — |
| User API Token | 第三方 API | 数据库(哈希) | ✅ | — |
| Bootstrap Token | Runner 初始化 | 数据库(哈希) | ✅ (TTL 5min) | ❌ Workflow参数 |
| Mail Runner Token | Runner 认证 | 数据库(哈希) | ✅ | 注册时获得 |
| DKIM Signing Token | Runner 调签名API | 数据库(哈希) | ✅ | 注册时获得 |
| GitHub PAT | 触发 Workflow | .env | ✅ | ❌ |
| Cloudflare Token | SPF 管理 | .env | ✅ | ❌ |
| DKIM Private Key | 邮件签名 | .env | ✅ | ❌ 绝不下发 |

---

*文档版本 V2.3 — 2026-07-23*
