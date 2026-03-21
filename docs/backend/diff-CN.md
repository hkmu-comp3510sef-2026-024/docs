# 设计差异：原版 (Node.js/Express) vs. Go 后端

## 1. 技术栈

| 组件 | 原版 (Node.js) | 新版 (Go) | 影响说明 |
|------|----------------|----------|----------|
| 编程语言 | JavaScript / Node.js 18+ | Go 1.26+ | 后端全面重写 |
| Web 框架 | Express.js 5.2.1 | Gin | Go 标准 REST 框架 |
| ORM | Prisma 7.4.2 | GORM | 最流行的 Go ORM，支持自动迁移 |
| 数据库 | MySQL / MariaDB | PostgreSQL | 更强的 JSON（jsonb）支持，用于 AuditLog |
| 认证 | JWT (jsonwebtoken 9.0.3) + bcryptjs | golang-jwt/jwt v5 + bcrypt | **见 §5 认证与会话** |
| 定时任务 | node-cron 4.2.1 | 内置 goroutine + time.Ticker | 逻辑相同，实现方式不同 |
| 数据验证 | Joi 18.0.2 | go-playground/validator | DTO struct tags vs. Joi schemas |
| 文件上传 | Multer 2.1.0 | CAS（内容寻址存储） | **见 §6 文件存储** |
| API 文档 | 未指定 | Swagger (swaggo) | 从 handler 注解自动生成 |
| 依赖注入 | 手动 / 未指定 | Google Wire | 编译期依赖注入 |

---

## 2. 项目架构

### 原版 — 标准 Express/MVC
```
src/
  routes/
  controllers/
  models/ (Prisma)
  middleware/
  services/
```

### 新版 — 清洁架构 + 领域驱动设计（DDD）
```
cmd/server/              # 程序入口 + Wire 引导
internal/
  domain/                # 纯 Go — 实体、接口、领域错误（无 ORM 标签）
    user/, book/, copy/, loan/, reservation/, fine/
    notification/, auditlog/, reminderpolicy/, assistant/, session/
    shared/enum.go       # 所有枚举类型
  usecase/               # 纯 Go — 业务规则，依赖领域接口
  repository/            # GORM 实现的领域仓库接口
    postgres/            # PostgreSQL 具体实现
  delivery/              # 外部适配器
    http/                # Gin HTTP 处理器
    cron/                # 定时任务运行器
    grpc/                # 预留（未来扩展）
  infrastructure/        # 领域接口的具体实现
    storage/             # LocalDiskStorage (CAS)
    jwt/                 # JWT 提供者
    hasher/              # bcrypt
    config/              # YAML 配置加载
pkg/
  validator/             # go-playground/validator 封装
  response/              # 统一 API 响应辅助函数
  pagination/            # 分页辅助工具
migrations/             # SQL 迁移文件
config.yaml
```

**核心架构差异：**
- Domain 层为**纯 Go 代码**，无框架依赖
- GORM 标签仅存在于 **repository 层**，不影响领域实体
- 所有外部适配器（HTTP 框架、存储、JWT）均通过领域层定义的接口**可插拔**（依赖反转）
- Google Wire 提供**编译期 DI**——无运行时反射式依赖注入容器

---

## 3. 数据库

| 方面 | 原版 | 新版 |
|------|------|------|
| 数据库引擎 | MySQL / MariaDB | PostgreSQL |
| ORM | Prisma（自动生成 CRUD） | GORM（手写查询） |
| AuditLog Before/After | 未指定 | `jsonb` 类型的列（`map[string]any`） |
| UUID 使用 | 未指定 | UUIDv7 用于实体，嵌入时间戳（如 AuditLog.ID、Session.ID） |

> **对前端无影响** — 表结构/实体相同，数据库变更对前端透明。

---

## 4. 实体模型（数据模型）

### 原版：12 个实体
### 新版：13 个实体（原 12 个 + `Session`）

### 新增实体：Session
```go
type Session struct {
    ID        uuid.UUID  // UUIDv7 — 内嵌时间戳，无需 CreatedAt
    UserID    uuid.UUID
    TokenHash string     // 原始 refresh token 的 SHA-256 哈希值，原始 token 不持久化
    UserAgent string     // 登录时记录的设备/浏览器信息
    IPAddress string     // 登录时记录的客户端 IP
    ExpiresAt time.Time  // 有效期 7 天
}
```
用途：刷新令牌存储，支持多设备登录（每用户最多 5 个活跃会话）。

