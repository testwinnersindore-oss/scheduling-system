# Winners Connect PDF-aligned implementation plan

## Goal
Implement the attached scheduling-system requirements in the existing Winners Connect command center while preserving the current branded shell, server/tRPC/Drizzle stack, and password-first security model.

## Design direction
- **Design movement:** authoritative academic operations command center: dark-blue rail, Monza red actions, silver/light-gray data surfaces, compact high-contrast typography.
- **Core principles:** permanent source-of-truth records, date-aware scheduling, explicit permissions, visible validation, and mobile-first operator workflows.
- **Color philosophy:** `#162456` signals institutional trust; `#D0021B` marks actions/exceptions; white and silver surfaces keep dense tables readable.
- **Layout paradigm:** persistent workspace rail plus a date-aware command header and scrollable data panels/drawers; mobile routes use a single-column flow with horizontal data-table scroll only where necessary.
- **Signature elements:** Winners wordmark, red action controls, status strips for live/upcoming/cancelled, and import/export format cards.
- **Interaction philosophy:** bulk actions are previewed and validated before commit; destructive deletes require Master Admin; edits remain auditable; forms and drawers always scroll and keep submit controls reachable.
- **Animation:** restrained drawer slide/fade, hover elevation on action cards, no motion that interferes with dense operator scanning.
- **Typography:** existing Inter/Space Grotesk/DM Mono system, with dark bold labels and clear table headers.
- **Brand essence:** Winners Connect is the institute's permanent, date-wise academic operations source of truth; precise, secure, operational.
- **Brand voice:** direct and accountable. Examples: “Upload the template, review validation, then save permanently.” and “Every class has a time, owner, resource, and status.”

## Required outcomes from the PDF and request
1. Add class start/end time and duration fields to scheduling forms; show live, upcoming, and cancelled states.
2. Support schedule carry-forward for the next day, configurable time adjustments of 1 hour, 2 hours, or 1 day, and automatic generation for the next three days. Mark carried-forward classes distinctly and allow editing.
3. Replace the fixed lunch rule with a configurable no-fixed-lunch model. Enforce a 10-minute faculty setup/travel buffer before and after each class; allow overlaps only when the buffer rule is satisfied by explicit edits.
4. Add Studio/Hall/Online Room resource management with building/group, occupied/available state, and time-window visibility.
5. Make all forms and drawers scrollable, keep action buttons reachable on mobile, and retain dashboard-friendly dark/bold typography.
6. Add tailored search, sorting, and filtering across faculty, courses, resources, attendance, and schedule records; eliminate blank/non-displaying data states when permanent records exist.
7. Mark daily faculty present automatically, while supporting absent, half-day, attendance time windows, replacement workflows, and class-time adjustment.
8. Add working CSV/Excel-compatible exports everywhere an export action is shown, including current filtered view and date context.
9. Add permanent Master Data storage through Drizzle/tRPC for faculty, courses, mappings, resources, and rules. Records survive future dates/changes; only authorized Master Admin actions can edit/delete.
10. Add bulk Excel upload for Faculty, Courses, and Resources. Provide downloadable templates and a visible format guide. Validate required columns, types, duplicates, resource modes, and row-level errors before permanent save.
11. Enforce password-first access on every route. Master Admin uses the protected server password; Admin and other created users use their assigned ID/password and only receive their assigned dashboard permissions. No route should expose dashboard data before login.

