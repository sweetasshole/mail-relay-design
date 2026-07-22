# Mail Relay Platform + Multi Mail Runner — 实施设计文档 V2.2

> **版本:** V2.2 (2026-07-23)
> **基准:** [V2.1 完整文档](mail-relay-design-v2.1.md)
> **定位:** V2.1 的架构评审修订（Cliff Notes）。新增和修改的部分以 V2.1 为基准。

## 修订驱动

V2.2 由以下架构评审触发，并明确了 **Mail Runner 使用 GitHub Hosted Runner**。

---

## 🔴 修订 #1 — MailDeliveryAttempt（新增核心模型）

### 问题

V2.1 中 MailDelivery 只存 `attempts: int`，无法记录每次 SMTP Attempt 的独立生命周期（哪个 Runner、什么时间、什么 SMTP 响应、为什么 UNKNOWN）。

### 新增模型

```
MailDeliveryAttempt
├── delivery_id
├── task_id
├── batch_id
├── runner_id
├── attempt_number: int
├── status: STARTED/DELIVERED/PERMANENT_FAILURE/TEMPORARY_FAILURE/UNKNOWN
├── smtp_code
├── smtp_enhanced_code
├── smtp_response
├── error_type
├── started_at
└── completed_at
```

### 计费关联

每次 `status = STARTED` → `QuotaTransaction.type = CONSUME, amount = 1`，关联 `attempt_id`。

### 示例

```
Attempt #1 (Runner A): STARTED → UNKNOWN (崩溃) → quota = 1
Attempt #2 (Runner B): STARTED → DELIVERED       → quota = 1
Total: 2 quota
```

---

## 🔴 修订 #2 — Cancel 状态机修正

### 问题

V2.1 允许 `ATTEMPTING → CANCELLED`，但 SMTP DATA 可能已发送完成只是没拿到回应。

### 修正规则

| 状态 | 用户取消 | 结果 |
|------|---------|------|
| `NOT_ATTEMPTED` | → | `CANCELLED` |
| `ATTEMPTING` | → | 优雅结束当前 SMTP → `DELIVERED` / `FAILURE` |
| `ATTEMPTING`（Agent 崩溃） | → | `UNKNOWN` |

`CANCELLED` **只能用于从未开始 SMTP 的收件人**。

---

## 🔴 修订 #3 — Heartbeat 与 Lease 分离

### 问题

V2.1 中 Runner Heartbeat 会刷新所有关联 Batch 的 Lease，导致卡死的 Batch 永远不会被回收。

### 新规则

```
Runner Heartbeat（30s）  → 仅证明 Runner 活着
Batch Progress（50封/5s）→ 刷新该 Batch 的 lease_expires_at
```

```
Lease Reaper（30s）：
  UPDATE mail_batches SET status = 'RECOVERING'
  WHERE lease_expires_at < NOW()
  AND status IN ('CLAIMED', 'RUNNING')
```

---

## 🔴 修订 #4 — 原子事务

### 问题

两个 Scheduler 或 Runner 可能同时 Claim 同一个 Batch。

### 解决

```sql
-- Batch Claim
UPDATE mail_batches
SET status = 'CLAIMED', runner_id = ?, lease_expires_at = NOW() + 60
WHERE id = ? AND status = 'QUEUED';
-- 只有 affected_rows = 1 才算成功

-- Lease Reaper
UPDATE mail_batches
SET status = 'RECOVERING'
WHERE lease_expires_at < NOW()
AND status IN ('CLAIMED', 'RUNNING');
```

依赖 PostgreSQL 行级锁 / SQLite 原子 UPDATE 保证并发安全。

---

## 🔴 修订 #5 — 统计一致性公式

### 明确

```
attempted_count = DELIVERED + PERMANENT_FAILURE + TEMPORARY_FAILURE + UNKNOWN + CANCELLED_AFTER_ATTEMPT

NOT_ATTEMPTED ≠ attempted
CANCELLED（未开始） ≠ attempted
```