### AuditLog 变更
```go
type AuditLog struct {
    ID         uuid.UUID      // UUIDv7（内嵌时间戳）
    UserID     uuid.UUID
    Action     string         // 如 "USER_FREEZE", "BOOK_UPDATE"
    EntityType string
    EntityID   string
    Reason     string         # 新增字段：操作原因说明
    BeforeData map[string]any // jsonb — 变更前数据快照（CREATE 时为 nil）
    AfterData  map[string]any // jsonb — 变更后数据快照（DELETE 时为 nil）
    IPAddress  string
    UserAgent  string
}
```
**变更：** 新增 `Reason` 字段；`BeforeData`/`AfterData` 类型改为 `map[string]any`（存储为 jsonb）。

### User 实体变更
**新增字段：** `FreezeReason string` — 管理员冻结会员账户时记录冻结原因。

---

## 5. 认证与会话（重大变更）

### 原版设计
```
POST /api/auth/login → 响应体返回 { token, role, userId }
Token 存储在 localStorage
通过 Authorization: Bearer <token> 请求头发送
每次登录一个 token
无明确的刷新机制（假设 token 长期有效）
```

### 新版设计：全 Cookie + 静默自动刷新

| Cookie 名称 | 类型 | HttpOnly | Secure | Max-Age | 内容 |
|------------|------|----------|--------|---------|------|
| `access_token` | JWT | Yes | Yes* | 3600（1小时） | HS256 签名 JWT |
| `refresh_token` | 不透明 | Yes | Yes* | 604800（7天） | 原始 refresh token（查询 key） |

*Secure 标志：生产环境启用，本地开发可配置关闭。

**JWT Payload：** `{ "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN", "sessionId": "uuid" }`

#### 静默自动刷新中间件流程
```
请求到达
  → 检查 access_token cookie（无状态，无需 DB 调用）
  → 如果有效 → 继续处理
  → 如果过期/缺失 → 检查 refresh_token cookie
      → 按 token hash 查找 Session（DB 调用）
      → 验证是否过期
      → 生成新 JWT + 设置新 cookie
      → 继续处理
```
- **99% 的请求：** 仅无状态 JWT 验证，零 DB 调用
- **仅在以下情况调用 DB：** access token 过期且存在有效 refresh token
- **前端不会收到因 token 过期导致的 401** — 后端自我修复
- **多设备支持：** 每用户最多 5 个活跃会话；第 6 次登录时清除最旧的会话

#### 刷新令牌安全机制
- 原始 token 仅存储在浏览器 HttpOnly Cookie 中（对 JavaScript 不可见）
- 服务器仅存储 `SHA-256(原始 token)` 哈希值（TokenHash）
- 即使数据库被攻破，也无法还原原始 token

#### 会话清理任务
- 每 **1 小时** 运行一次（`time.NewTicker`）
- **服务器启动时** 也会运行
- 删除所有 `ExpiresAt < now` 的 Session 记录

#### API 路由变更

| 原版 | 新版 | 说明 |
|------|------|------|
| `POST /api/auth/login` → body 返回 `{ token, role, userId }` | `POST /api/auth/login` → 设置 HttpOnly cookie，body 返回 `{ userId, role }` | 响应体少了 token 字段 |
| 所有请求带 `Authorization: Bearer <token>` 请求头 | **无需请求头** — 仅用 Cookie | 前端需移除 Bearer header 逻辑 |
| 无 logout-all | `DELETE /api/auth/logout-all` — 删除用户所有会话 | 新增 |
| 无会话管理 | 多会话追踪，每用户最多 5 个 | 新增 |
| `POST /api/auth/refresh`（前端显式调用） | **浏览器客户端无需此接口** — 中间件静默处理 | 移动端仍可用 |
| N/A | `POST /api/auth/refresh` | 保留供移动端使用（body: `{ refreshToken }`） |

