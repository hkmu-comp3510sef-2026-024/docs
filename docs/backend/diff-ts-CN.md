# 前端适配指南：原版 (Node.js/Express) vs. TypeScript/NestJS 后端

本文档聚焦于**前端开发者**需要关注的变更，与原版 Express 后端进行对比。

---

## 1. 技术栈变更（前端需知）

| 组件 | 原版 | 新版（TS/NestJS） | 前端影响 |
|------|------|-------------------|----------|
| 编程语言 | JavaScript | TypeScript 5.x | 无 — API 契约不变 |
| Web 框架 | Express.js | NestJS 10.x | 无 — API 契约不变 |
| ORM | Prisma | TypeORM | 无 — API 契约不变 |
| 数据库 | MySQL / MariaDB | PostgreSQL | 无 — API 契约不变 |
| 认证 | JWT + localStorage | @nestjs/jwt + HttpOnly Cookie | **见 §3** |
| 定时任务 | node-cron | @nestjs/schedule | 无 — 后台任务对前端透明 |
| 数据校验 | Joi | class-validator | 无 — 校验规则不变 |
| API 文档 | 未指定 | Swagger (@nestjs/swagger) | 新增：`GET /swagger/index.html` |

**总体结论：** 本次迁移的技术栈变更对前端**几乎透明**，API 契约保持不变。前端需要关注的核心变更集中在**认证机制**（Cookie 化）和**错误响应格式**（新增 `errorCode`）两方面。

---

## 2. API 路由变更汇总

### 2.1 行为不变的路由（前端代码无需修改）

- `POST /api/auth/register`
- `GET /api/books/search`
- `GET /api/books/{bookId}`
- `GET /api/member/profile`、`PUT /api/member/profile`
- `GET /api/member/membership`
- `GET /api/member/loans`、`POST /api/member/loans`、`POST /api/member/loans/{loanId}/renew`
- `GET /api/member/reservations`、`POST /api/member/reservations`、`DELETE /api/member/reservations/{reservationId}`
- `GET /api/member/notifications`、`PUT /api/notifications/{notificationId}/read`、`PUT /api/member/notifications/read-all`
- `GET /api/admin/members`、`POST /api/admin/members/{memberId}/approve`、`POST /api/admin/members/{memberId}/freeze`、`POST /api/admin/members/{memberId}/unfreeze`、`POST /api/admin/members/{memberId}/renew`
- `GET /api/admin/books`、`POST /api/admin/book`、`PUT /api/admin/book/{bookId}`、`PUT /api/admin/book/{bookId}/deactivate`
- `GET /api/admin/copies`、`POST /api/admin/copy`、`PUT /api/admin/copy/{copyId}/status`
- `GET /api/admin/fines`、`POST /api/admin/fines/{fineId}/pay`、`POST /api/admin/fines/{fineId}/waive`
- `GET /api/admin/reminder-policy`、`PUT /api/admin/reminder-policy`
- `GET /api/admin/audit-logs`、`GET /api/admin/audit-logs/{logId}`
- `POST /api/circulation/checkout`、`POST /api/circulation/return`、`GET /api/circulation/lookup`
- `GET /api/assistant/quick-questions`、`POST /api/assistant/sessions`、`POST /api/assistant/chat`、`GET /api/assistant/sessions/{sessionId}/messages`、`DELETE /api/assistant/sessions/{sessionId}`

### 2.2 新增/变更的路由

| 原版 | 新版（TS） | 说明 |
|------|-----------|------|
| — | `PUT /api/member/profile/avatar` | **新增** — 头像上传，`multipart/form-data`，返回 `{ avatarUrl }` |
| — | `PUT /api/admin/books/:bookId/cover` | **新增** — 书籍封面上传，`multipart/form-data`，更新 `coverUrl` |
| — | `POST /api/admin/notifications/announcements` | **新增** — 广播系统公告给所有活跃会员，body: `{ title, message }`，返回 `{ count }` |
| `GET /api/member/recommendations` | `GET /api/member/recommendations?seed=<string>` | **新增 `seed` 参数** — 用于"换一批"功能，见 §6 |
| — | `DELETE /api/auth/logout` | **新增** — 清除当前会话的 HttpOnly Cookie |
| — | `DELETE /api/auth/logout-all` | **新增** — 清除该用户所有会话 |
| — | `GET /api/admin/assistant/knowledge` | **新增** — 分页查询知识库条目（管理员） |
| — | `GET /api/admin/assistant/knowledge/:knowledgeId` | **新增** — 获取单条知识库条目 |
| — | `POST /api/admin/assistant/knowledge` | **新增** — 创建知识库条目 |
| — | `PUT /api/admin/assistant/knowledge/:knowledgeId` | **新增** — 更新知识库条目 |
| — | `DELETE /api/admin/assistant/knowledge/:knowledgeId` | **新增** — 删除知识库条目（硬删除） |

