# Library Management System — Go Backend Design Spec

## 1. Overview

A Go backend for a Library Book Borrowing & Membership Management System, built with Clean Architecture and Domain-Driven Design. The system serves a Vue 3 frontend (member portal + admin portal) via a REST API with auto-generated Swagger docs.

**Key functional modules (unchanged from original):**

- Authentication (register, login, JWT)
- Member profile & membership
- Book catalog search & detail
- Loan management (borrow, renew, return)
- Reservation management
- Notification system
- Fine management
- Reminder policy (configurable)
- Audit logging
- Book recommendation (rule-based, Innovation A)
- AI assistant / FAQ bot (Innovation C)

---

## 2. Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Go 1.26+ | Required by user |
| Database | PostgreSQL | Rich JSON support, works well with GORM |
| ORM | GORM | Most popular Go ORM, auto-migration |
| HTTP Framework | Gin | Standard Go REST framework |
| API Docs | Swagger (swaggo) | Auto-generated from annotations |
| DI | Google Wire | Compile-time dependency injection |
| Validation | go-playground/validator | Most common Go validation lib |
| Auth | golang-jwt/jwt v5 + bcrypt | Standard JWT + password hashing |
| Background Jobs | Built-in goroutines + time.Ticker | Lightweight; user wants to learn this pattern |
| File Storage | CAS (Content-Addressable Storage) | SHA-256 based local disk storage |

---

## 3. Project Structure

```
cmd/server/                      # Application entrypoint + Wire bootstrap
  main.go
  wire.go                        # Wire injectors (build tag: wireinject)

internal/
  domain/                        # PURE GO — entities, interfaces, domain errors
    user/
      entity.go
      errors.go
      repository.go              # UserRepository interface
    book/
      entity.go
      errors.go
      repository.go              # BookRepository interface
    copy/
      entity.go
      repository.go
    loan/
      entity.go
      errors.go
      repository.go
    reservation/
      entity.go
      errors.go
      repository.go
    fine/
      entity.go
      repository.go
    notification/
      entity.go
      errors.go
      repository.go
    auditlog/
      entity.go
      repository.go
    reminderpolicy/
      entity.go
      repository.go
    assistant/
      entity.go
      repository.go
    session/
      entity.go                  # Session entity (refresh token record)
      repository.go              # SessionRepository interface
    storage.go                   # StorageService interface (DOMAIN INTERFACE)
    shared/
      enum.go                    # All enums: UserRole, LoanStatus, CopyStatus, SessionStatus, etc.

  usecase/                       # PURE GO — business rules, depends on domain interfaces
    auth/
      interface.go
      login.go
      register.go
      refresh.go              # Refresh token use case
    book/
      interface.go
      search.go
      create.go
      update.go
    loan/
      interface.go
      borrow.go
      renew.go
      return.go
    reservation/
      interface.go
      create.go
      cancel.go
      my_reservations.go        # Returns user's reservations with queue position
    fine/
      interface.go
      calculate.go
      pay.go
      waive.go
    notification/
      interface.go
      send.go
      read_all.go              # Mark all notifications as read (bulk action)
    recommendation/
      interface.go               # Innovation A
      rule_engine.go
    assistant/
      interface.go               # Innovation C
      chat.go
    reminderpolicy/
      interface.go               # Innovation B
      update.go
    member/
      interface.go
      profile.go
      membership.go              # Freeze/unfreeze with FreezeReason
    circulation/
      interface.go
      checkout.go
      return.go
    auditlog/
      interface.go

  repository/                    # GORM implementations of domain repository interfaces
    postgres/
      user_repo.go
      book_repo.go
      copy_repo.go
      loan_repo.go
      reservation_repo.go
      fine_repo.go
      notification_repo.go
      auditlog_repo.go
      reminderpolicy_repo.go
      assistant_repo.go
      session_repo.go

  delivery/                      # ALL external-facing adapters (Gin is a pluggable detail)
    http/
      server.go                  # HTTP server bootstrap + graceful shutdown
      router.go                  # Route registration (all /api/* groups)
      gin/                       # Gin-specific HTTP handlers
        auth_handler.go
        book_handler.go
        loan_handler.go
        reservation_handler.go
        fine_handler.go
        notification_handler.go
        member_handler.go
        circulation_handler.go
        recommendation_handler.go
        assistant_handler.go
        reminderpolicy_handler.go
        auditlog_handler.go
      middleware/
        error.go                 # DomainError → unified JSON response adapter
        auth.go                  # JWT validation, inject userId + role into Gin ctx
        cors.go
        logger.go
    cron/                        # Background job runners
      job_runner.go
      due_reminder.go            # Runs daily at 08:00
      overdue_reminder.go        # Runs daily at 08:30
      reservation_expiry.go      # Runs every 30 minutes
      session_cleanup.go         # Session expiry cleanup
    grpc/                        # Future: gRPC delivery (stub only)

  infrastructure/                 # Concrete implementations of domain interfaces
    storage/
      local_disk.go              # LocalDiskStorage implements domain.StorageService
    jwt/
      provider.go                # JWT generation + validation
    hasher/
      bcrypt.go                  # Password hashing
    config/
      loader.go                  # YAML/env config loading
      wire.go                    # Wire provider sets

pkg/
  validator/
    validator.go                 # go-playground/validator wrapper, struct tags
  response/
    response.go                  # Unified API response helpers (Success, Error)
  pagination/
    pagination.go                # Page/pageSize parsing + metadata helpers

migrations/
  001_init.sql
  002_add_*.sql
  ...

config.yaml
```

