# TaskDock — Antigravity Agent Prompt
# Drop this file as AGENTS.md in your project root, or paste it directly into the Agent Manager.
# The PRD (TaskDock_PRD_v4.0.docx) should also be in the project root in a trusted folder.

---

## CONTEXT

You are a Senior Full Stack Engineer building **TaskDock** — an intelligent, multi-user productivity
and task management web application. The full Product Requirements Document is in this project
folder as `TaskDock_PRD_v4.0.docx`. Read it before doing any work. It is your single source of
truth for all requirements, data models, API contracts, and acceptance criteria.

If you cannot read the .docx, the key facts are summarised in this file.

---

## PRODUCT SUMMARY

TaskDock aggregates tasks from multiple sources (Jira, Notion, GitHub), intelligently schedules
them using Round Robin or Priority Queue strategies, alerts users before deadlines, and motivates
completion through XP, badges, and streaks. It is a multi-user application with Admin and Member
roles. Every user has their own isolated data and a personalised motivational dashboard.

---

## TECH STACK — NON-NEGOTIABLE

- **Framework**: Next.js 15 (App Router, React Server Components, TypeScript)
- **Styling**: Tailwind CSS 3.x + shadcn/ui
- **ORM**: Prisma 5.x
- **Primary DB**: Neon PostgreSQL (serverless) — local dev: Docker `postgres:16-alpine`
- **Cache / Sessions**: Upstash Redis — via `@upstash/redis` and `@auth/upstash-redis-adapter`
- **Auth**: NextAuth.js v5 with Google OAuth2 as the MVP provider
- **Validation**: Zod (shared schemas between client and server)
- **Drag & Drop**: dnd-kit
- **Charts**: Recharts 2.x
- **Animation**: Framer Motion
- **Data Fetching (client)**: SWR
- **Virtualised Lists**: react-window
- **Package Manager**: pnpm
- **Testing**: Vitest + Playwright
- **Deployment**: Vercel (Hobby — free tier)
- **Background Jobs**: Vercel Cron Jobs (vercel.json)
- **Real-time**: SSE via ReadableStream (Next.js Route Handlers)
- **Encryption**: Node.js crypto AES-256-GCM (for integration credentials at rest)

Do NOT use: SQLite, Turso, Supabase, MongoDB, Mongoose, Express, custom auth, WebSockets,
any paid infrastructure, or any library not listed here unless you explicitly confirm with me first.

---

## DATA MODEL — DECLARE ALL ON DAY 1

The full Prisma schema must be declared in Iteration 1 and migrated to Neon immediately.
Later iterations only add data — they never drop or rename columns.

```prisma
enum Role        { ADMIN  MEMBER }
enum TaskStatus  { PENDING  SCHEDULED  IN_PROGRESS  SNOOZED  COMPLETED  OVERDUE  ARCHIVED }
enum ScheduleMode{ RR  PQ }

model User {
  id           String    @id @default(uuid())
  name         String?
  email        String    @unique
  image        String?
  role         Role      @default(MEMBER)
  totalPoints  Int       @default(0)
  level        Int       @default(1)
  preferences  Json      @default("{}")
  createdAt    DateTime  @default(now())
  tasks        Task[]
  sources      IntegrationSource[]
  slots        ScheduleSlot[]
  rewards      RewardEvent[]
  notifications Notification[]
  auditLogs    AuditLog[] @relation("AdminLogs")
}

model Task {
  id               String       @id @default(uuid())
  userId           String
  sourceId         String?
  title            String
  description      String?
  status           TaskStatus   @default(PENDING)
  priority         Int          @default(3)
  deadline         DateTime?
  alertThreshold   Int          @default(30)
  projectTag       String?
  externalId       String?
  scheduledVersion Int          @default(0)
  schedulingMode   ScheduleMode?
  createdAt        DateTime     @default(now())
  updatedAt        DateTime     @updatedAt
  user             User         @relation(fields: [userId], references: [id])
  source           IntegrationSource? @relation(fields: [sourceId], references: [id])
  slots            ScheduleSlot[]
  rewards          RewardEvent[]
  notifications    Notification[]
}

model IntegrationSource {
  id             String   @id @default(uuid())
  userId         String
  type           String
  config         String   // AES-256-GCM encrypted JSON string
  pollIntervalMin Int     @default(15)
  lastPolledAt   DateTime?
  isActive       Boolean  @default(true)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  user           User     @relation(fields: [userId], references: [id])
  tasks          Task[]
}

model ScheduleSlot {
  id              String       @id @default(uuid())
  userId          String
  taskId          String
  slotStart       DateTime
  slotEnd         DateTime
  mode            ScheduleMode
  isCompleted     Boolean      @default(false)
  snapshotVersion Int          @default(0)
  user            User         @relation(fields: [userId], references: [id])
  task            Task         @relation(fields: [taskId], references: [id])
}

model RewardEvent {
  id           String   @id @default(uuid())
  userId       String
  taskId       String
  pointsEarned Int
  badge        String?
  reason       String
  awardedAt    DateTime @default(now())
  user         User     @relation(fields: [userId], references: [id])
  task         Task     @relation(fields: [taskId], references: [id])
}

model Notification {
  id        String   @id @default(uuid())
  userId    String
  taskId    String?
  type      String   // DEADLINE_ALERT | SYNC_COMPLETE | REWARD_GRANTED | SCHEDULER_RUN
  message   String
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id])
  task      Task?    @relation(fields: [taskId], references: [id])
}

model AuditLog {
  id           String   @id @default(uuid())
  adminId      String
  targetUserId String?
  action       String
  metadata     Json
  performedAt  DateTime @default(now())
  ipAddress    String?
  admin        User     @relation("AdminLogs", fields: [adminId], references: [id])
}
```