---

## 3. 认证机制（重大变更）

### 原版设计
```
POST /api/auth/login → 响应体返回 { token, role, userId }
Token 存储在 localStorage
通过 Authorization: Bearer <token> 请求头发送
```

### 新版设计：HttpOnly Cookie + 静默自动刷新

**Cookie 规格：**

| Cookie 名称 | 类型 | HttpOnly | Secure | Max-Age | 内容 |
|------------|------|----------|--------|---------|------|
| `access_token` | JWT | Yes | Yes* | 3600（1小时） | HS256 签名 JWT |
| `refresh_token` | 不透明 | Yes | Yes* | 604800（7天） | 原始 refresh token |

\* 生产环境启用，本地开发可配置关闭。

### 前端适配要点

1. **删除 localStorage 存储 token 的逻辑**
   ```javascript
   // ❌ 删除 — 原版做法
   localStorage.setItem('token', response.data.token);
   axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;

   // ✅ 新版 — 浏览器自动发送 Cookie，无需前端做任何事
   // axios 无需配置 Authorization header
   ```

2. **删除手动刷新 token 的逻辑**
   ```javascript
   // ❌ 删除 — 原版需要在 401 时手动刷新
   if (response.status === 401) {
     await refreshToken();
   }

   // ✅ 新版 — 后端中间件静默处理，前端不会收到因 token 过期导致的 401
   ```

3. **登录响应体解析逻辑不变**
   ```javascript
   // ✅ 保留 — 结构相同
   const { userId, role } = response.data;
   ```

4. **登出逻辑变更**
   ```javascript
   // ❌ 删除 — 原版只需清除 localStorage
   localStorage.removeItem('token');

   // ✅ 新版 — 调用后端登出接口，后端清除 Cookie
   await axios.delete('/api/auth/logout');
   localStorage.clear(); // 清除其他本地数据
   ```

5. **会话过期处理**
   ```javascript
   // 当收到以下 errorCode 时，跳转登录页
   if (error.errorCode === 'ERR_SESSION_EXPIRED' ||
       error.errorCode === 'ERR_INVALID_REFRESH_TOKEN' ||
       error.errorCode === 'ERR_UNAUTHORIZED') {
     window.location.href = '/login';
   }
   ```

6. **多设备支持**
   - 后端支持每用户最多 5 个活跃会话
   - 新增 `DELETE /api/auth/logout-all` 可一键登出所有设备
   - 前端可考虑在个人资料页添加"退出所有设备"按钮

### NestJS 实现细节（前端无感知）

- `@nestjs/jwt` 替代 `jsonwebtoken`
- NestJS 的 `JwtAuthGuard` 在每个请求前自动验证 Cookie 中的 JWT
- 刷新失败采用 **fail-open** 策略：JWT 验证错误不直接拒绝请求，而是尝试静默刷新
- 刷新成功后自动设置新 Cookie，前端无感知

---

## 4. 错误响应格式（新增 errorCode）

