# UI-UX: Keystone

> Frontend design spec for `keystone-web` (Next.js 16 App Router, React 19, Tailwind 4, shadcn/Radix).
> The UI **mirrors** backend rules for usability. It is never the security boundary.

---

## 1. Design Principles
1. **Explain every lock.** A disabled action always tells the user *why* (tooltip/inline text).
2. **Role-appropriate surfaces.** Each role sees only the controls it may use; the client view is deliberately minimal and calm.
3. **State is visible.** Status, blocked, version conflicts, and loading are always explicit, never silent.
4. **Fast scanning.** Dense but readable board; consistent badges and color semantics.
5. **Responsive & accessible.** Works from 360px up; keyboard navigable; WCAG AA contrast.

---

## 2. Brand & Design Tokens

**NodeWave brand colors are not specified in the brief.** Before styling, the agent must check NodeWave's website/brand assets and fill the tokens below. Everything uses CSS variables in one file so swapping is a 5-minute task.

`src/styles/tokens.css` (placeholders, replace brand values):
```css
:root {
  --brand-primary: <TODO from NodeWave>;
  --brand-primary-foreground: #ffffff;
  --brand-accent: <TODO>;
  --bg: #f8fafc;  --surface: #ffffff;  --border: #e2e8f0;
  --text: #0f172a; --text-muted: #64748b;

  /* semantic status colors (keep consistent everywhere) */
  --status-todo: #64748b;        /* slate */
  --status-progress: #2563eb;    /* blue */
  --status-done: #16a34a;        /* green */
  --status-blocked: #dc2626;     /* red */
  --status-conflict: #d97706;    /* amber */
}
.dark { /* optional dark mode */ }
```
Map tokens into Tailwind 4 `@theme`. Typography: one sans font (e.g. Inter/Geist), 14–16px base. Radius 8–12px. Subtle shadows.

---

## 3. Sitemap & Routes

```
/login                          public
/register                       public (creates INTERNAL users)
/projects                       list of my projects (role-scoped)
/projects/[projectId]           board (PM/INTERNAL) or client dashboard (CLIENT)
/projects/[projectId]/audit     PM only: project audit trail
/projects/[projectId]/standup   PM/INTERNAL: daily summary (bonus)
/projects/[projectId]/members   PM only (P1)
```
Route guard: unauthenticated → `/login?next=...`. Role-guard: wrong role → friendly 403 page (client trying `/audit`).

---

## 4. Screens

### 4.1 Auth (login / register)
```
┌──────────────────────────────────────┐
│            ▢ Keystone                │
│   Deliver high-value projects with   │
│   clarity.                           │
│  ┌────────────────────────────────┐  │
│  │ Email      [__________________]│  │
│  │ Password   [__________________]│  │
│  │ [        Sign in            ]  │  │
│  │ No account? Register           │  │
│  └────────────────────────────────┘  │
│  Demo accounts ▸ (collapsible list)  │
└──────────────────────────────────────┘
```
- React Hook Form + Zod; inline field errors; submit button shows spinner.
- A collapsible **"Demo accounts"** helper (PM / UI / FE / BE / Client, one-click fill) makes the reviewer's life easy. Only show it when `NEXT_PUBLIC_SHOW_DEMO=true`.
- Register: name, email, password, department select (UI/UX, Frontend, Backend).

### 4.2 Projects list
- Cards or table: name, description, progress bar, member count (PM/internal), blocked count badge.
- PM: **New Project** button. Search + pagination (follow the list contract).
- States: skeleton cards (loading), illustration + "No projects yet" (empty; PM sees CTA, others see "You haven't been added to a project"), error card with Retry.

### 4.3 Project Board (PM / Internal): the hero screen
```
┌ Header: ◂ Projects / Acme Mobile App Launch      [Audit] [Standup] [+ Task] ┐
│ Summary strip: 40% complete ▓▓▓▓░░░░░░  5 tasks · 1 blocked · 2 in progress │
│ Filters: [Search…] [Department ▾] [Assignee ▾] [Client-visible ▾] [Blocked only ☐]
├──────────────┬───────────────┬───────────────┬──────────────────────────────┤
│ TODO (2)     │ IN PROGRESS(1)│ DONE (2)      │  (Blocked shown as red badge  │
│ ┌──────────┐ │ ┌───────────┐ │ ┌───────────┐ │   + lock icon on TODO cards)  │
│ │🔒 C Front│ │ │A UI Design│ │ │E Brand    │ │                               │
│ │ end Slice│ │ │ 🟦 UI/UX  │ │ │ ✔ UI/UX   │ │                               │
│ │ BLOCKED  │ │ │ Dina      │ │ └───────────┘ │                               │
│ │ by A, B  │ │ └───────────┘ │               │                               │
│ └──────────┘ │               │               │                               │
└──────────────┴───────────────┴───────────────┴──────────────────────────────┘
```
Task card shows: title, department chip, assignee avatar+name, due date, client-visible eye icon (PM/internal only), **Blocked** state (red lock + "Blocked by 2"), dependency count.

