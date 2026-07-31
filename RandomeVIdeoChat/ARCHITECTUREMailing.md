# Notification Microservice — Architecture

A standalone, reusable service that any Exam Portal module (Exam, Quiz, Coding) can call to send **email** and **in-app** notifications. Calling modules never build notification logic of their own — they POST an event to one API, and this service handles templating, persistence, delivery, and retrieval.

---

## 1. Architecture at a glance

The service follows a layered, event-triggered design with a deliberately split delivery model:

- **Synchronous trigger.** A module calls the trigger API. The service validates the request, renders the template, writes the in-app notification to PostgreSQL, and returns an id + status **immediately**. This is the part the caller waits on, and it is fast (a single insert).
- **Asynchronous email.** Email delivery runs on a separate thread *after* the database transaction commits, so a slow or failing SMTP call never blocks or breaks the caller (AC3). A rolled-back notification never produces an email.

This separation is the central design decision: the caller gets a guaranteed, durable in-app record synchronously, while the slow external dependency (Gmail SMTP) is decoupled.

```
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Exam Module  │   │ Quiz Module  │   │ Coding Module│      Calling modules
   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘      (no notification
          └──────────────────┼──────────────────┘              logic of their own)
                             │  POST /api/v1/notifications
                             ▼
   ┌─────────────────────────────────────────────────────────┐
   │              NOTIFICATION MICROSERVICE (Spring Boot)      │
   │                                                          │
   │  REST API ─► Notification Service ─► Template Engine     │
   │                      │                                   │
   │            (1) render + (2) persist  ──────────┐         │
   │                      │                         │         │
   │                      │  publish AFTER_COMMIT    │         │
   │                      ▼                         ▼         │
   │            ┌───────────────────┐      ┌──────────────┐   │
   │            │ Async Email Worker│      │  PostgreSQL  │   │
   │            └─────────┬─────────┘      │ templates +  │   │
   └──────────────────────┼────────────────│ notifications│───┘
                          │                └──────┬───────┘
                          ▼                       │ GET (poll)
                ┌──────────────────┐              │
                │   Gmail SMTP     │              ▼
                │  → candidate mail │     ┌──────────────────┐
                └──────────────────┘     │  React Frontend  │
                                         │  Bell + Panel    │
                                         └──────────────────┘
```

---

## 2. Technology stack

| Layer | Choice | Notes |
|---|---|---|
| Backend | Spring Boot 3.x (Java 17+) | Spring Web, Spring Data JPA, Spring Mail, Spring `@Async` |
| Database | PostgreSQL | JPA/Hibernate, Flyway for migrations |
| Email | Gmail SMTP via `spring-boot-starter-mail` | App Password + STARTTLS, no paid provider (AC3) |
| API docs | springdoc-openapi | Renders Swagger UI; `springfox` is deprecated for Boot 3 |
| Frontend | React | React Query for data/cache, responsive bell + panel |
| Migrations/seed | Flyway | Schema + pre-configured templates as versioned scripts |

---

## 3. Components

**REST API layer.** Thin controllers that map HTTP to service calls and DTOs. One trigger endpoint plus retrieval, mark-read, and unread-count endpoints. Annotated for Swagger.

**Notification Service.** The orchestrator. Resolves the template for the requested type, renders placeholders with the supplied data, builds the notification entity, persists it, and raises an internal event for email delivery. Owns status transitions (UNREAD ⇄ READ) and unread-count queries.

**Template Engine.** Loads the active template for a notification type and substitutes `{{placeholder}}` tokens in both subject and body with values from the request payload. Kept simple (token replacement) but isolated behind an interface so it can be swapped for Thymeleaf/Freemarker later without touching callers.

**Async Email Worker.** A `@TransactionalEventListener(phase = AFTER_COMMIT)` running on a dedicated thread pool. Sends the rendered message through `JavaMailSender` (Gmail SMTP) and records the outcome in a delivery log. Failures are logged and retryable; they never surface to the original API caller.

**Persistence.** Spring Data JPA repositories over three tables (templates, notifications, delivery log).

**React Frontend.** A notification bell with an unread badge and a panel listing the user's notifications.

---

## 4. Data model

Notification `type`, `status`, and `channel` are stored as `VARCHAR` with `CHECK` constraints rather than native PostgreSQL `ENUM` types. This avoids the well-known friction between PG enums and Hibernate while keeping the same validation guarantees, and it maps cleanly to Java enums (`@Enumerated(EnumType.STRING)`).