### 原版错误响应
```json
{ "code": 400, "message": "Bad Request" }
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

### 前端适配要点

1. **使用 `errorCode` 作为 i18n key**
   ```javascript
   // ❌ 废弃 — 不再使用 message 作为用户文本
   ElMessage.error(response.message);

   // ✅ 新版 — 使用 errorCode 映射 i18n
   ElMessage.error(t(`errors.${response.errorCode}`));
   ```

2. **`data` 字段用于上下文信息**
   ```javascript
   // 示例：超期罚款错误
   if (error.errorCode === 'ERR_RENEWAL_LIMIT_REACHED') {
     ElMessage.warning(t('errors.ERR_RENEWAL_LIMIT_REACHED'));
   }
   ```

### 完整 errorCode 列表

| errorCode | HTTP 状态码 | 说明 |
|-----------|-------------|------|
| `ERR_USER_EMAIL_EXISTS` | 409 | 邮箱已被注册 |
| `ERR_USER_NOT_FOUND` | 404 | 用户不存在 |
| `ERR_USER_ACCOUNT_PENDING` | 422 | 账户等待管理员审核 |
| `ERR_USER_ACCOUNT_FROZEN` | 422 | 账户已被冻结 |
| `ERR_USER_ACCOUNT_EXPIRED` | 422 | 会员已过期 |
| `ERR_BOOK_NOT_FOUND` | 404 | 图书不存在 |
| `ERR_BOOK_INACTIVE` | 422 | 图书已下架 |
| `ERR_COPY_NOT_FOUND` | 404 | 副本（条码）不存在 |
| `ERR_NO_AVAILABLE_COPIES` | 422 | 没有可借的副本 |
| `ERR_LOAN_NOT_FOUND` | 404 | 借阅记录不存在 |
| `ERR_LOAN_NOT_ACTIVE` | 422 | 借阅记录不处于活跃状态 |
| `ERR_RENEWAL_LIMIT_REACHED` | 422 | 已达到最大续借次数 |
| `ERR_RENEWAL_BLOCKED_BY_RESERVATION` | 422 | 该书存在排队预约，续借被拒绝 |
| `ERR_RESERVATION_ALREADY_EXISTS` | 409 | 该图书已存在有效预约 |
| `ERR_RESERVATION_NOT_FOUND` | 404 | 预约记录不存在 |
| `ERR_RESERVATION_CANNOT_CANCEL` | 422 | 当前状态不允许取消预约 |
| `ERR_KNOWLEDGE_NOT_FOUND` | 404 | 知识库条目不存在 |
| `ERR_FINE_NOT_FOUND` | 404 | 罚款记录不存在 |
| `ERR_SESSION_EXPIRED` | 401 | 刷新令牌会话已过期 |
| `ERR_INVALID_REFRESH_TOKEN` | 401 | 刷新令牌无效或不存在 |
| `ERR_VALIDATION_FAILED` | 400 | 请求验证失败 |
| `ERR_UNAUTHORIZED` | 401 | JWT token 缺失或无效 |
| `ERR_FORBIDDEN` | 403 | 无权限访问 |
| `ERR_INTERNAL` | 500 | 意外的服务器错误 |

### Axios 拦截器适配示例

```typescript
// axios 拦截器
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const { response } = error;
    if (response?.data?.errorCode) {
      const { errorCode, data } = response.data;
      // 使用 errorCode 做 i18n
      ElMessage.error(t(`errors.${errorCode}`));
      // 特殊处理：会话过期跳转登录
      if (['ERR_SESSION_EXPIRED', 'ERR_INVALID_REFRESH_TOKEN', 'ERR_UNAUTHORIZED'].includes(errorCode)) {
        router.push('/login');
      }
      return Promise.reject({ errorCode, data });
    }
    return Promise.reject(error);
  }
);
```

---

## 5. 响应 DTO 变更

### 5.1 借阅响应（Loan）新增字段

借阅列表和详情响应中新增 `remainingRenewals` 字段：

```json
{
  "loanId": "uuid",
  "bookId": "uuid",
  "title": "书名",
  "borrowedAt": "2026-03-01T00:00:00Z",
  "dueDate": "2026-03-15T00:00:00Z",
  "status": 1,
  "renewalCount": 0,
  "remainingRenewals": 2
}
```

**前端适配：**
```javascript
// ✅ 续借按钮显示逻辑
const canRenew = loan.remainingRenewals > 0;
```

### 5.2 预约响应（Reservation）新增字段

预约列表响应中，状态为 `QUEUED`（排队中）的记录新增 `queuePosition` 字段：

```json
{
  "reservationId": "uuid",
  "bookId": "uuid",
  "title": "书名",
  "status": 1,
  "queuePosition": 3,
  "createdAt": "2026-03-01T00:00:00Z"
}
```

**前端适配：**
```javascript
// ✅ 排队位置显示
if (reservation.status === 1) { // QUEUED
  ElBadge.set(`排队第 ${reservation.queuePosition} 位`);
}
```

### 5.3 会员状态（UserStatus）枚举值

| 枚举值 | 整数 | 说明 |
|--------|------|------|
| `Pending` | 1 | 待审核 |
| `Active` | 2 | 有效 |
| `Expired` | 3 | 已过期 |
| `Frozen` | 4 | 已冻结 |

> ⚠️ 原版文档中 `status` 字段使用字符串（如 `"PENDING"`），新版 TS 后端返回整数。**前端需注意类型兼容性**，在显示时进行映射。

### 5.4 副本状态（CopyStatus）枚举值

| 枚举值 | 整数 | 说明 |
|--------|------|------|
| `Available` | 1 | 在馆可借 |
| `OnLoan` | 2 | 外借中 |
| `Maintenance` | 3 | 维修中 |
| `Lost` | 4 | 丢失 |
| `Removed` | 5 | 下架 |

### 5.5 预约状态（ReservationStatus）枚举值

| 枚举值 | 整数 | 说明 |
|--------|------|------|
| `Queued` | 1 | 排队中 |
| `ReadyForPickup` | 2 | 可取书 |
| `Completed` | 3 | 已完成 |
| `Expired` | 4 | 已过期 |
| `Cancelled` | 5 | 已取消 |

---

## 6. 图书推荐 — 换一批功能

### 路由变更

| 原版 | 新版 |
|------|------|
| `GET /api/member/recommendations` | `GET /api/member/recommendations?seed=<string>` |

### `seed` 参数说明

- **提供 `seed`**：传入随机字符串，后端通过 FNV-64a 哈希转为 `int64` 随机种子，实现**确定性**的推荐结果（点击"换一批"时传入不同 seed 得到不同推荐集）
- **省略 `seed`**：后端随机生成种子，每次返回不同推荐

### 前端实现示例

```javascript
// 推荐组件
const loadRecommendations = (seed) => {
  const params = seed ? { seed } : {};
  return axios.get('/api/member/recommendations', { params });
};

