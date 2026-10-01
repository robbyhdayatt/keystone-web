# SYSTEM-DESIGN: Keystone

> Technical blueprint. Implements the requirements in `PRD.md`.
> Two separate private repos: `keystone-api` (backend) and `keystone-web` (frontend). **No monorepo.** Copy the `docs/` folder into both repos.

---

## 1. Architecture Overview

```
┌──────────────────────┐   HTTPS + Bearer JWT   ┌──────────────────────────────┐
│ keystone-web         │ ─────────────────────▶ │ keystone-api                 │
│ Next.js 16 (Vercel)  │ ◀───────────────────── │ Bun + Hono (Railway/Render)  │
│ TanStack Query/Axios │      JSON envelope     │ Prisma ─▶ PostgreSQL         │
└──────────────────────┘                        └──────────────────────────────┘
```

**Backend layering** (strict, one direction):

```
Route (Hono) → Middleware (auth, rbac) → Controller (parse/validate/respond)
            → Service (business rules, permissions, transactions)
            → Repository (Prisma queries only)
            → Serializer (role-aware DTO / masking)  ← response is ALWAYS built here
```

Rules of thumb:
- Controllers never touch Prisma. Repositories never contain business rules.
- **Permissions and state rules live in the service layer** (via a `policy` module), so they are unit-testable and cannot be bypassed by a new route.
- **Responses are never returned raw from Prisma.** Always through a serializer.

---

## 2. Repository Structure

### 2.1 Backend `keystone-api`
```
src/
  index.ts                 # Bun entry, Hono app bootstrap
  app.ts                   # app composition, middleware, error handler
  config/env.ts            # Zod-validated env
  lib/
    prisma.ts              # Prisma client + soft-delete extension
    jwt.ts                 # sign/verify
    password.ts            # hash/verify
    errors.ts              # AppError classes + codes
    response.ts            # envelope helpers
    storage/               # StorageProvider (local, s3)
  middleware/
    auth.ts                # verify JWT, check denylist, set c.var.user
    requireRole.ts
    errorHandler.ts
  modules/
    auth/        (routes, controller, service, dto)
    users/
    projects/    (+ membership, summary)
    tasks/       (routes, controller, service, repository, policy, serializer, dto)
    dependencies/(service with cycle detection)
    attachments/
    comments/
    audit/       (service: record(), query)
    standup/
  policies/
    taskPolicy.ts          # canView, canEdit, canTransition
    projectPolicy.ts
prisma/
  schema.prisma
  migrations/              # includes SQL migration for audit append-only trigger
  seed.ts
tests/                     # unit tests (bun test)
docs/                      # PRD, SYSTEM-DESIGN, UI-UX, TASK-BREAKDOWN
```

### 2.2 Frontend `keystone-web`
```
src/
  app/                     # App Router
    (auth)/login, register
    (app)/projects, projects/[projectId], projects/[projectId]/audit, projects/[projectId]/standup
    layout.tsx, providers.tsx
  components/ui/           # shadcn
  components/              # TaskBoard, TaskCard, TaskDrawer, ConflictDialog, StatusButton, ...
  features/                # auth, projects, tasks, audit, standup (hooks + api + schemas)
  lib/axios.ts             # instance, auth header, 401/409 interceptors
  lib/permissions.ts       # UI mirror of rules (UX only, never security)
  stores/auth.store.ts     # Zustand (persist)
  styles/tokens.css        # brand tokens
```

---

## 3. Data Model (Prisma)

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

enum Role        { PM INTERNAL CLIENT }
enum Department  { PM UIUX FRONTEND BACKEND }
enum TaskStatus  { TODO IN_PROGRESS DONE }
enum AuditAction { CREATE UPDATE DELETE RESTORE }

model User {
  id           String      @id @default(cuid())
  email        String
  name         String
  passwordHash String
  avatarUrl    String?
  role         Role
  department   Department?              // null for CLIENT
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
  deletedAt    DateTime?

  memberships  ProjectMember[]
  assignedTasks Task[]     @relation("TaskAssignee")
  // NOTE: unique email among non-deleted rows → partial unique index in raw SQL migration:
  // CREATE UNIQUE INDEX user_email_active_key ON "User"(lower(email)) WHERE "deletedAt" IS NULL;
}

