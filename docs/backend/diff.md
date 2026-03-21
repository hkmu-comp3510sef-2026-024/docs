# Design Diff: Original (Node.js/Express) vs. Go Backend

## 1. Technology Stack

| Component | Original (Node.js) | New (Go) | Impact |
|-----------|-------------------|----------|--------|
| Language | JavaScript/Node.js 18+ | Go 1.26+ | Backend rewritten in Go |
| Web Framework | Express.js 5.2.1 | Gin | Standard Go REST framework |
| ORM | Prisma 7.4.2 | GORM | Most popular Go ORM, auto-migration |
| Database | MySQL / MariaDB | PostgreSQL | Richer JSON (jsonb) support for AuditLog |
| Authentication | JWT (jsonwebtoken 9.0.3) + bcryptjs | golang-jwt/jwt v5 + bcrypt | **See §5 Auth & Sessions** |
| Background Jobs | node-cron 4.2.1 | Built-in goroutines + time.Ticker | Same schedule logic, different Go idiom |
| Validation | Joi 18.0.2 | go-playground/validator | Struct tags on DTOs vs. Joi schemas |
| File Upload | Multer 2.1.0 | CAS (Content-Addressable Storage) | **See §6 File Storage** |
| API Docs | Not specified | Swagger (swaggo) | Auto-generated from handler annotations |
| DI | Manual / not specified | Google Wire | Compile-time dependency injection |

---

## 2. Project Architecture

### Original — Standard Express/MVC
```
src/
  routes/
  controllers/
  models/ (Prisma)
  middleware/
  services/
```

### New — Clean Architecture + DDD
```
cmd/server/              # Entrypoint + Wire bootstrap
internal/
  domain/                # PURE GO — entities, interfaces, domain errors (NO ORM tags)
    user/, book/, copy/, loan/, reservation/, fine/
    notification/, auditlog/, reminderpolicy/, assistant/, session/
    shared/enum.go       # All enumerated values
  usecase/               # PURE GO — business rules, depends on domain interfaces
  repository/            # GORM implementations of domain repository interfaces
    postgres/            # PostgreSQL-specific implementations
  delivery/              # External-facing adapters
    http/                # Gin HTTP handlers
    cron/                # Background job runners
    grpc/                # Future stub
  infrastructure/        # Concrete implementations
    storage/             # LocalDiskStorage (CAS)
    jwt/                 # JWT provider
    hasher/              # bcrypt
    config/              # YAML config loader
pkg/
  validator/             # go-playground/validator wrapper
  response/              # Unified API response helpers
  pagination/            # Pagination helpers
migrations/             # SQL migration files
config.yaml
```

**Key Architectural Differences:**
- Domain layer is **pure Go** with no framework dependencies
- GORM tags only in the **repository layer**, not on domain entities
- All external adapters (HTTP framework, storage, JWT) are **pluggable** via interfaces defined in the domain layer (Dependency Inversion)
- Google Wire provides **compile-time DI** — no runtime reflection-based DI container

---

## 3. Database

| Aspect | Original | New |
|--------|----------|-----|
| Engine | MySQL / MariaDB | PostgreSQL |
| ORM | Prisma (auto-generated CRUD) | GORM (manual query building) |
| AuditLog Before/After | Not specified | `jsonb` columns (`map[string]any`) |
| UUID Usage | Not specified | UUIDv7 for entities with embedded timestamps (e.g., AuditLog.ID, Session.ID) |

> **Frontend Impact:** None — same tables/entities. Database change is transparent to the frontend.

---

## 4. Entities (Data Model)

**Original:** 12 entities
**New:** 13 entities (all 12 original + `Session`)

### New Entity: Session
```go
type Session struct {
    ID        uuid.UUID  // UUIDv7 — embedded timestamp, no CreatedAt needed
    UserID    uuid.UUID
    TokenHash string     // SHA-256 of raw refresh token — raw token never persisted
    UserAgent string     // Device/browser captured at login
    IPAddress string     // Client IP captured at login
    ExpiresAt time.Time  // 7 days from creation
}
```
Purpose: Refresh token storage with multi-device support (max 5 sessions per user).

### AuditLog Changes
```go
type AuditLog struct {
    ID         uuid.UUID      // UUIDv7 (embedded timestamp)
    UserID     uuid.UUID
    Action     string         // e.g. "USER_FREEZE", "BOOK_UPDATE"
    EntityType string
    EntityID   string
    Reason     string         // Human-readable reason (NEW field)
    BeforeData map[string]any // jsonb — snapshot BEFORE (nil on CREATE)
    AfterData  map[string]any // jsonb — snapshot AFTER (nil on DELETE)
    IPAddress  string
    UserAgent  string
}
```
**Changes:** Added `Reason` field; `BeforeData`/`AfterData` now typed as `map[string]any` (stored as `jsonb`).

