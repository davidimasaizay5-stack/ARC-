# ARC Digital Platform — Scalable K–9 School Management System (Liberia)

## 1) Vision and Product Scope

ARC Digital Platform is a **mobile-first, low-bandwidth-ready, secure school operating system** for K–9 private schools moving from paper-based workflows to digital operations.

It is designed in two stages:

- **Stage 1 (Single School Modernization):** Replace manual enrollment, attendance, gradebooks, report cards, fee tracking, and parent communication.
- **Stage 2 (National EdTech Scale):** Expand into a **multi-tenant education platform** where each school, district, or ministry unit can operate independently on shared infrastructure.

Core principles:

1. **Offline-first and bandwidth-efficient UX** for low-connectivity contexts.
2. **Role-based data isolation and auditability** for trust and compliance.
3. **Modular architecture** so new services (LMS, exams, national reporting) can be added without rewrites.
4. **Observability + resilience** for production-grade scaling.

---

## 2) Role Model and Access Control

### Roles

- **Admin** (school leadership, bursar, registrar)
- **Teacher**
- **Parent/Guardian**
- **Student**

### Authorization Strategy

Use **RBAC + contextual constraints**:

- RBAC grants feature-level permissions (`attendance.write`, `grades.publish`, `fees.read`).
- Contextual rules restrict data scope:
  - Teachers: only assigned classes/subjects.
  - Parents: only linked children.
  - Students: only own records.
  - Admins: school-wide access (or scoped by campus in multi-campus setups).

### Security Controls

- JWT access tokens + rotating refresh tokens.
- Password hashing with Argon2id/Bcrypt.
- 2FA optional for Admin and finance actions.
- Full audit logs for grade changes, payment edits, and student record updates.
- Row-level filtering in API queries (never trust client role claims).

---

## 3) System Architecture Overview

## 3.1 High-Level Components

1. **Mobile Web/PWA Frontend**
   - Parent, teacher, student, and admin portals.
   - Service worker cache for offline forms and read-mostly pages.

2. **API Gateway / Backend-for-Frontend (BFF)**
   - Request routing, auth validation, throttling, versioning.
   - Aggregates data for dashboards to reduce client round trips.

3. **Core Domain Services**
   - Identity & Access Service
   - Enrollment & Student Records Service
   - Attendance Service
   - Grading & Report Service
   - Scheduling Service
   - Messaging Service
   - Fees & Payments Service
   - Notification Service (SMS/Email/Push)

4. **Data Layer**
   - PostgreSQL (primary transactional DB)
   - Redis (session cache, rate-limit counters, queue state)
   - Object storage (report PDFs, profile images)

5. **Asynchronous Infrastructure**
   - Message queue (RabbitMQ/SQS/Kafka-lite) for report generation, SMS dispatch, reconciliation jobs.

6. **Monitoring & Operations**
   - Centralized logs, metrics, tracing, alerting dashboards.

## 3.2 Deployment Model

- **Containerized services** (Docker) deployed via Kubernetes or managed container platform.
- Begin as a **modular monolith** (faster delivery), evolve to microservices for bottleneck domains (messaging, analytics, payments).
- Multi-tenant-ready via `tenant_id` in core tables + tenant-aware middleware.

## 3.3 Data Flow Examples

### Attendance Submission
Teacher app -> API Gateway -> Attendance Service -> PostgreSQL -> Event (`attendance.marked`) -> Notification Service (optional parent SMS).

### Report Card Generation
Teacher finalizes grades -> Grading Service stores computed results -> Async job generates PDF -> Object storage -> Parent/Student dashboard retrieves signed URL.

### Payment Log Update
Admin enters payment -> Fees Service writes ledger entry -> Event published -> Parent notification -> Dashboard balance recomputed.

---

## 4) Database Schema (Core Relational Design)

Below is a scalable baseline schema (PostgreSQL).

## 4.1 Identity & Tenant