**前端适配要点：**
- 删除 `localStorage.setItem('token', ...)` 和 `Authorization` 请求头逻辑
- 删除收到 401 后显式调用刷新接口的逻辑
- 登录响应体解析逻辑不变（仍为 `{ userId, role }`）
- 登出调用 `DELETE /api/auth/logout`（后端清除 cookie）
- 收到 `errorCode: "ERR_SESSION_EXPIRED"` 或 `"ERR_INVALID_REFRESH_TOKEN"` 时跳转登录页

---

## 6. 文件存储（新增：CAS 模式）

### 原版
- Multer 处理 `multipart/form-data` 上传
- 文件以原始文件名或生成名称存储
- 无去重机制

### 新版：内容寻址存储（CAS）
```
文件内容 → SHA-256 哈希 → 路径：{h[0:2]}/{h[2:4]}/{h[4:]}<ext>
示例：哈希值 "1823ad...b8c3"，扩展名 ".png"
     → "18/23/ad1537c537c02199358e2132e3cf25948c1e0b25134699dc79d1f634b8c3.png"
```

**特性：**
- **去重：** 如果文件内容已存在则跳过存储（相同哈希 = 相同文件）
- **无全文件内存加载：** 64KB 缓冲区循环读取
- **线程安全**（标注用于未来 S3 升级）
- 存储路径作为 `CoverURL` 返回（如 `"18/23/ad15...c3.png"`）

**对前端影响：**
- 头像上传和书籍封面上传行为不变（相同 multipart 流程）
- 响应中 `CoverURL` 字段格式变为哈希路径，而非传统文件名

---

## 7. 错误处理

### 原版错误响应
```json
{ "code": 400, "message": "Bad Request" }
{ "code": 401, "message": "Unauthorized" }
...
```

### 新版错误响应
```json
{
  "code": 422,
  "errorCode": "ERR_NO_AVAILABLE_COPIES",
  "message": "No available copies for this book",
  "data": { "bookId": "123" }
}
```

**核心差异：**
- `errorCode` 是**细粒度的、领域特定的字符串**（如 `ERR_NO_AVAILABLE_COPIES`、`ERR_USER_ACCOUNT_FROZEN`、`ERR_RENEWAL_LIMIT_REACHED`）
- `message` 仅为**标准英文**，用于开发/调试
- `data` 携带错误相关的结构化上下文信息
- 前端使用 `errorCode` 作为 **i18n 国际化 key** 来显示本地化消息

### 细粒度错误码（完整列表）

| errorCode | HTTP 状态码 | 说明 |
|-----------|-------------|------|
| `ERR_USER_EMAIL_EXISTS` | 409 | 邮箱已被注册 |
| `ERR_USER_NOT_FOUND` | 404 | 用户不存在 |
| `ERR_USER_ACCOUNT_PENDING` | 422 | 账户等待管理员审核 |
| `ERR_USER_ACCOUNT_FROZEN` | 422 | 账户已被冻结 |
| `ERR_USER_ACCOUNT_EXPIRED` | 422 | 会员已过期 |
| `ERR_BOOK_NOT_FOUND` | 404 | 图书不存在 |
| `ERR_BOOK_INACTIVE` | 422 | 图书已下架（停用） |
| `ERR_COPY_NOT_FOUND` | 404 | 副本（条码）不存在 |
| `ERR_NO_AVAILABLE_COPIES` | 422 | 没有可借的副本 |
| `ERR_LOAN_NOT_FOUND` | 404 | 借阅记录不存在 |
| `ERR_LOAN_NOT_ACTIVE` | 422 | 借阅记录不处于活跃状态 |
| `ERR_RENEWAL_LIMIT_REACHED` | 422 | 已达到最大续借次数 |
| `ERR_RESERVATION_ALREADY_EXISTS` | 409 | 该图书已存在有效预约 |
| `ERR_RESERVATION_NOT_FOUND` | 404 | 预约记录不存在 |
| `ERR_RESERVATION_CANNOT_CANCEL` | 422 | 当前状态不允许取消预约 |
| `ERR_FINE_NOT_FOUND` | 404 | 罚款记录不存在 |
| `ERR_SESSION_EXPIRED` | 401 | 刷新令牌会话已过期 |
| `ERR_INVALID_REFRESH_TOKEN` | 401 | 刷新令牌无效或不存在 |
| `ERR_VALIDATION_FAILED` | 400 | 请求验证失败 |
| `ERR_UNAUTHORIZED` | 401 | JWT token 缺失或无效 |
| `ERR_FORBIDDEN` | 403 | 无权限访问 |
| `ERR_INTERNAL` | 500 | 意外的服务器错误 |