model Project {
  id          String   @id @default(cuid())
  name        String
  description String?
  createdById String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  deletedAt   DateTime?

  members ProjectMember[]
  tasks   Task[]
}

model ProjectMember {          // links INTERNAL and CLIENT users to projects (PM sees all)
  id        String   @id @default(cuid())
  projectId String
  userId    String
  createdAt DateTime @default(now())
  deletedAt DateTime?
  project   Project  @relation(fields: [projectId], references: [id])
  user      User     @relation(fields: [userId], references: [id])
  @@unique([projectId, userId])
  @@index([userId])
}

model Task {
  id            String     @id @default(cuid())
  projectId     String
  title         String
  description   String?
  status        TaskStatus @default(TODO)
  department    Department               // owning discipline
  assigneeId    String?
  clientVisible Boolean    @default(false)
  dueDate       DateTime?
  version       Int        @default(1)   // optimistic lock
  createdById   String
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt
  completedAt   DateTime?
  deletedAt     DateTime?

  project   Project @relation(fields: [projectId], references: [id])
  assignee  User?   @relation("TaskAssignee", fields: [assigneeId], references: [id])
  prerequisites TaskDependency[] @relation("Dependent")    // tasks I depend on
  dependents    TaskDependency[] @relation("Prerequisite") // tasks that depend on me
  comments      Comment[]
  attachments   Attachment[]

  @@index([projectId, status])
  @@index([projectId, deletedAt])
  @@index([assigneeId])
}

model TaskDependency {
  id          String   @id @default(cuid())
  taskId      String                // the dependent (blocked) task
  dependsOnId String                // the prerequisite
  createdById String
  createdAt   DateTime @default(now())
  deletedAt   DateTime?
  task        Task @relation("Dependent",    fields: [taskId],      references: [id])
  dependsOn   Task @relation("Prerequisite", fields: [dependsOnId], references: [id])
  @@unique([taskId, dependsOnId])
  @@index([dependsOnId])
}

model Comment {                 // INTERNAL ONLY, never serialized for CLIENT
  id        String   @id @default(cuid())
  taskId    String
  authorId  String
  body      String
  createdAt DateTime @default(now())
  deletedAt DateTime?
  task      Task @relation(fields: [taskId], references: [id])
}

model Attachment {
  id           String   @id @default(cuid())
  taskId       String
  uploadedById String
  fileName     String
  mimeType     String
  sizeBytes    Int
  storageKey   String
  createdAt    DateTime @default(now())
  deletedAt    DateTime?
  task         Task @relation(fields: [taskId], references: [id])
}

model AuditLog {                // APPEND-ONLY (DB trigger enforces)
  id         String      @id @default(cuid())
  projectId  String?
  entityType String                  // "TASK" | "PROJECT" | "DEPENDENCY" | "ATTACHMENT" | ...
  entityId   String
  action     AuditAction
  field      String?                 // changed column; null for CREATE/DELETE of whole entity
  oldValue   Json?
  newValue   Json?
  userId     String                  // actor
  createdAt  DateTime    @default(now())
  @@index([entityType, entityId])
  @@index([projectId, createdAt])
}