### User Entity Changes
**New field:** `FreezeReason string` — recorded when admin freezes a member account.

---

## 5. Authentication & Sessions (Major Change)

### Original Design
```
POST /api/auth/login → returns { token, role, userId } in BODY
Token stored in localStorage
Sent via Authorization: Bearer <token> header
Single token per login
No explicit refresh endpoint usage (token assumed long-lived or unchecked)
```

### New Design: All-Cookie + Silent Auto-Refresh

| Cookie | Type | HttpOnly | Secure | Max-Age | Contents |
|--------|------|----------|--------|---------|---------|
| `access_token` | JWT | Yes | Yes* | 3600 (1h) | HS256-signed JWT |
| `refresh_token` | Opaque | Yes | Yes* | 604800 (7d) | Raw token (lookup key) |

*Secure flag: enabled in production, configurable off for local dev.

**Key JWT Payload:** `{ "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN", "sessionId": "uuid" }`

#### Silent Auto-Refresh Middleware Flow
```
Request arrives
  → Check access_token cookie (stateless, NO DB call)
  → If valid → proceed
  → If expired/missing → check refresh_token cookie
      → Lookup Session by token hash (DB call)
      → Validate expiry
      → Generate new JWT + set new cookie
      → Proceed
```
- **99% of requests:** zero DB calls (stateless JWT validation only)
- **DB call only:** when access token is expired AND valid refresh token exists
- **Frontend never sees 401 from token expiry** — server self-heals silently
- **Multi-device:** Up to 5 active sessions per user; 6th login evicts oldest session

#### Refresh Token Security
- Raw token stored ONLY in browser cookie (HttpOnly — invisible to JS)
- Server stores only `SHA-256(raw token)` hash (TokenHash)
- Even if DB is compromised, raw tokens cannot be recovered

#### Session Cleanup Job
- Runs every **1 hour** via `time.Ticker`
- Also runs on **server startup**
- Deletes all Session records where `ExpiresAt < now`

#### API Changes

