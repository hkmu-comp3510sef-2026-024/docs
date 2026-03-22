# Library Management System — TypeScript/NestJS Backend Design Spec

## 1. Overview

A TypeScript/NestJS backend for a Library Book Borrowing & Membership Management System, built with Clean Architecture and Domain-Driven Design. The system serves a Vue 3 frontend (member portal + admin portal) via a REST API.

**Key functional modules (unchanged from original):**

- Authentication (register, login, JWT with HttpOnly cookies)
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
| Language | TypeScript 5.x | Type safety, same as frontend team |
| Framework | NestJS 10.x | Built-in DI, modules, decorators, enforces Clean Architecture |
| Database | PostgreSQL | Reliable relational DB |
| ORM | TypeORM | NestJS ecosystem standard, decorator-based entities; `uuid` v9+ for UUIDv7 generation (via `@BeforeInsert()` hook on AuditLog + Session) |
| API Docs | Swagger (@nestjs/swagger) | Auto-generated from decorators |
| Validation | class-validator + class-transformer | NestJS standard validation |
| Auth | @nestjs/jwt + bcrypt | Standard JWT + password hashing |
| Background Jobs | @nestjs/schedule | Cron-based jobs, built into NestJS |
| File Storage | CAS (Content-Addressable Storage) | SHA-256 based local disk storage |
| Config | @nestjs/config + YAML | YAML config with env var overrides (e.g. `DB_PASSWORD` env var overrides `config.yaml`'s `database.password`) |

---

## 3. Project Structure

```
src/
  main.ts                         # Application bootstrap + graceful shutdown
```

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks();     // SIGINT/SIGTERM graceful shutdown

  // Global validation: strip unknown fields, transform types, enforce class-validator decorators
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));

  // CORS for Vue 3 frontend (different origin). Allows credentials (HttpOnly cookies) to cross origin.
  app.enableCors({ origin: true, credentials: true }); // TODO: lock down origin via env in production

  const server = await app.listen(3000);

  // Hard timeout: force exit after 8 seconds even if connections are still alive
  const HARD_TIMEOUT_MS = 8_000;
  const forceExit = setTimeout(() => {
    console.warn('[shutdown] Hard timeout reached, forcing process exit');
    process.exit(1);
  }, HARD_TIMEOUT_MS);
  forceExit.unref(); // don't block event loop from exiting normally

  // SIGTERM / SIGINT → start graceful shutdown
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT',  () => shutdown('SIGINT'));

  async function shutdown(signal: string) {
    console.log(`[shutdown] Received ${signal}, closing connections...`);
    clearTimeout(forceExit);
    await app.close();           // waits for in-flight requests to finish within NestJS timeout
    process.exit(0);
  }
}
bootstrap();
```
```
  app.module.ts                   # Root module

  config/
    configuration.ts             # YAML config loader; uses @nestjs/config customEnv() to wire env var overrides
    config.interface.ts           # Config type definitions

  domain/                         # Entities with TypeORM decorators, interfaces, domain errors
    user/
      entities/
        user.entity.ts
      errors/
        user.errors.ts
      repositories/
        user.repository.interface.ts
    book/
      entities/
        book.entity.ts
      errors/
        book.errors.ts
      repositories/
        book.repository.interface.ts
    copy/
      entities/
        copy.entity.ts
      repositories/
        copy.repository.interface.ts
    loan/
      entities/
        loan.entity.ts
      errors/
        loan.errors.ts
      repositories/
        loan.repository.interface.ts
    reservation/
      entities/
        reservation.entity.ts
      errors/
        reservation.errors.ts
      repositories/
        reservation.repository.interface.ts
    fine/
      entities/
        fine.entity.ts
      repositories/
        fine.repository.interface.ts
    notification/
      entities/
        notification.entity.ts
      errors/
        notification.errors.ts
      repositories/
        notification.repository.interface.ts
    auditlog/
      entities/
        audit-log.entity.ts
      repositories/
        audit-log.repository.interface.ts
    reminderpolicy/
      entities/
        reminder-policy.entity.ts
      repositories/
        reminder-policy.repository.interface.ts
    assistant/
      entities/
        assistant-session.entity.ts
        assistant-message.entity.ts
        assistant-knowledge.entity.ts
      repositories/
        assistant.repository.interface.ts
    session/
      entities/
        session.entity.ts
      repositories/
        session.repository.interface.ts
    storage/
      storage.service.interface.ts   # Domain interface
    shared/
      enums.ts                       # All enums: UserRole, LoanStatus, KnowledgeCategory, etc.
      domain-error.ts                # DomainError class
      uuid.ts                        # UUIDv7 generator (wraps uuid v9+ `v7()`)

  usecase/                          # PURE TS — business rules
    auth/
      auth.usecase.interface.ts
      login.usecase.ts
      register.usecase.ts
      refresh.usecase.ts
      logout.usecase.ts
    book/
      book.usecase.interface.ts
      search.usecase.ts
      create-book.usecase.ts
      update-book.usecase.ts
    loan/
      loan.usecase.interface.ts
      borrow.usecase.ts   # Auto-completes READY_FOR_PICKUP reservation for same book within the same transaction
      renew.usecase.ts    # Checks: loan must be ACTIVE, renewalCount < maxRenewals, AND no QUEUED reservation exists for the same book
      return.usecase.ts
    reservation/
      reservation.usecase.interface.ts
      create-reservation.usecase.ts
      cancel-reservation.usecase.ts   — on READY_FOR_PICKUP cancellation: promotes next QUEUED reservation to READY_FOR_PICKUP, sets new deadline, sends RESERVATION_READY notification
      my-reservations.usecase.ts
    fine/
      fine.usecase.interface.ts
      calculate-fine.usecase.ts
      pay-fine.usecase.ts
      waive-fine.usecase.ts
    notification/
      notification.usecase.interface.ts
      send-notification.usecase.ts
      read-all-notifications.usecase.ts
    recommendation/
      recommendation.usecase.interface.ts   # Innovation A
      rule-engine.usecase.ts
    assistant/
      assistant.usecase.interface.ts        # Innovation C
      chat.usecase.ts
    reminderpolicy/
      reminder-policy.usecase.interface.ts # Innovation B
      get-policy.usecase.ts
      update-policy.usecase.ts
    member/
      member.usecase.interface.ts
      get-profile.usecase.ts
      update-profile.usecase.ts
      get-membership.usecase.ts
    circulation/
      circulation.usecase.interface.ts
      checkout.usecase.ts
      return.usecase.ts
      lookup.usecase.ts
    auditlog/
      audit-log.usecase.interface.ts

  infrastructure/                    # Concrete implementations of domain interfaces
    orm/
      typeorm.module.ts
      repositories/
        user.repository.ts
        book.repository.ts
        copy.repository.ts
        loan.repository.ts
        reservation.repository.ts
        fine.repository.ts
        notification.repository.ts
        audit-log.repository.ts
        reminder-policy.repository.ts
        assistant.repository.ts
        session.repository.ts
    storage/
      local-disk.storage.ts           # CAS implementation
    jwt/
      jwt.provider.ts
    hasher/
      bcrypt.hasher.ts

  delivery/                           # External-facing adapters (NestJS Controllers)
    controllers/
      auth.controller.ts
      book.controller.ts
      loan.controller.ts
      reservation.controller.ts
      fine.controller.ts
      notification.controller.ts
      member.controller.ts
      circulation.controller.ts
      recommendation.controller.ts
      assistant.controller.ts
      assistant-knowledge.controller.ts  # Admin CRUD for knowledge base entries
      reminder-policy.controller.ts
      audit-log.controller.ts
    guards/
      jwt-auth.guard.ts               # JWT validation
      roles.guard.ts                  # Role-based access control
    filters/
      http-exception.filter.ts        # DomainError → unified JSON response
    interceptors/
      logging.interceptor.ts
    decorators/
      current-user.decorator.ts
      public.decorator.ts

  cron/
    jobs.module.ts
    due-reminder.job.ts               # Runs daily at 08:00
    overdue-reminder.job.ts           # Runs daily at 08:30
    reservation-expiry.job.ts        # Runs every 30 minutes
    session-cleanup.job.ts           # Runs every 1 hour

  common/
    dto/
      pagination.dto.ts
      response.dto.ts
      assistant/
        create-knowledge.dto.ts       # Admin CRUD: create knowledge entry
        update-knowledge.dto.ts       # Admin CRUD: update knowledge entry
        knowledge-query.dto.ts       # Admin CRUD: paginated list query
    shutdown/
      shutdown-flag.service.ts       # Singleton: OnModuleDestroy sets flag for cron job safe interruption

