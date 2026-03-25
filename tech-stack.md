# 技术栈说明文档

> 公司 AI Agent 系统 · 国内部署版 · 与 PRD v1.5 配套
> 若与 PRD 冲突，以 PRD 为准
> 原则：每层只用一个工具，选最成熟的，不引入可以避免的复杂度

---

## 总览

| 层级 | 技术选型 | 版本建议 |
|------|----------|----------|
| 前端框架 | React + Vite + Tailwind CSS | React 18 / Vite 5 / Tailwind 3 |
| 后端框架 | Node.js + Express | Node 20 LTS / Express 4 |
| 任务队列 | BullMQ | 5.x |
| LLM | 通义千问 qwen-plus（阿里云百炼） | — |
| Embedding | text-embedding-v3（阿里云百炼） | 1536 维 |
| OCR | 阿里云发票识别 API（增值税发票专项） | — |
| 主数据库 | 阿里云 RDS PostgreSQL + pgvector | PostgreSQL 15+ |
| 文件存储 | 阿里云 OSS | — |
| 缓存 / 队列后端 | Redis（ECS 自建） | Redis 7 |
| 邮件发送 | nodemailer + 网易企业邮箱 SMTP | nodemailer 6 |
| 审批 / 组织 | 钉钉开放平台 API | — |
| 部署 | 阿里云 ECS（2核4G） | Ubuntu 22.04 |

---

## 一、前端

### React + Vite + Tailwind CSS

**职责**：统一对话界面 + HR / 财务管理后台

**选型理由**
- 生态最成熟，初级开发者学习资料最多
- Vite 构建速度快，开发体验好
- Tailwind 无需维护 CSS 文件，适合小团队快速迭代

**关键依赖**

```json
{
  "react": "^18",
  "react-router-dom": "^6",
  "axios": "^1",
  "tailwindcss": "^3",
  "@vitejs/plugin-react": "^4"
}
```

**注意事项**
- 不引入 Redux，用 React Context + useState 管理状态，规模够用
- 移动端响应式适配，无需单独开发 App

---

## 二、后端

### Node.js + Express

**职责**：API 服务、Agent 编排、钉钉回调处理、定时任务触发

**选型理由**
- 轻量，初级开发者上手快
- 不引入 NestJS 等框架，避免增加不必要的复杂度
- 与前端同语言，降低上下文切换成本

**关键依赖**

```json
{
  "express": "^4",
  "openai": "^4",
  "bullmq": "^5",
  "ioredis": "^5",
  "nodemailer": "^6",
  "multer": "^1",
  "pg": "^8",
  "express-session": "^1",
  "zod": "^3"
}
```

**注意事项**
- 使用 `openai` 官方 SDK（阿里云百炼提供 OpenAI 兼容接口，只需改 `baseURL`）
- 钉钉 SSO 成功后，由后端创建服务端会话，并通过 HttpOnly Cookie 维持登录态
- 钉钉回调接口单独挂载，做幂等校验（`approval_instance_id + event_id`）
- 审批顺序、审批人来源、抄送规则均以钉钉模板 / 流程为准，系统不单独定义审批链
- 敏感配置（AppSecret / SMTP 密码 / API Key）全部通过环境变量注入，不写入代码

---

## 三、任务队列

### BullMQ + Redis

**职责**：邮件发送重试、钉钉状态轮询兜底、OCR 异步处理；审批催办 / 超时提醒是否由系统补充，待实施前确认

**选型理由**
- PRD 已明确邮件失败重试 3 次、10 分钟轮询兜底、OCR 异步处理，这三类需求适合统一放入队列
- 不加队列直接用 `setTimeout`，进程重启后任务全丢，生产必崩
- BullMQ 基于 Redis，与缓存复用同一个 Redis 实例，不增加额外依赖

**队列划分**

| 队列名 | 用途 | 重试策略 |
|--------|------|----------|
| `email-queue` | 邮件发送 | 失败重试 3 次，指数退避（1m / 3m / 10m） |
| `sync-queue` | 钉钉状态轮询兜底 | 每 10 分钟扫描 APPROVING 且 last_synced_at 超时的单据 |
| `ocr-queue` | 发票 OCR 异步处理 | 失败重试 2 次 |
| `reminder-queue`（可选） | 审批催办 / 超时提醒 | 仅在实施阶段确认系统需补充提醒时启用 |

**注意事项**
- Redis 部署在同一台 ECS 上，内网通信延迟 < 1ms
- 生产环境开启 BullMQ Dashboard（`@bull-board/express`）方便排查任务状态

---

## 四、AI 能力

### 4.1 LLM：通义千问（阿里云百炼）

**模型**：`qwen-plus`（性价比最优，支持 Function Calling）

**职责**：意图识别与路由、对话管理、邮件 / 周报内容生成、知识库问答

**接入方式**：阿里云百炼提供 OpenAI 兼容接口，代码只需改两行：