**前端适配要点：**
- 错误处理必须使用 `errorCode`（而非 `message`）作为用户可见文本的 key
- 可使用 `data` 字段中的上下文信息（如 `{ bookId }`）自定义界面提示

---

## 8. API 路由变更汇总

### 行为不变的路由（相同路径、相同功能）
- `POST /api/auth/register` — 功能不变
- `GET /api/books/search` — 功能不变
- `GET /api/books/{bookId}` — 功能不变
- `GET /api/member/profile`、`PUT /api/member/profile` — 不变
- `GET /api/member/membership` — 不变
- `GET /api/member/loans`、`POST /api/member/loans`、`POST /api/member/loans/{loanId}/renew` — 不变
- `GET /api/member/reservations`、`POST /api/member/reservations`、`DELETE /api/member/reservations/{reservationId}` — 不变
- `GET /api/member/notifications`、`PUT /api/notifications/{notificationId}/read`、`PUT /api/member/notifications/read-all` — 不变
- `GET /api/admin/members`、`POST /api/admin/members/{memberId}/approve`、`POST /api/admin/members/{memberId}/freeze`、`POST /api/admin/members/{memberId}/unfreeze`、`POST /api/admin/members/{memberId}/renew` — 不变
- `GET /api/admin/books`、`POST /api/admin/book`、`PUT /api/admin/book/{bookId}`、`PUT /api/admin/book/{bookId}/deactivate` — 不变
- `GET /api/admin/copies`、`POST /api/admin/copy`、`PUT /api/admin/copy/{copyId}/status` — 不变
- `GET /api/admin/fines`、`POST /api/admin/fines/{fineId}/pay`、`POST /api/admin/fines/{fineId}/waive` — 不变
- `GET /api/admin/reminder-policy`、`PUT /api/admin/reminder-policy` — 不变
- `GET /api/admin/audit-logs`、`GET /api/admin/audit-logs/{logId}` — 不变
- `POST /api/circulation/checkout`、`POST /api/circulation/return`、`GET /api/circulation/lookup` — 不变
- `GET /api/assistant/quick-questions`、`POST /api/assistant/sessions`、`POST /api/assistant/chat`、`GET /api/assistant/sessions/{sessionId}/messages`、`DELETE /api/assistant/sessions/{sessionId}` — 不变

### 新增/变更的认证路由
| 原版 | 新版 | 说明 |
|------|------|------|
| — | `DELETE /api/auth/logout` | 新增 — 清除当前会话的 cookie |
| — | `DELETE /api/auth/logout-all` | 新增 — 清除该用户所有会话的 cookie |
| — | `POST /api/auth/refresh` | 仅移动端使用；浏览器客户端由中间件静默处理 |

### 图书推荐路由增强
| 原版 | 新版 |
|------|------|
| `GET /api/member/recommendations` | `GET /api/member/recommendations?seed=<string>` |

`seed` 为可选参数。如果提供，则通过 FNV-64 哈希将字符串转为 `int64` 作为随机种子，实现确定性选择（用于"换一批"功能）。如果省略，后端随机生成。

---

## 9. 枚举类型设计

### 原版
简单字符串或整数常量，隐式值。

### 新版：以零值 Undefined 为安全设计
```go
type UserRole int
const (
    UserRoleUndefined UserRole = iota // 0 — 无效默认值
    UserRoleMember                     // 1
    UserRoleLibrarian                  // 2
    UserRoleAdmin                      // 3
)
```

**安全特性：** 任何意外未初始化的枚举变量默认为 0（`Undefined`），始终**无效**并通过验证失败，而非静默映射为 `ADMIN` 或 `ACTIVE`。

所有枚举字段在 repository/use-case 边界通过 `if !role.IsValid()` 进行校验。

所有枚举均生成 `String()` 方法和 `IsValid()` 校验方法。