// 换一批按钮
let seed = '';
const handleRefresh = () => {
  seed = Math.random().toString(36).substring(2);
  loadRecommendations(seed).then(...);
};
```

---

## 7. 文件上传变更

### 头像上传

**接口：** `PUT /api/member/profile/avatar`
**Content-Type：** `multipart/form-data`
**请求体：** `file` 字段（图片文件）

```javascript
const formData = new FormData();
formData.append('file', fileInput.files[0]);
const response = await axios.put('/api/member/profile/avatar', formData, {
  headers: { 'Content-Type': 'multipart/form-data' }
});
// 返回 { avatarUrl: "18/23/ad15...c3.png" }
```

### 书籍封面上传（管理员）

**接口：** `PUT /api/admin/books/:bookId/cover`
**Content-Type：** `multipart/form-data`
**请求体：** `file` 字段（图片文件）

### CAS 存储路径格式

新版采用内容寻址存储（CASS），`CoverURL` 格式变为哈希路径：
```
原始文件名：cover.png
存储路径：  18/23/ad1537c537c02199358e2132e3cf25948c1e0b25134699dc79d1f634b8c3.png
```

> ⚠️ 前端图片展示组件需确保能正确拼接完整 URL（如 `/uploads/18/23/ad15...c3.png`）。

---

## 8. 系统公告广播（管理员新功能）

**接口：** `POST /api/admin/notifications/announcements`
**权限：** LIBRARIAN / ADMIN
**请求体：**
```json
{
  "title": "系统维护通知",
  "message": "系统将于本周六 22:00-23:00 进行维护。"
}
```
**响应：**
```json
{
  "code": 200,
  "data": { "count": 42 }
}
```
表示成功向 42 位活跃会员发送了通知。

---

## 9. 知识库管理（管理员新功能）

管理员可对智能问答助手的知识库进行 CRUD 操作：

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/admin/assistant/knowledge` | GET | 分页查询知识库 |
| `/api/admin/assistant/knowledge/:knowledgeId` | GET | 获取单条 |
| `/api/admin/assistant/knowledge` | POST | 创建条目 |
| `/api/admin/assistant/knowledge/:knowledgeId` | PUT | 更新条目 |
| `/api/admin/assistant/knowledge/:knowledgeId` | DELETE | 删除条目 |

**知识分类（KnowledgeCategory）：**
- `LOAN_RULES` — 借阅规则
- `RESERVATION` — 预约规则
- `FINE` — 罚款规则
- `SYSTEM_USAGE` — 系统使用
- `OTHER` — 其他

> 前端管理员界面需新增知识库管理页面。