- `tenants` (`id`, `name`, `type`, `status`, `created_at`)
- `users` (`id`, `tenant_id`, `first_name`, `last_name`, `phone`, `email`, `password_hash`, `status`, `last_login_at`, `created_at`)
- `roles` (`id`, `name`) — Admin, Teacher, Parent, Student
- `user_roles` (`id`, `user_id`, `role_id`, `tenant_id`)
- `audit_logs` (`id`, `tenant_id`, `actor_user_id`, `action`, `entity_type`, `entity_id`, `before_json`, `after_json`, `ip_address`, `created_at`)

Indexes:

- `users(tenant_id, phone)` unique where phone not null.
- `users(tenant_id, email)` unique where email not null.
- `audit_logs(tenant_id, created_at)` for compliance reporting.

## 4.2 Student & Enrollment

- `students` (`id`, `tenant_id`, `admission_no`, `first_name`, `last_name`, `dob`, `gender`, `status`, `enrollment_date`, `photo_url`)
- `guardians` (`id`, `tenant_id`, `user_id`, `relationship_type`, `address`, `preferred_contact_method`)
- `student_guardians` (`id`, `student_id`, `guardian_id`, `is_primary`)
- `academic_years` (`id`, `tenant_id`, `name`, `start_date`, `end_date`, `is_active`)
- `terms` (`id`, `tenant_id`, `academic_year_id`, `name`, `start_date`, `end_date`)
- `classes` (`id`, `tenant_id`, `name`, `grade_level`, `homeroom_teacher_id`, `capacity`)
- `enrollments` (`id`, `tenant_id`, `student_id`, `class_id`, `academic_year_id`, `status`, `enrolled_at`, `withdrawn_at`)

Indexes:

- `students(tenant_id, admission_no)` unique.
- `enrollments(tenant_id, academic_year_id, class_id)` for class rosters.

## 4.3 Attendance

- `attendance_sessions` (`id`, `tenant_id`, `class_id`, `term_id`, `session_date`, `recorded_by_user_id`, `created_at`)
- `attendance_records` (`id`, `tenant_id`, `attendance_session_id`, `student_id`, `status`, `remarks`)
  - `status` in {present, absent, late, excused}

Indexes:

- `attendance_records(tenant_id, student_id, attendance_session_id)` unique.
- `attendance_sessions(tenant_id, class_id, session_date)`.

## 4.4 Grading & Reports

- `subjects` (`id`, `tenant_id`, `name`, `code`, `grade_level`)
- `class_subjects` (`id`, `tenant_id`, `class_id`, `subject_id`, `teacher_id`)
- `assessments` (`id`, `tenant_id`, `class_subject_id`, `term_id`, `title`, `type`, `max_score`, `weight`, `scheduled_date`)
- `grades` (`id`, `tenant_id`, `assessment_id`, `student_id`, `raw_score`, `grade_letter`, `comment`, `entered_by_user_id`, `entered_at`)
- `report_cards` (`id`, `tenant_id`, `student_id`, `term_id`, `gpa`, `position_in_class`, `teacher_comment`, `principal_comment`, `pdf_url`, `published_at`)

Indexes:

- `grades(tenant_id, assessment_id, student_id)` unique.
- `report_cards(tenant_id, student_id, term_id)` unique.

## 4.5 Scheduling

- `time_slots` (`id`, `tenant_id`, `day_of_week`, `start_time`, `end_time`)
- `class_timetables` (`id`, `tenant_id`, `class_id`, `term_id`, `time_slot_id`, `subject_id`, `teacher_id`, `room`) 

Constraints:

- No teacher double-booking: unique over (`tenant_id`, `teacher_id`, `term_id`, `time_slot_id`).
- No class double-booking: unique over (`tenant_id`, `class_id`, `term_id`, `time_slot_id`).

## 4.6 Messaging