---

## 4. Domain Layer Details

### 4.1 Entities

All 13 entities (12 original + 1 new for session management):

`User`, `Book`, `Copy`, `Loan`, `Reservation`, `Fine`, `Notification`, `AuditLog`, `ReminderPolicy`, `AssistantSession`, `AssistantMessage`, `AssistantKnowledge`, `Session`

**AuditLog — Data Snapshots (Before/After):**
```go
type AuditLog struct {
    ID          uuid.UUID       // UUIDv7 (embedded timestamp, no separate Timestamp field needed)
    UserID      uuid.UUID       // Who performed the action
    Action      string           // e.g. "USER_FREEZE", "BOOK_UPDATE", "LOAN_RETURN"
    EntityType  string           // e.g. "User", "Book", "Loan"
    EntityID    string           // ID of the affected entity
    Reason      string           // Human-readable reason for the action (e.g. "Suspected abuse", "User requested")
    BeforeData  map[string]any  // JSON snapshot BEFORE the change (nil for CREATE)
    AfterData   map[string]any  // JSON snapshot AFTER the change (nil for DELETE)
    IPAddress   string           // Client IP address at time of action
    UserAgent   string           // Client user agent at time of action
}

// GetTimestamp extracts the embedded Unix timestamp from the UUIDv7 ID.
// This provides a consistent operation time without a separate Timestamp column.
// UUIDv7 layout: bytes 0-5 (48 bits) = timestamp_ms, bytes 6-7 = ver + var, bytes 8-15 = random
// The caller passes uuid.UUID (from google/uuid or gofrs/uuid) which must be UUIDv7 variant.
func (a *AuditLog) GetTimestamp() time.Time {
    var unixMs int64
    unixMs |= int64(a.ID[0]) << 40
    unixMs |= int64(a.ID[1]) << 32
    unixMs |= int64(a.ID[2]) << 24
    unixMs |= int64(a.ID[3]) << 16
    unixMs |= int64(a.ID[4]) << 8
    unixMs |= int64(a.ID[5])
    return time.UnixMilli(unixMs)
}
```
`BeforeData` and `AfterData` are `map[string]any` stored as `jsonb` in PostgreSQL. Both may be `nil` — `BeforeData` is `nil` for CREATE operations, `AfterData` is `nil` for DELETE operations. UUIDv7 embeds a Unix timestamp in the ID, so a separate `Timestamp` field is unnecessary.

**User — FreezeReason:**
The `User` entity includes a `FreezeReason string` field. When an admin sets a member's account status to `FROZEN`, the `FreezeReason` must be recorded. The `membership.go` use case accepts this string as a required input parameter.

**Book:**
```go
type Book struct {
    ID          uuid.UUID  // UUID (primary key)
    ISBN        string     // ISBN number (unique)
    Title       string     // Book title
    Author      string     // Author name
    Category    string     // Category name
    Description string     // Book description
    PublishYear int        // Publication year
    CoverURL    string     // Cover image path (CAS storage path)
    IsActive    bool       // false = deactivated (invisible to member search)
    CreatedAt   time.Time  // Record creation time
    UpdatedAt   time.Time  // Last update time
}
```