migrations/
  001_init.sql
  002_add_*.sql

config.yaml
test/
  ...
```

**Key NestJS vs Go differences:**
- `delivery/controllers/` replaces Go's `delivery/http/gin/` handlers
- Route registration via `@Controller('api/books')` decorator — no centralized router file
- Guards replace middleware for auth/roles
- Exception Filters replace the Go error middleware
- Modules (`@Module()`) replace Wire dependency injection

---

## 4. Domain Layer

### 4.1 Entities

All 13 entities (same as Go design):

`User`, `Book`, `Copy`, `Loan`, `Reservation`, `Fine`, `Notification`, `AuditLog`, `ReminderPolicy`, `AssistantSession`, `AssistantMessage`, `AssistantKnowledge`, `Session`

**AuditLog — Data Snapshots (Before/After):**

> **UUIDv7 generation:** `AuditLog` and `Session` are high-volume 流水 tables requiring time-ordered IDs. They use `generateUUIDv7()` from `domain/shared/uuid.ts` (wraps `uuid` v9+ `v7()`) in a `@BeforeInsert()` hook, replacing TypeORM's default `@PrimaryGeneratedColumn('uuid')` (UUIDv4). All other entities keep `@PrimaryGeneratedColumn('uuid')` — no ordering requirement.
>
> **Embedded timestamp:** UUIDv7 embeds a Unix timestamp (milliseconds) in bytes 0–5 of the UUID. A companion method `getUUIDv7Timestamp(id: string): Date` in `domain/shared/uuid.ts` decodes this timestamp, making it unnecessary to store a separate `createdAt` column for AuditLog — the ID itself is the timestamp carrier. The `createdAt` column on AuditLog is retained for convenience (DB-level queries) but is functionally redundant with the embedded timestamp.

```typescript
// domain/auditlog/entities/audit-log.entity.ts
@Entity('audit_logs')
export class AuditLog {
  @Column({ type: 'uuid', primary: true })
  id: string;   // UUIDv7 — timestamp embedded, sortable by time

  @BeforeInsert()
  generateId() {
    // uuidv7 embeds current timestamp — replaces TypeORM auto-generated UUIDv4
    this.id = generateUUIDv7();
  }

  @Column({ name: 'user_id', type: 'uuid' })
  userId: string;

  @Column({ name: 'action', type: 'varchar' })
  action: string;                      // e.g. "USER_FREEZE", "BOOK_UPDATE"

  @Column({ name: 'entity_type', type: 'varchar' })
  entityType: string;                  // e.g. "User", "Book"

  @Column({ name: 'entity_id', type: 'varchar' })
  entityId: string;

  @Column({ name: 'reason', type: 'varchar', nullable: true })
  reason: string;                      // Human-readable reason

  @Column({ name: 'before_data', type: 'jsonb', nullable: true })
  beforeData: Record<string, any>;      // JSON snapshot BEFORE (nil for CREATE)

  @Column({ name: 'after_data', type: 'jsonb', nullable: true })
  afterData: Record<string, any>;       // JSON snapshot AFTER (nil for DELETE)

  @Column({ name: 'ip_address', type: 'varchar', nullable: true })
  ipAddress: string;

  @Column({ name: 'user_agent', type: 'varchar', nullable: true })
  userAgent: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

**User — Full Entity:**
```typescript
// domain/user/entities/user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', unique: true })
  email: string;

  @Column({ name: 'password_hash', type: 'varchar' })
  passwordHash: string;

  @Column({ type: 'varchar' })
  name: string;

  @Column({ name: 'student_id', type: 'varchar', nullable: true })
  studentId: string;          // University/student ID (used by frontend UI)

  @Column({ type: 'varchar', nullable: true })
  phone: string;              // Contact phone number

  @Column({ type: 'varchar', nullable: true })
  address: string;            // Mailing address

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.Undefined,
  })
  role: UserRole;

  @Column({
    type: 'enum',
    enum: UserStatus,
    default: UserStatus.Pending,
  })
  status: UserStatus;

  @Column({ name: 'freeze_reason', type: 'varchar', nullable: true })
  freezeReason: string;        // Required when admin sets status to FROZEN

  @Column({ name: 'membership_expires_at', type: 'timestamp', nullable: true })
  membershipExpiresAt: Date;

  @Column({ name: 'avatar_url', type: 'varchar', nullable: true })
  avatarUrl: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

**Book:**
```typescript
// domain/book/entities/book.entity.ts
@Entity('books')
export class Book {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', unique: true })
  isbn: string;

  @Column({ type: 'varchar' })
  title: string;

  @Column({ type: 'varchar' })
  author: string;