- Columns = real statuses. **Blocked is a visual state on TODO cards**, optionally also a "Blocked only" filter.
- Mobile: columns become horizontal swipe tabs (TODO / In progress / Done) with a sticky tab bar.
- Clicking a card opens the **Task Drawer** (sheet from right; full-screen on mobile).
- Drag-and-drop is optional (P2). If added, dropping onto a forbidden column must show the same reason toast and snap back.

### 4.4 Task Drawer (detail)
Sections (tabs or stacked): **Details · Dependencies · Attachments · Comments · History**.

```
┌ Frontend Slicing                              [✕] ┐
│ Status: TODO   🔒 Blocked                          │
│ ┌ Blocked by ───────────────────────────────────┐  │
│ │ • UI Design        In Progress  → open        │  │
│ │ • Backend API      To Do        → open        │  │
│ └───────────────────────────────────────────────┘  │
│ [ Start task ] (disabled)  ⓘ Finish UI Design and  │
│                             Backend API first.     │
│ Description (read-only for engineers; PM: Edit)    │
│ Department: Frontend   Assignee: Fia   Due: —      │
│ Client visible: ◉ (PM toggle)                      │
│ Version 3                                          │
└────────────────────────────────────────────────────┘
```
**Status action button logic (UI mirror of BR-20..23)**, implement in `lib/permissions.ts`, unit-test it:

| Situation | Button | Tooltip / hint |
|---|---|---|
| Blocked, user wants to start | Disabled "Start task" | "Waiting for: UI Design, Backend API" |
| Not blocked, internal user in same department | "Start task" | — |
| Different department | Hidden or disabled | "This task belongs to Frontend" |
| In progress, I'm the assignee | "Mark as done" | — |
| In progress, I'm PM | Disabled "Mark as done" | "Only the assignee can complete this task" |
| In progress, someone else's task (internal) | Disabled | "Assigned to Dina" |
| Done | "Reopen" (PM/assignee) | — |

If the backend still returns 422/403 (stale UI), show the server `message` in a toast and refetch.

PM-only controls: Edit (title/description/department/assignee/due/client-visible), Dependencies picker (multi-select of other tasks in the project; show cycle error inline), Delete (confirm dialog; "archived, not erased").

### 4.5 Edit + Conflict flow (concurrency)
Edit form carries the loaded `version`. On `409 VERSION_CONFLICT`:

```
┌ ⚠ This task changed while you were editing ──────┐
│ Dina changed Status: In Progress → Done (v4)     │
│                                                  │
│  Your edit            Latest on server           │
│  Description (yours)  Description (current)      │
│  [ … ]                [ … ]                      │
│                                                  │
│ [ Reload latest ]   [ Re-apply my changes ]      │
└──────────────────────────────────────────────────┘
```
- **Reload latest:** discard local edits, refetch.
- **Re-apply my changes:** load server version into the form, keep user's text in the field, user saves again (new version).
- The dialog component is `ConflictDialog` and receives `details.current` from the 409 response.

### 4.6 Audit Trail (PM)
Table: Time · User · Entity · Field · Old → New. Filters: user, field, date range. Pagination. Old/new values rendered readably (status chips, user names resolved server-side or via lookup). Read-only; no edit/delete affordances.
Also a **History** tab inside the task drawer (timeline list).

### 4.7 Client Guest view
Intentionally simple and calm. **No avatars, names, departments, comments, attachments, dependency info.**
```
┌ Acme Mobile App Launch ─────────────────────────────┐
│  ┌─────────────────────┐                            │
│  │   40% Complete       │   2 of 5 tasks done        │
│  │   ▓▓▓▓░░░░░░         │   To do 2 · In progress 1  │
│  └─────────────────────┘                            │
│  Shared tasks                                        │
│  ┌──────────────────────────────────────────────┐   │
│  │ UI Design             ● In progress   Due …  │   │
│  │ Backend API           ○ To do                 │   │
│  │ Frontend Slicing      ○ To do                 │   │
│  │ Brand Guideline       ✔ Done                  │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```
- Read-only. Search by title, filter by status only.
- Task detail shows title, description, status, due date.
- The UI trusts the API (already masked). Do not render any field that the DTO doesn't contain.

### 4.8 Standup (bonus)
Date picker (default yesterday). Three department columns/cards; each lists "Completed yesterday" and "Blocked today (and by what)". Copy-as-JSON button for the structured output.

---

## 5. Component Inventory