```js
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: process.env.DASHSCOPE_API_KEY,
  baseURL: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
});
```

**模型选择参考**

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 意图识别 / 路由 | qwen-turbo | 低延迟，任务简单 |
| 对话 / 内容生成 | qwen-plus | 效果与成本均衡 |
| 复杂推理 / 长文档 | qwen-max | 按需使用，成本较高 |

**注意事项**
- 上下文保留最近 10 轮对话历史
- Tool Use（Function Calling）写法与 OpenAI 一致，直接复用
- 不传员工身份证、工资等敏感字段给 LLM

---

### 4.2 Embedding：text-embedding-v3（阿里云百炼）

**职责**：知识库文档切片向量化、用户提问向量化、语义相似度检索

**维度**：1536 维，与 pgvector 默认配置兼容

**接入方式**：同 LLM，复用同一个 OpenAI SDK 实例，传入 `model: 'text-embedding-v3'`

**注意事项**
- 文档上传时异步生成 Embedding，不阻塞上传响应
- 检索时余弦相似度阈值建议初始值 0.75，上线前根据实际效果调整
- 一期文档规模 < 500，pgvector 性能完全够用，无需额外优化

---

### 4.3 OCR：阿里云发票识别 API

**职责**：上传发票图片后自动提取结构化字段

**选型理由**
- 有增值税发票专项 API，直接返回结构化 JSON，准确率远高于通用 Vision 模型
- 不需要用 LLM 解析图片，延迟更低、成本更低

**返回字段**

| 字段 | 说明 |
|------|------|
| `invoiceNumber` | 发票号码 |
| `invoiceDate` | 开票日期 |
| `totalAmount` | 价税合计 |
| `taxAmount` | 税额 |
| `sellerName` | 销售方名称 |
| `buyerName` | 购买方名称 |

**失败处理**：OCR 失败时标记 `ocr_failed = true`，允许员工手动录入全部字段后继续提交

---

## 五、数据存储

### 5.1 阿里云 RDS PostgreSQL + pgvector

**职责**：所有业务数据（请假单、报销单、发票记录、知识库索引、审计日志等）

**选型理由**
- Supabase 无国内节点，访问慢且数据出境有合规风险，必须替换
- 阿里云 RDS PostgreSQL 支持 pgvector 插件，一个数据库同时解决业务数据和向量检索

**规格建议**：2核4G，存储 100GB SSD，按量付费

**pgvector 配置**

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- 文档切片向量索引
CREATE INDEX ON document_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

**核心数据表**

| 表名 | 说明 |
|------|------|
| `users` | 员工信息与角色（从钉钉同步） |
| `leave_requests` | 请假单（含状态机字段） |
| `expense_reports` | 报销单（含状态机字段 + ocr_failed） |
| `invoices` | 发票元信息 + OCR 结构化结果 + OSS 路径 |
| `documents` | 知识库文档元信息 |
| `document_chunks` | 文档切片 + pgvector 向量 |
| `approval_logs` | 钉钉回调原始事件（幂等表） |
| `email_logs` | 邮件发送日志 |
| `audit_logs` | 系统操作审计 |

**状态机关键字段**（`leave_requests` / `expense_reports` 共用）

```sql
status                VARCHAR      -- 主状态：DRAFT / APPROVING / APPROVED / REJECTED / CANCELLED
approval_instance_id  VARCHAR      -- 钉钉审批实例 ID
current_node_name     VARCHAR      -- 当前节点名（展示用，不参与状态流转）
current_approver      VARCHAR      -- 当前审批人（展示用）
last_synced_at        TIMESTAMPTZ  -- 最后同步时间（轮询兜底依据）
rejection_reason      TEXT         -- 拒绝原因（REJECTED 状态时展示）
```

---

### 5.2 阿里云 OSS

**职责**：发票图片存储（合规保留 5 年）

**选型理由**：替代 Supabase Storage，数据存储在国内，满足合规要求

**配置要点**
- Bucket 设置为私有，通过后端生成临时签名 URL 供前端访问
- 开启版本控制，防止误删
- 生命周期规则：5 年后转为低频存储（降低费用）

---

### 5.3 Redis（ECS 自建）

**职责**：BullMQ 队列后端、接口限流、钉钉 Token 缓存

**部署方式**：与应用部署在同一台 ECS，内网通信

**配置要点**
- 开启持久化（AOF），防止重启丢失队列任务
- 设置最大内存限制（建议 512MB），超出时使用 `allkeys-lru` 淘汰策略
- 不对外暴露端口，只允许本机访问

---

## 六、集成服务

### 6.1 钉钉开放平台

**职责**：SSO 登录、组织架构同步、审批流创建、审批结果回调、工作通知推送

**鉴权方式**：企业内部应用，AppKey + AppSecret，存服务端环境变量；钉钉 SSO 成功后由后端创建服务端会话，并通过 HttpOnly Cookie 维持登录态

**关键 API**

