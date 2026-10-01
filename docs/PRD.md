# PRD: Keystone — Project Delivery Hub

> Product Requirements Document. Source of truth for *what* we build and *why*.
> Companion docs: `SYSTEM-DESIGN.md` (how), `UI-UX.md` (look & feel), `TASK-BREAKDOWN.md` (work plan).
> Language of docs: English (better for AI agents). Keep identifiers (FR-xx, BR-xx) stable; reference them in commits and PRs.

---

## 1. Overview

**Keystone** is the operational backbone for managing deliverables of high-value projects. A multi-discipline team (Product Management, UI/UX, Frontend, Backend) collaborates on tasks, while a **Client Guest** gets a safe, read-only window into progress.

The challenge is **not CRUD**. The challenge is correctly enforcing:

1. **State-based permissions** (RBAC + ABAC: who you are + which team + current task status).
2. **Inter-task dependencies** (a task is automatically *Blocked* until prerequisites are Done).
3. **Concurrency safety** (no silent overwrites; 409 Conflict on stale writes).
4. **Immutable audit trail + soft deletes** (nothing is ever really deleted or silently changed).
5. **Server-side data masking** for clients (internal identities never leave the API).

### 1.1 Goals
- G1: Every rule above is enforced **in the backend** and mirrored in the UI (never UI-only).
- G2: A reviewer can log in as each role on the live deployment and see the rules working within minutes.
- G3: Clean, type-safe, well-documented code across two separate repos (backend, frontend).

### 1.2 Non-goals
- Real-time sync (WebSockets), notifications/email, billing, SSO, mobile apps.
- Complex project templates, time tracking, Gantt charts.
- Drag-and-drop is *nice to have*, not required (buttons/menus are the primary status control).

---

## 2. Personas & Roles

| Persona | Role enum | Department | Summary |
|---|---|---|---|
| Product Manager | `PM` | `PM` | Plans work, defines dependencies, controls what clients see. Cannot complete work. |
| UI/UX Designer | `INTERNAL` | `UIUX` | Executes design tasks. |
| Frontend Engineer | `INTERNAL` | `FRONTEND` | Executes frontend tasks; usually blocked by design/backend tasks. |
| Backend Engineer | `INTERNAL` | `BACKEND` | Executes backend tasks. |
| Client Guest | `CLIENT` | none | External stakeholder; sees aggregate progress and explicitly shared tasks only. |

---

## 3. User Stories

### Authentication
- US-01: As a user, I can register, log in, and log out securely.
- US-02: As a visitor, I am redirected to login when opening a protected page; the API rejects unauthenticated calls with 401.

### Product Manager
- US-10: As a PM, I can create/edit/archive (soft-delete) projects and assign members (internal + client guests).
- US-11: As a PM, I can create/edit/archive tasks (title, description, department, assignee, due date, client-visible flag).
- US-12: As a PM, I can define which tasks a task depends on (and the system prevents cycles).
- US-13: As a PM, I can flag a task as **Client-Visible**.
- US-14: As a PM, I can see the full audit trail of a task and of a project.
- US-15: As a PM, I am prevented from moving a task from In Progress to Done (only the executor can).

### Internal Team
- US-20: As an engineer/designer, I see only projects I am a member of and only tasks inside them.
- US-21: As an engineer, I can start a task (→ In Progress) only if all its prerequisites are Done; otherwise the button is disabled (UI) and the API rejects it (backend).
- US-22: As the assignee, I can mark my In Progress task as Done.
- US-23: As an engineer, I can upload work attachments and comment internally, but I **cannot** change title/description.

### Client Guest
- US-30: As a client, I see aggregate metrics of **my own project only** (e.g. "50% Complete").
- US-31: As a client, I see only tasks flagged Client-Visible, with no internal identities, departments, comments, or attachments.
- US-32: As a client, I can never access another client's/project's data (tenant isolation).

### Concurrency
- US-40: As a PM editing a description while an engineer changes status, I am told about the conflict (409) instead of overwriting their change, and I can reload and re-apply my edit.

### Bonus
- US-50: As a PM, I can request a **Daily Standup Summary** for a project: what was completed yesterday and what is blocked today, per department.

---

## 4. Functional Requirements

Priority: **P0** = must ship, **P1** = should ship, **P2** = bonus.