---

## FOLDER STRUCTURE — SET UP THIS WAY FROM THE START

```
taskdock/
├── app/
│   ├── (auth)/login/page.tsx          — Google sign-in page
│   ├── (dashboard)/page.tsx            — Motivational Dashboard (root route)
│   ├── board/page.tsx                  — Kanban Task Board
│   ├── integrations/page.tsx           — Integration settings
│   ├── settings/page.tsx               — User preferences
│   ├── admin/page.tsx                  — Admin Dashboard (role-gated)
│   └── api/
│       ├── tasks/route.ts              — GET, POST
│       ├── tasks/[id]/route.ts         — PATCH, DELETE
│       ├── integrations/route.ts       — GET, POST
│       ├── integrations/[id]/route.ts  — PATCH, DELETE
│       ├── scheduler/run/route.ts      — POST
│       ├── rewards/route.ts            — GET
│       ├── notify/route.ts             — GET (SSE)
│       ├── dashboard/stats/route.ts    — GET (KPI batch)
│       ├── dashboard/upcoming/route.ts — GET (next 5 slots)
│       ├── dashboard/burndown/route.ts — GET (weekly chart data)
│       ├── dashboard/quote/route.ts    — GET (daily quote)
│       ├── admin/users/route.ts        — GET
│       ├── admin/users/[id]/route.ts   — PATCH
│       └── cron/
│           ├── poll/route.ts           — POST (Vercel Cron)
│           ├── deadlines/route.ts      — POST (Vercel Cron)
│           └── scheduler/route.ts      — POST (Vercel Cron)
├── components/
│   ├── dashboard/                      — All dashboard zone components
│   ├── board/                          — TaskBoard, TaskCard, Column
│   ├── modals/                         — AddTask, EditTask, RewardCelebration
│   ├── admin/                          — UserTable, AuditLogTable
│   └── ui/                             — shadcn/ui re-exports
├── lib/
│   ├── prisma.ts                       — Prisma client singleton + userId middleware
│   ├── auth/
│   │   ├── config.ts                   — NextAuth config (Google provider)
│   │   └── roles.ts                    — isAdmin(), requireRole() utilities
│   ├── integrations/
│   │   ├── types.ts                    — IntegrationAdapter interface + RawTask type
│   │   ├── registry.ts                 — Adapter registry
│   │   ├── adapters/jira.ts
│   │   ├── adapters/notion.ts
│   │   └── adapters/github.ts
│   ├── scheduler/
│   │   ├── full-replan.ts              — Full re-plan algorithm
│   │   └── incremental.ts              — Incremental slot assignment (Iteration 3)
│   ├── rewards/
│   │   └── engine.ts                   — XP calculation + badge evaluation
│   ├── crypto.ts                       — AES-256-GCM encrypt/decrypt for integration configs
│   └── validations/
│       └── task.schema.ts              — Zod schemas
├── data/
│   └── quotes.json                     — 365 daily motivational quotes
├── middleware.ts                        — Session + role enforcement on all routes
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── docker-compose.yml                  — Local Postgres + Redis
├── vercel.json                         — Cron job definitions
└── .env.local.example                  — All required env vars documented
```