```sql
-- TEMPLATES: one active template per type (subject + body with placeholders)
CREATE TABLE notification_template (
    id          BIGSERIAL PRIMARY KEY,
    type        VARCHAR(50)  NOT NULL,
    subject     VARCHAR(255) NOT NULL,
    body        TEXT         NOT NULL,
    channel     VARCHAR(20)  NOT NULL DEFAULT 'BOTH'
                 CHECK (channel IN ('EMAIL','IN_APP','BOTH')),
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT chk_template_type CHECK
        (type IN ('EXAM_ASSIGNED','RESULT_READY','FLAGGED_FOR_REVIEW','DEADLINE_REMINDER'))
);
CREATE UNIQUE INDEX uq_template_active_type
    ON notification_template (type) WHERE is_active = TRUE;

-- NOTIFICATIONS: the in-app record, one row per delivered notification
CREATE TABLE notification (
    id                BIGSERIAL PRIMARY KEY,
    recipient_user_id BIGINT       NOT NULL,
    recipient_email   VARCHAR(255),
    type              VARCHAR(50)  NOT NULL,
    subject           VARCHAR(255),
    message           TEXT         NOT NULL,
    status            VARCHAR(20)  NOT NULL DEFAULT 'UNREAD'
                       CHECK (status IN ('UNREAD','READ')),
    channel           VARCHAR(20)  NOT NULL DEFAULT 'BOTH',
    metadata          JSONB,                       -- the dynamic data used to render
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    read_at           TIMESTAMPTZ,
    CONSTRAINT chk_notif_type CHECK
        (type IN ('EXAM_ASSIGNED','RESULT_READY','FLAGGED_FOR_REVIEW','DEADLINE_REMINDER'))
);
-- Index 1 powers the unread-count query; Index 2 powers the reverse-chron panel
CREATE INDEX idx_notif_user_status  ON notification (recipient_user_id, status);
CREATE INDEX idx_notif_user_created ON notification (recipient_user_id, created_at DESC);

-- DELIVERY LOG: tracks async email outcome without polluting the notification row
CREATE TABLE notification_delivery (
    id              BIGSERIAL PRIMARY KEY,
    notification_id BIGINT NOT NULL REFERENCES notification(id) ON DELETE CASCADE,
    channel         VARCHAR(20) NOT NULL,
    delivery_status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                     CHECK (delivery_status IN ('PENDING','SENT','FAILED')),
    attempts        INT NOT NULL DEFAULT 0,
    error_message   TEXT,
    last_attempt_at TIMESTAMPTZ
);
```

`metadata` (JSONB) stores the exact data used to render the message, which makes the history fully auditable later (AC9).

---

## 5. API design

All endpoints are versioned under `/api/v1` and documented via springdoc/Swagger UI (typically at `/swagger-ui.html`).

| Method | Path | Purpose | Serves |
|---|---|---|---|
| POST | `/api/v1/notifications` | Trigger a notification | AC1, AC8 |
| GET | `/api/v1/notifications?userId=&status=&page=&size=` | Retrieve user notifications, paged, reverse-chron | AC6, AC9 |
| GET | `/api/v1/notifications/unread-count?userId=` | Unread count for badge | AC5 |
| PATCH | `/api/v1/notifications/{id}/read` | Mark one as read | AC7 |
| PATCH | `/api/v1/notifications/{id}/unread` | Mark one as unread | AC7 |
| PATCH | `/api/v1/notifications/read-all?userId=` | Mark all as read | AC7 |
| GET | `/api/v1/templates` | List templates | AC2 |
| POST / PUT | `/api/v1/templates` | Manage templates | AC2 |

**Trigger request** — type + recipient + dynamic data, with optional channel selection:

```json
POST /api/v1/notifications
{
  "type": "RESULT_READY",
  "recipient": { "userId": 1024, "email": "candidate@example.com" },
  "data": {
    "candidateName": "Asha",
    "examName": "Java Fundamentals",
    "score": "88%"
  },
  "channels": ["EMAIL", "IN_APP"]
}
```

**Trigger response** — id + status returned synchronously; email is queued, not awaited (AC1, AC3):

```json
HTTP 201 Created
{
  "notificationId": 55012,
  "status": "CREATED",
  "delivery": { "inApp": "STORED", "email": "QUEUED" }
}
```

**Unread-count response:**

```json
{ "userId": 1024, "unreadCount": 7 }
```

A consistent error envelope (timestamp, status, error code, message) is returned for validation failures, unknown types, and missing templates.

---

## 6. Notification lifecycle