| Original | New |
|----------|-----|
| `POST /api/auth/login` → `{ token, role, userId }` in body | `POST /api/auth/login` → sets HttpOnly cookies, returns `{ userId, role }` in body |
| `Authorization: Bearer <token>` header on all requests | **No header needed** — cookie-only |
| No logout-all | `DELETE /api/auth/logout-all` — deletes ALL sessions for user |
| No session management | Multi-session tracking, max 5 per user |
| `POST /api/auth/refresh` (explicit call) | **Not needed for browser clients** — middleware handles silently |
| N/A | `POST /api/auth/refresh` still available for mobile clients (body: `{ refreshToken })` |

**Frontend Adaptation Required:**
- Remove `localStorage.setItem('token', ...)` and `Authorization` header logic
- Remove manual token refresh on 401
- Login response now contains `{ userId, role }` in body (same as before)
- Logout calls `DELETE /api/auth/logout` (clears cookies server-side)
- On any API error with `errorCode: "ERR_SESSION_EXPIRED"` or `"ERR_INVALID_REFRESH_TOKEN"`, redirect to login

---

## 6. File Storage (NEW: CAS Pattern)

### Original
- Multer handles `multipart/form-data` uploads
- Files stored with original filename or generated name
- No deduplication

### New: Content-Addressable Storage (CAS)
```
File Content → SHA-256 hash → path: {h[0:2]}/{h[2:4]}/{h[4:]}<ext>
Example: hash "1823ad...b8c3", ext ".png"
         → "18/23/ad1537c537c02199358e2132e3cf25948c1e0b25134699dc79d1f634b8c3.png"
```

**Properties:**
- **Deduplication:** If file content already exists, storage is skipped (same hash = same file)
- **No full file in memory:** 64KB buffer loop
- **Thread-safety noted** for future S3 upgrade
- Storage path returned as `CoverURL` (e.g., `"18/23/ad15...c3.png"`)

**Frontend Impact:**
- Avatar upload and book cover upload behavior unchanged (same multipart flow)
- `CoverURL` returned in book/profile responses will be a hash-based path instead of a traditional filename

---

## 7. Error Handling

### Original Error Responses
```json
{ "code": 400, "message": "Bad Request" }
{ "code": 401, "message": "Unauthorized" }
...
```

### New Error Responses
```json
{
  "code": 422,
  "errorCode": "ERR_NO_AVAILABLE_COPIES",
  "message": "No available copies for this book",
  "data": { "bookId": "123" }
}
```

**Key Differences:**
- `errorCode` is a **granular, domain-specific string** (e.g., `ERR_NO_AVAILABLE_COPIES`, `ERR_USER_ACCOUNT_FROZEN`, `ERR_RENEWAL_LIMIT_REACHED`)
- `message` is **standard English** for dev/debug only
- `data` carries structured context from the error
- Frontend uses `errorCode` as an **i18n key** to display localized messages

### Granular Error Codes (Exhaustive List)

| errorCode | HTTP Status | Description |
|-----------|-------------|-------------|
| `ERR_USER_EMAIL_EXISTS` | 409 | Email already registered |
| `ERR_USER_NOT_FOUND` | 404 | User does not exist |
| `ERR_USER_ACCOUNT_PENDING` | 422 | Account awaiting admin approval |
| `ERR_USER_ACCOUNT_FROZEN` | 422 | Account is frozen |
| `ERR_USER_ACCOUNT_EXPIRED` | 422 | Membership has expired |
| `ERR_BOOK_NOT_FOUND` | 404 | Book does not exist |
| `ERR_BOOK_INACTIVE` | 422 | Book is not active (deactivated) |
| `ERR_COPY_NOT_FOUND` | 404 | Copy (barcode) does not exist |
| `ERR_NO_AVAILABLE_COPIES` | 422 | No available copies to borrow |
| `ERR_LOAN_NOT_FOUND` | 404 | Loan record not found |
| `ERR_LOAN_NOT_ACTIVE` | 422 | Loan is not in active state |
| `ERR_RENEWAL_LIMIT_REACHED` | 422 | Maximum renewal count exceeded |
| `ERR_RESERVATION_ALREADY_EXISTS` | 409 | Active reservation already exists for this book |
| `ERR_RESERVATION_NOT_FOUND` | 404 | Reservation not found |
| `ERR_RESERVATION_CANNOT_CANCEL` | 422 | Reservation cannot be cancelled in current state |
| `ERR_FINE_NOT_FOUND` | 404 | Fine record not found |
| `ERR_SESSION_EXPIRED` | 401 | Refresh token session has expired |
| `ERR_INVALID_REFRESH_TOKEN` | 401 | Refresh token is invalid or not found |
| `ERR_VALIDATION_FAILED` | 400 | Request validation failed |
| `ERR_UNAUTHORIZED` | 401 | Missing or invalid JWT token |
| `ERR_FORBIDDEN` | 403 | Caller does not have permission |
| `ERR_INTERNAL` | 500 | Unexpected server error |

**Frontend Adaptation Required:**
- Error handling must use `errorCode` (not `message`) for user-facing text
- `data` field may contain contextual info (e.g., `{ bookId }`) for custom UI copy

---

## 8. API Routes — Summary of Changes

### Unchanged Routes (Same Path, Same Behavior)
- `POST /api/auth/register` — behavior unchanged
- `GET /api/books/search` — behavior unchanged
- `GET /api/books/{bookId}` — behavior unchanged
- `GET /api/member/profile`, `PUT /api/member/profile` — unchanged
- `GET /api/member/membership` — unchanged
- `GET /api/member/loans`, `POST /api/member/loans`, `POST /api/member/loans/{loanId}/renew` — unchanged
- `GET /api/member/reservations`, `POST /api/member/reservations`, `DELETE /api/member/reservations/{reservationId}` — unchanged
- `GET /api/member/notifications`, `PUT /api/notifications/{notificationId}/read`, `PUT /api/member/notifications/read-all` — unchanged
- `GET /api/admin/members`, `POST /api/admin/members/{memberId}/approve`, `POST /api/admin/members/{memberId}/freeze`, `POST /api/admin/members/{memberId}/unfreeze`, `POST /api/admin/members/{memberId}/renew` — unchanged
- `GET /api/admin/books`, `POST /api/admin/book`, `PUT /api/admin/book/{bookId}`, `PUT /api/admin/book/{bookId}/deactivate` — unchanged
- `GET /api/admin/copies`, `POST /api/admin/copy`, `PUT /api/admin/copy/{copyId}/status` — unchanged
- `GET /api/admin/fines`, `POST /api/admin/fines/{fineId}/pay`, `POST /api/admin/fines/{fineId}/waive` — unchanged
- `GET /api/admin/reminder-policy`, `PUT /api/admin/reminder-policy` — unchanged
- `GET /api/admin/audit-logs`, `GET /api/admin/audit-logs/{logId}` — unchanged
- `POST /api/circulation/checkout`, `POST /api/circulation/return`, `GET /api/circulation/lookup` — unchanged
- `GET /api/assistant/quick-questions`, `POST /api/assistant/sessions`, `POST /api/assistant/chat`, `GET /api/assistant/sessions/{sessionId}/messages`, `DELETE /api/assistant/sessions/{sessionId}` — unchanged

### New/Modified Auth Routes
| Original | New | Notes |
|----------|-----|-------|
| — | `DELETE /api/auth/logout` | New — clears current session cookies |
| — | `DELETE /api/auth/logout-all` | New — clears ALL user sessions |
| — | `POST /api/auth/refresh` | Mobile clients only; browser clients use silent middleware |

### Recommendations Route Enhancement
| Original | New |
|----------|-----|
| `GET /api/member/recommendations` | `GET /api/member/recommendations?seed=<string>` |

`seed` is optional. If provided, it drives deterministic selection for "换一批" (refresh) functionality. If omitted, a random seed is used server-side.

---

## 9. Enumerated Values (Enum Design)

### Original
Simple string or integer constants with implicit values.

### New: Zero-Based Undefined as Security Design
```go
type UserRole int
const (
    UserRoleUndefined UserRole = iota // 0 — INVALID default
    UserRoleMember                     // 1
    UserRoleLibrarian                  // 2
    UserRoleAdmin                      // 3
)
```

**Security Property:** Any accidentally uninitialized enum variable defaults to 0 (`Undefined`), which is always **invalid** and fails validation — it never silently map to `ADMIN` or `ACTIVE`.

Every entity with an enum field validates with `if !role.IsValid()` at repository/use-case boundaries.

All enums have a generated `String()` method and `IsValid()` check.

**Enums defined in `domain/shared/enum.go`:**
- `UserRole` (Member=1, Librarian=2, Admin=3)
- `UserStatus` (Pending=1, Active=2, Expired=3, Frozen=4)
- `LoanStatus` (Active=1, Returned=2)
- `CopyStatus` (Available=1, OnLoan=2, Maintenance=3, Lost=4, Removed=5)
- `ReservationStatus` (Queued=1, ReadyForPickup=2, Completed=3, Expired=4, Cancelled=5)
- `FineStatus` (Unpaid=1, Paid=2, Waived=3)
- `NotificationType` (DueReminder=1, OverdueReminder=2, ReservationReady=3, ReservationExpired=4, SystemAnnouncement=5)
- `SessionStatus` (Active=1, Expired=2, Revoked=3)

**Frontend Impact:** None — enum values are internal. API returns integer values for enums in JSON (e.g., `status: 2`), which the frontend maps to display strings.

---

## 10. Background Jobs (Scheduler)

| Job | Original | New |
|-----|----------|-----|
| Due Reminder | `node-cron` daily 08:00 | `time.NewTimer` reset every 24h |
| Overdue Reminder | `node-cron` daily 08:30 | `time.NewTimer` reset every 24h |
| Reservation Expiry | `node-cron` every 30 min | `time.NewTicker` every 30 min |
| Session Cleanup | N/A | `time.NewTicker` every 1 hour + runs on startup |

**Functional behavior is identical.** Implementation idiom differs (Go goroutines vs. node-cron).

---

## 11. Validation

### Original
Joi schemas defined separately from route handlers:
```javascript
const schema = Joi.object({
  email: Joi.string().email(),
  password: Joi.string().min(6)
});
```

### New
go-playground/validator with struct tags on DTO types in `pkg/dto/`:
```go
type RegisterRequest struct {
    Email    string `validate:"required,email"`
    Password string `validate:"required,min=6"`
    Name     string `validate:"required"`
    Phone    string `validate:"required"`
}
```

**Frontend Impact:** None — validation is a backend concern. Same validation rules apply.

---

## 12. Swagger API Documentation

### Original
Not specified.

### New
All HTTP handlers annotated with Swagger (`swaggo`) comments. Running `swag init` auto-generates `/swagger/*` endpoints.

**Routes exposed:**
- `GET /swagger/index.html` — Swagger UI
- `GET /swagger/doc.json` — OpenAPI spec

**Frontend Benefit:** Auto-generated interactive API docs at `/swagger/index.html`.

---

## 13. Domain-Driven Design Separation

### Key Principle: Domain = Pure Go
Domain entities (`internal/domain/`) are plain Go structs with **no ORM tags, no framework imports**. All business logic lives here.

Repository interfaces are defined in `domain/<entity>/repository.go`:
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

GORM implementations live in `repository/postgres/`. The domain never imports GORM.

**Frontend Impact:** None — this is a backend architecture concern. The API contract remains unchanged.

---

## 14. Configuration

### Original
Not specified (likely `.env` files).

### New
`config.yaml` with environment variable overrides:
```yaml
database:
  host: "localhost"
  password: "${DB_PASSWORD}"   # env var override
app:
  host: "0.0.0.0"
  port: 8080
jwt:
  secret: "${JWT_SECRET}"
  accessTokenExpiry: 3600      # 1 hour
  refreshTokenExpiry: 604800   # 7 days
storage:
  baseDir: "./uploads"
  maxFileSize: 10485760        # 10MB
```

---

## 15. Feature Behavior Differences

### 15.1 Book Recommendations — Refresh ("换一批")
- **Original:** Not specified how refresh works
- **New:** `GET /api/member/recommendations?seed=<string>`
  - If `seed` is provided: deterministic random selection via FNV-64 hash → `int64` seed
  - If `seed` is omitted: server generates random `int64` seed
  - Frontend can pass a random string on each "换一批" click for deterministic refresh

### 15.2 Freeze Reason
- **Original:** Not specified
- **New:** Admin **must** provide a `reason` string when freezing a member. Stored in `User.FreezeReason` and audit log.

### 15.3 Audit Log — Reason Field
- **Original:** Basic audit log
- **New:** Every audit log entry includes a `Reason` field (human-readable reason for the action, e.g., "Suspected abuse", "User requested")

### 15.4 Audit Log — Data Snapshots
- **Original:** Not specified
- **New:** `BeforeData` and `AfterData` as `map[string]any` stored in `jsonb` columns
  - `BeforeData` is `nil` for CREATE operations
  - `AfterData` is `nil` for DELETE operations

### 15.5 Session Entity (Refresh Tokens)
- **Original:** Not modeled as an entity (tokens stored client-side only)
- **New:** Explicit `Session` entity tracks all refresh tokens server-side
  - Enables multi-device support (max 5)
  - Enables "logout all devices" functionality
  - Enables session expiry cleanup job

---

## 16. Frontend Adaptation Checklist

### Authentication
- [ ] **Remove** `localStorage.getItem('token')` and `Authorization: Bearer` header
- [ ] **Remove** manual token refresh logic on 401
- [ ] **Keep** login response body parsing (`{ userId, role }`) — same shape
- [ ] **Add** cookie handling — browser automatically sends cookies on same-origin requests
- [ ] **Add** redirect to login on `errorCode: "ERR_SESSION_EXPIRED"` or `"ERR_INVALID_REFRESH_TOKEN"`

### Error Handling
- [ ] **Replace** `response.message` user-facing text with `response.errorCode` mapped to i18n keys
- [ ] **Use** `response.data` for contextual info when available

### Recommendations Refresh
- [ ] "换一批" button: generate a random string and pass as `?seed=<random>`
- [ ] Or omit `seed` param to get a random set server-side

### File Uploads
- [ ] No behavioral change needed — same multipart upload flow
- [ ] `CoverURL` field format changes (hash-based path vs. filename)

### Swagger Docs
- [ ] New URL available: `GET /swagger/index.html`

---

## 17. Summary of All Changes

| Category | Change |
|----------|--------|
| Language | JavaScript → Go |
| Framework | Express.js → Gin |
| Database | MySQL → PostgreSQL |
| ORM | Prisma → GORM |
| Architecture | MVC → Clean Architecture + DDD |
| Auth | localStorage + Bearer header → HttpOnly cookies + silent middleware |
| Sessions | Not modeled → Explicit Session entity, max 5 per user, multi-device |
| Refresh Token | Explicit API call → Silent auto-refresh middleware |
| File Storage | Multer (filename-based) → CAS (content-addressable, SHA-256 hash) |
| Error Model | HTTP status + message → Granular domain error codes + i18n |
| Validation | Joi schemas → go-playground/validator (struct tags) |
| DI | Manual → Google Wire (compile-time) |
| API Docs | None → Swagger (swaggo auto-generated) |
| Background Jobs | node-cron → goroutines + time.Ticker |
| Session Cleanup | N/A → Hourly cleanup job |
| AuditLog | Basic → Added `Reason` field, `BeforeData`/`AfterData` as `map[string]any` jsonb |
| User Entity | Basic → Added `FreezeReason` field |
| Enums | Implicit values → 0-based Undefined as security design |
| Domain | Coupled to ORM → Pure Go domain (no ORM tags) |
| Recommendations | Basic → `seed` parameter for deterministic refresh |
| Swagger | N/A → Auto-generated at `/swagger/*` |

---