  @Column({ type: 'varchar' })
  category: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ name: 'publish_year', type: 'int' })
  publishYear: number;

  @Column({ name: 'cover_url', type: 'varchar', nullable: true })
  coverUrl: string;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive: boolean;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

**Copy:**
```typescript
// domain/copy/entities/copy.entity.ts
@Entity('copies')
export class Copy {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'book_id', type: 'uuid' })
  bookId: string;

  @ManyToOne(() => Book)
  @JoinColumn({ name: 'book_id' })
  book: Book;

  @Column({ type: 'varchar', unique: true })
  barcode: string;

  @Column({ type: 'varchar' })
  location: string;

  @Column({
    type: 'enum',
    enum: CopyStatus,
    default: CopyStatus.Available,
  })
  status: CopyStatus;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

**Loan:**
```typescript
// domain/loan/entities/loan.entity.ts
@Entity('loans')
export class Loan {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'user_id', type: 'uuid' })
  userId: string;

  @ManyToOne(() => User)
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ name: 'copy_id', type: 'uuid' })
  copyId: string;

  @ManyToOne(() => Copy)
  @JoinColumn({ name: 'copy_id' })
  copy: Copy;

  @Column({ name: 'book_id', type: 'uuid' })
  bookId: string;

  @ManyToOne(() => Book)
  @JoinColumn({ name: 'book_id' })
  book: Book;

  @Column({
    type: 'enum',
    enum: LoanStatus,
    default: LoanStatus.Active,
  })
  status: LoanStatus;

  @Column({ name: 'borrowed_at', type: 'timestamp' })
  borrowedAt: Date;

  @Column({ name: 'due_date', type: 'timestamp' })
  dueDate: Date;

  @Column({ name: 'returned_at', type: 'timestamp', nullable: true })
  returnedAt: Date | null;

  @Column({ name: 'renewal_count', type: 'int', default: 0 })
  renewalCount: number;

  @Column({ name: 'max_renewals', type: 'int', default: 2 })
  maxRenewals: number;
}
```

**Notification:**
```typescript
// domain/notification/entities/notification.entity.ts
@Entity('notifications')
export class Notification {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'user_id', type: 'uuid' })
  userId: string;

  @ManyToOne(() => User)
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({
    type: 'enum',
    enum: NotificationType,
    default: NotificationType.Undefined,
  })
  type: NotificationType;

  @Column({ type: 'varchar' })
  title: string;                    // Notification title

  @Column({ type: 'text' })
  message: string;                  // Notification body

  @Column({ name: 'is_read', type: 'boolean', default: false })
  isRead: boolean;

  @Column({ type: 'jsonb', nullable: true })
  metadata: Record<string, any>;    // e.g. { bookId, reservationId }

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

**AssistantSession:**
```typescript
// domain/assistant/entities/assistant-session.entity.ts
@Entity('assistant_sessions')
export class AssistantSession {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'user_id', type: 'uuid', nullable: true })
  userId: string | null;     // nil for anonymous sessions

  @ManyToOne(() => User, { nullable: true })
  @JoinColumn({ name: 'user_id' })
  user: User | null;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

**AssistantMessage:**
```typescript
// domain/assistant/entities/assistant-message.entity.ts
@Entity('assistant_messages')
export class AssistantMessage {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'session_id', type: 'uuid' })
  sessionId: string;

  @ManyToOne(() => AssistantSession)
  @JoinColumn({ name: 'session_id' })
  session: AssistantSession;

  @Column({ type: 'varchar' })
  role: string;                // "user" | "assistant"

  @Column({ type: 'text', nullable: true })
  question: string;            // User's question (for role=user)

  @Column({ type: 'text', nullable: true })
  answer: string;              // Assistant's answer (for role=assistant)

  @Column({
    type: 'varchar',
    nullable: true,
  })
  category: string;            // e.g. "LOAN_RULES", "RESERVATION" (for role=assistant)

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

**AssistantKnowledge:**
```typescript
// domain/assistant/entities/assistant-knowledge.entity.ts
@Entity('assistant_knowledge')
export class AssistantKnowledge {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar' })
  question: string;                 // Canonical question text

  @Column({ type: 'text' })
  answer: string;                    // Canonical answer text

  @Column({
    type: 'enum',
    enum: KnowledgeCategory,
    default: KnowledgeCategory.Other,
  })
  category: KnowledgeCategory;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

**Session (Refresh Token):**
```typescript
// domain/session/entities/session.entity.ts
@Entity('sessions')
export class Session {
  @Column({ type: 'uuid', primary: true })
  id: string;   // UUIDv7 — timestamp embedded, sortable by time

  @BeforeInsert()
  generateId() {
    // UUIDv7 — timestamp embedded, sortable by time
    this.id = generateUUIDv7();
  }

  @Column({ name: 'user_id', type: 'uuid' })
  userId: string;

  @ManyToOne(() => User)
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ name: 'token_hash', type: 'varchar' })
  tokenHash: string;                   // SHA-256 of raw refresh token

  @Column({ name: 'user_agent', type: 'varchar', nullable: true })
  userAgent: string;

  @Column({ name: 'ip_address', type: 'varchar', nullable: true })
  ipAddress: string;

  @Column({ name: 'expires_at', type: 'timestamp' })
  expiresAt: Date;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

Entities in `domain/<entity>/entities/` use TypeORM decorators directly (e.g. `@Entity`, `@Column`, `@ManyToOne`). There is no separate pure-TS-to-TypeORM mapper layer — domain entities ARE the TypeORM entities. This trades strict Clean Architecture isolation for development velocity.

**Transaction boundary rule (mandatory):** All Use Cases involving multi-entity writes — including Borrow, Return, Reservation Cancellation (queue shift), and Renewal — MUST be wrapped in a TypeORM `EntityManager` transaction (`em.transaction(async (tx) => { ... })`) to ensure atomicity. If any step in the transaction fails, all prior changes are rolled back, preventing partial/inconsistent state across Copy, Loan, Fine, Notification, and AuditLog tables.

### 4.2 Enumerated Values

All enums defined in `domain/shared/enums.ts`. **First value (0) is always `Undefined`** — deliberate security design.