| Component | Notes |
|---|---|
| `AppShell` | sidebar (desktop) / top bar + sheet (mobile), user menu with role badge + logout |
| `AuthGuard`, `RoleGuard` | route protection |
| `ProjectCard`, `ProgressBar` | |
| `TaskBoard`, `TaskColumn`, `TaskCard` | cards are keyboard focusable |
| `StatusBadge`, `DepartmentChip`, `BlockedBadge` | consistent semantics via tokens |
| `StatusButton` | encapsulates the table in §4.4; **the one key component to unit-test** |
| `TaskDrawer` (+ tabs) | shadcn Sheet |
| `DependencyPicker` | Combobox multi-select |
| `ConflictDialog` | §4.5 |
| `AttachmentUploader` | drag/drop + progress, size/type validation |
| `AuditTable`, `HistoryTimeline` | |
| `EmptyState`, `ErrorState`, `Skeletons` | reusable for every view |
| `ClientTaskList`, `ClientSummary` | masked views |

shadcn primitives: Button, Input, Select, Dialog, Sheet, Tabs, Tooltip, Badge, Avatar, Table, DropdownMenu, Toast (sonner), Skeleton, Popover/Command, Form.

---

## 6. State, Data & Interaction Rules

- **TanStack Query** keys: `['projects', params]`, `['project', id]`, `['tasks', projectId, params]`, `['task', id]`, `['audit', projectId, params]`, `['summary', projectId]`.
- Mutations send `version`. On success: invalidate `task`, `tasks`, `summary`. On `409`: open `ConflictDialog`. On `422 TASK_BLOCKED`: toast + refetch.
- **Optimistic UI** only for harmless UI preferences. For status changes prefer *pessimistic* (wait for server) to avoid confusing rollbacks; show a pending spinner on the button.
- **Zustand:** `auth.store` (token, user, hydrate, logout) and a small `ui.store` (drawer open/selected task, filters). Server data stays in TanStack Query, not Zustand.
- Axios instance: base URL `NEXT_PUBLIC_BE_URL`, request interceptor adds token, response interceptor handles `401` (logout + redirect) and normalizes errors to `{ status, code, message, details }`.
- Forms: React Hook Form + Zod resolvers; schemas colocated in `features/*/schemas.ts`.

---

## 7. Loading / Empty / Error Matrix (every view must have all three)

| View | Loading | Empty | Error |
|---|---|---|---|
| Projects | card skeletons | "No projects yet" (+CTA for PM) | message + Retry |
| Board | column skeletons | "No tasks yet. Create the first task" (PM) | message + Retry |
| Task drawer | skeleton blocks | n/a | inline error + Retry |
| Attachments | list skeleton | "No attachments yet" | inline error |
| Audit | table skeleton rows | "No activity yet" | message + Retry |
| Client view | skeleton | "No tasks have been shared with you yet" | message + Retry |
| Standup | skeleton | "Nothing completed or blocked for this date" | message + Retry |

---

## 8. Microcopy (examples)
- Blocked: "Blocked by 2 tasks"
- Disabled start: "Waiting for: UI Design, Backend API"
- PM complete denied: "Only the assignee can mark this task as done."
- Conflict title: "This task changed while you were editing"
- Delete confirm: "Archive this task? It will be hidden but kept in the audit trail."
- 403 page: "You don't have access to this page."

---

## 9. Responsive & Accessibility
- Breakpoints: mobile < 640, tablet 640–1024, desktop > 1024.
- Board: 3 columns on desktop, horizontally scrollable columns on tablet, tabbed on mobile.
- Drawer becomes full-screen sheet on mobile.
- Touch targets ≥ 44px; focus rings visible; dialogs trap focus (Radix handles it).
- Never convey state by color alone: pair with icon + text (lock icon + "Blocked").
- `aria-disabled` + tooltip-accessible reason text for locked buttons (tooltips must be reachable via keyboard focus; also render the reason as visible helper text in the drawer).

---

## 10. Demo Assets Checklist (deliverable)

**Screenshots (3–5):**
1. Login page (with demo accounts helper).
2. Task board showing dependency info (e.g., Task C with "Blocked by 2").
3. A **Blocked** task drawer with the disabled button and reason.
4. Client Guest view (clean, no internal info).
5. (Optional) Conflict dialog or audit trail.

**Screen recording (< 3 min) script (~2:45):**
1. 0:00 Live URL, login as **PM**; show board, dependencies on Task C (blocked).
2. 0:25 PM tries to complete an In Progress task → disabled/denied. Optionally show the API 403 in devtools.
3. 0:45 Login as **Frontend**: Task C start button locked with reason. Show direct API call rejected (Network tab or curl) → `422 TASK_BLOCKED`.
4. 1:10 Login as **UI/UX**: complete Task A, then **Backend**: complete Task B.
5. 1:35 Back as **Frontend**: Task C unlocked, start it.
6. 1:50 **Conflict demo:** two browser windows (PM editing description, engineer sets status). PM saves → conflict dialog.
7. 2:10 PM opens **Audit trail**: show old/new values.
8. 2:25 Login as **Client**: aggregate progress, only shared tasks; open Network tab to show masked JSON.
9. 2:40 Wrap-up.
