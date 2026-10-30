# Job Tracker

@AGENTS.md

## What this project is

A **personal job application tracker** built with **Next.js (App Router)**. I log the roles I apply for, move them across a Kanban board as they progress, and keep notes and a timeline for each one.

It's a **single-user app** (only I can log in), deployed on Vercel.

This is a **portfolio and learning project**. The goals are, in order:
1. **Learning Next.js properly.** I must be able to explain every part in an interview: Server vs Client Components, Server Actions, caching and revalidation, routing, auth.
2. **Learning PostgreSQL** (I haven't used it before) through Prisma.
3. A clean, well-tested, well-documented public repo that recruiters can look at.
4. An app I actually use during my job search.

## About me

- Senior full-stack engineer: TypeScript, Node.js, NestJS, React, AWS (Lambda, S3, CloudWatch).
- Used TypeORM at work, so Prisma concepts (models, migrations, relations) should map across. Relate to TypeORM where that helps.
- **Next.js is new to me** beyond docs and small practice projects. **PostgreSQL is new to me.**
- I'm used to a separate React SPA + Node/NestJS API. Explain where Next.js differs from that model.
- I use Claude Code day to day and I own the output: I review every line and define the test scenarios.

## How you should work with me

**I write the code. You guide and review.** Don't write or edit application code, tests, config or schema files for me unless I explicitly ask for a specific piece.

- **Work in small steps**, one milestone at a time. Don't jump ahead.
- **Start each milestone in plan mode:** outline the approach, the files I'll need to create or change, any packages to install, and the Next.js concepts involved, then wait for my OK before I start.
- **Guide, don't solve.** When I'm stuck, point me to the concept or the relevant docs section first, then give a hint. Only show a short example snippet if I ask for one, and keep it separate from my files.
- **Review what I write** when I ask (e.g. "review my changes"). Read the diff (`git diff` or the files I name) and give feedback grouped as:
  1. **Must fix:** bugs, security issues (auth or validation missing in a Server Action, server code leaking to the client, secrets, logging personal data)
  2. **Should fix:** Next.js or React anti-patterns, unnecessary `"use client"`, missing error/loading states, missing tests
  3. **Nice to have:** readability, naming, small simplifications
  For each point, say *why* and reference the file and line. Don't apply the fixes yourself; I'll make the changes and ask for a re-review.
- You may run read-only or checking commands (`git diff`, `npm run lint`, `npm test`, `npm run build`, type-check) to support a review, and tell me what failed and why.
- **Explain the why**, especially Next.js-specific behaviour: whether code runs on the server or the client, why a component needs `"use client"`, what gets cached and when it revalidates.
- At the end of each milestone, **ask me 2–3 short questions** to check I understand what we built.
- Keep code simple and readable over clever. Prefer fewer dependencies.
- Use the **current stable versions** of Next.js and Prisma, and check the official docs when an API may have changed. Don't rely on memory for version-specific details.
- Never commit secrets. Never print database URLs, tokens or keys in logs.

## Architecture

**One full-stack Next.js app. No separate backend.**

- **Reads:** Server Components query the database directly through Prisma.
- **Writes:** Server Actions (`"use server"`), with input validated by Zod, then `revalidatePath` to refresh affected pages.
- **Route Handlers** (`app/api/...`) only if something outside the UI needs an endpoint. None are planned for v1.
- **Client Components** only where interactivity needs them (drag and drop, modals, filters). Keep them small and pass server-fetched data in as props.
- Deployed to Vercel, where server code runs as serverless functions (similar to AWS Lambda).

Why no separate backend: it's a single-user CRUD app with one client. A separate API would add deploys, CORS and shared auth without a design reason. (Phase 2 below adds a NestJS service for background jobs, where a split *is* justified.)

## Technical decisions

| Area | Choice |
|---|---|
| Framework | Next.js, App Router, TypeScript (strict) |
| Styling | Tailwind CSS |
| Database | PostgreSQL, hosted free (Neon or Supabase; decide in M2) |
| ORM | Prisma (migrations committed to the repo) |
| Validation | Zod, shared between forms and Server Actions |
| Drag and drop | dnd-kit |
| Auth | Single user, GitHub OAuth, with an allowlist of my account only. Library decided in M3 after checking current docs (Auth.js or Better Auth). |
| Unit/component tests | Vitest + React Testing Library |
| E2E tests | Playwright, for the main flows only |
| Lint/format | ESLint + Prettier |
| Hosting | Vercel (preview deployment per PR) |

## Data model (initial design)

```
Application
  id           String   (cuid)
  company      String
  role         String
  jobUrl       String?
  location     String?
  workMode     REMOTE | HYBRID | ONSITE
  status       WISHLIST | APPLIED | INTERVIEWING | OFFER | REJECTED | WITHDRAWN
  appliedAt    DateTime?
  notes        String?
  createdAt    DateTime
  updatedAt    DateTime
  events       Event[]

Event                    (timeline entries)
  id            String   (cuid)
  applicationId String   -> Application (cascade delete)
  type          APPLIED | RECRUITER_CALL | INTERVIEW | TECH_TEST | OFFER | REJECTION | FOLLOW_UP | OTHER
  date          DateTime
  note          String?
```

Single user for v1, so there is no `userId`. Keep the schema easy to extend with one later.

## Repo structure (target)

```
job-tracker/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                  # redirect to /board
│   ├── board/page.tsx            # Kanban view
│   ├── applications/
│   │   ├── page.tsx              # list view with search and filters
│   │   ├── new/page.tsx
│   │   └── [id]/
│   │       ├── page.tsx          # detail + timeline
│   │       ├── edit/page.tsx
│   │       ├── loading.tsx
│   │       └── not-found.tsx
│   └── error.tsx
├── components/                   # UI components (mark client ones clearly)
├── lib/
│   ├── db.ts                     # Prisma client singleton
│   ├── actions/                  # Server Actions
│   └── validation.ts             # Zod schemas
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts                   # fake sample data only
├── tests/                        # Vitest
├── e2e/                          # Playwright
├── .github/workflows/ci.yml
├── .env.example                  # variable names only, no values
├── .nvmrc                        # Node version for nvm (24)
├── AGENTS.md                     # managed by Next.js, points to bundled docs
├── CLAUDE.md                     # this file
├── README.md
└── LICENSE                       # MIT
```

## Milestones

For each milestone: plan together → I implement → you review → I fix → you re-review → check-in questions.

- **M1: Scaffold and deploy.** Create the Next.js app (TypeScript, Tailwind, ESLint). A static `/board` page with hard-coded cards. Deploy to Vercel so the pipeline works from day one.
- **M2: Database.** Set up hosted Postgres and Prisma. Write the schema, run the first migration, seed fake data. Board and list pages read from the database in Server Components. Production holds fake seed data only.
- **M3: Auth.** GitHub login restricted to my account. Protect every page and every data read, and create one auth helper (e.g. `requireUser()`) that every Server Action from M4 onward calls first. Library decided at the start of M3 after checking current docs (Auth.js or Better Auth). No real data in production before M3 ships.
- **M4: Create and edit.** Forms for adding and editing applications using Server Actions (auth check first, then Zod validation), error messages and `revalidatePath`.
- **M5: Drag and drop.** Move cards between columns with dnd-kit (Client Component). Persist the new status via an authenticated Server Action, with an optimistic update.
- **M6: Detail page and timeline.** Dynamic route `/applications/[id]`, add/delete timeline events (authenticated actions), `loading.tsx`, `not-found.tsx`, `error.tsx`.
- **M7: Tests, CI and README.** Unit tests for validation and actions, including "unauthenticated call is rejected". Component tests for key UI, 2–3 Playwright flows (signed-in). CI runs lint, type-check, test and build. README with screenshots, setup steps and architecture notes.
- **M8 (optional):** Stats page (response rate, time per stage), list filters in the URL, CSV export.

### Phase 2 (later, separate repo or package)

A small **NestJS service built from scratch** for background work: scheduled follow-up reminders and AI pre-fill from a pasted job description. Not part of v1.

## Things to watch out for

- **Server Actions are public POST endpoints.** Check auth and validate input inside every action, even though the UI hides them.
- From M4 onward, every Server Action starts with the auth helper, then Zod validation. No exceptions, including "small" actions like status changes and event deletes.
- **Don't leak server-only code to the client.** Prisma and env vars stay in server files. Never import `lib/db.ts` into a Client Component.
- **This data is personal** (companies, recruiter names, interview notes). Don't log application contents, and never commit real data. The seed file uses fake companies only.
- **Prisma in development:** use a client singleton to avoid exhausting connections during hot reload.
- **Serverless + Postgres:** use the provider's pooled connection string for the app, and a direct connection for migrations, as the provider and Prisma docs recommend.
- Keep `.env` out of git. Commit `.env.example` with variable names only.

## Current setup (verified 2026-10-30)

- Next.js 16.4.3, App Router, Turbopack. Cache Components and Partial Prefetching are ON in `next.config.ts`: uncached data reads need `<Suspense>` or `"use cache"`.
- Check version-matched docs in `node_modules/next/dist/docs/` before relying on memory.
- Node 24: `.nvmrc` contains `24` (nvm format, used locally); `engines.node` is `24.x` (semver range, read by Vercel).
- Tailwind v4, CSS-first config, loaded via `@tailwindcss/turbopack` (no PostCSS config).
- ESLint 9 flat config. Keep the version `eslint-config-next` expects. `next build` does not lint.

## Commands

- `npm run dev` · `npm run build` · `npm run lint`
- `npm run typecheck` · `npm run format:check` (add these scripts with Prettier)

## Status

Current milestone: M1 (scaffold and deploy).