**枚举定义于 `domain/shared/enum.go`：**
- `UserRole`（Member=1, Librarian=2, Admin=3）
- `UserStatus`（Pending=1, Active=2, Expired=3, Frozen=4）
- `LoanStatus`（Active=1, Returned=2）
- `CopyStatus`（Available=1, OnLoan=2, Maintenance=3, Lost=4, Removed=5）
- `ReservationStatus`（Queued=1, ReadyForPickup=2, Completed=3, Expired=4, Cancelled=5）
- `FineStatus`（Unpaid=1, Paid=2, Waived=3）
- `NotificationType`（DueReminder=1, OverdueReminder=2, ReservationReady=3, ReservationExpired=4, SystemAnnouncement=5）
- `SessionStatus`（Active=1, Expired=2, Revoked=3）

**对前端无影响** — 枚举为内部实现。API 在 JSON 中返回整数值（如 `status: 2`），前端自行映射为显示字符串。

---

## 10. 定时任务（后台任务）

| 任务 | 原版 | 新版 |
|------|------|------|
| 到期提醒 | `node-cron` 每天 08:00 | `time.NewTimer` 每 24 小时重置 |
| 逾期提醒 | `node-cron` 每天 08:30 | `time.NewTimer` 每 24 小时重置 |
| 预约超时顺延 | `node-cron` 每 30 分钟 | `time.NewTicker` 每 30 分钟 |
| 会话清理 | 无 | `time.NewTicker` 每 1 小时 + 服务器启动时运行 |

**功能行为完全相同。** 实现方式不同（Go goroutine vs. node-cron）。

---

## 11. 数据校验

### 原版
Joi schemas 与路由处理器分离定义：
```javascript
const schema = Joi.object({
  email: Joi.string().email(),
  password: Joi.string().min(6)
});
```

### 新版
go-playground/validator 在 `pkg/dto/` 的 DTO struct 上使用 tags：
```go
type RegisterRequest struct {
    Email    string `validate:"required,email"`
    Password string `validate:"required,min=6"`
    Name     string `validate:"required"`
    Phone    string `validate:"required"`
}
```

**对前端无影响** — 校验属于后端内部逻辑，校验规则不变。

---

## 12. Swagger API 文档

### 原版
未指定。

### 新版
所有 HTTP handler 带有 Swagger（`swaggo`）注解。运行 `swag init` 自动生成 `/swagger/*` 端点文档。

**暴露路由：**
- `GET /swagger/index.html` — Swagger UI
- `GET /swagger/doc.json` — OpenAPI 规范文档

**对前端的好处：** 可在 `/swagger/index.html` 获取交互式 API 文档。

---

## 13. 领域驱动设计分离

### 核心原则：Domain = 纯 Go
领域实体（`internal/domain/`）是纯 Go 结构体，**无 ORM 标签、无框架引用**。所有业务逻辑均在此层。

仓库接口定义在 `domain/<entity>/repository.go`：
```go
// domain/book/repository.go
type BookRepository interface {
    FindByID(ctx context.Context, id string) (*Book, error)
    Search(ctx context.Context, q *BookSearchQuery) ([]*Book, int64, error)
    Create(ctx context.Context, b *Book) error
    Update(ctx context.Context, b *Book) error
    Deactivate(ctx context.Context, id string) error
}
```

GORM 实现在 `repository/postgres/`。domain 层不导入 GORM。

**对前端无影响** — 属于后端架构内部设计，API 契约不变。

---

## 14. 配置

### 原版
未指定（可能为 `.env` 文件）。

### 新版
`config.yaml` 支持环境变量覆盖：
```yaml
database:
  host: "localhost"
  password: "${DB_PASSWORD}"   # 环境变量覆盖
app:
  host: "0.0.0.0"
  port: 8080
jwt:
  secret: "${JWT_SECRET}"
  accessTokenExpiry: 3600      # 1 小时
  refreshTokenExpiry: 604800    # 7 天
storage:
  baseDir: "./uploads"
  maxFileSize: 10485760         # 10MB
```

---

## 15. 功能行为差异