| API | 用途 |
|-----|------|
| `POST /v1.0/oauth2/accessToken` | 获取企业 Access Token |
| `GET /v1.0/contact/users/{userId}` | 获取员工信息与组织关系 |
| `POST /v1.0/workflow/processInstances` | 创建审批实例 |
| `GET /v1.0/workflow/processInstances/{id}` | 查询审批状态（轮询兜底） |
| `POST /v1.0/message/workNotice` | 推送工作通知给审批人 |

**审批边界**
- 审批顺序、审批人来源、抄送规则均以钉钉模板 / 流程为准
- 系统仅负责发起审批、同步结果、展示节点信息，不自行定义审批链
- 组织关系异常应先在钉钉中修正，一期系统不保留兜底审批人

**回调幂等处理**

```js
// 权威幂等键：approval_instance_id + event_id
const idempotencyKey = `${approvalInstanceId}:${eventId}`;

// 先查 approval_logs（幂等表 / 审计表）
const exists = await db.approvalLogs.findUnique({
  where: { idempotency_key: idempotencyKey },
});
if (exists) return res.status(200).json({ ok: true });

await db.approvalLogs.create({
  data: {
    idempotency_key: idempotencyKey,
    approval_instance_id: approvalInstanceId,
    event_id: eventId,
    payload: rawPayload,
  },
});

// 如需提速，可额外写 Redis 做热点缓存，但 Redis 不是唯一依据
```

---

### 6.2 网易企业邮箱（SMTP）

**职责**：审批结果通知、行政待办邮件；审批催办 / 超时提醒是否由系统补充待实施前确认

**配置**

```js
const transporter = nodemailer.createTransport({
  host: 'smtp.qiye.163.com',
  port: 465,
  secure: true,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});
```

**发送策略**：所有邮件通过 BullMQ `email-queue` 异步发送，失败自动重试 3 次

---

## 七、部署

### 阿里云 ECS（2核4G）

**规格**：2核 CPU / 4GB 内存 / 40GB 系统盘 + 100GB 数据盘 / Ubuntu 22.04

**选型理由**
- Railway 在国内访问慢且无 ICP，不适合国内生产部署
- ECS 2核4G 满足 5–20 人并发使用
- 与 RDS、OSS 同在阿里云，内网通信，延迟低

**部署方式**：PM2 管理 Node.js 进程，Nginx 反向代理，Let's Encrypt HTTPS

```
ECS 内部结构
├── Nginx（80/443，反向代理）
├── Node.js API（PM2，端口 3000）
├── BullMQ Worker（PM2，独立进程）
└── Redis（端口 6379，仅本机访问）
```

**环境变量清单**

```bash
# 阿里云百炼
DASHSCOPE_API_KEY=

# 阿里云 OCR
ALIBABA_CLOUD_ACCESS_KEY_ID=
ALIBABA_CLOUD_ACCESS_KEY_SECRET=

# 阿里云 OSS
OSS_BUCKET=
OSS_REGION=
OSS_ACCESS_KEY_ID=
OSS_ACCESS_KEY_SECRET=

# 数据库
DATABASE_URL=

# Redis
REDIS_URL=redis://127.0.0.1:6379

# 钉钉
DINGTALK_APP_KEY=
DINGTALK_APP_SECRET=

# 邮件
SMTP_USER=
SMTP_PASS=

# Session
SESSION_SECRET=
```

**注意事项**
- 所有敏感变量通过 `.env` 文件注入，不写入代码，不提交 Git
- 定期备份 RDS 快照（每日自动备份，保留 7 天）
- 上线前压测：`ab -n 1000 -c 20`，验证 20 人并发下 p95 响应时间达标

---

## 八、不引入的技术（及原因）

| 技术 | 不引入原因 |
|------|-----------|
| Docker / Kubernetes | 初级开发者运维成本过高，ECS + PM2 够用 |
| NestJS / Fastify | Express 满足需求，不引入框架复杂度 |
| Redux / Zustand | React Context 够用，规模不需要状态管理库 |
| Pinecone / Weaviate | pgvector 支持 < 500 文档，不引入外部向量库 |
| Supabase | 无国内节点，数据出境合规风险 |
| Railway / Vercel | 国内访问慢，无 ICP |
| Upstash Redis | 国内延迟高，用自建 Redis 代替 |
| GraphQL | REST 够用，不引入额外复杂度 |
| 微服务架构 | 5–20 人规模，单体应用更易维护 |

---

## 九、技术栈一句话总结

```
React + Node.js/Express + BullMQ
+ 通义千问 qwen-plus + text-embedding-v3 + 阿里云发票 OCR
+ 阿里云 RDS PostgreSQL/pgvector + 阿里云 OSS + 自建 Redis
+ nodemailer/网易邮箱 + 钉钉开放平台
+ 阿里云 ECS 部署
```

> 11 个选型，全部国内可用，无需翻墙，数据不出境，每层只有一个工具。
