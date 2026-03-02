# ARC Digital Platform — Scalable K–9 School Management Architecture

## 1) System Architecture Overview

### 1.1 Vision and constraints
ARC Digital Platform is designed for a K–9 private school in Liberia migrating from paper/manual operations, while deliberately laying foundations for multi-school and national-level growth.

Design constraints:
- **Low bandwidth and intermittent connectivity** (2G/3G and unstable power).
- **Mobile-first usage** (teachers and parents mostly on Android phones).
- **Data privacy and student safety** are mandatory from day one.
- **Gradual digitization** path (staff transitioning from manual workflows).

### 1.2 Architecture style
Use a **modular monolith first**, with strict domain boundaries and API contracts, then split high-load domains into microservices when needed.

Core domains (logical modules):
1. Identity & Access (RBAC)
2. Enrollment & Student Records
3. Attendance
4. Grading & Report Cards
5. Class Scheduling
6. Parent-Teacher Messaging
7. Fees & Payment Logs
8. Analytics & Reporting
9. Notifications (SMS/Email/Push)

This keeps early operations simple but preserves future decomposability.

### 1.3 High-level component diagram (textual)
- **Client apps**
  - Mobile Web PWA (parents, teachers, students)
  - Admin web console (desktop + tablet)
- **API Gateway / BFF layer**
  - AuthN/AuthZ enforcement
  - Rate limiting
  - Request shaping and caching
- **Application layer** (modular domains above)
- **Data layer**
  - PostgreSQL (primary relational store)
  - Redis (session cache, queues, ephemeral state)
  - Object storage (report PDFs, document uploads)
- **Async layer**
  - Message queue / job runner for heavy tasks (report generation, bulk SMS)
- **Integration layer**
  - Payment providers (mobile money/bank import)
  - SMS gateway

### 1.4 Multi-tenant national scaling model
Implement **tenant isolation by school**:
- `tenant_id` in all business tables.
- Row-level security and tenant-aware query guards.
- Ability to move large tenants to dedicated database/schema later.

Scaling path:
- Phase 1: single DB with tenant partition keys.
- Phase 2: read replicas + table partitioning (attendance, messages, audit logs).
- Phase 3: service extraction (messaging, analytics) + per-region deployment.

### 1.5 Security architecture
- **Authentication**: email/phone + password, optional OTP for admins.
- **Authorization**: RBAC + school scoping + class-level permissions.
- **Data protection**:
  - TLS 1.2+ in transit.
  - Encryption at rest.
  - Passwords hashed with Argon2/bcrypt.
  - PII field minimization and strict access logs.
- **Auditability**:
  - `audit_log` table for critical actions (grade edits, fee adjustments, user role changes).
- **Resilience**:
  - Automated backups and point-in-time recovery.
  - Offline-friendly client queues for attendance and messaging drafts.

---

## 2) Database Schema (Core)

> All tables include `id (UUID)`, `tenant_id`, `created_at`, `updated_at` unless noted.

### 2.1 Identity and role-based access

#### `users`
- `id`, `tenant_id`
- `email` (nullable), `phone` (nullable, indexed)
- `password_hash`
- `status` (`active`, `inactive`, `locked`)
- `last_login_at`

#### `roles`
- `id`, `tenant_id`
- `name` (`admin`, `teacher`, `parent`, `student`, extensible)

#### `user_roles`
- `user_id`, `role_id` (many-to-many)

#### `permissions`
- `id`, `code` (e.g., `grade.write`, `attendance.take`)

#### `role_permissions`
- `role_id`, `permission_id`

### 2.2 Academic structure

#### `schools`
- `id`, `name`, `address`, `country`, `timezone`

#### `academic_years`
- `id`, `tenant_id`, `name`, `start_date`, `end_date`, `is_active`

#### `terms`
- `id`, `academic_year_id`, `name`, `start_date`, `end_date`

#### `classes`
- `id`, `tenant_id`, `name`, `grade_level`, `homeroom_teacher_id`

#### `subjects`
- `id`, `tenant_id`, `name`, `code`

#### `class_subjects`
- `class_id`, `subject_id`, `teacher_id`

#### `schedules`
- `id`, `class_id`, `subject_id`, `teacher_id`
- `day_of_week`, `start_time`, `end_time`, `room`

### 2.3 Student records and enrollment

#### `students`
- `id`, `tenant_id`, `user_id` (nullable until account creation)
- `admission_no` (unique per tenant)
- `first_name`, `last_name`, `dob`, `gender`
- `current_class_id`, `enrollment_status`

#### `parents`
- `id`, `tenant_id`, `user_id`
- `first_name`, `last_name`, `relationship_primary`

#### `student_parents`
- `student_id`, `parent_id`, `is_primary_contact`