保证 Batch / Task / Delivery 三者数字永远一致。

---

## 🔴 修订 #6 — DKIM 私钥不下发 Runner

### 核心原则

**DKIM 私钥永久只保存在 Oracle，不下发到 Mail Runner。**

GitHub Hosted Runner 是临时实例，私钥下发存在泄露风险。

### 架构

```
用户创建任务
   → Oracle 加载 DKIM 私钥
   → Oracle 对邮件内容签名
   → 生成已签名 .eml
   → 存入对象存储
   → Runner 通过 Presigned URL 下载
   → Runner SMTP 发送（无需 DKIM 能力）
```

### 新增凭证

| 凭证 | 存储 | 下发？ |
|------|------|:------:|
| DKIM Private Key | .env (chmod 600) | ❌ 不下发 |

---

## 🔴 修订 #7 — GitHub Hosted Runner SPF

### 明确

**Mail Runner 使用 GitHub Hosted Runner。**

影响：
1. IP 生命周期短（分钟到小时）
2. IP 池巨大且动态
3. DNS 更新和 SPF 生效存在天然时差

### 新增：SPF 生效验证

```
Runner IP 注册 → 更新 SPF → DNS Resolver 查询（1.1.1.1 / 8.8.8.8）→ 确认可见 → Runner = READY
```

### 长期方案

推荐采用子域名架构：

```
主域名 SPF：v=spf1 include:_spf.yourdomain.com ~all
子域名 SPF：_spf.yourdomain.com TXT "v=spf1 ip4:RunnerA ip4:RunnerB ~all"
```

---

## 🔴 修订 #8 — PostgreSQL 生产环境

### 决策

| 环境 | 数据库 |
|------|--------|
| 开发 | SQLite |
| 生产 | **PostgreSQL** |

原因：Scheduler / Lease / Quota / Batch Claim 涉及并发事务，SQLite 写锁争用严重。

---

## 🟡 强烈建议（修订中包含）

| 修订 | 内容 |
|------|------|
| 内容对象存储 | 大邮件内容走对象存储，数据库只存 `content_object_key` |
| 动态并发控制 | `HEALTHY → max=3, SUSPECTED → max=1, UNHEALTHY → max=0` |
| Phase 顺序优化 | 多 Batch + Lease 提前到 Phase 4（原 Phase 5） |

---

## 实施阶段（修订后）

| Phase | 内容 |
|-------|------|
| 1 | 基础设施 + Auth（15+ 张表，Session + Bootstrap Token + Runner Token） |
| 2 | Runner 生命周期 + Health + SPF 生效验证 |
| 3 | MailTask + MailBatch + Quota（Snapshot + Idempotency） |
| 4 | **Scheduler + Lease + 多 Batch**（提前） |
| 5 | Go Agent + SMTP（含 MailDeliveryAttempt 记录） |
| 6 | 故障转移 + Cancel + Results CSV |
| 7 | SPF 子域名 + DKIM 签名 + Sender Identity |
| 8 | Admin + Audit + Observability |

---

## 凭证总结（V2.2 最终版）

| 凭证 | 用途 | 存储 | 可撤销 | 下发？ |
|------|------|------|:------:|:------:|
| User Session | Web 登录 | Database | ✅ | — |
| Admin Session | 管理后台 | Database | ✅ | — |
| User API Token | 第三方 API | 数据库(哈希) | ✅ | — |
| Bootstrap Token | Runner 初始化 | 数据库(哈希) | ✅ (TTL 5min) | ❌ Workflow参数 |
| Mail Runner Token | Runner 认证 | 数据库(哈希) | ✅ | — |
| GitHub PAT | 触发 Workflow | .env | ✅ | ❌ |
| Cloudflare Token | SPF 管理 | .env | ✅ | ❌ |
| DKIM Private Key | 邮件签名 | .env | ✅ | ❌ |

---

*文档版本 V2.2 — 2026-07-23*
