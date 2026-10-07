# Winners Connect — Local Verification and Code Review

**Date:** 2026-10-05  
**Project:** `winnersco` / Winners Connect by Winners Institute Indore

## Executive summary

The project installs, type-checks, tests, builds, starts locally, and serves the React application successfully. The browser smoke test reached the app and rendered the main workspace routes without console errors.

However, the current application behaves primarily as a **front-end prototype**. Most visible operational features use in-memory React state and `localStorage`; they do not call the server-side tRPC procedures or persist to the Drizzle/MySQL schema. The backend contains several starter/placeholder procedures that return empty data or no-op results. As a result, the app is suitable for UI demonstration and local preview, but it is **not yet a reliable production system of record**.

## Verification performed

### Commands

| Check | Result | Notes |
|---|---:|---|
| `pnpm install --frozen-lockfile` | Passed | 749 packages installed; lockfile matched |
| `pnpm check` | Passed | TypeScript completed with no errors |
| `pnpm test` | Passed | 2 test files, 6 tests passed |
| `pnpm build` | Passed with warnings | Vite/esbuild build succeeded |
| `PORT=4173 pnpm dev` | Passed | Express/Vite server started and remained healthy |
| `curl http://127.0.0.1:4173/` | Passed | HTTP 200, application HTML returned |
| Browser navigation | Passed | Sandbox browser loaded the public app URL |
| Browser console | Passed | No console output/errors on initial overview render |

### Build warnings

The production build reports two non-blocking warnings:

1. `/api/platform/config.js` is included as a classic script without `type="module"`.
2. The main JavaScript bundle is approximately 550 kB after minification, above Vite’s 500 kB advisory threshold.

### Routes smoke-tested

The following routes rendered successfully in the browser:

- `/` — Overview
- `/schedule` — Schedule board
- `/scheduling` — Scheduling control
- `/attendance` — Attendance desk
- `/master-data` — Master data (tested both restricted and Master Admin views)
- `/courses` — Courses & lectures
- `/notes` — Notes compliance

The remaining registered routes are present in `client/src/App.tsx` and use the same `Home` shell: `/faculty`, `/analytics`, `/live`, `/audit`, `/settings`, `/login`, `/master-admin`, and `/access/:userId`.

The UI consistently rendered empty states because the checked-out seed collections are empty and there was no configured database/session data in this local environment.

## Architecture summary

### Frontend

- React 19 + TypeScript + Vite.
- Wouter provides client-side routing.
- `client/src/pages/Home.tsx` is the primary application shell and owns most operational state.
- Radix UI primitives and custom CSS provide the component system.
- Recharts is used for dashboard visualizations.
- `localStorage` is used for date selection, faculty, schedule, master data, access users, and role preview state.
- `client/src/lib/winners-data.ts` and `client/src/lib/master-data.ts` hold seed/demo models. In this archive, the main seed arrays are empty.

### Backend

- Express server in `server/_core/index.ts`.
- tRPC router in `server/routers.ts`.
- Drizzle ORM with MySQL schema in `drizzle/schema.ts`.
- Platform authentication/session helpers under `server/_core`.
- Storage, maps, LLM, image generation, OAuth, and notification helpers are platform-oriented integrations.
- Database access is lazy and optional: if `DATABASE_URL` is absent, `getDb()` returns `null` and several procedures fall back to empty/demo responses.

### Data model

The schema models users, faculties, centres, halls, operators, courses, lectures, attendance, schedule slots, lecture assignments, notes, audit logs, resources, faculty-course mappings, replacement logs, rule catalog, and settings. The data model is materially richer than the currently wired frontend.

### Intended flow

1. Master data defines faculty, courses, mappings, resources, and rules.
2. Attendance captures availability.
3. Scheduling control resolves faculty/course/resource/operator assignments.
4. Schedule board and live display consume the schedule.
5. Notes compliance tracks online lecture artifacts.
6. Audit logs capture changes.

That flow is represented in the UI and schema, but only partially implemented end-to-end.

## Potential bugs and implementation gaps

### Critical — visible operations are not connected to the backend

`Home.tsx`, `MasterData.tsx`, and `Scheduling.tsx` manage their records with React state and browser storage. There are no active `trpc.winners.*` calls in those pages. For example, attendance, schedule generation, faculty edits, course creation, and resource changes update local state and/or `localStorage`, not the database.

**Impact:** changes are browser/device-specific, disappear when local storage is cleared, are invisible to other users, and do not populate the Drizzle tables or server audit log.

**Evidence:**

- `client/src/pages/Home.tsx:260-271` loads/saves date data in `localStorage`.
- `client/src/pages/Home.tsx:277-279` implements attendance and generation locally.
- `client/src/pages/MasterData.tsx:19-22` persists master-data state in `localStorage`.
- `client/src/pages/Scheduling.tsx:16-24` persists schedule rows locally.
- The server mutations exist in `server/routers.ts:50-119`, but the operational pages do not invoke them.

### Critical — authorization is currently client-controlled in the workspace UI

The role selector can change the active role in the browser, and `/master-admin` exposes a client-side unlock gate. The access portal compares a plaintext password stored in `localStorage` and writes role/access flags back to `localStorage`.

**Impact:** this is not a secure authorization boundary. A user who can load the app can manipulate browser storage or client state to change the displayed role. Backend procedures do have protected checks, but the main UI does not use them for its operational actions.