```typescript
// domain/shared/enums.ts

// --- User ---
export enum UserRole {
  Undefined = 0,    // security: invalid default
  Member = 1,
  Librarian = 2,
  Admin = 3,
}

export enum UserStatus {
  Undefined = 0,
  Pending = 1,
  Active = 2,
  Expired = 3,
  Frozen = 4,
}

// --- Loan ---
export enum LoanStatus {
  Undefined = 0,
  Active = 1,
  Returned = 2,
}

// --- Copy ---
export enum CopyStatus {
  Undefined = 0,
  Available = 1,
  OnLoan = 2,
  Maintenance = 3,
  Lost = 4,
  Removed = 5,
}

// --- Reservation ---
export enum ReservationStatus {
  Undefined = 0,
  Queued = 1,
  ReadyForPickup = 2,
  Completed = 3,
  Expired = 4,
  Cancelled = 5,
}

// --- Fine ---
export enum FineStatus {
  Undefined = 0,
  Unpaid = 1,
  Paid = 2,
  Waived = 3,
}

// --- Notification ---
export enum NotificationType {
  Undefined = 0,
  DueReminder = 1,
  OverdueReminder = 2,
  ReservationReady = 3,
  ReservationExpired = 4,
  SystemAnnouncement = 5,
}

// --- Session ---
export enum SessionStatus {
  Undefined = 0,
  Active = 1,
  Expired = 2,
  Revoked = 3,
}

// --- Return Condition ---
export enum ReturnCondition {
  Undefined = 0,
  Normal = 1,    // Good condition — copy status returns to AVAILABLE
  Damaged = 2,  // Damaged — copy status set to MAINTENANCE for repair
  Lost = 3,     // Reported lost by member — copy status set to LOST
}

// --- Assistant Knowledge ---
export enum KnowledgeCategory {
  LoanRules = 'LOAN_RULES',    // borrowing rules, due dates, renewals
  Reservation = 'RESERVATION', // how to reserve, queue, pickup
  Fine = 'FINE',              // fine calculation, payment, waiver
  SystemUsage = 'SYSTEM_USAGE', // how to register, login, use features
  Other = 'OTHER',
}
```

### 4.3 DomainError

Same design as Go version. The backend is fully language-agnostic — `message` is English for dev/debug, frontend owns i18n.

```typescript
// domain/shared/domain-error.ts

export class DomainError extends Error {
  constructor(
    public readonly code: string,      // Granular, e.g. "ERR_NO_AVAILABLE_COPIES"
    message: string,                    // English for dev/debug
    public readonly context: Record<string, any> = {},
    public readonly cause: Error | undefined = undefined,
  ) {
    super(message);
    this.name = 'DomainError';
  }
}
```

**Error code naming:** `ERR_<NOUN>_<REASON>` in SCREAMING_SNAKE_CASE.

**Error codes (same exhaustive list as Go version):**

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `ERR_USER_EMAIL_EXISTS` | 409 | Email already registered |
| `ERR_USER_NOT_FOUND` | 404 | User does not exist |
| `ERR_USER_ACCOUNT_PENDING` | 422 | Account awaiting admin approval |
| `ERR_USER_ACCOUNT_FROZEN` | 422 | Account is frozen |
| `ERR_USER_ACCOUNT_EXPIRED` | 422 | Membership has expired |
| `ERR_BOOK_NOT_FOUND` | 404 | Book does not exist |
| `ERR_BOOK_INACTIVE` | 422 | Book is not active |
| `ERR_COPY_NOT_FOUND` | 404 | Copy (barcode) does not exist |
| `ERR_NO_AVAILABLE_COPIES` | 422 | No available copies to borrow |
| `ERR_LOAN_NOT_FOUND` | 404 | Loan record not found |
| `ERR_LOAN_NOT_ACTIVE` | 422 | Loan is not in active state |
| `ERR_RENEWAL_LIMIT_REACHED` | 422 | Maximum renewal count exceeded |
| `ERR_RENEWAL_BLOCKED_BY_RESERVATION` | 422 | Book has active queued reservation; renewal denied |
| `ERR_RESERVATION_ALREADY_EXISTS` | 409 | Active reservation already exists |
| `ERR_RESERVATION_NOT_FOUND` | 404 | Reservation not found |
| `ERR_RESERVATION_CANNOT_CANCEL` | 422 | Cannot cancel in current state |
| `ERR_KNOWLEDGE_NOT_FOUND` | 404 | Knowledge base entry not found |
| `ERR_FINE_NOT_FOUND` | 404 | Fine record not found |
| `ERR_SESSION_EXPIRED` | 401 | Refresh token session has expired |
| `ERR_INVALID_REFRESH_TOKEN` | 401 | Refresh token is invalid or not found |
| `ERR_VALIDATION_FAILED` | 400 | Request validation failed |
| `ERR_UNAUTHORIZED` | 401 | Missing or invalid JWT |
| `ERR_FORBIDDEN` | 403 | Caller does not have permission |
| `ERR_INTERNAL` | 500 | Unexpected server error |

### 4.4 StorageService Interface (Domain Interface)

```typescript
// domain/storage/storage.service.interface.ts

export interface StorageService {
  save(
    reader: Readable,        // Node.js Readable stream — NEVER a full Buffer
    originalFilename: string,
  ): Promise<string>;        // Returns relative CAS path e.g. "18/23/ad15...c3.png"
}
```

**Mandatory implementation rules — memory safety:**
- **No full file in memory.** Pipe the upload stream to a **temporary file** while simultaneously computing the SHA-256 hash via a stream-mode hash. Once the stream finishes and the final hash is computed, use `fs.rename()` to move the temporary file to its final computed CAS destination path (`{h[0:2]}/{h[2:4]}/{h[4:]}<ext>`). If the destination already exists (deduplication check via `fs.stat()` before rename), safely delete the temp file and reuse the existing path.
- **SHA-256 via stream mode:** `crypto.createHash('sha256')` accepts piped data; call `hash.digest('hex')` after the pipeline completes. Never `crypto.createHash('sha256').update(fullBuffer)`.
- **64KB internal buffer:** `stream.pipeline` uses a default 64KB high-water mark; do NOT increase this to avoid OOM on large files.
- **Thread-safety note:** Local disk single-instance only. Future S3 upgrade will require a different approach.

Path generation: SHA-256 hash → `{h[0:2]}/{h[2:4]}/{h[4:]}<ext>`. The extension `<ext>` MUST be extracted using `path.extname()`, converted to lowercase, and validated against an allowlist (e.g. `.jpg`, `.jpeg`, `.png`, `.gif`, `.pdf`) before path construction. Deduplication by checking file existence before write.

### 4.5 DTO Validation & Enum Boundary Defense

**Why this matters in TypeScript:** In JavaScript/TypeScript, a missing JSON field in the request body results in `undefined` — NOT the enum's `0` value. This bypasses the `Undefined = 0` safety check because `undefined !== 0`. Therefore, every incoming DTO must go through strict `class-validator` checks **at the Controller layer before entering any Use Case**.

**Required validation patterns:**