---

## 10. 前端适配完整检查清单

### 认证模块
- [ ] **移除** `localStorage.getItem('token')`
- [ ] **移除** `axios.defaults.headers.common['Authorization']` 配置
- [ ] **移除** 手动 token 刷新逻辑
- [ ] **保留** 登录响应体解析（`{ userId, role }`）
- [ ] **实现** 登出时调用 `DELETE /api/auth/logout`
- [ ] **实现** 收到 `ERR_SESSION_EXPIRED` / `ERR_INVALID_REFRESH_TOKEN` / `ERR_UNAUTHORIZED` 时跳转登录页
- [ ] **可选** 实现"退出所有设备"功能（调用 `DELETE /api/auth/logout-all`）

### 错误处理
- [ ] **改造** Axios 拦截器，使用 `errorCode` 而非 `message` 作为 i18n key
- [ ] **实现** 所有 errorCode 的中文 i18n 映射
- [ ] **处理** `ERR_RENEWAL_BLOCKED_BY_RESERVATION` 错误（该书存在排队预约）

### 借阅模块
- [ ] **适配** 借阅响应中的 `remainingRenewals` 字段计算续借按钮状态
- [ ] **移除** 前端自行计算 `maxRenewals - renewalCount` 的逻辑

### 预约模块
- [ ] **适配** 预约响应中的 `queuePosition` 字段显示排队位置

### 枚举映射
- [ ] **改造** 所有状态字段的显示映射，从字符串改为整数枚举

### 头像/封面上传
- [ ] **适配** 头像上传使用 `PUT /api/member/profile/avatar`
- [ ] **适配** 书籍封面上传使用 `PUT /api/admin/books/:bookId/cover`
- [ ] **适配** 图片 URL 拼接逻辑（CAS 哈希路径格式）

### 图书推荐
- [ ] **实现** "换一批"按钮传入 `seed` 参数

### 管理员功能
- [ ] **新增** 系统公告广播功能界面
- [ ] **新增** 知识库管理界面（CRUD）

### Swagger 文档
- [ ] **访问** `GET /swagger/index.html` 查阅完整 API 文档

---

## 11. 与 Go 后端版本的差异说明

本 TS/NestJS 版本与 Go 版本在**前端可见的 API 层面完全一致**，核心变更相同：

| 变更项 | Go 版 | TS/NestJS 版 | 前端影响 |
|--------|-------|--------------|----------|
| 认证机制 | ✅ HttpOnly Cookie + 静默刷新 | ✅ HttpOnly Cookie + 静默刷新 | 无差异 |
| 错误格式 | ✅ `errorCode` | ✅ `errorCode` | 无差异 |
| 响应 DTO | ✅ `remainingRenewals` + `queuePosition` | ✅ `remainingRenewals` + `queuePosition` | 无差异 |
| 文件上传 | ✅ CAS | ✅ CAS | 无差异 |
| `seed` 参数 | ✅ 支持 | ✅ 支持 | 无差异 |

**唯一后端内部差异（前端无感知）：**
- TypeScript 使用 `@nestjs/jwt`，Go 使用 `golang-jwt/jwt`
- NestJS 使用 Guards/Exception Filters 做认证/错误处理，Go 使用中间件
- TypeORM 替代 GORM

---

## 12. 变更汇总

| 类别 | 变更内容 | 前端影响等级 |
|------|----------|-------------|
| 认证 | localStorage → HttpOnly Cookie | ⭐⭐⭐ 重大 |
| 错误处理 | 新增 `errorCode` 字段 | ⭐⭐⭐ 重大 |
| 借阅响应 | 新增 `remainingRenewals` | ⭐⭐ 中等 |
| 预约响应 | 新增 `queuePosition` | ⭐⭐ 中等 |
| 枚举值 | 字符串 → 整数 | ⭐⭐ 中等 |
| 头像上传 | 新接口 `PUT /api/member/profile/avatar` | ⭐ 低 |
| 封面上传 | 新接口 `PUT /api/admin/books/:bookId/cover` | ⭐ 低 |
| 推荐刷新 | `seed` 参数支持 | ⭐ 低 |
| 系统公告 | 新接口广播通知 | ⭐ 低 |
| 知识库管理 | 新增 5 个管理员接口 | ⭐ 低 |
| API 文档 | 新增 Swagger UI | ⭐ 无 |