---

## ENVIRONMENT VARIABLES

Create `.env.local.example` with these — I will fill in real values:

```env
DATABASE_URL=postgresql://taskdock:taskdock@localhost:5432/taskdock    # local Docker
# Prod: postgresql://[user]:[pass]@[host]/[db]?sslmode=require

REDIS_URL=https://[name].upstash.io
UPSTASH_REDIS_REST_TOKEN=

NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=                    # openssl rand -base64 32

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

INTEGRATION_ENCRYPT_KEY=            # 32-byte hex: openssl rand -hex 32

RESEND_API_KEY=                     # Phase 2 - email magic link
NEXT_PUBLIC_SWR_INTERVAL=30000
CRON_SECRET=                        # Shared secret for /api/cron/* route auth
```

---

## MULTI-USER & SECURITY RULES — ALWAYS ENFORCE THESE

1. **Every Prisma query** in API routes must include `WHERE userId = session.user.id`.
   Use a Prisma middleware in `lib/prisma.ts` that auto-injects this for all model queries.
2. **Next.js middleware** at `middleware.ts` must validate the session on every `api/*`
   and `admin/*` request before it reaches any Route Handler. Return 401 if no session,
   403 if wrong role.
3. **Admin routes** (`/admin`, `/api/admin/*`) require `role === 'ADMIN'` — reject with 403 otherwise.
4. **PostgreSQL RLS policies** must be applied after migration for the Task, IntegrationSource,
   ScheduleSlot, RewardEvent, and Notification tables.
5. **Integration credentials** must be AES-256-GCM encrypted before writing to DB,
   decrypted only at poll time. Never return raw credentials to the client.
6. **Zod validation** on every API route input — reject invalid payloads with 400 before
   any DB operation.

---

## ITERATIVE DELIVERY — BUILD IN THIS ORDER

### ITERATION 1 — Usable MVP (Weeks 1–3)
**Goal**: A fully deployed, multi-user task management app on Vercel. Real users can sign in,
use it, and benefit from it immediately.

**Build in this order:**
1. Project scaffold: Next.js 15 + TypeScript + Tailwind + shadcn/ui + pnpm + Docker Compose
2. Full Prisma schema (all models) + first migration to Neon
3. NextAuth.js v5 + Google OAuth2 + Upstash Redis session adapter
4. Next.js middleware (session + role enforcement)
5. Task CRUD API (`/api/tasks`, `/api/tasks/[id]`) with Zod + Prisma userId middleware
6. Kanban Task Board UI (`/board`) — 4 columns, dnd-kit drag-drop, task cards
7. Add/Edit Task modals
8. Admin Dashboard skeleton (`/admin`) — user list, role promotion, deactivation
9. Motivational Dashboard (`/`) — Hero Strip, 5 KPI cards, Focus Now (earliest PENDING task),
   Upcoming list (5 PENDING tasks sorted by createdAt), daily quote, level progress panel at 0 XP
10. Deploy to Vercel — Git tag `v1.0.0`

**Release criteria (all must pass before Iteration 2):**
- Any user signs in with Google → sees personalised dashboard with greeting, 5 KPI cards,
  Focus Now card, and daily quote
- Two users cannot see each other's tasks or dashboard data
- User can create, edit, prioritise, delete tasks; tasks appear on Kanban board
- Admin can view all users, promote/demote, deactivate accounts
- App is live at Vercel production URL

---

### ITERATION 2 — Integrated Product (Weeks 4–6)
**Goal**: Everything from Iteration 1 plus automatic task aggregation from real work tools.

**Build on top of Iteration 1:**
1. `IntegrationAdapter` interface + `RawTask` type in `lib/integrations/types.ts`
2. Jira adapter (`lib/integrations/adapters/jira.ts`) — REST API v3 + API token
3. Vercel Cron poll handler (`/api/cron/poll`) — iterates active sources, upserts tasks,
   deduplicates by `externalId + sourceId`, uses Redis SET TTL for notification dedup