#### `enrollments`
- `id`, `student_id`, `class_id`, `academic_year_id`, `term_id`
- `enrollment_date`, `status`

#### `student_documents`
- `id`, `student_id`, `doc_type`, `file_url`, `verified_by`

### 2.4 Attendance and grading

#### `attendance_sessions`
- `id`, `class_id`, `subject_id` (nullable for homeroom), `date`, `taken_by`

#### `attendance_records`
- `id`, `attendance_session_id`, `student_id`
- `status` (`present`, `absent`, `late`, `excused`)
- `remarks`

#### `assessments`
- `id`, `class_id`, `subject_id`, `term_id`
- `title`, `type` (`quiz`, `test`, `exam`, `assignment`), `max_score`, `weight`

#### `grades`
- `id`, `assessment_id`, `student_id`, `score`, `grade_letter`, `comment`
- `entered_by`, `approved_by`

#### `report_cards`
- `id`, `student_id`, `term_id`, `generated_at`, `generated_by`, `pdf_url`

### 2.5 Fees and payments

#### `fee_structures`
- `id`, `tenant_id`, `class_id`, `academic_year_id`, `term_id`
- `fee_type` (`tuition`, `transport`, `exam`, etc.), `amount`

#### `invoices`
- `id`, `student_id`, `term_id`, `total_amount`, `due_date`, `status`

#### `invoice_items`
- `id`, `invoice_id`, `fee_structure_id`, `amount`

#### `payments`
- `id`, `invoice_id`, `amount`, `payment_method`, `transaction_ref`
- `paid_at`, `received_by`, `reconciliation_status`

#### `payment_logs`
- `id`, `payment_id`, `event_type`, `payload_json`, `event_time`

### 2.6 Messaging and communication

#### `message_threads`
- `id`, `tenant_id`, `context_type` (`student`, `class`, `general`), `context_id`

#### `thread_participants`
- `thread_id`, `user_id`, `role`

#### `messages`
- `id`, `thread_id`, `sender_user_id`, `body`, `sent_at`, `is_read`

#### `notifications`
- `id`, `user_id`, `channel` (`sms`, `email`, `push`, `in_app`)
- `template_code`, `status`, `sent_at`

### 2.7 Governance and operations

#### `audit_logs`
- `id`, `tenant_id`, `actor_user_id`, `action`, `resource_type`, `resource_id`
- `old_values_json`, `new_values_json`, `ip_address`, `created_at`

#### `sync_events`
- for offline data sync conflict handling
- `id`, `device_id`, `entity_type`, `entity_id`, `operation`, `payload_json`, `status`

---

## 3) API Structure

### 3.1 API style
- **REST API v1** for operational simplicity.
- JSON responses, cursor-based pagination.
- Versioned routes: `/api/v1/...`
- Idempotency keys for payment and bulk operations.

