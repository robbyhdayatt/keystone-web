# TASK-BREAKDOWN: Keystone (72-hour plan, agent-ready)

> Work plan for building Keystone with AI agents (e.g. Antigravity).
> Read first: `PRD.md` → `SYSTEM-DESIGN.md` → `UI-UX.md`. IDs: **BE-xx** (backend repo), **FE-xx** (frontend repo), **OPS-xx**, **DOC-xx**.
> Mark tasks `[x]` when done. Each task has **Acceptance Criteria (AC)**. Do not mark done unless AC is verified.

---

## 0. How to work with the agents

### 0.1 Working agreements (paste into the agent's rules / system instructions, per repo)
```
- Source of truth: docs/PRD.md, docs/SYSTEM-DESIGN.md, docs/UI-UX.md. If code and docs conflict, ask or update docs explicitly.
- TypeScript strict. No `any` unless justified in a comment. Biome must pass.
- Backend layering: route → controller → service → repository → serializer. No Prisma in controllers. No raw Prisma objects in responses.
- All authorization and business rules are enforced in the BACKEND. Frontend only mirrors them.
- Every task mutation: one transaction = version check + update + audit rows.
- Conventional Commits (feat:, fix:, chore:, docs:, test:, refactor:), small commits, one concern each. Reference IDs, e.g. "feat(tasks): enforce blocked rule (BE-12)".
- Never invent the query-param contract of @nodewave/prisma-ezfilter. Read the NodeWave "Filtering, Pagination & Searching" doc and the package README first.
- Never hard-delete. Never edit or delete AuditLog rows.
- After each task: run lint, typecheck, tests; summarize what changed and how it was verified.
```