```typescript
// Every enum field MUST use @IsDefined() + @IsEnum()
@Column({ type: 'int' })
@IsDefined({ message: 'role is required' })
@IsEnum(UserRole, { message: 'Invalid user role' })
role: UserRole;

// Boolean fields should also be validated
@Column({ type: 'boolean', default: false })
@IsBoolean()
isActive: boolean;

// UUID fields must be validated
@Column({ name: 'book_id', type: 'uuid' })
@IsUUID()
bookId: string;
```

**Validation rules:**
1. **All enum fields:** `@IsDefined()` + `@IsEnum()` — rejects both `undefined` and out-of-range values
2. **All foreign key UUIDs:** `@IsUUID()` from `class-validator`
3. **All required strings:** `@IsNotEmpty()` with `@MaxLength()`
4. **Numeric bounds:** `@Min()` / `@Max()` for fine amounts, renewal counts, etc.
5. **DTOs live in `common/dto/`** and are decorated with `class-validator` + `class-transformer` (`@Type(() => Class)`) decorators

**Transformation before Use Case:** Use `class-transformer`'s `plainToClass()` via NestJS's built-in `ValidationPipe` to automatically transform and validate incoming request bodies.

---

### 4.6 Repository Interfaces

Each repository interface is defined in `domain/<entity>/repositories/<entity>.repository.interface.ts`.

> **Transactional context:** All write-based methods (`create`, `update`, `delete`, `save`) accept an optional `EntityManager` parameter. When called within a transaction, pass the transaction's `EntityManager`; otherwise omit it and the repository uses its own connection. This allows all multi-entity writes within a transaction to share the same connection and rolling back on error.

```typescript
// domain/user/repositories/user.repository.interface.ts

export interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(user: User, em?: EntityManager): Promise<void>;
  update(user: User, em?: EntityManager): Promise<void>;
}

// domain/book/repositories/book.repository.interface.ts

export interface BookSearchQuery {
  keyword?: string;
  category?: string;
  author?: string;
  availableOnly?: boolean;
  page: number;
  pageSize: number;
}

export interface BookRepository {
  findById(id: string): Promise<Book | null>;
  search(q: BookSearchQuery): Promise<{ books: Book[]; total: number }>;
  create(book: Book, em?: EntityManager): Promise<void>;
  update(book: Book, em?: EntityManager): Promise<void>;
  deactivate(id: string, em?: EntityManager): Promise<void>;
}

// domain/copy/repositories/copy.repository.interface.ts

export interface CopyRepository {
  findById(id: string): Promise<Copy | null>;
  findByBarcode(barcode: string): Promise<Copy | null>;
  create(copy: Copy, em?: EntityManager): Promise<void>;
  update(copy: Copy, em?: EntityManager): Promise<void>;

  // Pessimistic lock: SELECT ... FOR UPDATE on copy row during checkout/return.
  // Prevents race condition where two concurrent requests both read AVAILABLE
  // and create duplicate loans for the same physical copy.
  findByBarcodeForUpdate(barcode: string, em: EntityManager): Promise<Copy | null>;
}

// domain/loan/repositories/loan.repository.interface.ts

export interface LoanRepository {
  findById(id: string): Promise<Loan | null>;
  findActiveByUserId(userId: string): Promise<Loan[]>;
  create(loan: Loan, em?: EntityManager): Promise<void>;
  update(loan: Loan, em?: EntityManager): Promise<void>;
}

// domain/reservation/repositories/reservation.repository.interface.ts

export interface ReservationRepository {
  findById(id: string): Promise<Reservation | null>;
  findQueuedByBookId(bookId: string): Promise<Reservation[]>;   // ordered by createdAt ASC (FIFO)
  findReadyByUserId(userId: string): Promise<Reservation | null>;
  create(reservation: Reservation, em?: EntityManager): Promise<void>;
  update(reservation: Reservation, em?: EntityManager): Promise<void>;
  delete(id: string, em?: EntityManager): Promise<void>;
}

// domain/fine/repositories/fine.repository.interface.ts

export interface FineRepository {
  findById(id: string): Promise<Fine | null>;
  findByUserId(userId: string): Promise<Fine[]>;
  create(fine: Fine, em?: EntityManager): Promise<void>;
  update(fine: Fine, em?: EntityManager): Promise<void>;
}

// domain/notification/repositories/notification.repository.interface.ts

export interface NotificationRepository {
  findById(id: string): Promise<Notification | null>;
  findByUserId(userId: string, limit?: number): Promise<Notification[]>;
  create(notification: Notification, em?: EntityManager): Promise<void>;
  update(notification: Notification, em?: EntityManager): Promise<void>;
  deleteOldByUserId(userId: string, before: Date, em?: EntityManager): Promise<void>;
}

// domain/auditlog/repositories/audit-log.repository.interface.ts

export interface AuditLogRepository {
  findById(id: string): Promise<AuditLog | null>;
  search(filters: AuditLogSearchQuery): Promise<{ logs: AuditLog[]; total: number }>;
  create(log: AuditLog, em?: EntityManager): Promise<void>;
}

// domain/reminderpolicy/repositories/reminder-policy.repository.interface.ts

export interface ReminderPolicyRepository {
  findActive(): Promise<ReminderPolicy | null>;
  update(policy: ReminderPolicy, em?: EntityManager): Promise<void>;
}

// domain/assistant/repositories/assistant.repository.interface.ts

export interface KnowledgeSearchQuery {
  category?: KnowledgeCategory;
  keyword?: string;       // search in question + answer
  page: number;
  pageSize: number;
}

export interface AssistantRepository {
  // AssistantSession
  createSession(session: AssistantSession, em?: EntityManager): Promise<void>;
  findSessionById(id: string): Promise<AssistantSession | null>;
  deleteSession(id: string, em?: EntityManager): Promise<void>;

  // AssistantMessage
  createMessage(msg: AssistantMessage, em?: EntityManager): Promise<void>;
  findMessagesBySessionId(sessionId: string): Promise<AssistantMessage[]>;

  // AssistantKnowledge (Admin CRUD)
  findKnowledgeById(id: string): Promise<AssistantKnowledge | null>;
  searchKnowledge(q: KnowledgeSearchQuery): Promise<{ entries: AssistantKnowledge[]; total: number }>;
  createKnowledge(entry: AssistantKnowledge, em?: EntityManager): Promise<void>;
  updateKnowledge(entry: AssistantKnowledge, em?: EntityManager): Promise<void>;
  deleteKnowledge(id: string, em?: EntityManager): Promise<void>;
}

// domain/session/repositories/session.repository.interface.ts

export interface SessionRepository {
  findById(id: string): Promise<Session | null>;
  findByTokenHash(hash: string): Promise<Session | null>;
  create(session: Session, em?: EntityManager): Promise<void>;
  delete(id: string, em?: EntityManager): Promise<void>;
  countByUserId(userId: string): Promise<number>;       // active session count for max-5 eviction
  deleteOldestByUserId(userId: string, em?: EntityManager): Promise<void>;  // evict oldest session when count >= 5
}
```