model RevokedToken {            // logout denylist
  jti       String   @id
  expiresAt DateTime
  @@index([expiresAt])
}
```

**Append-only trigger** (raw SQL in a migration):
```sql
CREATE OR REPLACE FUNCTION audit_log_immutable() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'AuditLog is append-only';
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER audit_log_no_update_delete
BEFORE UPDATE OR DELETE ON "AuditLog"
FOR EACH ROW EXECUTE FUNCTION audit_log_immutable();
```

---

## 4. Authentication & Sessions

- `POST /auth/login`: verify password → sign JWT `{ sub, role, department, jti }`, expiry 24h → return `{ token, user }`.
- Library: `jsonwebtoken`. Secret from `JWT_SECRET`.
- `auth` middleware: read `Authorization: Bearer`, verify, check `RevokedToken` for `jti`, load user (not soft-deleted), set `c.var.user`.
- `POST /auth/logout`: insert `jti` + `exp` into `RevokedToken`. Periodic cleanup of expired rows (on login or cron).
- Passwords: argon2id or bcrypt (cost ≥ 10). Never return `passwordHash`.
- **Frontend:** token in Zustand (persisted). Axios interceptor adds the header; on `401` clears the store and redirects to `/login`. A route guard component protects `(app)` pages; optionally also a small session cookie for Next middleware redirects.

---

## 5. Authorization Model (RBAC + ABAC + State)

Evaluate in this order inside the service layer; fail fast with a specific error code.

1. **Authenticated?** (401)
2. **Role gate (RBAC):** route-level `requireRole(...)`.
3. **Tenant / membership gate (ABAC):** PM → all projects. INTERNAL/CLIENT → must be an active `ProjectMember`, else **404** (do not reveal existence).
4. **Attribute gate (ABAC):** department match, assignee match.
5. **State gate:** current `status`, `isBlocked`, transition matrix.
6. **Concurrency gate:** `version` check (inside the transaction).

### 5.1 Permission matrix

| Action | PM | INTERNAL | CLIENT |
|---|---|---|---|
| List projects | all | member only | member only (masked) |
| Create/edit/archive project, manage members | ✅ | ❌ | ❌ |
| View project summary | ✅ full | ✅ full | ✅ aggregate only |
| List/view tasks | ✅ all | ✅ in member projects | ✅ `clientVisible` only, masked |
| Create/edit task fields (title, description, assignee, dept, due, clientVisible) | ✅ | ❌ | ❌ |
| Set dependencies | ✅ | ❌ | ❌ |
| Change status | per matrix (never IN_PROGRESS→DONE) | per matrix | ❌ |
| Upload attachment | ✅ | ✅ (member project) | ❌ |
| Soft-delete attachment/task/project | ✅ | ❌ | ❌ |
| Comments (internal) | ✅ | ✅ | ❌ |
| Audit logs | ✅ | task history of own tasks (P1) | ❌ |
| Standup summary | ✅ | ✅ | ❌ |

---

## 6. Task State Machine

Stored statuses: `TODO`, `IN_PROGRESS`, `DONE`. **Blocked is derived** (`isBlocked`).

| From → To | PM | INTERNAL | Extra conditions | Error on failure |
|---|---|---|---|---|
| TODO → IN_PROGRESS | ✅ | ✅ | not blocked; internal: dept match, unassigned or self (auto-assign) | `TASK_BLOCKED` (422), `FORBIDDEN_DEPARTMENT` (403) |
| IN_PROGRESS → DONE | ❌ | ✅ | **must be assignee** | `PM_CANNOT_COMPLETE` (403), `NOT_ASSIGNEE` (403) |
| IN_PROGRESS → TODO | ✅ | ✅ (assignee) | pause/unclaim | `FORBIDDEN` |
| DONE → IN_PROGRESS | ✅ | ✅ (assignee) | reopen; not blocked | `TASK_BLOCKED` |
| any other | ❌ | ❌ | | `INVALID_TRANSITION` (422) |

Implementation sketch (`policies/taskPolicy.ts`):

```ts
export function assertCanTransition(ctx: {
  user: AuthUser; task: TaskWithMeta; to: TaskStatus; isMember: boolean;
}) {
  const { user, task, to } = ctx;
  if (user.role === 'CLIENT') throw forbidden();
  const allowed = TRANSITIONS[task.status]?.includes(to);
  if (!allowed) throw new AppError(422, 'INVALID_TRANSITION');

  if (task.status === 'IN_PROGRESS' && to === 'DONE') {
    if (user.role === 'PM') throw new AppError(403, 'PM_CANNOT_COMPLETE');
    if (task.assigneeId !== user.id) throw new AppError(403, 'NOT_ASSIGNEE');
  }
  if (to === 'IN_PROGRESS' && task.isBlocked)
    throw new AppError(422, 'TASK_BLOCKED', { blockedBy: task.blockedBy });
  if (user.role === 'INTERNAL') {
    if (!ctx.isMember) throw notFound();
    if (task.department !== user.department) throw new AppError(403, 'FORBIDDEN_DEPARTMENT');
    if (task.assigneeId && task.assigneeId !== user.id) throw new AppError(403, 'NOT_ASSIGNEE');
  }
}
```

---

## 7. Dependencies

- Table `TaskDependency(taskId → dependsOnId)`.
- **Set operation:** `PUT /tasks/:id/dependencies { dependsOnIds: string[], version }` replaces the set in one transaction (diff → create/soft-delete rows, audit each add/remove, bump task version).
- **Validation:** same project, not self, no cycles. Cycle check: DFS/BFS from each new prerequisite following *its* prerequisites; if we reach `taskId`, reject with `DEPENDENCY_CYCLE`.
- **isBlocked computation:** one query for a page of tasks:
  ```sql
  SELECT d."taskId", array_agg(p.id) AS blockers
  FROM "TaskDependency" d JOIN "Task" p ON p.id = d."dependsOnId"
  WHERE d."deletedAt" IS NULL AND p."deletedAt" IS NULL AND p.status <> 'DONE'
    AND d."taskId" = ANY($1) GROUP BY d."taskId";
  ```
  (or Prisma equivalent). Merge into task DTOs: `isBlocked`, `blockedBy: [{id, title, status}]`.
- **Backend enforcement:** the status service recomputes blockers **inside the transaction** (`SELECT ... FOR SHARE` on prerequisites or a re-query) before allowing `→ IN_PROGRESS`.
- Reopening a prerequisite makes dependents in `TODO` blocked automatically (derived). See PRD A8 for in-progress dependents.

---

## 8. Concurrency: Optimistic Locking

Every mutating task endpoint takes `version` in the body (or `If-Match` header; pick one and be consistent: **body `version`**).

```ts
// repository
const res = await tx.task.updateMany({
  where: { id, version: input.version, deletedAt: null },
  data: { ...changes, version: { increment: 1 } },
});
if (res.count === 0) {
  const current = await tx.task.findFirst({ where: { id, deletedAt: null } });
  if (!current) throw notFound();
  throw new AppError(409, 'VERSION_CONFLICT', { current: serialize(current) });
}
```

Properties:
- Check-and-update is **atomic** (single SQL `UPDATE ... WHERE version = ?`), so there is no read-then-write race.
- Applies to: field edits, status changes, dependency changes, soft delete.
- 409 body includes the latest state so the UI can offer "reload" or "re-apply my edit".
- Example scenario: PM and Engineer both read `version=3`. Engineer → Done (v4). PM saves description with `version=3` → **409**, nothing overwritten.
- Test (must exist): fire two concurrent updates with `Promise.all`; exactly one 200 and one 409.

---

## 9. Audit Trail & Soft Delete

### 9.1 Audit
`audit.record(tx, { userId, projectId, entityType, entityId, action, changes })` where `changes` is a list of `{ field, old, new }`. One `AuditLog` row **per changed field**.

- Diff helper compares old vs new for tracked fields: `title, description, status, assigneeId, department, clientVisible, dueDate, deletedAt`, plus dependency add/remove (`field: "dependencies"`).
- Always called **inside the same transaction** as the mutation. If the audit write fails, the mutation rolls back.
- Never exposes update/delete endpoints. DB trigger enforces immutability.
- Values stored as JSON (`oldValue`/`newValue`). For assignee changes, store ids (and optionally names snapshot in a `meta` field if needed).
- Query: `GET /projects/:id/audit-logs` and `GET /tasks/:id/history` (paginated, PM-only; filter by user, field, date).

### 9.2 Soft delete
- `deletedAt` on every model.
- Prisma Client Extension (`$extends` query hook) injects `deletedAt: null` into `findMany/findFirst/count/...` for soft-deletable models. Provide an explicit `prisma.$unfiltered` escape hatch for audit/admin reads.
- "Delete" endpoints set `deletedAt = now()` (and bump version for tasks) and write an audit row (`action: DELETE`).
- Deleting a task also soft-deletes its dependency edges so dependents are not blocked forever by a deleted prerequisite (BR-10 ignores deleted prerequisites).

---

## 10. Data Masking for Client Guest

**Principle: whitelist at the serializer, never blacklist.**

```ts
// modules/tasks/serializer.ts
export const ClientTaskDTO = z.object({
  id: z.string(),
  title: z.string(),
  description: z.string().nullable(),
  status: z.enum(['TODO','IN_PROGRESS','DONE']),
  dueDate: z.string().nullable(),
}).strict();