**Evidence:**

- `client/src/pages/Home.tsx:257` derives the role from `localStorage`.
- `client/src/pages/Home.tsx:281` unlocks Master Admin in client state.
- `client/src/pages/Home.tsx:239` implements access-password and dashboard selection in the browser.
- `server/routers.ts:13-20` contains server-side role checks, but those checks do not protect the local-only UI mutations.

### High — multi-dashboard assignment is not represented consistently

The UI supports `dashboards?: Role[]` and multiple dashboard checkboxes, but the database schema has only one `users.appRole` enum value, and `createUser` accepts a single `dashboard` value.

**Impact:** a user created through the server cannot reliably retain multiple dashboard assignments. The local access portal and database can disagree.

**Evidence:**

- `drizzle/schema.ts` defines a single `appRole` column.
- `server/routers.ts:40-41` exposes one `dashboard` input and stores one `appRole`.
- `Home.tsx:239` and the Settings UI use arrays of dashboards locally.

### High — several backend procedures are placeholders

The server returns empty/no-op results for key workflows:

- `winners.schedule` always returns `slots: 0` and `status: "empty"` (`server/routers.ts:72`).
- `winners.generateSchedule` records an audit event but returns `generated: false` and `slots: 0` (`server/routers.ts:80-83`).
- `winners.courses` and `winners.resources` return empty collections when no DB is configured (`server/routers.ts:42-43`).
- `winners.masterData` returns rules only, not the full master-data set (`server/routers.ts:44-48`).

**Impact:** production behavior will not match the UI’s implied capabilities until the frontend and backend are integrated and the scheduling algorithm is implemented.

### Medium — audit trail is split and not durable

The UI audit trail is a capped in-memory array of 20 entries (`Home.tsx:272, 276`), while the backend audit trail writes to `auditLogs` only when backend mutations are called. Local UI actions therefore do not appear in the server audit log.

### Medium — date isolation is only partially implemented

The app intentionally stores many records under date-specific browser-storage keys, but several components initialize date-scoped state once and rely on local fallback data. This is fragile for cross-date changes and multi-tab synchronization. The system has no server-side date query layer for the frontend operational screens.

### Medium — validation is duplicated and inconsistent

The backend validates schedule inputs in `validateSchedule`, while the frontend uses a separate warning implementation in `Scheduling.tsx`. The two implementations can diverge. The backend validation also does not clearly validate malformed time strings, duplicate resource/faculty conflicts, or per-faculty lecture limits despite the rule catalog exposing those concepts.

### Medium — tests cover platform helpers, not product workflows

The six passing tests cover auth/platform behavior only. There are no tests for:

- date switching and isolation;
- role-based navigation;
- faculty/course/resource CRUD;
- schedule generation and validation;
- attendance and replacement flows;
- notes compliance;
- multi-dashboard access;
- database persistence.

### Low — production bundle size and script warning

The build succeeds but should be improved with code splitting and correction of the platform config script tag. This is not currently a functional blocker.

## Feature status from this run

| Feature area | UI route | Rendered | Data-backed end-to-end |
|---|---|---:|---:|
| Overview | `/` | Yes | No |
| Schedule board | `/schedule` | Yes | No; empty/no-op schedule source |
| Scheduling control | `/scheduling` | Yes | No; localStorage state |
| Attendance | `/attendance` | Yes | No; local React state |
| Faculty roster | `/faculty` | Registered | No evidence of server wiring |
| Master data | `/master-data` | Yes | No; localStorage state |
| Courses & lectures | `/courses` | Yes | No; empty local seed in this archive |
| Notes compliance | `/notes` | Yes | No; backend mutation exists but UI is not wired |
| Analysis dashboard | `/analytics` | Registered | No evidence of server wiring |
| Audit log | `/audit` | Registered | UI-only activity unless backend mutation is used |
| Live display | `/live` | Registered | Consumes local schedule state |
| Settings / users | `/settings` | Registered | Local access-user workflow; server user procedure differs |
| Login | `/login` | Registered | Platform login helper exists; not exercised without credentials/config |

## Recommended next steps

1. **Choose one source of truth:** wire all operational screens to tRPC/Drizzle, or explicitly label the app as a local preview.
2. Add query/mutation hooks for faculties, courses, mappings, resources, attendance, schedule slots, assignments, notes, and audit logs.
3. Remove client-side role elevation and plaintext localStorage passwords; enforce access exclusively through authenticated server sessions and server role checks.
4. Introduce a normalized multi-dashboard assignment table or join table and update the user APIs accordingly.
5. Implement real schedule generation and persist generated assignments transactionally, including lunch locks, resource conflicts, faculty limits, and replacement workflows.
6. Add integration tests against a test database or repository layer for date isolation, permissions, CRUD, validation, and persistence.
7. Replace duplicated frontend/backend validation with shared schemas and rule evaluation.
8. Add error/loading states for database/network failures and surface whether data is local, empty, or persisted.
9. Split the frontend bundle and fix the `/api/platform/config.js` script warning before production deployment.

## Local server URL

The app was served locally on port `4173` and verified through the sandbox public URL:

<https://4173-i41ecboqu6pn8mz8e772b-15e7684d.sg2.manus.computer/>