---

## 5. Authentication

### 5.1 Design Principle: All-Cookie + Silent Auto-Refresh

Identical to Go version. Both tokens live in HttpOnly cookies. Frontend never handles token storage.

### 5.2 Cookie Specification

| Cookie Name | Type | HttpOnly | Secure | SameSite | Max-Age | Contents |
|-------------|------|----------|--------|----------|---------|----------|
| `access_token` | Access JWT | Yes | Yes* | Lax | 3600 (1h) | JWT string |
| `refresh_token` | Opaque token | Yes | Yes* | Lax | 604800 (7d) | Raw refresh token |

*Secure flag: enabled in production; configurable off for local dev.

**`access_token` is validated from cookie only — the `Authorization: Bearer <token>` header is never read.** This is a hard requirement: the JwtAuthGuard reads `req.cookies['access_token']` exclusively.

### 5.3 JWT (Access Token)

- Signed with HS256
- Lifetime: **1 hour** (configurable)
- **Not passed in Authorization header** — validated from cookie only
- Payload: `{ "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN", "sessionId": "uuid" }`

### 5.4 Session Entity

```typescript
// domain/session/entities/session.entity.ts
@Entity('sessions')
export class Session {
  @Column({ type: 'uuid', primary: true })
  id: string;

  @BeforeInsert()
  generateId() {
    // UUIDv7 — timestamp embedded, sortable by time
    this.id = generateUUIDv7();
  }

  @Column({ name: 'user_id', type: 'uuid' })
  userId: string;

  @ManyToOne(() => User)
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ name: 'token_hash', type: 'varchar' })
  tokenHash: string;      // SHA-256 of raw refresh token — raw token never persisted

  @Column({ name: 'user_agent', type: 'varchar', nullable: true })
  userAgent: string;

  @Column({ name: 'ip_address', type: 'varchar', nullable: true })
  ipAddress: string;

  @Column({ name: 'expires_at', type: 'timestamp' })
  expiresAt: Date;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

Max **5 active sessions per user** — oldest evicted on 6th login.

### 5.5 JWT Auth Guard — Silent Auto-Refresh Flow (Fail-Open)

**Key design principle (aligned with Go §11.5):** All JWT verification errors during the refresh flow must **fail open** — log a warning and allow the request to proceed, rather than reject it. Only throw `ERR_UNAUTHORIZED` when both the access token AND the refresh attempt have failed (i.e., no valid session exists at all).

This applies to: malformed JWT tokens, algorithm mismatches, expired JWTs before refresh, and JWT signing failures during refresh. The frontend never sees a 401 caused by a transient JWT service error.

```typescript
// delivery/guards/jwt-auth.guard.ts

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    @Inject('SESSION_REPOSITORY') private sessionRepo: SessionRepository,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const ctx = context.switchToHttp();
    const req = ctx.getRequest<Request>();
    const res = ctx.getResponse<Response>();

    // Step 1: Statelessly validate JWT from cookie — wrapped in try-catch.
    // Non-fatal errors (malformed token, algorithm mismatch) fail open per Go §11.5.
    let claims: JwtClaims | null = null;
    const accessToken = req.cookies?.['access_token'];
    try {
      claims = await this.validateToken(accessToken);
    } catch (err) {
      // Malformed/expired JWT — fail open, attempt silent refresh.
      console.warn('[auth] JWT verify error (failing open):', err);
    }

    if (!claims) {
      // Step 2: Token missing, expired, or invalid — attempt silent refresh.
      const refreshed = await this.trySilentRefresh(req, res);
      if (refreshed) {
        const newToken = req.cookies?.['access_token'];
        try {
          claims = await this.validateToken(newToken);
        } catch (err) {
          // New token from refresh also invalid — fail open.
          console.warn('[auth] Post-refresh JWT verify error (failing open):', err);
        }
      }
    }

    if (!claims) {
      // Only throw when: no token at all AND refresh cookie missing or session expired.
      throw new DomainError('ERR_UNAUTHORIZED', 'No valid session', {});
    }

    req['userId'] = claims.userId;
    req['role'] = claims.role;
    req['sessionId'] = claims.sessionId;
    return true;
  }

  private async trySilentRefresh(req: Request, res: Response): Promise<boolean> {
    const rawToken = req.cookies?.['refresh_token'];
    if (!rawToken) return false;

    const hash = sha256(rawToken);
    const session = await this.sessionRepo.findByTokenHash(hash);
    if (!session || new Date() > session.expiresAt) return false;

    try {
      const newToken = this.jwtService.sign({
        userId: session.userId,
        role: session.user.role,
        sessionId: session.id,
      });
      res.cookie('access_token', newToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 3600 * 1000,
      });
      return true;
    } catch (err) {
      // Fail-open: JWT generation error (service jitter) — log and allow request.
      console.warn('[auth] Silent refresh JWT sign error (failing open):', err);
      return false;
    }
  }

  private async validateToken(token: string | undefined): Promise<JwtClaims | null> {
    if (!token) return null;
    return this.jwtService.verify<JwtClaims>(token);
  }
}