- `conversations` (`id`, `tenant_id`, `type`, `created_by_user_id`, `created_at`)
- `conversation_participants` (`id`, `conversation_id`, `user_id`, `joined_at`)
- `messages` (`id`, `tenant_id`, `conversation_id`, `sender_user_id`, `body`, `sent_at`, `read_at`)

Indexes:

- `messages(conversation_id, sent_at desc)`.

## 4.7 Fees & Payment Logs

- `fee_structures` (`id`, `tenant_id`, `academic_year_id`, `class_id`, `name`, `amount`, `currency`, `due_date`)
- `student_fee_assignments` (`id`, `tenant_id`, `student_id`, `fee_structure_id`, `amount_due`, `discount_amount`, `status`)
- `payments` (`id`, `tenant_id`, `student_id`, `received_by_user_id`, `payment_method`, `reference_no`, `amount_paid`, `currency`, `paid_at`, `notes`)
- `payment_allocations` (`id`, `tenant_id`, `payment_id`, `student_fee_assignment_id`, `amount_allocated`)

Indexes:

- `payments(tenant_id, student_id, paid_at desc)`.
- `payments(tenant_id, reference_no)` unique where reference_no not null.

---

## 5) API Structure (Versioned, Role-Aware)

Base path: `/api/v1`

## 5.1 Auth & User

