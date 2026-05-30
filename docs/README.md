# Auti - Personal Task Tracker

**Auti** is a personal task-tracking SPA (Single Page Application) built with:

- **SvelteKit 5** + **TypeScript**
- **TailwindCSS v4**
- **Supabase** (PostgreSQL + Auth)
- **Cloudflare Workers** (deployment)

## What It Does

Auti helps you track tasks with time sessions and recurring schedules. It organizes tasks into **categories**, each with a goal (times per period or duration per period). Tasks can be:

- **Undated** - one-off tasks that appear every day until completed
- **Scheduled** - one-off tasks that appear on a specific date
- **Recurring** - daily/weekly/monthly repeating tasks

You can:

- Start/stop timers on any task
- Mark tasks as done or skipped for the day
- Track progress toward category goals
- View history of past completions

## Core Concepts

### Categories

A category groups related tasks and defines a goal:

- **Goal type**: `times` (count completions) or `seconds` (track duration)
- **Goal period**: `week`, `month`, or `year`
- **Goal value**: target number (e.g., 5 times per week, 2 hours per month)

### Tasks

Each task belongs to a category and has:

- **Name** and optional **description**
- **Repeat frequency**: `daily`, `weekly`, `monthly`, or `once`
- **Repeat interval**: every N days/weeks/months
- **Weekday mask**: 7-bit mask for weekly tasks (bit0=Mon)
- **Month-day mask**: 31-bit mask for monthly tasks
- **Start/End dates**: when the task becomes active/inactive
- **Completed at**: timestamp when a once-task was marked done (never shows again)

### Sessions

Time tracking for tasks. Each session has:

- `started_at`: Unix timestamp
- `ended_at`: Unix timestamp (null if running)

### History

Daily status log for tasks:

- `date`: YYYY-MM-DD string
- `status`: `done` or `skipped`

---

# Architecture

## File Structure

```
src/
├── lib/
│   ├── auth.svelte.ts          # GitHub OAuth singleton
│   ├── supabase.ts             # Supabase client
│   ├── types/index.ts           # TypeScript interfaces
│   ├── utils/index.ts           # Helpers (cn() for classes)
│   ├── models/                  # Svelte 5 reactive stores (classes)
│   │   ├── categories.svelte.ts
│   │   ├── tasks.svelte.ts
│   │   ├── sessions.svelte.ts
│   │   ├── history.svelte.ts
│   │   └── progress.svelte.ts
│   └── components/
│       ├── Modal.svelte         # Reusable modal shell
│       ├── TaskModal.svelte     # Task create/edit (3 tabs: Edit/Sessions/History)
│       ├── CategoryModal.svelte # Category create/edit
│       └── ui/                  # UI primitives
│           ├── button/
│           └── avatar/
└── routes/
    ├── (auth)/                 # Auth pages
    │   └── auth/+page.svelte  # GitHub sign-in
    └── (protected)/            # Protected pages
        ├── +layout.svelte     # Nav bar, clock, dark mode
        ├── +page.svelte       # TODAY view (main dashboard)
        ├── profile/+page.svelte
        └── categories/
            ├── +page.svelte    # Categories list with progress bars
            └── [id]/+page.svelte # Single category + its tasks
```

## Data Flow

1. **Auth** → `auth.svelte.ts` holds the user session
2. **Stores** (`$lib/models/*.svelte.ts`) are Svelte 5 class-based reactive stores:
   - Each store has `$state` for data
   - Methods for CRUD via Supabase
   - Exported as singleton instances (`export const categories = new CategoryStore()`)
3. **Pages** import stores and use `$derived` to compute derived state
4. **Components** receive data via props, emit events via `onclose` callbacks

## Routing

| Route              | Description                                                             |
| ------------------ | ----------------------------------------------------------------------- |
| `/`                | Today view - all tasks grouped by section (Undated/Recurring/Scheduled) |
| `/categories`      | List of categories with goal progress bars                              |
| `/categories/[id]` | Single category detail + its tasks                                      |
| `/profile`         | User profile (placeholder)                                              |
| `/auth`            | GitHub OAuth sign-in                                                    |