export function serializeTask(task, viewer) {
  if (viewer.role === 'CLIENT') return ClientTaskDTO.parse(pick(task, [...]));
  return InternalTaskDTO.parse(toInternal(task));   // includes assignee {id,name,avatar,department}, isBlocked, blockedBy, version...
}
```

Client guarantees:
- Query is constrained at the **database level**: `where: { projectId, clientVisible: true, deletedAt: null }` plus membership check.
- Excluded fields: assignee, avatar, department, comments, attachments, audit, dependencies, `version`, `createdById`.
- **Filter/search/sort whitelist per role.** For CLIENT only `title` (search), `status` (filter), `dueDate` (sort). Reject others with 400. This prevents inference attacks such as `?filter=assignee.name:John`.
- Summary for client: `{ totalTasks, doneTasks, percentComplete, byStatus }`; no names, no departments.
- Test: automated test serializes a fixture with every internal field populated and asserts the client JSON contains none of those keys/strings.

---

## 11. API Specification

### 11.1 Conventions
- Base path: `/api/v1`. JSON only.
- **Envelope:**
  ```json
  { "success": true, "message": "OK", "data": {}, "meta": { "page": 1, "pageSize": 10, "total": 42, "totalPages": 5 } }
  ```
  Errors: `{ "success": false, "message": "...", "code": "VERSION_CONFLICT", "errors": [...], "details": {...} }`
- **Lists / filtering / search / pagination:** use `@nodewave/prisma-ezfilter` and follow the **NodeWave "Filtering, Pagination & Searching – Standard Documentation"** (link in the brief). **Agent TODO:** read that doc and the package README first and match the exact query param names; this spec intentionally does not guess them. Wrap ezfilter behind one `buildListQuery(config, role)` helper that applies the per-role field whitelist.
- Validation errors: `400` with Zod issues. Unknown task/project: `404`.

### 11.2 Error codes
| HTTP | code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | bad payload/query |
| 401 | `UNAUTHENTICATED` / `TOKEN_REVOKED` | missing/invalid token |
| 403 | `FORBIDDEN`, `FORBIDDEN_DEPARTMENT`, `NOT_ASSIGNEE`, `PM_CANNOT_COMPLETE` | permission failures |
| 404 | `NOT_FOUND` | missing or not visible to the viewer |
| 409 | `VERSION_CONFLICT` | stale `version` |
| 422 | `TASK_BLOCKED`, `INVALID_TRANSITION`, `DEPENDENCY_CYCLE`, `CROSS_PROJECT_DEPENDENCY` | business rule violation |

### 11.3 Endpoints

**Auth**
| Method | Path | Notes |
|---|---|---|
| POST | `/auth/register` | `{email,name,password,department}` → INTERNAL user |
| POST | `/auth/login` | → `{token,user}` |
| POST | `/auth/logout` | revoke `jti` |
| GET | `/auth/me` | current user |

**Users** (PM)
| GET | `/users` | filter by role/department (assignee picker) |
| POST | `/users` | create CLIENT/INTERNAL (P1) |

**Projects**
| GET | `/projects` | role-scoped, paginated |
| POST | `/projects` | PM |
| GET | `/projects/:id` | member/PM |
| PATCH | `/projects/:id` | PM |
| DELETE | `/projects/:id` | PM, soft |
| GET/POST/DELETE | `/projects/:id/members` | PM |
| GET | `/projects/:id/summary` | all roles; masked for CLIENT |

**Tasks**
| GET | `/projects/:id/tasks` | list (role-aware DTO + whitelist) |
| POST | `/projects/:id/tasks` | PM |
| GET | `/tasks/:id` | detail |
| PATCH | `/tasks/:id` | PM; fields + `version` |
| POST | `/tasks/:id/status` | `{status, version}`; transition matrix |
| PUT | `/tasks/:id/dependencies` | PM; `{dependsOnIds, version}` |
| DELETE | `/tasks/:id` | PM soft; `{version}` |
| GET | `/tasks/:id/history` | PM |

**Collaboration**
| GET/POST | `/tasks/:id/comments` | PM + INTERNAL |
| POST | `/tasks/:id/attachments` | multipart; PM + INTERNAL |
| GET | `/attachments/:id/download` | authz re-checked |
| DELETE | `/attachments/:id` | PM soft |

**Audit & Standup**
| GET | `/projects/:id/audit-logs` | PM; filters: userId, entityType, field, date range |
| GET | `/projects/:id/standup?date=YYYY-MM-DD` | default = yesterday in `APP_TZ` (default `Asia/Jakarta`) |

**Health:** `GET /health`.

### 11.4 Key payloads

`GET /projects/:id/tasks` item (PM/INTERNAL):
```json
{
  "id": "tsk_c", "title": "Frontend Slicing", "status": "TODO", "department": "FRONTEND",
  "assignee": { "id": "usr_fe", "name": "Fia", "avatarUrl": null, "department": "FRONTEND" },
  "clientVisible": true, "dueDate": null, "version": 3,
  "isBlocked": true,
  "blockedBy": [ { "id": "tsk_a", "title": "UI Design", "status": "IN_PROGRESS" },
                 { "id": "tsk_b", "title": "Backend API", "status": "TODO" } ]
}
```
Same task for CLIENT:
```json
{ "id": "tsk_c", "title": "Frontend Slicing", "description": "...", "status": "TODO", "dueDate": null }
```

`POST /tasks/:id/status` blocked → `422`:
```json
{ "success": false, "code": "TASK_BLOCKED", "message": "Task is blocked by unfinished prerequisites",
  "details": { "blockedBy": [ { "id": "tsk_a", "title": "UI Design", "status": "IN_PROGRESS" } ] } }