4. Integrations settings page (`/integrations`) — add/edit/delete/enable/disable sources,
   manual "Sync Now" button
5. Notion adapter + GitHub Issues adapter
6. Admin: integration health panel (last poll time per source, no credentials exposed)
7. Dashboard update: Hero Strip shows "Last synced X min ago", KPI cards include integrated tasks
8. Deploy — Git tag `v2.0.0`

**Release criteria:**
- Jira tasks appear on board + KPI cards within one 15-min poll cycle, no duplicates
- Dashboard Hero Strip shows accurate last-sync timestamp
- "Tasks Due Today" counts tasks from all sources
- User can add/configure/disable/delete integrations

---

### ITERATION 3 — Smart Scheduling Product (Weeks 7–9)
**Goal**: Everything from Iteration 2 plus intelligent scheduling and real-time deadline alerts.

**Build on top of Iteration 2:**
1. Full re-plan scheduler (`lib/scheduler/full-replan.ts`) — both RR and PQ modes
2. Incremental scheduler (`lib/scheduler/incremental.ts`) — uses `task.scheduledVersion`
   and Redis key `scheduler:version:{userId}`
3. Scheduler API (`/api/scheduler/run`) — accepts `{ mode: 'rr'|'pq', strategy: 'full'|'incremental' }`
4. Scheduler mode toggle in board toolbar — persists to `user.preferences`
5. Vercel Cron deadline checker (`/api/cron/deadlines`) — every 5 min, Redis SET dedup
6. SSE endpoint (`/api/notify`) — ReadableStream, per-user scoped, events:
   `DEADLINE_ALERT | SYNC_COMPLETE | REWARD_GRANTED | SCHEDULER_RUN`
7. Task card urgency UI — amber border (< 2× threshold), red border + Framer Motion
   pulse animation + countdown ticker (< threshold)
8. Notification history sidebar drawer — lists NOTIFICATION records, mark-as-read
9. Dashboard update: Focus Now card powered by `ScheduleSlot.slotStart`, Upcoming Schedule
   list shows time-boxed slots, Weekly Burndown chart goes live (`/api/dashboard/burndown`)
10. Deploy — Git tag `v3.0.0`

**Release criteria:**
- RR mode distributes slots evenly across project tags
- PQ mode orders by `priority×10 + urgency_bonus` correctly
- Deadline alert fires as browser toast within 30s of cron evaluation
- Focus Now shows task with earliest `ScheduleSlot.slotStart`
- Burndown chart renders correct actual vs ideal lines for current ISO week

---

### ITERATION 4 — Gamified Production Product (Weeks 10–12)
**Goal**: Everything from Iteration 3 plus full gamification, analytics, polish, and production hardening.

**Build on top of Iteration 3:**
1. Reward engine (`lib/rewards/engine.ts`):
   - XP = 10 (base) + 5 (early, if completed > alertThreshold before deadline)
            + min(streak_days × 3, 15) (streak bonus from Redis key TTL 48h)
   - Level up every 100 XP — update `user.totalPoints` + `user.level`
   - Streak key: Redis `streak:{userId}` TTL 48h — increment on task completion
2. Badge evaluation — conditions: Speed Demon, On the Clock, Project Juggler, Streak Starter,
   On Fire, Clean Sweep, First Blood (Iteration 1), Power Admin
3. Reward celebration modal — Framer Motion: XP gained, badge (if any), level + animated progress bar
4. Dashboard full analytics: BarChart (tasks/day last 7 days), PieChart (tasks by project tag),
   animated XP progress bar with Framer Motion fill, badge chips, streak flame pulse
5. Upstash Redis rate limiter on all API routes — sliding window 100 req/min per userId
6. First-login onboarding modal — 3 steps: connect integration → add task → see scheduled slot
7. Dark mode — full `dark:` Tailwind coverage, system preference detection + manual toggle
8. Admin Audit Log UI — paginated, filterable by action type, date, user
9. Playwright E2E tests — all smoke tests in Section 12 of PRD
10. PostgreSQL RLS policy hardening review
11. README.md — complete local setup guide, env vars, migration steps, Vercel deploy
12. Deploy — Git tag `v4.0.0` — this is the v1.0 production release