## Implementation approach
- `server/scheduler.ts`: replace fixed lunch assumptions with `setupBufferMinutes` and configurable work window; add carry-forward cloning and 3-day generation helpers; retain deterministic resource/faculty validation.
- `drizzle/schema.ts` and migrations: add resource building/group/time-window fields, schedule carry-forward metadata, and durable import/audit metadata as needed without removing existing records.
- `server/routers.ts`: add protected bulk upsert/import-preview, permanent master-data queries/mutations, resource availability query, multi-day generation/carry-forward procedures, attendance auto-mark/adjustment procedures, and export-safe filtered query boundaries.
- `client/src/pages/MasterData.tsx`: add file upload with `xlsx` parsing, template downloads, format guide, validation preview, permanent save, sorting/filtering, and scroll-safe modal behavior.
- `client/src/pages/Scheduling.tsx`: add multi-day date range/carry-forward controls, class time/duration editor, buffer display, status filters, resource occupancy filters, and export of the filtered schedule.
- `client/src/pages/Home.tsx` and shared UI: remove fixed-lunch copy, add route-wide password gate/session handling, automatic daily-present behavior, working exports, and mobile-friendly scroll containers.
- `client/src/lib/export.ts`: centralize UTF-8 BOM CSV export, Excel-compatible filenames, filtered-view exports, and template generation.
- `client/src/lib/master-data.ts`: define canonical import column names, format descriptions, and conversion helpers.
- `server/*.test.ts`: cover buffer scheduling, no fixed lunch, 3-day carry-forward, permanent upserts, import validation, attendance states, resource occupancy, route security, and export formatting.

## Data formats shown in the website
- **Faculty.xlsx:** `employeeCode`, `name`, `initials`, `primarySubject`, `defaultMode` (`online`/`offline`/`hybrid`), `availability` (`08:00-20:00`), `replacementEligible` (`Yes`/`No`), `maxLectures`, `notes`.
- **Courses.xlsx:** `code`, `name`, `studentCount`, `subject`, `branch`, `mode` (`online`/`offline`/`hybrid`), `status` (`active`/`draft`/`archived`), `notes`.
- **Resources.xlsx:** `resourceCode`, `resourceName`, `type` (`hall`/`studio`/`online_room`), `buildingGroup`, `availabilityFrom`, `availabilityTo`, `status` (`available`/`live`/`maintenance`), `notes`.
- Accepted upload formats: `.xlsx`, `.xls`, `.csv`; first row must be the exact header row. The UI will show a downloadable template and row-level errors before commit.

## Project structure
- `client/src/components/SiteLoginPage.tsx`: password-first route gate and assigned-user login.
- `client/src/pages/Home.tsx`: shell, date context, route access, attendance and export actions.
- `client/src/pages/Scheduling.tsx`: multi-day schedule and class-time editor.
- `client/src/pages/MasterData.tsx`: permanent master data, bulk import/export, filters, and templates.
- `client/src/lib/export.ts`: reusable export/template helpers.
- `server/scheduler.ts`: pure schedule generation, buffers, carry-forward.
- `server/routers.ts`: durable queries/mutations and import validation.
- `drizzle/schema.ts`, `drizzle/*.sql`: permanent MySQL schema.
- `server/*.test.ts`: domain and integration coverage.

## Constraints
- No fixed lunch rule remains in UI, scheduler validation, or fallback copy.
- Preserve existing data; import is upsert-by-code/employeeCode/resourceCode and never silently deletes unrelated records.
- LocalStorage may remain only as a clearly labeled offline preview fallback; permanent saves go through the database.
- Do not commit secrets or password values.
- Keep the existing managed WebDev project, database, production routing, and password-first session cookie behavior.


## Scheduling adjustment rules (2026-10-06)
- If a mapped faculty member is absent, an offline lecture first searches for a free same-course mapped faculty member, then a free same-subject replacement; the replacement is marked in the schedule and audit trail.
- If an online lecture's mapped faculty is absent, it is deferred to the next day instead of being assigned to an unrelated replacement. Multi-day generation carries that deferred lecture into the next day.
- The scheduler enforces a maximum of 8 lectures per faculty per day, both for automatic generation and manual persisted schedule saves.
- Faculty receive a protected rest window at least every 3 hours; the scheduler will not continue a faculty's uninterrupted teaching block beyond that interval and preserves the 10-minute total class buffer (5 minutes before plus 5 minutes after).