- `POST /auth/login`
- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /me`
- `GET /users/:id` (admin-scope)

## 5.2 Enrollment & Records

- `POST /students`
- `GET /students?class_id=&status=&q=`
- `GET /students/:id`
- `PATCH /students/:id`
- `POST /enrollments`
- `GET /classes/:id/roster`

## 5.3 Attendance

- `POST /attendance/sessions`
- `POST /attendance/sessions/:id/records`
- `GET /attendance/students/:student_id?term_id=`
- `GET /attendance/classes/:class_id/summary?term_id=`

## 5.4 Grading & Reports

- `POST /assessments`
- `POST /assessments/:id/grades/bulk`
- `GET /students/:id/grades?term_id=`
- `POST /reports/generate` (async)
- `GET /reports/:student_id/:term_id`
- `POST /reports/:id/publish`

## 5.5 Scheduling

- `POST /timetables`
- `GET /classes/:id/timetable?term_id=`
- `GET /teachers/:id/timetable?term_id=`

## 5.6 Messaging

- `POST /conversations`
- `GET /conversations`
- `POST /conversations/:id/messages`
- `GET /conversations/:id/messages?cursor=`

## 5.7 Fees & Payments

- `POST /fees/structures`
- `POST /fees/assignments`
- `GET /students/:id/fees/ledger`
- `POST /payments`
- `GET /payments?student_id=&from=&to=`

## 5.8 Dashboards (BFF Aggregates)

- `GET /dashboards/admin`
- `GET /dashboards/teacher`
- `GET /dashboards/parent`
- `GET /dashboards/student`

Each dashboard endpoint returns pre-aggregated widgets optimized for low-latency mobile rendering.

---

## 6) Frontend Structure (Mobile-First)

Recommended client: **PWA + responsive UI**, installable on Android devices used by teachers/parents.

## 6.1 App Shell and Modules

- `/auth` — login, reset, session handling
- `/admin`
  - Enrollment
  - Student Records
  - Class Setup
  - Fees & Payment Logs
  - Reports & Analytics
- `/teacher`
  - My Classes
  - Attendance Capture
  - Gradebook
  - Timetable
  - Parent Messaging
- `/parent`
  - Child Summary
  - Attendance & Performance
  - Fees Ledger
  - Messages
- `/student`
  - Timetable
  - Attendance
  - Grades/Report Cards

## 6.2 UX Patterns for Low Bandwidth

- Text-first views with progressive enhancement.
- Skeleton loaders + retry states.
- Store-and-forward forms (attendance/grades cached locally until connectivity returns).
- Compressed API payloads (gzip/br), pagination, and field-select support.
- Image upload resizing on-device before submit.

## 6.3 State/Data Strategy

- Query caching (TanStack Query/RTK Query).
- Optimistic updates for message sending and attendance toggles.
- Role-based route guards from server-provided claims.

---

## 7) Recommended Tech Stack (Low-Bandwidth Optimized)

## Backend

- **Language/Framework:** Node.js (NestJS or Fastify) or Go (Fiber/Gin).
  - For faster team onboarding in West African startup ecosystems, Node/NestJS is often pragmatic.
- **Database:** PostgreSQL.
- **Cache/Queue:** Redis + BullMQ (or RabbitMQ).
- **Auth:** JWT + refresh tokens.
- **Report generation:** headless HTML-to-PDF worker.

## Frontend

- **Web:** React + Vite + TypeScript, PWA enabled.
- **UI:** Tailwind CSS or lightweight component system.
- **Forms:** React Hook Form + Zod validation.

## Infrastructure

- Dockerized services.
- Nginx/Cloudflare edge caching for static assets.
- Object storage compatible with S3.
- CI/CD via GitHub Actions.

## Communications

- SMS integration (local telecom aggregator) for attendance alerts, fee reminders, and one-time codes.
- Optional WhatsApp integration for future parental engagement.

---

## 8) Role-Specific Dashboards

## Admin Dashboard

- Total enrollment, daily attendance rate, unpaid fees summary.
- Recent payments and defaulters list.
- Pending report publication and teacher completion status.
- Quick actions: add student, record payment, publish reports.

## Teacher Dashboard

- Today’s classes and timetable.
- Attendance pending classes.
- Assessment marking progress.
- Message inbox (parents/admin).

## Parent Dashboard

- Child attendance this term.
- Latest grades/report card availability.
- Outstanding balance + payment history.
- Message thread with class teacher.

## Student Dashboard

- Today’s timetable.
- Attendance summary.
- Subject performance snapshots.
- Announcements.

---

## 9) Scalability Path to National Product

1. **Tenant Isolation:** add strict tenant scoping in every table and query.
2. **Regional Deployment:** deploy in-country or region-near cloud zones for latency and data policy needs.
3. **Configurable Academic Models:** support different term structures, grading scales, and fee models per school/region.
4. **Pluggable Integrations:** national exam APIs, mobile money providers, ministry data exports.
5. **Analytics Warehouse:** replicate transactional data into warehouse for ministry-level KPI dashboards.
6. **Event-Driven Expansion:** stream domain events for real-time alerts and large-scale analytics.

---

## 10) Key Design Decisions and Rationale

1. **Modular monolith first, microservices later**
   - Reduces early complexity while preserving clear domain boundaries for future extraction.

2. **PostgreSQL as source of truth**
   - Strong consistency required for grades, payments, and student identity.

3. **PWA over native-first**
   - Faster rollout, lower maintenance cost, broad Android compatibility, and acceptable offline behavior.

4. **Asynchronous jobs for heavy operations**
   - PDF generation and bulk notifications are decoupled to keep user-facing APIs responsive.

5. **RBAC + contextual authorization**
   - Prevents both horizontal and vertical privilege misuse in sensitive school data.

6. **Low-bandwidth optimization as baseline**
   - Compression, pagination, caching, and offline queues directly address Liberia connectivity realities.

7. **Auditability for trust**
   - Immutable logs for grade/payment changes support governance, parent trust, and regulatory readiness.

---

## 11) Suggested Delivery Roadmap

- **Phase 1 (8–12 weeks):** Auth, enrollment, attendance, basic gradebook, admin/teacher dashboards.
- **Phase 2 (6–8 weeks):** Reports, parent portal, messaging, fee ledger and payments logging.
- **Phase 3 (6–10 weeks):** Multi-tenant hardening, analytics, integrations (SMS/mobile money), national pilot readiness.