### 3.2 Authentication and session endpoints
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout`
- `POST /api/v1/auth/refresh`
- `POST /api/v1/auth/request-otp`

### 3.3 Core domain endpoints (examples)

#### Enrollment & records
- `GET /api/v1/students`
- `POST /api/v1/students`
- `GET /api/v1/students/{id}`
- `PATCH /api/v1/students/{id}`
- `POST /api/v1/students/{id}/enrollments`
- `GET /api/v1/enrollments?class_id=&term_id=`

#### Attendance
- `POST /api/v1/attendance/sessions`
- `POST /api/v1/attendance/sessions/{id}/records`
- `GET /api/v1/attendance/students/{student_id}/summary`

#### Grading & reports
- `POST /api/v1/assessments`
- `POST /api/v1/assessments/{id}/grades`
- `PATCH /api/v1/grades/{id}`
- `POST /api/v1/reports/cards/generate`
- `GET /api/v1/reports/cards/{student_id}?term_id=`

#### Scheduling
- `GET /api/v1/schedules?class_id=`
- `POST /api/v1/schedules`
- `PATCH /api/v1/schedules/{id}`

#### Messaging
- `POST /api/v1/messages/threads`
- `GET /api/v1/messages/threads`
- `POST /api/v1/messages/threads/{id}/messages`

#### Fees & payments
- `POST /api/v1/fees/structures`
- `POST /api/v1/invoices/generate`
- `GET /api/v1/invoices?student_id=`
- `POST /api/v1/payments`
- `GET /api/v1/payments/logs?invoice_id=`

### 3.4 RBAC and policy matrix
- Admin: full access + user/role/configuration management.
- Teacher: class-bound attendance/grades/schedule + messaging with assigned parents/students.
- Parent: own children records, attendance summary, report cards, fee invoices, messages.
- Student: personal timetable, grades/report cards, attendance summary.

Enforce at:
1. Route guard (role-level).
2. Service layer policy (ownership/class assignment).
3. Query filters (`tenant_id`, relationship joins).

### 3.5 Reliability and observability
- Request tracing IDs.
- Structured logs (JSON).
- Metrics: p95 latency, failed logins, payment callback failure rate, message delivery success.
- Webhook signature validation for payment integrations.

---

## 4) Frontend Structure (Mobile-first)

### 4.1 Client strategy
- **Progressive Web App (PWA)** as primary client to minimize installation friction and data usage.
- Responsive design with touch-first navigation and compressed assets.
- Optional Android wrapper later (Trusted Web Activity / React Native shell).

### 4.2 Frontend modules
- `auth/` (login, password reset, OTP)
- `dashboard/`
  - `dashboard-admin`
  - `dashboard-teacher`
  - `dashboard-parent`
  - `dashboard-student`
- `students/` (records, enrollment)
- `attendance/`
- `grades/`
- `reports/`
- `schedule/`
- `fees/`
- `messages/`
- `settings/`

### 4.3 State and data loading patterns
- Query caching (stale-while-revalidate) for low bandwidth.
- Optimistic UI for attendance marking and messaging.
- Background sync queue for offline actions.
- Local storage encryption for sensitive cached data.

### 4.4 UX principles for school operations
- Role-specific home screens showing only top tasks.
- One-tap daily attendance flow.
- Parent dashboard with quick fee balance and child progress snapshot.
- Clear status badges (paid/unpaid, present/absent, published/draft).
- Localization-ready text architecture for future regional languages.

---

## 5) Recommended Tech Stack (Low-bandwidth optimized)

### 5.1 Backend
- **Language/framework**: Node.js + NestJS (or Fastify) for structured modular architecture.
- **Database**: PostgreSQL 15+.
- **Cache/queue**: Redis + BullMQ.
- **ORM**: Prisma/TypeORM (with explicit indexing and migration discipline).
- **Auth**: JWT access + refresh tokens; optional OTP provider.

### 5.2 Frontend
- **Framework**: Next.js (App Router) or React + Vite PWA.
- **UI**: Tailwind CSS (small bundle strategy) + accessible component primitives.
- **Data fetching**: TanStack Query.
- **PWA**: Workbox for caching/sync.

### 5.3 Infra/DevOps
- Containerized deployment (Docker).
- Reverse proxy: Nginx.
- CI/CD: GitHub Actions.
- Hosting path:
  - Early stage: single-region VPS/managed container.
  - Growth: Kubernetes (managed) + autoscaling + managed Postgres.
- Monitoring: Prometheus/Grafana + Sentry.

### 5.4 Cost and connectivity optimization
- Image/document compression at upload.
- Gzip/Brotli and aggressive static caching.
- SMS fallback for critical notifications where push/data fails.
- Batch writes for attendance sync when network returns.

---

## 6) Why these design decisions

1. **Modular monolith first** reduces delivery risk for a transitioning school while preserving a clean path to microservices.
2. **Tenant-aware data model** enables scaling from one school to many without redesign.
3. **PWA-first approach** fits Liberia’s mobile and bandwidth realities better than heavy native apps.
4. **RBAC + audit logging** protects children’s records and supports compliance/governance.
5. **Async jobs and queueing** prevent slow report generation or bulk messaging from degrading user experience.
6. **Offline-capable workflows** are essential for unreliable connectivity in classrooms.
7. **Payment logging and reconciliation fields** prepare the platform for formal finance controls as institutions scale.

---

## 7) Role-specific dashboard blueprint

### Admin dashboard
- Enrollment funnel, attendance rate, fee collection, overdue invoices.
- Staff actions queue (pending approvals, report publication status).
- School calendar and timetable conflicts.

### Teacher dashboard
- Today’s classes.
- Attendance pending list.
- Assessments awaiting grading.
- Parent messages requiring response.

### Parent dashboard
- Child attendance trend.
- Latest grades/report card status.
- Outstanding balance and payment history.
- Messages/announcements.

### Student dashboard
- Personal timetable.
- Assignment/assessment highlights.
- Attendance summary.
- Published results and teacher remarks.

---

## 8) National product readiness roadmap

### Phase A (single school foundation)
- Core SIS features + billing + messaging.
- PWA rollout and staff training.

### Phase B (multi-school expansion)
- Tenant admin portal.
- School onboarding wizard and data import templates.
- Regional reporting dashboards.

### Phase C (national ecosystem)
- Government/regulator reporting adapters.
- Learning analytics and early warning indicators.
- Public API partnerships (content providers, payment rails, identity systems).

This phased strategy keeps implementation practical for current school needs while structurally ready for national EdTech adoption.