**Copy:**
```go
type Copy struct {
    ID        uuid.UUID  // UUID (primary key)
    BookID    uuid.UUID  // FK → Book
    Barcode   string     // Unique barcode string (used at circulation)
    Location  string     // Shelf location (e.g. "A-123")
    Status    CopyStatus // AVAILABLE | ON_LOAN | MAINTENANCE | LOST | REMOVED
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

**Loan:**
```go
type Loan struct {
    ID            uuid.UUID   // UUID (primary key)
    UserID        uuid.UUID   // FK → User (member)
    CopyID        uuid.UUID   // FK → Copy
    BookID        uuid.UUID   // FK → Book (denormalized for display)
    Status        LoanStatus // ACTIVE | RETURNED
    BorrowedAt    time.Time  // When the book was borrowed
    DueDate       time.Time  // Calculated due date
    ReturnedAt    *time.Time // nil until returned
    RenewalCount  int        // Number of times renewed (max 2)
    MaxRenewals   int        // Maximum allowed renewals (fixed at 2)
}
```

**Notification:**
```go
type Notification struct {
    ID        uuid.UUID         // UUID (primary key)
    UserID    uuid.UUID         // FK → User (recipient)
    Type      NotificationType  // DUE_REMINDER | OVERDUE_REMINDER | RESERVATION_READY | RESERVATION_EXPIRED | SYSTEM_ANNOUNCEMENT
    Title     string            // Notification title
    Message   string            // Notification body
    IsRead    bool              // Read status
    Metadata  map[string]any    // Additional data (e.g. bookId, reservationId)
    CreatedAt time.Time
}
```

**AssistantSession:**
```go
type AssistantSession struct {
    ID        uuid.UUID // UUID (primary key)
    UserID    uuid.UUID // FK → User (nil for anonymous)
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

**AssistantMessage:**
```go
type AssistantMessage struct {
    ID         uuid.UUID  // UUID (primary key)
    SessionID  uuid.UUID  // FK → AssistantSession
    Role       string     // "user" | "assistant"
    Question   string     // User's question (for role=user)
    Answer     string     // Assistant's answer (for role=assistant)
    Category   string     // Question category (e.g. "LOAN_RULES", "RESERVATION", "FINE")
    CreatedAt  time.Time
}
```

**ReminderPolicy:**
```go
type ReminderPolicy struct {
    ID                uuid.UUID // UUID (primary key)
    DueDaysBefore     []int     // e.g. [3, 1, 0] — days before due date
    OverdueDaysAfter  []int     // e.g. [1, 3, 7] — days after due date
    DailyFineAmount   float64   // Fine per overdue day
    MaxFineAmount     float64   // Cap on total fine
    GraceDays         int       // Days after due date before fines apply
    UpdatedAt         time.Time
}
```

Each entity is a plain Go struct with no ORM tags at the domain layer. GORM tags are applied only in the repository implementation layer.

### 4.2 DomainError

**The backend is fully language-agnostic.** All `Message` values are standard English strings for development/debugging. The frontend owns i18n — it maps `errorCode` (`DomainError.Type`) to localized user-facing strings.

`errorCode` is a **granular, domain-specific** code representing a specific business event or failure reason — not a generic HTTP status label. HTTP status is derived from `errorCode` internally (via a mapping table), not the other way around.

```go
// Defined in domain/shared/errors.go

type DomainError struct {
    Type    string         // Granular, domain-specific error code (i18n key), e.g. "ERR_NO_AVAILABLE_COPIES"
    Message string         // Standard English message for dev/debug; frontend uses Type for i18n
    Context map[string]any // Structured data (e.g. {"bookId": "123", "copyId": "abc"})
    Err     error          // Wrapped underlying error for call-chain tracing
}

func (e *DomainError) Error() string { return e.Message }
func (e *DomainError) Unwrap() error { return e.Err }
func (e *DomainError) ErrorCode() string { return e.Type }

// Constructor
func NewDomainError(code, msg string, ctx map[string]any, err error) *DomainError
```

**Error code naming convention:** `ERR_<NOUN>_<REASON>` in SCREAMING_SNAKE_CASE.

**Granular error codes (domain-specific, exhaustive list per entity):**

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
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

**How HTTP status is determined:** Each error code maps to an HTTP status at the infrastructure/middleware layer via an internal `errorCodeToHTTP` map. The mapping above is the authoritative reference. Handler code only returns domain error codes — HTTP status derivation is an infrastructure concern.

### 4.3 StorageService Interface (Domain Interface)

```go
// Defined in domain/storage.go

type StorageService interface {
    // Save reads from the provided reader, stores the file using CAS,
    // and returns the relative path (e.g. "18/23/ad15...c3.png").
    // If the file already exists at the target path, it skips the write
    // (deduplication) and returns the path directly.
    Save(ctx context.Context, reader io.Reader, originalFilename string) (string, error)
}
```

**Path generation rule (strict):**
- Compute SHA-256 hash of file content
- Split into `{h[0:2]}/{h[2:4]}/{h[4:]}<ext>`
- Example: hash `1823ad...b8c3`, ext `.png` → `18/23/ad1537c537c02199358e2132e3cf25948c1e0b25134699dc79d1f634b8c3.png`

### 4.4 Repository Interfaces (Domain Interfaces)

Each repository interface is defined in its respective `domain/<entity>/repository.go`. Example:

```go
// domain/book/repository.go
type BookSearchQuery struct {
    Keyword      string  // full-text search on title, author, ISBN
    CategoryID   string
    Author      string  // exact or partial author name filter
    AvailableOnly bool   // filter: only books with at least 1 available copy
    Page         int
    PageSize     int
}

type BookRepository interface {
    FindByID(ctx context.Context, id string) (*Book, error)
    Search(ctx context.Context, q *BookSearchQuery) ([]*Book, int64, error)
    Create(ctx context.Context, b *Book) error
    Update(ctx context.Context, b *Book) error
    Deactivate(ctx context.Context, id string) error
}
```

### 4.5 Enumerated Values

All enums are defined in `domain/shared/enum.go`. **The first iota value (0) is always `Undefined`** — this is a deliberate security design: any accidentally uninitialized enum variable defaults to 0, which is always invalid and fails validation rather than silently mapping to a privileged state (e.g., ADMIN). All enum types have a generated `String()` method and a `IsValid()` check.

```go
// domain/shared/enum.go

// --- User ---

// UserRole: undefined=0 (security: invalid default), member=1, librarian=2, admin=3
type UserRole int
const (
    UserRoleUndefined UserRole = iota // 0 — invalid default
    UserRoleMember                     // 1
    UserRoleLibrarian                  // 2
    UserRoleAdmin                      // 3
)

// UserStatus: undefined=0, pending=1, active=2, expired=3, frozen=4
type UserStatus int
const (
    UserStatusUndefined UserStatus = iota // 0 — invalid default
    UserStatusPending                      // 1
    UserStatusActive                       // 2
    UserStatusExpired                      // 3
    UserStatusFrozen                       // 4
)

// --- Loan ---

// LoanStatus: undefined=0, active=1, returned=2
type LoanStatus int
const (
    LoanStatusUndefined LoanStatus = iota // 0 — invalid default
    LoanStatusActive                       // 1
    LoanStatusReturned                     // 2
)

// --- Copy ---

// CopyStatus: undefined=0, available=1, on_loan=2, maintenance=3, lost=4, removed=5
type CopyStatus int
const (
    CopyStatusUndefined CopyStatus = iota // 0 — invalid default
    CopyStatusAvailable                    // 1
    CopyStatusOnLoan                       // 2
    CopyStatusMaintenance                  // 3
    CopyStatusLost                         // 4
    CopyStatusRemoved                      // 5
)

// --- Reservation ---

// ReservationStatus: undefined=0, queued=1, ready_for_pickup=2, completed=3, expired=4, cancelled=5
type ReservationStatus int
const (
    ReservationStatusUndefined ReservationStatus = iota // 0 — invalid default
    ReservationStatusQueued                             // 1
    ReservationStatusReadyForPickup                     // 2
    ReservationStatusCompleted                          // 3
    ReservationStatusExpired                            // 4
    ReservationStatusCancelled                         // 5
)

// --- Fine ---

// FineStatus: undefined=0, unpaid=1, paid=2, waived=3
type FineStatus int
const (
    FineStatusUndefined FineStatus = iota // 0 — invalid default
    FineStatusUnpaid                       // 1
    FineStatusPaid                         // 2
    FineStatusWaived                       // 3
)

// --- Notification ---

// NotificationType: undefined=0, due_reminder=1, overdue_reminder=2, reservation_ready=3, reservation_expired=4, system_announcement=5
type NotificationType int
const (
    NotificationTypeUndefined NotificationType = iota // 0 — invalid default
    NotificationTypeDueReminder                      // 1
    NotificationTypeOverdueReminder                  // 2
    NotificationTypeReservationReady                  // 3
    NotificationTypeReservationExpired                // 4
    NotificationTypeSystemAnnouncement                // 5
)

// --- Session ---

// SessionStatus: undefined=0, active=1, expired=2, revoked=3
type SessionStatus int
const (
    SessionStatusUndefined SessionStatus = iota // 0 — invalid default
    SessionStatusActive                         // 1
    SessionStatusExpired                        // 2
    SessionStatusRevoked                        // 3
)
```

**Validation:** Every entity with an enum field runs `if !role.IsValid()` or equivalent at the repository/use-case boundary. The `Undefined` zero-value guarantees that any missing initialization is caught as a validation error rather than silently granting `ADMIN` or `UserStatusActive`.

---

## 5. Delivery Layer — HTTP

### 5.1 Server Bootstrap

`delivery/http/server.go`:

```go
func NewServer(
    cfg *config.Config,
    router *gin.Engine,
) *http.Server

func (s *Server) Start(ctx context.Context) error
    // 1. Start HTTP server in goroutine
    // 2. Block on <-ctx.Done() (receives SIGINT/SIGTERM)
    // 3. On signal: create shutdownCtx with 8s hard timeout
    // 4. Call s.Shutdown(shutdownCtx) — lets existing connections finish within timeout
    // 5. If hard timeout fires, s.Shutdown returns context.DeadlineExceeded → force close
    // 6. Return nil (clean exit)
```

### 5.2 Router

`delivery/http/router.go` mounts the following route groups:

```
/api/auth/*                      — public
  POST /api/auth/register         — register new member
  POST /api/auth/login            — login (sets HttpOnly cookies, returns { userId, role } in body)
  DELETE /api/auth/logout         — logout current session
  DELETE /api/auth/logout-all     — logout all sessions
  POST /api/auth/refresh          — refresh tokens (mobile clients only)
/api/books/*                     — public search; member detail
  GET /api/books/search           — search books (keyword, category, author, availableOnly, page, pageSize)
  GET /api/books/{bookId}         — book detail + real-time copy stats
/api/member/*                    — authenticated member
  GET  /api/member/profile        — get / update profile
  PUT  /api/member/profile
  GET  /api/member/membership     — membership status
  GET  /api/member/loans          — my loans (active/returned)
  POST /api/member/loans          — borrow a book (body: { bookId, copyId? }); if copyId omitted, system auto-assigns first available copy
  POST /api/member/loans/{loanId}/renew — renew loan
  GET  /api/member/reservations   — my reservations with queue position
  POST /api/member/reservations   — create reservation
  DELETE /api/member/reservations/{reservationId}
  GET  /api/member/notifications  — notification list
  PUT  /api/notifications/{notificationId}/read    — mark single notification as read
  PUT  /api/member/notifications/read-all          — mark all notifications as read
  GET  /api/member/recommendations — personalized recommendations
/api/admin/*                     — authenticated librarian / admin
  GET  /api/admin/members         — member list (filter by status, keyword)
  POST /api/admin/members/{memberId}/approve
  POST /api/admin/members/{memberId}/freeze       — requires reason
  POST /api/admin/members/{memberId}/unfreeze
  POST /api/admin/members/{memberId}/renew
  GET  /api/admin/books           — book management list
  POST /api/admin/book            — create book
  PUT  /api/admin/book/{bookId}
  PUT  /api/admin/book/{bookId}/deactivate
  GET  /api/admin/copies          — copy management list
  POST /api/admin/copy            — create copy
  PUT  /api/admin/copy/{copyId}/status
  GET  /api/admin/fines           — fine list
  POST /api/admin/fines/{fineId}/pay
  POST /api/admin/fines/{fineId}/waive
  GET  /api/admin/reminder-policy
  PUT  /api/admin/reminder-policy
  GET  /api/admin/audit-logs
  GET  /api/admin/audit-logs/{logId}
/api/circulation/*               — authenticated librarian / admin
  POST /api/circulation/checkout
  POST /api/circulation/return
  GET  /api/circulation/lookup?barcode={barcode}  — lookup loan by barcode (pre-return validation)
/api/assistant/*                 — public (no auth required)
  GET  /api/assistant/quick-questions
  POST /api/assistant/sessions
  POST /api/assistant/chat
  GET  /api/assistant/sessions/{sessionId}/messages
  DELETE /api/assistant/sessions/{sessionId}
```

### 5.3 Middleware Stack

Applied in order:

1. **CORS** — allow frontend origin
2. **Logger** — request log (method, path, status, latency)
3. **Auth** — JWT validation; unauthenticated routes skip this via a public group
4. **Error** — catches all panics + `*DomainError`, returns unified JSON

### 5.4 Error Middleware (DomainError Adapter)

```go
// delivery/http/middleware/error.go

func ErrorMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if r := recover(); r != nil {
                domErr := domain.ErrInternal(fmt.Errorf("%v", r))
                writeError(c, domErr)
                return
            }
        }()
        c.Next()
        // After handler returns, check c.Errors
        // Use errors.As to unwrap *DomainError
        // Write: {"code": N, "errorCode": e.Type, "message": e.Message, "data": e.Context}
    }
}

func writeError(c *gin.Context, e *domain.DomainError) {
    httpStatus := errorCodeToHTTP[e.Type]  // lookup table at infrastructure layer
    c.JSON(httpStatus, response.Error(e.Type, e.Message, e.Context))
}
```

**`errorCodeToHTTP` lookup table (in `infrastructure/` or `pkg/response/`):**

| errorCode prefix/pattern | HTTP Status |
|--------------------------|-------------|
| `ERR_VALIDATION_FAILED` | 400 |
| `ERR_UNAUTHORIZED` | 401 |
| `ERR_FORBIDDEN` | 403 |
| `ERR_*_NOT_FOUND` | 404 |
| `ERR_USER_EMAIL_EXISTS`, `ERR_RESERVATION_ALREADY_EXISTS` | 409 |
| `ERR_*_EXPIRED`, `ERR_*_FROZEN`, `ERR_*_LIMIT_REACHED`, `ERR_NO_AVAILABLE_COPIES`, `ERR_*_CANNOT_*` | 422 |
| `ERR_INTERNAL` | 500 |

---

## 6. Use Case Layer

Each use case is a struct holding only domain-level dependencies (repository interfaces, domain services). Public methods form the use case interface.

**Example — circulation/checkout:**

```go
// usecase/circulation/interface.go
type CirculationUseCase interface {
    Checkout(ctx context.Context, memberId string, copyBarcode string) (*Loan, error)
    Return(ctx context.Context, copyBarcode string, condition string) (*ReturnResult, error)
    LookupByBarcode(ctx context.Context, barcode string) (*Loan, error)
}

// delivery/circulation/checkout.go
type checkoutUseCase struct {
    loanRepo       LoanRepository
    copyRepo       CopyRepository
    userRepo       UserRepository
    reservationSvc ReservationService  // domain interface
    notifSvc       NotificationService
    fineSvc        FineService
    auditSvc       AuditLogService
}

func (uc *checkoutUseCase) Checkout(ctx context.Context, memberId, copyBarcode string) (*Loan, error) {
    // business rules: member must be ACTIVE, copy must be AVAILABLE
    // creates loan, updates copy status, handles reservation auto-complete
    // all errors are domain errors
}
```

---

## 7. Infrastructure — Storage (CAS Implementation)

`infrastructure/storage/local_disk.go` implements `domain.StorageService`:

```go
type LocalDiskStorage struct {
    baseDir string
    maxSize int64  // max file size in bytes
}

// Save reads the file in buffered chunks (no full-file-in-memory),
// computes SHA-256 hash via crypto/sha256,
// constructs path using h[0:2]/h[2:4]/h[4:]<ext>,
// creates directories via os.MkdirAll,
// deduplicates by checking os.Stat before write,
// returns the relative path.
func (s *LocalDiskStorage) Save(ctx context.Context, reader io.Reader, originalFilename string) (string, error)
```

Key requirements:
- **No full file in memory** — use a 64KB buffer loop
- **Deduplication** — `os.Stat` before write; skip write if file exists
- **Thread-safety** — not required for local disk single-instance; note in comments for future S3 upgrade

---

## 8. Infrastructure — DI with Wire

### 8.1 Wire Provider Sets

```go
// infrastructure/config/wire.go
//go:build wireinject

func InitializeServer(cfgPath string) (*deliveryhttp.Server, error) {
    wire.Build(
        configSet,      // loads config.yaml
        domainSet,      // entity constructors, enum values
        repositorySet,  // GORM DB + repository implementations
        infrastructureSet, // storage, JWT, hasher
        usecaseSet,     // use case implementations
        deliveryHTTPServerSet, // router, server
    )
    return nil, nil
}
```

### 8.2 Config Loading

`config.yaml` loaded via a custom YAML loader into a `Config` struct. All sensitive values support env var overrides (e.g., `database.password` from `DB_PASSWORD` env).

---

## 9. Background Jobs

### 9.1 JobRunner

```go
type JobRunner struct {
    dueReminder       *DueReminderJob
    overdueReminder   *OverdueReminderJob
    reservationExpiry *ReservationExpiryJob
}

func (jr *JobRunner) Start(ctx context.Context)
func (jr *JobRunner) Stop()
```

All three jobs are started as goroutines in `cmd/server/main.go` after Wire injection. On `os.Signal` (SIGINT/SIGTERM), `Stop()` is called which cancels each job's context.

### 9.2 Job Schedule

| Job | Schedule | Implementation |
|-----|----------|----------------|
| Due Reminder | Daily 08:00 | `time.NewTimer` reset every 24h |
| Overdue Reminder | Daily 08:30 | `time.NewTimer` reset every 24h |
| Reservation Expiry | Every 30 min | `time.NewTicker` |
| Session Cleanup | Every 1 hour | `time.NewTicker`, also runs on startup |

---

## 10. API Response Format

**Language-agnostic:** `message` is always standard English for development/debug. The frontend uses `errorCode` as the i18n lookup key.

### Success

```json
{
  "code": 200,
  "message": "Success",
  "data": { ... }
}
```

### Error

```json
{
  "code": 422,
  "errorCode": "ERR_NO_AVAILABLE_COPIES",
  "message": "No available copies for this book",
  "data": {
    "bookId": "123"
  }
}
```

| Field | Description |
|-------|-------------|
| `code` | HTTP status code (derived from `errorCode`) |
| `errorCode` | Granular domain error code (i18n key for frontend) |
| `message` | Standard English debug message |
| `data` | Structured context from `DomainError.Context` |

### Pagination

All list endpoints accept `?page=1&pageSize=20` and return:

```json
{
  "code": 200,
  "message": "Success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "pageSize": 20,
    "totalPages": 5
  }
}
```

---

## 11. Authentication

**Design principle: All-Cookie + Silent Auto-Refresh**

Both tokens live in HttpOnly cookies. The frontend never handles token storage or refresh logic — the auth middleware handles everything server-side.

> **Why not follow the original Node.js design?** The original design returned `{ token, role, userId }` in the response body and stored the JWT in `localStorage` / sent via `Authorization: Bearer` header. This approach has critical security and UX flaws:
>
> 1. **XSS theft** — `localStorage` is accessible to JavaScript; any XSS vulnerability allows attackers to steal tokens
> 2. **Manual refresh** — frontend must detect 401 errors and explicitly call a refresh endpoint, creating race conditions and UX flicker
> 3. **Token proliferation** — every API request needs the `Authorization` header; a missed header is a silent auth failure
>
> The Go backend eliminates all three by storing tokens exclusively in `HttpOnly` cookies (invisible to JavaScript), handling refresh silently in middleware, and returning only `{ userId, role }` in the body for frontend session initialization.

### 11.1 Cookie Specification

| Cookie Name | Type | HttpOnly | Secure | SameSite | Max-Age | Contents |
|-------------|------|----------|--------|----------|---------|----------|
| `access_token` | Access JWT | Yes | Yes* | Lax | 3600 (1h) | JWT string |
| `refresh_token` | Opaque token | Yes | Yes* | Lax | 604800 (7d) | Raw refresh token string |

*Secure flag: enabled in production; configurable off for local development.

### 11.2 JWT (Access Token)

- Signed with HS256
- Lifetime: **1 hour** (configurable)
- **Not passed in Authorization header** — validated from `access_token` cookie only
- Payload: `{ "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN", "sessionId": "uuid" }`

### 11.3 Session Entity (Refresh Token Record)

```go
// domain/session/entity.go

type Session struct {
    ID         uuid.UUID  // UUIDv7 (embedded timestamp, no separate CreatedAt needed)
    UserID     uuid.UUID  // FK → User
    TokenHash  string     // SHA-256 of the raw refresh token — raw token is never persisted
    UserAgent  string     // Device/browser identifier captured at login
    IPAddress  string     // Client IP captured at login
    ExpiresAt  time.Time  // Refresh token expiry (7 days from creation)
}
```

**Multi-device:** A user may have multiple active sessions. Max **5 active sessions per user** — oldest is evicted on 6th login.

### 11.4 SessionRepository Interface

```go
// domain/session/repository.go

type SessionRepository interface {
    Create(ctx context.Context, s *Session) error
    FindByID(ctx context.Context, id uuid.UUID) (*Session, error)
    FindByTokenHash(ctx context.Context, hash string) (*Session, error)
    DeleteByID(ctx context.Context, id uuid.UUID) error
    DeleteByUserID(ctx context.Context, userID uuid.UUID) error     // logout all devices
    CountByUserID(ctx context.Context, userID uuid.UUID) (int, error)
    DeleteOldestByUserID(ctx context.Context, userID uuid.UUID) error // evict on session limit
    DeleteExpired(ctx context.Context) error                           // cleanup job
}
```

### 11.5 Auth Middleware — Silent Auto-Refresh Flow

```go
// delivery/http/middleware/auth.go

func AuthMiddleware(jwtProvider *JWTProvider, sessionRepo SessionRepository) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Step 1: Try to validate access_token cookie (stateless, no DB)
        accessToken, err := c.Cookie("access_token")
        if err != nil {
            // No cookie → attempt silent refresh
            trySilentRefresh(c, sessionRepo, jwtProvider)
            // After refresh attempt, re-read the cookie
            accessToken, _ = c.Cookie("access_token")
            if accessToken == "" {
                // Still no token → unauthenticated
                domErr := domain.ErrUnauthorized("No valid session", nil)
                writeError(c, domErr)
                c.Abort()
                return
            }
        }

        // Step 2: Validate JWT (stateless)
        claims, err := jwtProvider.Validate(accessToken)
        if err != nil {
            // Expired or invalid → attempt silent refresh
            trySilentRefresh(c, sessionRepo, jwtProvider)
            accessToken, _ = c.Cookie("access_token")
            claims, err = jwtProvider.Validate(accessToken)
            if err != nil {
                domErr := domain.ErrUnauthorized("Token invalid or expired", nil)
                writeError(c, domErr)
                c.Abort()
                return
            }
        }

        // Step 3: Inject claims into Gin context
        c.Set("userId", claims.UserID)
        c.Set("role", claims.Role)
        c.Set("sessionId", claims.SessionID)
        c.Next()
    }
}

func trySilentRefresh(c *gin.Context, sessionRepo SessionRepository, jwtProvider *JWTProvider) {
    // ⚠️ CRITICAL: This function MUST only read cookies (c.Cookie).
    // It MUST NOT access c.Request.Body. The body is owned by the downstream handler.
    // Read refresh_token cookie
    rawToken, err := c.Cookie("refresh_token")
    if rawToken == "" {
        return // No refresh token → cannot refresh
    }

    // Lookup session by token hash
    hash := sha256String(rawToken)
    session, err := sessionRepo.FindByTokenHash(c.Request.Context(), hash)
    if err != nil || session == nil {
        return // Invalid refresh token
    }

    // Check expiry
    if time.Now().After(session.ExpiresAt) {
        sessionRepo.DeleteByID(c.Request.Context(), session.ID)
        return // Expired session, cannot refresh
    }

    // Generate new JWT
    newToken, exp, err := jwtProvider.Generate(session.UserID, session.ID)
    if err != nil {
        return // Generation failed → fail open (let request proceed without auth)
    }

    // Set new access_token cookie
    c.SetCookie("access_token", newToken, int(exp-time.Now().Unix()), "/", "", false, true)
}
```

**Key properties:**
- **Stateless path:** 99% of requests hit Step 1-3 only — zero DB calls
- **DB lookup only on expired JWT + valid refresh cookie** — rare, only when access token expires
- **Silent:** frontend never sees a 401 from token expiry — the server self-heals the session
- **Fail open on refresh:** if token generation fails during silent refresh, the expired token is not revoked — this is intentional to avoid breaking the user experience; the next request retries

### 11.6 Login Flow

```
1. Validate email + password
2. Check account status → ErrUserAccountPending / ErrUserAccountFrozen / ErrUserAccountExpired
3. Generate raw refresh token (crypto-safe random string)
4. Create Session record (TokenHash = SHA-256(raw), ExpiresAt = now + 7d, UserAgent + IP from request)
5. If session count ≥ 5 → DeleteOldestByUserID
6. Generate JWT (1h expiry) with { userId, role, sessionId }
7. Set access_token cookie + refresh_token cookie
8. Return: { "code": 200, "message": "Success", "data": { "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN" } }
```

Tokens are in HttpOnly cookies (set via `SetCookie`). The body additionally returns `userId` and `role` for frontend session initialization.

### 11.7 Refresh Endpoint - **NOT REQUIRED for browser clients**

`POST /api/auth/refresh`
**Request:** `refreshToken: "<raw-token>"` (JSON body)
**Response:** Sets `access_token` + `refresh_token` cookies, returns `{ "code": 200, "message": "Success", "data": null }`

**Note:** Browser clients rely on the silent refresh middleware and do not call this endpoint directly.

### 11.8 Logout

**`DELETE /api/auth/logout`** — deletes the current session by `sessionId` (from JWT claims), clears both cookies.

**`DELETE /api/auth/logout-all`** — deletes all sessions for the user, clears both cookies.

Both set cookie `Max-Age: 0` to instruct the browser to delete.

### 11.9 Session Cleanup Job

Runs every **1 hour** via `time.Ticker`. Deletes all `Session` records where `ExpiresAt < now`. Also runs on server startup to catch any missed entries.

---

## 12. Innovation Modules

### Innovation A — Book Recommendation

Rule-based engine in `usecase/recommendation/rule_engine.go`:
- Same-category popular books (based on user's loan history)
- Same-author books (based on user's loan history)
- Recently added books (last 30 days)
- Cold start: top popular + newest books
- Returns exactly 5 unique recommendations with `reason` and `reasonType`
- Excludes already-borrowed books

**Refresh ("换一批") support:**
- `GET /api/recommendations?seed=<string>` — accepts an optional `seed` query parameter
- If `seed` is provided, it is mapped to an `int64` value via `fnv.New64a()` (hash of the string cast to `int64`) and used as the random seed for deterministic selection
- If `seed` is omitted, `rand.Int64()` (`math/rand/v2`, not crypto) is used to generate a random seed server-side
- The `int64` seed drives offset-based random selection in the rule engine

### Innovation B — Reminder Policy

`domain/reminderpolicy/entity.go` holds:
- `DueDaysBefore []int` — e.g. [3, 1, 0]
- `OverdueDaysAfter []int` — e.g. [1, 3, 7]
- `DailyFineAmount decimal`
- `MaxFineAmount decimal`
- `GraceDays int`

Stored in DB. Jobs read current policy on each run.

### Innovation C — AI Assistant (FAQ Bot)

`usecase/assistant/chat.go`:
- Classifies question into category (loan rules, reservation, fine, system usage)
- Searches `AssistantKnowledge` table for matching Q&A
- Falls back to a generic "please contact librarian" response
- Session management via `AssistantSession` table

**Quick Questions endpoint:**
- `GET /api/assistant/quick-questions` — returns a list of predefined question tags to help users start a conversation (e.g. "How to register?", "Fine rules", "How to renew a book?")
- Handled by `AssistantHandler.GetQuickQuestions`

---

## 13. Pending Implementation Details (Frontend Contract)

These are agreed upon in principle but exact request/response DTOs will be defined during implementation:

- **Swagger annotations** on all handler functions for auto-generated docs
- **Request DTOs** in `pkg/dto/` with validation struct tags
- **File upload** handled via `multipart/form-data` on profile (avatar) and book (cover) endpoints
- **Notification types**: DUE_REMINDER, OVERDUE_REMINDER, RESERVATION_READY, RESERVATION_EXPIRED, SYSTEM_ANNOUNCEMENT
- **State machines** (user, loan, reservation, copy, fine) preserved from original design

---

## 14. Summary of Architectural Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Language | Go | Required by user |
| DB | PostgreSQL | User preferred over MySQL |
| ORM | GORM | Most popular Go ORM |
| API | REST + Gin | User preferred |
| API Docs | Swagger (swaggo) | Auto-generated, frontend-friendly |
| DI | Google Wire | User requested to learn |
| Validation | go-playground/validator | User preferred |
| Auth | golang-jwt/jwt v5 + bcrypt + HttpOnly cookie sessions | All-cookie, silent auto-refresh middleware, session per device, max 5 sessions |
| Architecture | Clean Architecture + DDD | User preferred |
| Delivery decoupling | `delivery/` with `http/`, `cron/`, `grpc/` | Gin is a pluggable external detail |
| Storage interface | `domain/storage.go` | Dependency Inversion Principle |
| Storage impl | CAS on local disk | User-specified CAS pattern |
| Error model | Custom `DomainError` + Gin middleware | Granular domain-specific error codes (e.g. ERR_NO_AVAILABLE_COPIES), language-agnostic; frontend maps errorCode → i18n |
| Background jobs | Built-in goroutines + time.Ticker | User requested to learn |