```

`409`:
```json
{ "success": false, "code": "VERSION_CONFLICT", "message": "Task was modified by someone else",
  "details": { "current": { "id": "tsk_x", "version": 4, "status": "DONE", "description": "..." } } }
```

---

## 12. Standup Auto-Summary (Bonus)

Input: project id + date `D` (default yesterday, `APP_TZ`). Window: `[D 00:00, D+1 00:00)`.

- **completedYesterday:** `AuditLog` rows where `entityType='TASK' AND field='status' AND newValue='"DONE"'` within the window → group by task department.
- **blockedToday:** computed *now* (not from the log): non-Done tasks with `isBlocked` true, grouped by department, including `blockedBy`.
- Optionally also include `startedYesterday` and `changesCount`.

```json
{
  "projectId": "prj_1", "date": "2026-09-30",
  "departments": {
    "UIUX": { "completedYesterday": [ { "taskId": "tsk_a", "title": "UI Design", "by": "Dina" } ], "blockedToday": [] },
    "FRONTEND": { "completedYesterday": [], "blockedToday": [ { "taskId": "tsk_c", "title": "Frontend Slicing", "blockedBy": ["UI Design","Backend API"] } ] },
    "BACKEND": { "completedYesterday": [], "blockedToday": [] }
  }
}
```
A background job is optional; the dedicated endpoint is enough.

---

## 13. File Storage
`StorageProvider` interface: `put(file) → key`, `getStream(key)`, `delete(key)`.
- Default: local disk (`UPLOAD_DIR`), mount a Railway volume in production. Optional S3-compatible adapter.
- Validate size (≤10 MB) and MIME allowlist. Store original name in DB, random key on disk.
- Downloads go through the API with authorization (no public URLs).

---

## 14. Seed Data (must work on the live deployment)

| Account | Role | Dept | Password |
|---|---|---|---|
| pm@keystone.test | PM | PM | set in `SEED_PASSWORD` env, document in submission |
| ui@keystone.test | INTERNAL | UIUX | same |
| fe@keystone.test | INTERNAL | FRONTEND | same |
| be@keystone.test | INTERNAL | BACKEND | same |
| client@keystone.test | CLIENT | — | same |
| client2@keystone.test | CLIENT | — | same (belongs to Project 2, proves tenant isolation) |

Project 1 "Acme Mobile App Launch" (client = client@): tasks
- A "UI Design" (UIUX, IN_PROGRESS, assignee ui@, clientVisible)
- B "Backend API Integration" (BACKEND, TODO, assignee be@, clientVisible)
- C "Frontend Slicing" (FRONTEND, TODO, **depends on A and B** → Blocked, clientVisible)
- D "Internal Security Review" (BACKEND, DONE, **not** client-visible)
- E "Brand Guideline" (UIUX, DONE, clientVisible)

Project 2 "Globex Portal" (client = client2@) with 2–3 tasks. Add a few comments and audit rows so every screen has content. Seed must be idempotent (upsert).

---

## 15. Deployment

**Backend (Railway/Render/Fly):**
- Provision PostgreSQL. Env: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `CORS_ORIGIN`, `UPLOAD_DIR`, `APP_TZ`, `SEED_PASSWORD`, `PORT`.
- Build/start: `bun install && bunx prisma generate` → release: `bunx prisma migrate deploy && bun run prisma/seed.ts` → `bun run src/index.ts`.
- Check `/health` and log in as every seeded role on the live URL.

**Frontend (Vercel):** `NEXT_PUBLIC_BE_URL=https://<api-host>/api/v1`. Verify CORS and that Authorization headers pass.