**Release criteria (v1.0 production release):**
- Task completion → correct XP, possible badge, level progression with animation
- Streak increments on consecutive days, resets on missed day
- Dashboard all 6 zones load < 1.5s LCP, KPI revalidate 30s, CLS < 0.1
- Dashboard responsive: 3-col ≥ 1280px, 2-col 768–1279px, 1-col < 768px
- Dark mode: no WCAG AA contrast violations
- All Playwright E2E tests pass on production Vercel URL
- README: new dev can run locally in < 15 minutes

---

## DASHBOARD — 6 ZONES

Build these as independent React components in `components/dashboard/`:

| Zone | Component | Data source | Active from |
|------|-----------|-------------|-------------|
| Top Nav | `TopNav.tsx` | session | Iteration 1 |
| Hero Strip | `HeroStrip.tsx` | session + `/api/dashboard/stats` | Iteration 1 |
| KPI Cards (×5) | `KpiCards.tsx` | `/api/dashboard/stats` SWR 30s | Iteration 1 |
| Focus Now | `FocusNow.tsx` | `/api/dashboard/upcoming` | Iteration 1 (→ Iteration 3) |
| Upcoming List | `UpcomingList.tsx` | `/api/dashboard/upcoming` react-window | Iteration 2 |
| Analytics Row | `AnalyticsRow.tsx` | burndown + stats | Iteration 3 (→ Iteration 4) |

KPI cards: Tasks Due Today (blue), Overdue (red), XP This Week (purple),
           Day Streak (orange), Total Completed (green).

---

## SCHEDULER ALGORITHMS

### Full Re-Plan (active from Iteration 1 seed, default in Iterations 1–2):
```
1. DELETE ScheduleSlots WHERE userId = :uid AND isCompleted = false
2. FETCH Tasks WHERE userId = :uid AND status IN (PENDING, SCHEDULED)
3. IF mode = RR: group by projectTag, interleave (round-robin across groups)
   IF mode = PQ: sort by score DESC where score = priority*10 + urgency_bonus
4. Assign ScheduleSlots: slotStart = now + index * avgTaskDuration
5. SET task.scheduledVersion = currentVersion for all re-planned tasks
```

### Incremental (Iteration 3+, strategy: 'incremental'):
```
1. READ currentVersion from Redis key 'scheduler:version:{userId}' (or 0)
2. FETCH Tasks WHERE userId = :uid AND scheduledVersion < currentVersion
3. FIND last existing ScheduleSlot.slotEnd as insertion anchor
4. Apply RR or PQ ordering to NEW tasks only
5. INSERT ScheduleSlots from anchor
6. SET task.scheduledVersion = currentVersion
7. INCR Redis key 'scheduler:version:{userId}'
```

---

## KEY RULES FOR THE AGENT

1. **Always read the PRD first** before starting any task. It is the authority.
2. **Never start a new iteration** until all release criteria for the current one pass.
3. **Never use raw SQL** — Prisma ORM only, except for RLS policy DDL.
4. **Never expose secrets** to the client — credentials stay server-side only.
5. **Always use TypeScript** — no `any` types, no `// @ts-ignore`.
6. **Additive schema only** — never drop or rename existing columns after Iteration 1.
7. **Test business logic** — scheduler and reward engine must have Vitest unit tests.
8. **One file per feature** in `lib/` — keep it modular and independently testable.
9. **Commit at each deliverable** — clean commits with descriptive messages.
10. **Git tag at each iteration release** — `v1.0.0`, `v2.0.0`, `v3.0.0`, `v4.0.0`.

---

## START COMMAND

Begin with Iteration 1. Your first task:

> "Review the TaskDock PRD v4.0 in this project folder, then scaffold the full Next.js 15
> project with TypeScript, Tailwind CSS, shadcn/ui, pnpm, Prisma with the complete schema
> declared, Docker Compose for local Postgres and Redis, NextAuth.js v5 with Google OAuth2,
> and the Next.js middleware for session + role enforcement. Set up the full folder structure
> as specified. Do not add any features beyond what Iteration 1 requires. Once scaffolded,
> run the Prisma migration against the local Docker Postgres and confirm the schema is applied.
> Then confirm what needs to be done next in Iteration 1."