1. A module sends `POST /api/v1/notifications` with `type`, `recipient`, and `data`.
2. The service validates the payload and looks up the active template for that type. An unknown type or missing template returns a 4xx error.
3. The Template Engine substitutes `{{placeholders}}` in subject and body using `data`.
4. The notification is inserted into PostgreSQL with `status = UNREAD` (this is the durable in-app record).
5. The API returns `201` with the notification id and status — the caller is now done.
6. **After the transaction commits**, the email event fires on a worker thread. The worker sends the message via Gmail SMTP and writes `SENT`/`FAILED` to the delivery log.
7. The candidate's React client, polling the unread-count and panel endpoints, reflects the new notification.

---

## 7. Templates and placeholders (AC2)

Templates live in the database with a subject and body containing `{{tokens}}`. Tokens are replaced at send time with values from the request `data`. Missing tokens resolve to a safe blank (and are logged) rather than rendering literal `{{...}}` to the recipient.

Four templates are pre-configured (seeded via Flyway):

| Type | Subject | Body (excerpt) |
|---|---|---|
| `EXAM_ASSIGNED` | New exam assigned: {{examName}} | Hi {{candidateName}}, you've been assigned **{{examName}}**, due {{deadline}}. |
| `RESULT_READY` | Your result for {{examName}} is ready | Hi {{candidateName}}, your result for {{examName}} is available. Score: {{score}}. |
| `FLAGGED_FOR_REVIEW` | Your submission was flagged for review | Hi {{candidateName}}, your submission for {{examName}} was flagged: {{reviewReason}}. |
| `DEADLINE_REMINDER` | Reminder: {{examName}} deadline approaching | Hi {{candidateName}}, {{examName}} is due {{deadline}}. Please submit on time. |

Seed example:

```sql
INSERT INTO notification_template (type, subject, body, channel) VALUES
('RESULT_READY',
 'Your result for {{examName}} is ready',
 'Hi {{candidateName}}, your result for {{examName}} is now available. Score: {{score}}.',
 'BOTH');
-- ... three more rows for EXAM_ASSIGNED, FLAGGED_FOR_REVIEW, DEADLINE_REMINDER
```

---

## 8. Asynchronous email delivery (AC3)

The recommended mechanism is an **internal application event handled after commit**, on a dedicated thread pool:

- The service publishes a `NotificationCreatedEvent` after persisting.
- A listener annotated `@TransactionalEventListener(phase = AFTER_COMMIT)` and `@Async` handles it, so it runs only once the row is safely committed, and on a separate thread.
- Define a bounded `ThreadPoolTaskExecutor` (e.g. core 4 / max 8 / queue 100) so email work can't exhaust request threads.

This is preferable to a bare `@Async` service call because it guarantees no email is sent for a transaction that later rolls back. For higher volume you can later swap the in-process event for a message queue (RabbitMQ/Kafka) or a DB outbox + scheduler — without changing the trigger API contract.

**Gmail SMTP configuration** (credentials from environment, never committed):

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_APP_PASSWORD}   # Google App Password (requires 2FA enabled)
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

Note Gmail requires an **App Password** (basic password auth was removed) and imposes a daily send cap (roughly 500 messages/day on a free account). That is fine for development and low volume per the requirement; production-scale volume would warrant a transactional email provider, but the SMTP-based design here keeps callers unchanged if you migrate.

---

## 9. Frontend architecture

**NotificationBell** renders a badge with the unread count. It polls `GET /unread-count` on an interval (≈30s) using React Query. Polling keeps the implementation simple and dependency-free; Server-Sent Events or WebSocket can replace it later for instant updates without changing the backend contract.

**NotificationPanel** opens from the bell and fetches `GET /notifications` (paged, reverse-chronological — AC6). Each row shows message, type, and timestamp. Unread items are visually distinct (bold + an accent dot); read items are muted. Clicking an item triggers an **optimistic** update (mark it read in the UI immediately), fires `PATCH /{id}/read`, then invalidates the unread-count query so the badge updates right away (AC5, AC7).

**Responsiveness (AC11).** The panel renders as a dropdown on desktop and a full-width drawer / bottom sheet on mobile, via CSS media queries (or Tailwind breakpoints), with touch-friendly tap targets.

A single API client module centralizes all calls. The `userId` comes from the authenticated session/JWT rather than being typed by the client.

---

## 10. Reusability and the integration contract (AC8)

Every module integrates the same way: one POST to `/api/v1/notifications` with a `type`, `recipient`, and `data` map. There is no module-specific code path inside the service — adding a new notification kind means adding a template row and an enum value, not new endpoints. This is what prevents duplication across Exam, Quiz, and Coding modules. Publish the request/response schema (Swagger) as the contract other teams build against.

---

## 11. Security and cross-cutting concerns