**Deploy a skeleton on Day 1** (health endpoint + login page) to flush out infra problems early.

---

## 16. Testing Strategy
- **Unit (bun test), must-have:** `taskPolicy` transition matrix (incl. PM_CANNOT_COMPLETE, TASK_BLOCKED), cycle detection, diff→audit generation, client serializer masking, optimistic lock conflict.
- **Integration (optional):** run service against a test DB with concurrent updates.
- **Frontend (one key component):** `StatusButton` / `TaskCard` — disabled + reason when blocked; enabled when not.
- **CI (optional):** GitHub Actions: install → `biome ci` → typecheck → build (+ tests).

---

## 17. Security Checklist
- [ ] All routes behind `auth` except register/login/health.
- [ ] Every task/project access passes membership check; non-members get 404.
- [ ] No raw Prisma objects in responses.
- [ ] Client filter/sort/search whitelist.
- [ ] Zod validation on every body/query/param.
- [ ] Passwords hashed; JWT secret from env; token revocation works.
- [ ] CORS locked to frontend origin.
- [ ] File upload size/type limits; authorized downloads.
- [ ] Audit + mutation in one transaction; trigger blocks audit tampering.
- [ ] Rate limit login (P1).

---

## 18. Architecture Summary for the Submission Doc (copy-ready)

1. **RBAC + ABAC:** `requireRole` middleware + `policies/*` evaluating role, membership, department, assignee, and task state.
2. **State-based permissions:** explicit transition matrix; blocked tasks rejected in the service layer inside the transaction; UI mirrors with disabled buttons + tooltip.
3. **Dependencies:** `TaskDependency` join table, same-project + cycle validation, derived `isBlocked`.
4. **Concurrency:** integer `version`, atomic `UPDATE ... WHERE version = ?`, `409 VERSION_CONFLICT` with latest state.
5. **Audit trail:** per-field `AuditLog` written in the same transaction; append-only via DB trigger; soft deletes everywhere via Prisma extension.
6. **Client masking:** whitelist serializer + DB-level `clientVisible` filter + per-role filter whitelist; verified by tests.