### 4.1 Authentication & Sessions
| ID | Requirement | Pri |
|---|---|---|
| FR-01 | Register (email, name, password, department). Public register creates `INTERNAL` users only. PM and CLIENT accounts are seeded or created by a PM. | P0 |
| FR-02 | Login returns a signed JWT (with `jti`, `sub`, `role`, `department`, expiry). | P0 |
| FR-03 | Logout revokes the token server-side (denylist by `jti`) and clears client state. | P0 |
| FR-04 | `GET /auth/me` returns the current user. | P0 |
| FR-05 | Protected routes on API (middleware) and UI (route guard). | P0 |
| FR-06 | PM can create Client Guest accounts and attach them to a project. | P1 |

### 4.2 Projects & Membership
| ID | Requirement | Pri |
|---|---|---|
| FR-10 | CRUD (soft delete) for projects by PM. | P0 |
| FR-11 | Membership: PM adds/removes members. Internal/Client users see only member projects. PM sees all projects. | P0 |
| FR-12 | Project summary endpoint: totals per status, percent complete, blocked count. Client receives masked aggregate only. | P0 |

### 4.3 Tasks
| ID | Requirement | Pri |
|---|---|---|
| FR-20 | PM creates/edits tasks: title, description, department, assignee, dueDate, clientVisible. | P0 |
| FR-21 | Status model: `TODO`, `IN_PROGRESS`, `DONE`. **Blocked is derived**, not stored (see BR-10). | P0 |
| FR-22 | Status transitions follow the transition matrix (BR-20). | P0 |
| FR-23 | Internal users cannot edit title/description/assignee/clientVisible/dependencies. | P0 |
| FR-24 | Attachments: upload (internal + PM), list, download, soft-delete (PM). Max 10 MB. | P0 |
| FR-25 | Internal comments (PM + internal only). | P1 |
| FR-26 | List endpoints support filter, search, sort, pagination per the NodeWave standard. | P0 |

### 4.4 Dependencies
| ID | Requirement | Pri |
|---|---|---|
| FR-30 | PM sets prerequisites for a task (many-to-many, same project only). | P0 |
| FR-31 | Reject self-dependency and cycles (A→B→A, longer loops too). | P0 |
| FR-32 | A task with any non-Done prerequisite is **Blocked**: cannot enter In Progress. Enforced in backend; button disabled in UI with a reason. | P0 |
| FR-33 | API exposes `isBlocked` and `blockedBy[]` on tasks (internal/PM view). | P0 |

### 4.5 Concurrency
| ID | Requirement | Pri |
|---|---|---|
| FR-40 | Every task has an integer `version`. Every mutating request includes the `version` it was based on. | P0 |
| FR-41 | On mismatch → `409 Conflict` with code `VERSION_CONFLICT` and the latest task state in the body. | P0 |
| FR-42 | UI shows a conflict dialog (reload latest / keep my edit and re-apply). | P0 |

### 4.6 Audit Trail & Soft Delete
| ID | Requirement | Pri |
|---|---|---|
| FR-50 | Every task field change writes an `AuditLog` row: userId, timestamp, entity, changed column, old value, new value. | P0 |
| FR-51 | Audit rows are append-only: no update/delete API, and DB-level trigger blocks UPDATE/DELETE. | P0 |
| FR-52 | All entities use soft delete (`deletedAt`). Default queries exclude deleted rows. Soft deletes are also audited. | P0 |
| FR-53 | PM can view audit logs per task and per project (paginated, filterable). | P0 |

### 4.7 Client Guest & Masking
| ID | Requirement | Pri |
|---|---|---|
| FR-60 | Client sees only own-project aggregate and tasks with `clientVisible = true`. | P0 |
| FR-61 | Client responses are built from **whitelist** DTOs: no assignee, avatar, department, comments, attachments, audit, dependency ids. | P0 |
| FR-62 | Client list endpoints restrict filter/search/sort fields (no filtering by hidden fields, which would leak data by inference). | P0 |
| FR-63 | Access to a project the client is not a member of returns `404` (not `403`). | P0 |

### 4.8 Standup Summary (Bonus)
| ID | Requirement | Pri |
|---|---|---|
| FR-70 | `GET /projects/:id/standup?date=` returns structured JSON from audit logs of the previous day: `completedYesterday[]` and `blockedToday[]` grouped by department. | P2 |

### 4.9 UX Requirements
| ID | Requirement | Pri |
|---|---|---|
| FR-80 | Loading, empty, and error states on every data view. | P0 |
| FR-81 | Responsive layout (mobile → desktop). | P0 |
| FR-82 | NodeWave brand colors applied via design tokens. | P0 |

---

## 5. Business Rules

