# Winners Connect by Winners Institute Indore

React / Express / tRPC / Drizzle starter, adapted from the Sandbox web-db-user template.

- `pnpm dev`: development server; honors `PORT` (default 3000).
- `pnpm build` / `pnpm start`: build and serve `dist/index.js` and `dist/public/`.
- `pnpm db:migrate`: apply checked-in migrations. `pnpm db:push`: generate and apply new schema changes.
- `pnpm check` / `pnpm test`: types and application tests.

Start with the Webdev skill's default-template guide. Platform login, storage, payments and service contracts live in its shared references; read the relevant capability before extending its helper.

`server/_core/publicConfig.ts` exposes only named public runtime values. Private keys stay server-side. The platform serves managed `/manus-storage/` assets; the application does not register a second proxy.

Platform configuration is readable and editable through `webdev.config`. Default settings are initial values, not enforced constraints. The agent may modify the files, commands and configuration or follow the flexible guide for another stack.

## Database-backed scheduling

Scheduling Control reads date-scoped entries through `winners.schedule`, while automatic generation uses `winners.generateSchedule`. With `DATABASE_URL` configured, apply the checked-in `drizzle/0004_schedule_entries.sql` migration before using automatic generation:

```bash
pnpm db:migrate
```

The generator uses active faculty, courses, faculty-course mappings, attendance, and available resources. It preserves the 08:00–20:00 working window, the protected 13:00–13:40 lunch lock, resource-mode rules, absent-faculty exclusions, and the eight-lecture daily limit. Without a database/session, the UI remains usable as an explicit local preview and labels schedule changes as local rather than persisted.