- **Service-to-service auth on the trigger API.** The endpoint that lets a caller send email *as* a recipient must be protected (internal network only, a shared API key/header, or OAuth2 client credentials), so external clients can't spoof recipients.
- **User scoping on retrieval.** Derive `userId` from the caller's JWT for read/mark-read endpoints rather than trusting a query param — otherwise a user could read another user's notifications. (The `userId` param is shown above for clarity, but enforce ownership server-side.)
- **Secrets** (SMTP App Password, DB credentials) live in environment variables / a secret manager, never in source.
- **Validation and rate limiting** on the trigger endpoint to reject malformed payloads and prevent abuse.
- **Data integrity (AC9).** In-app persistence is synchronous and transactional, so a successful API response means the record is durably stored; email delivery, being async, never risks the stored record.

---

## 12. Project structure

```
notification-service/
├── src/main/java/com/examportal/notification/
│   ├── controller/        # REST endpoints (trigger, retrieve, read, count)
│   ├── service/           # NotificationService, TemplateService
│   ├── template/          # TemplateEngine (placeholder substitution)
│   ├── email/             # EmailSender, async event listener
│   ├── domain/            # JPA entities (Notification, Template, Delivery)
│   ├── repository/        # Spring Data repositories
│   ├── dto/               # request/response objects
│   ├── event/             # NotificationCreatedEvent
│   ├── config/            # async executor, mail, OpenAPI/Swagger
│   └── exception/         # global handler + error envelope
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/      # Flyway: V1__schema.sql, V2__seed_templates.sql
├── src/test/java/...      # unit + integration tests
└── frontend/
    └── src/
        ├── components/    # NotificationBell, NotificationPanel, NotificationItem
        ├── api/           # notificationApi.js
        └── hooks/         # useUnreadCount, useNotifications
```

---

## 13. Testing strategy (AC12)

Tests must cover **send, read-status, and unread-count** scenarios explicitly:

- **Unit tests.** Template rendering (correct substitution, missing-token handling); service status transitions (UNREAD → READ persists `read_at`); unread-count calculation.
- **Integration tests** with `@SpringBootTest` + **Testcontainers** (real PostgreSQL) and MockMvc:
  - Triggering creates a row and returns id + `CREATED` status.
  - Retrieval returns the user's notifications, reverse-chronological and paged.
  - Unread-count returns the correct number and decrements after a mark-read.
  - Mark-read flips status, persists, and is reflected in the next count call.
  - Email is asserted with a mocked `JavaMailSender` (or GreenMail) so content matches the rendered template **without** hitting real SMTP.

---

## 14. Deliverables checklist

| Deliverable | Where it comes from |
|---|---|
| Database scripts | Flyway `V1__schema.sql` (tables/indexes) + `V2__seed_templates.sql` |
| API documentation | springdoc-generated Swagger UI + exported OpenAPI JSON |
| Test cases | Unit + integration suite covering send / read / unread |
| ≥4 pre-configured templates | Seeded for the four notification types |

---

## 15. AC traceability

| AC | Requirement | Satisfied by |
|---|---|---|
| AC1 | Trigger API (type, recipient, data → id + status) | §5 trigger endpoint, §6 lifecycle |
| AC2 | Templates with placeholders, ≥4 configured | §7 templates + Flyway seed |
| AC3 | Email via Gmail SMTP, non-blocking | §8 async after-commit listener + SMTP config |
| AC4 | In-app storage (type, message, timestamp, status, recipient) | §4 `notification` table |
| AC5 | Bell with unread count, updates and resets | §9 bell polling, §5 count endpoint |
| AC6 | Panel, reverse-chron, read/unread distinct | §9 panel, §5 retrieval, `created_at DESC` index |
| AC7 | Mark read/unread, persisted, count updates | §5 PATCH endpoints, §9 optimistic update |
| AC8 | Single reusable API for all modules | §10 integration contract |
| AC9 | Persist all data, integrity, history per user | §4 model + JSONB, §11 transactional store |
| AC10 | REST APIs documented in Swagger | §2 springdoc, §5 endpoints |
| AC11 | Responsive bell + panel | §9 responsive drawer/dropdown |
| AC12 | All deliverables complete | §13 tests, §14 checklist |

---

## 16. Suggested build order

1. **Schema first** — Flyway migrations for the three tables + seed the four templates.
2. **Domain + persistence** — entities and repositories.
3. **Template engine + trigger service + trigger API** — get an end-to-end in-app notification working and verifiable in the DB.
4. **Async email** — wire the after-commit listener and Gmail SMTP; confirm the API returns before email sends.
5. **Retrieval / read / unread-count APIs + Swagger** — finish the backend surface.
6. **React bell + panel** — polling, optimistic mark-read, responsive layout.
7. **Tests + docs + hardening** — integration suite, export OpenAPI, add auth and rate limiting.