---

# Data Models

## Database Schema (PostgreSQL)

### category

| Column      | Type    | Description                |
| ----------- | ------- | -------------------------- |
| id          | UUID    | Primary key                |
| user_id     | UUID    | FK to auth.users           |
| name        | TEXT    | Category name              |
| color       | TEXT    | Hex color code             |
| goal_type   | VARCHAR | `times` or `seconds`       |
| goal_value  | INT     | Target number              |
| goal_period | VARCHAR | `week`, `month`, or `year` |

### task

| Column            | Type    | Description                           |
| ----------------- | ------- | ------------------------------------- |
| id                | UUID    | Primary key                           |
| category_id       | UUID    | FK to category                        |
| name              | TEXT    | Task name                             |
| description       | TEXT    | Optional notes                        |
| repeat_freq       | VARCHAR | `daily`, `weekly`, `monthly`, or null |
| repeat_interval   | INT     | Every N periods                       |
| repeat_weekdays   | INT     | 7-bit mask (bit0=Mon)                 |
| repeat_month_days | INT     | 31-bit mask (bit0=day1)               |
| start_date        | BIGINT  | Unix timestamp (midnight)             |
| end_date          | BIGINT  | Unix timestamp (midnight)             |
| completed_at      | BIGINT  | When once-task was completed          |

### session

| Column     | Type   | Description                      |
| ---------- | ------ | -------------------------------- |
| id         | UUID   | Primary key                      |
| task_id    | UUID   | FK to task                       |
| started_at | BIGINT | Unix timestamp                   |
| ended_at   | BIGINT | Unix timestamp (null if running) |

### history

| Column  | Type    | Description         |
| ------- | ------- | ------------------- |
| id      | UUID    | Primary key         |
| task_id | UUID    | FK to task          |
| date    | DATE    | YYYY-MM-DD          |
| status  | VARCHAR | `done` or `skipped` |

**Unique constraint**: `(task_id, date)` - one entry per task per day

---

# Key Functions

## `shouldFireOnDate(task, date)` - `src/lib/models/history.svelte.ts`

Determines if a task should appear on a given date:

| Task Type            | Logic                                     |
| -------------------- | ----------------------------------------- |
| Once undated         | Always true (until `completed_at` is set) |
| Once with start_date | True only on that exact date              |
| Daily                | Every N days from start_date              |
| Weekly               | Matches weekday mask, every N weeks       |
| Monthly              | Matches month-day mask, every N months    |

## `describeSchedule(task)` - `src/routes/(protected)/+page.svelte`

Returns human-readable schedule string:

- `daily, interval=1` → `"every day"`
- `daily, interval=3` → `"every 3 days"`
- `weekly, [Mon, Thu]` → `"Mon & Thu"`
- `weekly, interval=2` → `"Mon & Thu · 2wk"`
- `monthly, [1, 15]` → `"1st & 15th"`
- `once, start_date` → `"12 Mar"`

---

# UI Components

## Modal (shell)

- Reusable modal with title, color dot, backdrop click + Escape to close
- Optional footer snippet
- Configurable max-width and max-height
- Fade + scale transitions

## TaskModal

- **3 tabs**: Edit, Sessions, History
- **Edit tab**: name, description, category picker, repeat frequency, weekday/month-day pickers, date ranges
- **Sessions tab**: list of time sessions with live timer
- **History tab**: past 60 days of done/skipped entries

## CategoryModal

- Name, color swatches (8 colors)
- Goal type toggle (Times/Duration)
- Goal value inputs + period selector

---

# Development

## Commands

```bash
bun run dev      # Start dev server
bun run build    # Build for production
bun run check    # Type-check with svelte-check
bun run lint     # Run ESLint
```

## Environment Variables

```
SUPABASE_URL=your_supabase_url
SUPABASE_ANON_KEY=your_anon_key
```

## Deployment

The app deploys to **Cloudflare Workers** via Wrangler. See `wrangler.jsonc`.