| ID | Rule |
|---|---|
| BR-01 | Authorization = Role (RBAC) + Department & membership & assignment (ABAC) + current Task Status (state-based). Evaluated server-side on every request. |
| BR-02 | Public registration can never create `PM` or `CLIENT`. |
| BR-10 | `isBlocked(task) = exists prerequisite where prerequisite.status != DONE and prerequisite.deletedAt is null`. Computed on read and re-checked inside the transaction on every transition. |
| BR-11 | A blocked task cannot move to `IN_PROGRESS`, regardless of role (PM included). |
| BR-12 | Dependencies must be within the same project and must be acyclic. |
| BR-20 | **Transition matrix** (see SYSTEM-DESIGN §6): `TODO→IN_PROGRESS`, `IN_PROGRESS→DONE`, `IN_PROGRESS→TODO`, `DONE→IN_PROGRESS` (reopen). Anything else is invalid. |
| BR-21 | `IN_PROGRESS→DONE` is allowed **only for the assignee** (an `INTERNAL` user). PM is always denied (`PM_CANNOT_COMPLETE`). |
| BR-22 | An internal user can start a task only if: project member, task.department == user.department, task unassigned or assigned to them (starting auto-assigns). |
| BR-23 | Internal users may only change status and add attachments/comments. |
| BR-30 | Every mutation of a task requires the current `version`; success increments it by 1. |
| BR-40 | Hard deletes never happen. Audit logs are append-only. |
| BR-50 | Client data is whitelisted by DTO in the API layer. Frontend hiding is never relied upon. |

---

## 6. Non-Functional Requirements

- **Security:** bcrypt/argon2 password hashing, JWT secret in env, CORS limited to frontend origin, input validated with Zod, rate limit on login (P1), no sensitive data in logs.
- **Consistency:** mutation + audit rows + version bump happen in **one DB transaction**.
- **Type safety:** TypeScript `strict` on both repos; shared contract described in SYSTEM-DESIGN §10.
- **Performance:** list endpoints paginated; indexes on `projectId`, `status`, `deletedAt`, `(entityType, entityId)`, `createdAt`.
- **Maintainability:** layered architecture, conventional commits, Biome formatting, small functions with unit tests on critical logic.
- **Observability:** structured request logs, `/health` endpoint.

---

## 7. Assumptions & Open Questions (decisions made; revisit if the brief clarifies)

| # | Ambiguity in brief | Decision |
|---|---|---|
| A1 | Can PM start a task (TODO→IN_PROGRESS)? | Yes, subject to the Blocked rule. Only IN_PROGRESS→DONE is denied for PM. |
| A2 | Is "Blocked" a stored status? | No. Derived from dependencies, exposed as `isBlocked` and shown as a lane/badge. |
| A3 | Does the client's "% complete" count hidden tasks? | Yes: percent = DONE / total non-deleted tasks of the project (true progress). Only numbers are exposed. Easy to switch to visible-only. |
| A4 | Who can register? | Only `INTERNAL`. PM/Client via seed or PM-created. |
| A5 | Logout with stateless JWT | Denylist table keyed by `jti` until token expiry. |
| A6 | Safe merge vs 409 | 409 only (explicitly allowed by brief). |
| A7 | File storage | Abstract `StorageProvider`; default local disk/volume, S3-compatible optional. |
| A8 | Reopening a Done task that has in-progress dependents | Allowed; dependents in TODO become blocked; dependents already IN_PROGRESS get a "prerequisite reopened" warning flag (no auto-revert). |
| A9 | NodeWave brand colors / pagination standard | Not included in the brief text. Agents must fetch the linked "Filtering, Pagination & Searching" doc and NodeWave brand assets before implementing; tokens are centralized so swapping is trivial. |

---

## 8. Success Criteria (what the reviewer will check)

1. Seeded PM / Internal / Client accounts all log in on the **live** deployment.
2. Frontend task button is disabled when blocked **and** direct API call returns a rejection.
3. PM attempting IN_PROGRESS→DONE is rejected (UI + API).
4. Two stale writes → one succeeds, other gets `409`.
5. Audit rows exist for status/description/assignee changes; cannot be altered.
6. Soft-deleted items disappear from lists but remain in DB.
7. Client network responses contain **no** internal names/avatars/departments/comments.
8. Docs, screenshots (3–5), and a ≤3 min screen recording are delivered.

---

## 9. Deliverables

- Backend repo (private) + Frontend repo (private); collaborators: `rigenski`, `nodewavescout`.
- Live URLs + seeded credentials per role.
- Architecture overview (derived from `SYSTEM-DESIGN.md`).
- 3–5 screenshots: auth, board with dependencies, Blocked state, Client Guest view.
- Screen recording < 3 minutes.
- Submission form: https://tally.so/r/LZ87Xz — deadline **72 hours** from receipt.