interface JwtClaims {
  userId: string;
  role: string;
  sessionId: string;
}
```

### 5.6 Login Flow

```
1. Validate email + password
2. Check account status → ErrUserAccountPending / ErrUserAccountFrozen / ErrUserAccountExpired
3. Generate raw refresh token (crypto-safe random)
4. Create Session record (TokenHash = SHA-256(raw), ExpiresAt = now + 7d)
5. If session count ≥ 5 → Delete oldest session for user
6. Generate JWT (1h expiry) with { userId, role, sessionId }
7. Set access_token + refresh_token HttpOnly cookies
8. Return: { "code": 200, "message": "Success", "data": { "userId": "uuid", "role": "MEMBER|LIBRARIAN|ADMIN" } }
```

### 5.7 Logout

- `DELETE /api/auth/logout` — deletes current session, clears cookies
- `DELETE /api/auth/logout-all` — deletes all sessions for user, clears cookies

Both set cookie `Max-Age: 0`.

---

## 6. API Routes

Routes registered via NestJS `@Controller()` decorators — no centralized router file.

```
/api/auth/*                    — public
  POST /api/auth/register       — register new member
  POST /api/auth/login          — login (sets HttpOnly cookies, returns { userId, role })
  DELETE /api/auth/logout       — logout current session
  DELETE /api/auth/logout-all   — logout all sessions
  POST /api/auth/refresh        — refresh tokens (mobile clients only)

/api/books/*                   — public search; member detail
  GET /api/books/search         — search books
  GET /api/books/:bookId        — book detail + real-time copy stats

/api/member/*                  — authenticated member
  GET  /api/member/profile
  PUT  /api/member/profile
  PUT  /api/member/profile/avatar          — multipart/form-data; accepts image file, returns { avatarUrl }
  GET  /api/member/membership
  GET  /api/member/loans    — response includes `remainingRenewals` (maxRenewals − renewalCount)
  POST /api/member/loans    — body: { bookId, copyId? }; if copyId omitted, auto-assigns first available copy; if member has a READY_FOR_PICKUP reservation for this book, that reservation is atomically transitioned to COMPLETED within the same transaction
  POST /api/member/loans/:loanId/renew
  GET  /api/member/reservations   — response includes `queuePosition` for QUEUED reservations
  POST /api/member/reservations
  DELETE /api/member/reservations/:reservationId  — if reservation is READY_FOR_PICKUP, triggers immediate queue shift: next QUEUED reservation is promoted to READY_FOR_PICKUP, new deadline set, RESERVATION_READY notification sent
  GET  /api/member/notifications
  PUT  /api/notifications/:notificationId/read
  PUT  /api/member/notifications/read-all
  GET  /api/member/recommendations

/api/admin/*                   — authenticated librarian / admin
  GET  /api/admin/assistant/knowledge              — list knowledge base (paginated)
  GET  /api/admin/assistant/knowledge/:knowledgeId — get single entry
  POST /api/admin/assistant/knowledge             — create knowledge entry
  PUT  /api/admin/assistant/knowledge/:knowledgeId — update entry
  DELETE /api/admin/assistant/knowledge/:knowledgeId — delete entry (hard delete)
  GET  /api/admin/members
  POST /api/admin/members/:memberId/approve    — body: { endDate: string (ISO date) }
  POST /api/admin/members/:memberId/freeze     — body: { reason: string }
  POST /api/admin/members/:memberId/unfreeze
  POST /api/admin/members/:memberId/renew
  GET  /api/admin/books
  POST /api/admin/books
  PUT  /api/admin/books/:bookId
  PUT  /api/admin/books/:bookId/cover          — multipart/form-data; accepts image file, updates coverUrl
  PUT  /api/admin/books/:bookId/deactivate
  GET  /api/admin/copies
  POST /api/admin/copies
  PUT  /api/admin/copies/:copyId/status
  GET  /api/admin/fines
  POST /api/admin/fines/:fineId/pay
  POST /api/admin/fines/:fineId/waive
  GET  /api/admin/reminder-policy
  PUT  /api/admin/reminder-policy
  POST /api/admin/notifications/announcements   — broadcast SYSTEM_ANNOUNCEMENT to all active members
  GET  /api/admin/audit-logs
  GET  /api/admin/audit-logs/:logId

/api/circulation/*             — authenticated librarian / admin
  POST /api/circulation/checkout   — body: { memberId: string, copyBarcode: string }
  POST /api/circulation/return    — body: { copyBarcode: string, condition: ReturnCondition }
  GET  /api/circulation/lookup?barcode={barcode}

/api/assistant/*               — public (no auth required)
  GET  /api/assistant/quick-questions
  POST /api/assistant/sessions
  POST /api/assistant/chat
  GET  /api/assistant/sessions/:sessionId/messages
  DELETE /api/assistant/sessions/:sessionId
```

---

## 7. API Response Format

**Language-agnostic:** `message` is always English. Frontend uses `errorCode` for i18n.

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
  "data": { "bookId": "123" }
}
```

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

## 8. Background Jobs

Using `@nestjs/schedule` for cron-based jobs.

`ShutdownFlagService` is a singleton that implements `OnModuleDestroy`. When the server receives SIGTERM/SIGINT and NestJS initiates graceful shutdown, `onModuleDestroy()` sets `isShuttingDown() = true`. Long-running jobs (DueReminder, OverdueReminder) check this flag at the **start of each batch iteration** and exit safely before committing any batch state, preventing dirty data on shutdown.

| Job | Schedule | Safe Interruption |
|-----|----------|-------------------|
| Due Reminder | Daily 08:00 | ✅ Checks `ShutdownFlagService` at each batch start (BATCH_SIZE=100) |
| Overdue Reminder | Daily 08:30 | ✅ Checks `ShutdownFlagService` at each batch start (BATCH_SIZE=100) |
| Reservation Expiry | Every 30 min | Short-running (<1s), no interrupt needed |
| Session Cleanup | Every 1 hour + on startup | Short-running (<1s), no interrupt needed; also runs on server startup to purge any expired sessions missed while the server was down. Also purges anonymous AssistantSession records (where userId is null) older than 24 hours to prevent orphaned chat data leak. |

**CAS Garbage Collection (long-term maintenance):** A periodic job (recommended: weekly) should scan the CAS storage directory and compare all files against the set of paths currently referenced in `User.avatarUrl` and `Book.coverUrl`. Any file not referenced by any entity should be deleted. This prevents orphaned files from accumulating when avatars are updated or books get new covers. Run infrequently (weekly) to avoid performance impact.

### Innovation A — Book Recommendation

Rule-based engine in `usecase/recommendation/rule-engine.usecase.ts`:
- Same-category popular books (based on loan history)
- Same-author books (based on loan history)
- Recently added books (last 30 days)
- Cold start: top popular + newest books
- Returns exactly 5 unique recommendations with `reason` and `reasonType`
- Excludes already-borrowed books

**"换一批" (Refresh) support:**
- `GET /api/member/recommendations?seed=<string>` — optional seed for deterministic selection
- If `seed` provided: FNV-64a hash → int64 seed for deterministic random offset
- If `seed` omitted: `Math.random()` seed generated server-side

### Innovation B — Reminder Policy

**Fine Calculation Formula** (applied in `calculate-fine.usecase.ts`):
```
Fine Amount = max(0, Overdue Days − Grace Days) × Daily Fine Amount
```
- Capped at `maxFineAmount` from ReminderPolicy
- Grace Days defaults to 0; only strictly positive overdue days incur fine
- Example: 5 overdue days, 2 grace days, $1/day daily fine → `max(0, 5-2) × 1 = $3`

```typescript
// domain/reminderpolicy/entities/reminder-policy.entity.ts
@Entity('reminder_policies')
export class ReminderPolicy {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'due_days_before', type: 'jsonb', default: [3, 1, 0] })
  dueDaysBefore: number[];

  @Column({ name: 'overdue_days_after', type: 'jsonb', default: [1, 3, 7] })
  overdueDaysAfter: number[];

  @Column({ name: 'daily_fine_amount', type: 'decimal', precision: 10, scale: 2 })
  dailyFineAmount: number;

  @Column({ name: 'max_fine_amount', type: 'decimal', precision: 10, scale: 2 })
  maxFineAmount: number;

  @Column({ name: 'grace_days', type: 'int', default: 0 })
  graceDays: number;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

### Innovation C — AI Assistant (FAQ Bot)

Same knowledge-based approach as Go version:
- Classifies question into category (loan rules, reservation, fine, system usage)
- Searches `AssistantKnowledge` table for matching Q&A
- Falls back to "please contact librarian" response
- Session management via `AssistantSession` table
- `GET /api/assistant/quick-questions` — predefined quick question tags

---

## 10. Fine-grained Error Codes & HTTP Status Mapping

The `HttpExceptionFilter` maps `DomainError.code` to HTTP status:

```typescript
// delivery/filters/http-exception.filter.ts

@Catch(DomainError)
export class DomainErrorFilter implements ExceptionFilter {
  catch(error: DomainError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const status = this.errorCodeToHTTP(error.code);
    res.status(status).json(
      Response.error(error.code, error.message, error.context),
    );
  }

  private errorCodeToHTTP(code: string): number {
    if (code === 'ERR_VALIDATION_FAILED') return 400;
    if (code === 'ERR_UNAUTHORIZED') return 401;
    if (code === 'ERR_FORBIDDEN') return 403;
    if (code.endsWith('_NOT_FOUND')) return 404;
    if (code === 'ERR_USER_EMAIL_EXISTS' || code === 'ERR_RESERVATION_ALREADY_EXISTS') return 409;
    if (['ERR_USER_ACCOUNT_PENDING', 'ERR_USER_ACCOUNT_FROZEN', 'ERR_USER_ACCOUNT_EXPIRED',
         'ERR_NO_AVAILABLE_COPIES', 'ERR_RENEWAL_LIMIT_REACHED', 'ERR_RESERVATION_CANNOT_CANCEL',
         'ERR_LOAN_NOT_ACTIVE', 'ERR_BOOK_INACTIVE'].includes(code)) return 422;
    return 500;
  }
}
```

---

## 11. State Machines

Identical to original design:

- **User:** PENDING → ACTIVE → EXPIRED/FROZEN (→ ACTIVE via unfreeze/renew)
- **Loan:** ACTIVE ↔ RETURNED, ACTIVE → ACTIVE (renew)
- **Reservation:** QUEUED → READY_FOR_PICKUP → COMPLETED/EXPIRED/CANCELLED
- **Copy:** AVAILABLE ↔ ON_LOAN, AVAILABLE ↔ MAINTENANCE, ON_LOAN → AVAILABLE/MAINTENANCE/LOST
- **Fine:** UNPAID → PAID/WAIVED

---

## 12. Pending Implementation Details

- **Graceful shutdown with hard timeout** — implemented in §3 (main.ts snippet with 8s `setTimeout` force-exit)
- **Enum boundary defense** — `class-validator` `@IsDefined()` + `@IsEnum()` mandatory on all enum fields (implemented in §4.5)
- **UUIDv7 generation** — install `uuid` v9+; create `domain/shared/uuid.ts` wrapping `v7()`; replace `@PrimaryGeneratedColumn('uuid')` on `AuditLog` and `Session` with `@Column({ type: 'uuid', primary: true })` + `@BeforeInsert()` calling `generateUUIDv7()`; other entities keep TypeORM default
- **JWT fail-open** — implemented in §5.5 (JwtAuthGuard wraps ALL `validateToken()` calls in try-catch, including the initial stateless verify; only throws `ERR_UNAUTHORIZED` when both access token AND refresh fail)
- **Cron safe interruption** — create `common/shutdown/shutdown-flag.service.ts` singleton implementing `OnModuleDestroy`; inject into DueReminder and OverdueReminder; check `isShuttingDown()` at each batch start (BATCH_SIZE=100); bail out before updating any state
- **AssistantKnowledge admin CRUD** — create `common/dto/assistant/{create,update,query}-knowledge.dto.ts`; add `KnowledgeCategory` enum to `domain/shared/enums.ts`; add `ERR_KNOWLEDGE_NOT_FOUND` to `domain/assistant/errors/`; extend `AssistantRepository` interface with `{find,search,create,update,delete}Knowledge`; implement in `infrastructure/orm/repositories/assistant.repository.ts`; create `delivery/controllers/assistant-knowledge.controller.ts` at `api/admin/assistant/knowledge` (5 endpoints: list, getOne, create, update, delete) with `@Roles(UserRole.Admin, UserRole.Librarian)`
- Swagger annotations on all controller methods
- All DTOs in `common/dto/` with `class-validator` decorators (struct tags)

**UpdateProfile DTO** (`common/dto/update-profile.dto.ts`):
```typescript
// common/dto/update-profile.dto.ts
export class UpdateProfileDto {
  @IsOptional()
  @IsNotEmpty()
  @MaxLength(100)
  name?: string;

  @IsOptional()
  @IsNotEmpty()
  @MaxLength(50)
  studentId?: string;        // University/student ID

  @IsOptional()
  @IsNotEmpty()
  @MaxLength(20)
  phone?: string;

  @IsOptional()
  @IsNotEmpty()
  @MaxLength(255)
  address?: string;
}
// Avatar is handled separately via multipart/form-data upload endpoint
// PUT /api/member/profile/avatar — accepts file, returns { avatarUrl }
```

**Reservation response DTO** must include `queuePosition: number` (position among `QUEUED` reservations for the same book, ordered by `createdAt ASC`)

**Loan response DTO** must include `remainingRenewals: number` = `maxRenewals − renewalCount`

**Broadcast announcement** — `POST /api/admin/notifications/announcements` — Request body: `{ title: string, message: string }`. Creates one `Notification` per active member (`UserStatus.Active`) with `NotificationType.SystemAnnouncement`, then returns `{ count: number }` of notifications created. If no active members, returns `{ count: 0 }` with code 200.
- File upload: must use `stream.pipeline()` + `crypto.createHash('sha256')` stream mode (no full file in memory)
- Renewal use case: must check `SELECT ... WHERE bookId = ? AND status = QUEUED` before allowing renewal; raise `ERR_RENEWAL_BLOCKED_BY_RESERVATION` if found
- Unit tests: key use cases covered before implementation