### 15.1 图书推荐 — 换一批功能
- **原版：** 未指定刷新机制
- **新版：** `GET /api/member/recommendations?seed=<string>`
  - 提供 `seed`：通过 FNV-64 哈希将字符串转为 `int64`，用于确定性随机选择
  - 不提供 `seed`：后端随机生成 `int64` 种子
  - 前端可在每次"换一批"点击时传入随机字符串，实现确定性刷新

### 15.2 冻结原因（Freeze Reason）
- **原版：** 未指定
- **新版：** 管理员冻结会员时**必须**提供 `reason` 字符串。存储在 `User.FreezeReason` 和审计日志中。

### 15.3 审计日志 — 操作原因字段
- **原版：** 基础审计日志
- **新版：** 每条审计日志包含 `Reason` 字段（操作原因说明，如"Suspected abuse"、"User requested"）

### 15.4 审计日志 — 数据快照
- **原版：** 未指定
- **新版：** `BeforeData` 和 `AfterData` 为 `map[string]any` 类型，存储在 jsonb 列
  - CREATE 操作时 `BeforeData` 为 `nil`
  - DELETE 操作时 `AfterData` 为 `nil`

### 15.5 Session 实体（刷新令牌）
- **原版：** 未建模为实体（token 仅存储在客户端）
- **新版：** 显式 `Session` 实体追踪所有刷新令牌
  - 支持多设备（最多 5 个）
  - 支持"登出所有设备"功能
  - 支持会话过期清理任务

---

## 16. 前端适配检查清单

### 认证相关
- [ ] **移除** `localStorage.getItem('token')` 和 `Authorization: Bearer` 请求头
- [ ] **移除** 收到 401 后手动调用 token 刷新的逻辑
- [ ] **保留** 登录响应体解析（`{ userId, role }`）— 结构不变
- [ ] **添加** cookie 处理 — 浏览器自动在同一源请求时发送 cookie
- [ ] **添加** 收到 `errorCode: "ERR_SESSION_EXPIRED"` 或 `"ERR_INVALID_REFRESH_TOKEN"` 时跳转登录页

### 错误处理
- [ ] **替换** 使用 `response.message` 作为用户文本的方式，改为用 `response.errorCode` 映射 i18n key
- [ ] **使用** `response.data` 中的上下文信息（如有）

### 推荐刷新
- [ ] "换一批"按钮：生成随机字符串，以 `?seed=<随机>` 传递
- [ ] 或不传 `seed` 参数，直接获取后端随机推荐结果

### 文件上传
- [ ] 无需行为变更 — multipart 上传流程相同
- [ ] 注意：`CoverURL` 字段格式变为哈希路径

### Swagger 文档
- [ ] 新 URL：`GET /swagger/index.html`

---

## 17. 全部变更汇总

| 类别 | 变更内容 |
|------|----------|
| 编程语言 | JavaScript → Go |
| Web 框架 | Express.js → Gin |
| 数据库 | MySQL → PostgreSQL |
| ORM | Prisma → GORM |
| 架构 | MVC → 清洁架构 + DDD |
| 认证 | localStorage + Bearer header → HttpOnly cookie + 静默中间件 |
| 会话 | 未建模 → 显式 Session 实体，每用户最多 5 个，支持多设备 |
| Token 刷新 | 前端显式调用 API → 中间件静默自动处理 |
| 文件存储 | Multer（基于文件名）→ CAS（SHA-256 内容寻址，支持去重） |
| 错误模型 | HTTP 状态码 + message → 细粒度领域错误码 + i18n |
| 数据校验 | Joi schemas → go-playground/validator（struct tags） |
| 依赖注入 | 手动 → Google Wire（编译期） |
| API 文档 | 无 → Swagger（swaggo 自动生成） |
| 定时任务 | node-cron → goroutine + time.Ticker |
| 会话清理 | 无 → 每小时清理任务 |
| 审计日志 | 基础 → 新增 `Reason` 字段，`BeforeData`/`AfterData` 改为 `map[string]any` jsonb |
| User 实体 | 基础 → 新增 `FreezeReason` 字段 |
| 枚举 | 隐式值 → 零值 Undefined 安全设计 |
| 领域层 | 耦合 ORM → 纯 Go 领域（无 ORM 标签） |
| 图书推荐 | 基础 → 支持 `seed` 参数实现确定性刷新 |
| Swagger | 无 → `/swagger/*` 自动生成文档 |

---