### 0.2 Suggested agent setup
- Open **two workspaces**: `keystone-api` and `keystone-web`. Put the 4 docs into `docs/` of both repos.
- Use **one agent per repo** (backend agent, frontend agent). Frontend starts after the API contract (BE-01..BE-06) is stable; until then it can build UI against the contract in SYSTEM-DESIGN §11 with mock data.
- Ask the agent to **plan first** and show the plan before large changes; review diffs yourself at every phase boundary (the rules in this system are subtle, so don't blindly accept).
- Keep each prompt scoped to 1–3 tasks from this file. Paste the task ID + AC + the relevant doc section.
- Don't let agents run unrestricted destructive commands (database resets in production, force pushes).
- Verify critical rules yourself with curl/Postman, not only by trusting agent claims.

### 0.3 Prompt template
```
Context: Keystone backend. Read docs/SYSTEM-DESIGN.md §{sections} and docs/PRD.md {FR/BR ids}.
Task: {ID + title}.
Requirements: {paste task description}.
Acceptance criteria: {paste AC}.
Constraints: follow the working agreements. Write/extend unit tests for the rules touched.
Output: implement, run lint/typecheck/tests, then give a short summary + how to verify manually.
```

---

## 1. Timeline Overview (72h)

| Block | Hours | Focus |
|---|---|---|
| Phase 0 | 0–3 | Repos, tooling, skeleton **deployed** |
| Phase 1 | 3–14 | Backend core: schema, auth, projects, tasks CRUD |
| Phase 2 | 14–28 | The hard rules: dependencies, state machine, locking, audit, soft delete, masking |
| Phase 3 | 20–46 | Frontend (starts overlapping once API contract is stable) |
| Phase 4 | 46–56 | Integration, bonus (standup), tests, CI |
| Phase 5 | 56–68 | Deploy hardening, seeds, QA on live, docs, screenshots, video |
| Buffer | 68–72 | Submit early. Never use the final hour for anything new |

**Priority order if time runs short:** Auth+RBAC → Task+dependency+state validation → Optimistic locking → Audit+soft delete → Client masking → Frontend polish/deploy/docs/video → Standup bonus.

---

## 2. Phase 0: Setup & Skeleton Deploy (0–3h)

- [ ] **OPS-01 Create repos.** Two private GitHub repos `keystone-api`, `keystone-web`. Invite `rigenski`, `nodewavescout`.
  AC: both repos exist, collaborators invited (pending OK), README stub, `docs/` copied.
- [ ] **BE-00 Backend bootstrap.** Bun + Hono + TS strict + Prisma + Zod env + Biome + Husky + Commitlint; `/health` endpoint; folder structure from SYSTEM-DESIGN §2.1.
  AC: `bun run dev` serves `/health`; lint, typecheck pass; commit hook enforces Conventional Commits.
- [ ] **FE-00 Frontend bootstrap.** Next.js 16 + React 19 + TS strict + Tailwind 4 + shadcn + TanStack Query + Axios + Zustand + RHF + Zod + Biome + Husky + Commitlint; providers; token file; placeholder login page.
  AC: `bun/npm run dev` works; lint, typecheck, build pass.
- [ ] **OPS-02 Skeleton deploy.** Deploy API (+ Postgres) and web; set `NEXT_PUBLIC_BE_URL`, CORS.
  AC: frontend on public URL can call live `/health`.
- [ ] **OPS-03 Fetch external specs.** Read NodeWave "Filtering, Pagination & Searching" doc + `@nodewave/prisma-ezfilter` README; grab NodeWave brand colors. Write notes into `docs/NOTES.md` (param names, response meta shape, color hexes).
  AC: notes file committed; tokens in `tokens.css` updated.

---

## 3. Phase 1: Backend Core (3–14h)

- [ ] **BE-01 Prisma schema + migration.** Implement SYSTEM-DESIGN §3 (all models/enums), indexes, partial unique email index, audit append-only trigger migration.
  AC: `prisma migrate dev` OK; manual `UPDATE/DELETE` on `AuditLog` raises an exception.
- [ ] **BE-02 Core libs.** env, prisma client + soft-delete extension, error classes + global error handler, response envelope, jwt, password hashing, Zod validation helper.
  AC: soft-deleted rows excluded by default; errors follow envelope with `code`.
- [ ] **BE-03 Auth module.** register (INTERNAL only), login, logout (denylist), me; `auth` middleware with revocation check; `requireRole`.
  AC: token of logged-out user → 401 `TOKEN_REVOKED`; register cannot set role; passwords never returned.
- [ ] **BE-04 Seed.** Idempotent seed per SYSTEM-DESIGN §14 (6 users, 2 projects, tasks, deps, comments).
  AC: `bun run seed` twice causes no errors/duplicates; all 6 accounts can log in.
- [ ] **BE-05 Projects + membership + summary.** CRUD (soft), members, role-scoped list with ezfilter, `/summary` (full for PM/internal, aggregate for client).
  AC: internal sees only member projects; non-member → 404; client summary has only numbers.
- [ ] **BE-06 Tasks CRUD (PM) + list/detail.** Create, patch (title/description/assignee/department/due/clientVisible), list with filters/search/sort/pagination, detail. Role-aware serializers (internal vs client DTO) from day one.
  AC: internal cannot PATCH (403); list conforms to NodeWave pagination contract; DTOs never return raw Prisma objects.
- [ ] **BE-07 Per-role list whitelist.** `buildListQuery(config, role)` restricting filter/search/sort fields (client: title/status/dueDate only).
  AC: client request `?...assignee...` → 400; PM/internal full set works.

---

## 4. Phase 2: The Hard Rules (14–28h) ⭐ core of the evaluation

- [ ] **BE-10 Audit service.** `audit.record(tx, …)` + diff helper tracking fields (title, description, status, assigneeId, department, clientVisible, dueDate, deletedAt, dependencies). One row per changed field.
  AC: patching 2 fields creates 2 rows with correct old/new, userId, timestamp; written in the same transaction (simulate failure → no mutation persisted).
- [ ] **BE-11 Optimistic locking.** `version` required on every task mutation; atomic `updateMany where version`; `409 VERSION_CONFLICT` with `details.current`.
  AC: concurrent test (`Promise.all`) yields exactly one success + one 409; nothing overwritten; version increments by 1.
- [ ] **BE-12 Dependencies.** `PUT /tasks/:id/dependencies` (PM only): same-project, no self, cycle detection, diff add/remove, audit entries, version bump. Derived `isBlocked` + `blockedBy` in DTOs (efficient batch query).
  AC: A→B→C→A rejected with `DEPENDENCY_CYCLE`; cross-project rejected; Task C blocked until A and B are DONE; unit tests for cycle detection.
- [ ] **BE-13 State machine + policy.** `taskPolicy.assertCanTransition` (SYSTEM-DESIGN §6) used by `POST /tasks/:id/status`; blocked check re-evaluated inside the transaction; auto-assign on start; set/clear `completedAt`.
  AC (each has a test): PM IN_PROGRESS→DONE → 403 `PM_CANNOT_COMPLETE`; Frontend start while blocked → 422 `TASK_BLOCKED` with blockers; non-assignee complete → 403; wrong department → 403; invalid transition → 422; status change writes audit + bumps version.
- [ ] **BE-14 Soft delete flows.** Delete endpoints for task/project/attachment; cascade soft-delete of dependency edges; audit DELETE.
  AC: deleted entities absent from lists/detail (404) but present in DB; deleting a prerequisite unblocks dependents.
- [ ] **BE-15 Masking verification.** Client serializer whitelist (`.strict()` Zod) + DB-level `clientVisible: true` filter + membership check → 404.
  AC: test with fully populated fixture proves client JSON has no assignee/avatar/department/comments/attachments/audit/dependency/version fields; client of project 2 cannot read project 1 (404).
- [ ] **BE-16 Attachments.** Multipart upload (≤10MB, MIME allowlist), `StorageProvider` (local), authorized download, soft delete (PM). Audit entry on add/remove.
  AC: internal member can upload; client → 403/404; download re-checks authorization.
- [ ] **BE-17 Comments (internal).** List/create; never included in any client response.
  AC: client has no access; not present in client task DTO.
- [ ] **BE-18 Audit read APIs.** `GET /projects/:id/audit-logs`, `GET /tasks/:id/history` (PM) with filters/pagination.
  AC: filter by user/field/date works; no write endpoints exist for audit.

**Checkpoint (hour ~28):** run a manual script (curl/Bruno/Postman collection) covering all P0 rules against the local DB. Commit the collection to `docs/api-collection/`. Deploy to live and re-run.

---

## 5. Phase 3: Frontend (can start ~hour 20 against the API contract)

- [ ] **FE-01 Axios + auth store + guards.** Axios instance with interceptors (401 → logout, normalized errors incl. `code`, `details`); Zustand persisted auth store; `AuthGuard`, `RoleGuard`; app shell with role badge + logout.
  AC: reload keeps session; expired/revoked token → redirected to `/login`; unauthenticated deep link → `/login?next=`.
- [ ] **FE-02 Login & Register pages.** RHF + Zod; demo accounts helper (flag-controlled).
  AC: validation messages, loading state, error toast on bad credentials; register creates INTERNAL user.
- [ ] **FE-03 Projects list.** Search + pagination, progress bar, loading/empty/error states.
  AC: all three states demonstrable; PM sees "New project".
- [ ] **FE-04 Board (PM/Internal).** Columns, task cards, filters, summary strip, Blocked visual state, mobile tabs.
  AC: Task C shows "Blocked by 2"; responsive at 360px; skeleton/empty/error.
- [ ] **FE-05 Task drawer + StatusButton.** Implement the button matrix (UI-UX §4.4) in `lib/permissions.ts`; reason tooltips + visible helper text; mutation sends `version`; 422/403 toasts.
  AC: blocked → disabled with reason; PM cannot complete; assignee can complete; **unit test for StatusButton** (blocked/unblocked).
- [ ] **FE-06 PM editing.** Edit form (title, description, department, assignee, due, clientVisible), dependency picker with cycle error, delete confirm.
  AC: Internal users do not see edit controls; PM edits succeed and bump version.
- [ ] **FE-07 ConflictDialog.** Handle 409 across edit/status/dependencies; reload vs re-apply.
  AC: two browser windows reproduce conflict; no data lost; dialog shows server's latest.
- [ ] **FE-08 Attachments & comments.** Uploader with progress + validation, list, download, comments tab.
  AC: errors from API displayed; empty states.
- [ ] **FE-09 Audit page + History tab.** Table with filters + pagination; timeline in drawer.
  AC: PM only; old→new values readable.
- [ ] **FE-10 Client dashboard.** Summary card + shared task list + detail; no internal fields rendered.
  AC: Network tab shows masked payloads; layout clean on mobile.
- [ ] **FE-11 Polish.** Brand tokens applied, favicon/title, consistent badges, a11y pass (focus, contrast, aria), responsive pass.
  AC: Lighthouse a11y ≥ 90 on main pages (best effort).

---

## 6. Phase 4: Integration, Bonus, Tests, CI (46–56h)

- [ ] **BE-20 Standup endpoint (bonus).** Per SYSTEM-DESIGN §12.
  AC: seeded activity yields `completedYesterday` and `blockedToday` grouped by department; date param works; timezone configurable.
- [ ] **FE-12 Standup page (bonus).**
  AC: renders three departments with empty/loading/error states.
- [ ] **BE-21 Unit tests.** At minimum: transition matrix, cycle detection, audit diff, client masking, optimistic lock.
  AC: `bun test` green; run in CI.
- [ ] **FE-13 Component test.** `StatusButton` (or TaskCard).
  AC: test passes in CI.
- [ ] **OPS-04 CI (optional).** GitHub Actions per repo: install → biome ci → typecheck → build (→ test).
  AC: green badge on main.
- [ ] **QA-01 End-to-end manual pass** against local, then live (see §8 checklist).

---

## 7. Phase 5: Deploy Hardening, Docs, Media (56–68h)

- [ ] **OPS-05 Production deploy.** Migrations + seed on live DB; env vars; CORS; uploads volume; confirm all 6 seeded accounts log in on live.
  AC: reviewer can follow credentials list and log in as PM, internal, client on the live URL.
- [ ] **DOC-01 Submission document** (single doc/email body): live URLs (FE + BE), credentials per role, repo links, architecture overview (copy from SYSTEM-DESIGN §18 and expand with 1 short example per mechanism), assumptions list (PRD §7), how to run tests.
- [ ] **DOC-02 Screenshots (3–5)** per UI-UX §10 captured from the **live** app.
- [ ] **DOC-03 Screen recording < 3 min** following the script in UI-UX §10. Rehearse once, record, trim.
- [ ] **DOC-04 READMEs** in both repos: setup, env vars, scripts, seed credentials, API collection link.
- [ ] **OPS-06 Submit** via https://tally.so/r/LZ87Xz **at least 3–4 hours before deadline**.

---

## 8. Final QA Checklist (run on LIVE)

**Auth**
- [ ] Register works (INTERNAL only). Login works for all 6 seeded accounts.
- [ ] Logout then reuse old token via curl → 401.
- [ ] Protected pages redirect to login; protected API returns 401.

**RBAC / ABAC / State**
- [ ] Internal user sees only member projects; others → 404.
- [ ] Internal PATCH on task description → 403.
- [ ] Frontend user: Task C start disabled while A or B not Done; direct `POST /tasks/:id/status` → 422 `TASK_BLOCKED`.
- [ ] After A and B are Done, Task C start succeeds.
- [ ] PM IN_PROGRESS→DONE rejected in UI and API (403).
- [ ] Wrong-department user cannot start task (403).

**Dependencies**
- [ ] Cycle attempt rejected. Cross-project rejected.

**Concurrency**
- [ ] Stale `version` returns 409 and UI shows ConflictDialog; no overwritten data.

**Audit / soft delete**
- [ ] Status, description, assignee changes each create audit rows with old/new/user/time.
- [ ] Direct SQL UPDATE/DELETE on AuditLog fails.
- [ ] Deleted task disappears from lists but exists in DB.

**Client**
- [ ] Client sees aggregate + only `clientVisible` tasks.
- [ ] Client payloads contain no names/avatars/departments/comments (inspect Network tab).
- [ ] Client filtering by hidden field → 400. Client2 cannot access Project 1 (404).

**UX**
- [ ] Loading, empty, error states on all views. Responsive at 360 / 768 / 1280.

**Delivery**
- [ ] Both repos private with both collaborators invited. Conventional commit history. Docs, screenshots, video, links complete.

---

## 9. Risk Register

| Risk | Mitigation |
|---|---|
| ezfilter query contract unclear | Read docs first (OPS-03); isolate behind `buildListQuery`; test early with a simple list |
| Deploy issues (CORS, migrations, volumes) | Skeleton deploy in Phase 0; redeploy after each phase |
| Agent writes UI-only enforcement | Review BE-13/BE-12 personally; require API-level tests |
| Race condition tests flaky | Use real DB + `Promise.all`; assert counts, not ordering |
| Scope creep (drag-drop, notifications) | Only P0 first; P1/P2 only after QA checklist is green |
| Last-minute media problems | Record video + screenshots in Phase 5, not at the end |

---

## 10. Definition of Done
All P0 requirements in `PRD.md` verified via §8 on the live deployment; tests green; docs/screenshots/video delivered; submitted ahead of the 72h deadline.
