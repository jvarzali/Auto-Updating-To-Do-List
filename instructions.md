# Smart To-Do — Project Specification

A to-do list for students that you can fill in by hand or set up to fill itself in. It pulls assignments from class websites, Canvas, and Google Calendar, and it creates recurring tasks on whatever schedule you set. Every task can have an assigned date and a due date, and you can look at your tasks in three ways: a checklist, a due-date calendar, and a timeline calendar.

---

## 1. Goals

- **One place for everything.** Manual tasks, imported assignments, and recurring chores all live in one list.
- **Little upkeep.** Once a source is connected, new assignments show up without the user re-entering them.
- **Clear deadlines.** Due dates are always visible, and anything overdue or coming up soon stands out.
- **Views for different needs.** A quick checklist for today, a calendar for deadlines, and a timeline that shows how long you have for each task.

## 2. Core Concepts

### 2.1 To-Do Item

| Field           | Type                | Notes                                                        |
|-----------------|---------------------|--------------------------------------------------------------|
| `id`            | string (UUID)       | Unique identifier                                            |
| `title`         | string              | Required                                                     |
| `description`   | string (markdown)   | Optional notes and links                                     |
| `assignedDate`  | date/datetime       | Optional. When the task becomes available or was assigned. If empty, the timeline view uses the creation date |
| `dueDate`       | date/datetime       | Optional. Drives the calendar and timeline views             |
| `status`        | enum                | `todo`, `in_progress`, `done`                                |
| `completedAt`   | datetime            | Set when the status changes to `done`                        |
| `priority`      | enum                | `low`, `medium`, `high`                                      |
| `tags`          | string[]            | Free-form labels                                             |
| `course`        | string / ref        | Optional link to a course (for color coding and filtering)   |
| `source`        | enum                | `manual`, `canvas`, `google_calendar`, `website`, `recurring` |
| `sourceRef`     | object              | External ID, URL, and last-synced timestamp, used to avoid duplicates |
| `recurrenceId`  | string              | Set if the item was created by a recurrence rule             |
| `userEdited`    | boolean             | `true` if the user changed an imported item (see §4.5)       |

### 2.2 Course / Category

Groups tasks and gives them a color. Created by hand or automatically from Canvas courses or calendar names.

| Field     | Notes                                  |
|-----------|----------------------------------------|
| `id`      |                                        |
| `name`    | e.g. "CS 101"                          |
| `color`   | Used in every view                     |
| `sources` | Linked integrations (Canvas course ID, calendar ID, URL) |

### 2.3 Source (Integration)

A connected feed the app checks on a schedule. See §4.

### 2.4 Recurrence Rule

A template that creates new to-dos on a schedule. See §5.

---

## 3. Manual Entry

- **Quick add bar:** type a title and press Enter. Optional natural-language parsing:
  - `Read ch. 4 due fri` sets the due date to the coming Friday
  - `Lab report #CS101 !high` adds a course tag and high priority
- **Full editor:** every field from §2.1, including date pickers for the assigned and due dates.
- **Edit, delete, duplicate, and mark complete** from any view.
- **Bulk actions:** select several items to complete, delete, re-tag, or reschedule them at once.

---

## 4. Automatic Import

Every source has:
- An **enable/disable** toggle
- A **sync interval** (e.g. every 30 min, hourly, daily) plus a manual "Sync now" button
- A **default course/tag** for imported items
- A **last synced** status and an error message if the last sync failed

**When syncs run:** syncing happens in the browser, only while the app is open: once at launch, then on each source's interval. The serverless functions only act as a proxy (OAuth token exchange, and fetching Canvas, iCal, and website data to get around CORS). They never store items or run on their own schedule, and all imported items are saved to local IndexedDB.

**Date-only due dates:** if an imported due date has no time (e.g. "Oct 31" or an all-day calendar event), it is due at **11:59 PM** that day in the user's time zone.

**Any student can set this up:** nothing is hard-coded to one school. The Canvas instance URL, calendars, and website rules are all configured by the user.

### 4.1 Canvas LMS

- **Auth:** a personal access token (simplest), or OAuth2 if the school's Canvas instance allows developer keys.
- **Instance URL:** e.g. `https://school.instructure.com`.
- **Data pulled:**
  - Active courses (`GET /api/v1/courses?enrollment_state=active`)
  - Assignments per course (`GET /api/v1/courses/:id/assignments`), using `due_at` as the due date and `unlock_at` (or `created_at`) as the assigned date
  - Optional: quizzes, discussions, and planner items (`GET /api/v1/planner/items`)
- **Completion:** optionally mark an item done automatically once Canvas shows it as submitted.

### 4.2 Google Calendar

- **Auth:** Google OAuth2 with the `calendar.readonly` scope.
- **Setup:** pick which calendars to import from.
- **Mapping:** event end time becomes the due date; event start time (or the creation date) becomes the assigned date.
- **Filters:** import every event, or only events whose title matches a keyword or regex (e.g. `due`, `HW`, `exam`).
- **Optional two-way sync (later phase):** write manual to-dos with due dates back to a chosen calendar.

### 4.3 Class Websites

Many courses post assignments on a plain web page instead of an LMS.

- **Input:** the page URL plus how to find assignments on it:
  - **iCal/RSS feed** if the site has one (most reliable)
  - **CSS selector rules** for the title, due date, and link (e.g. `table.schedule tr`, then `td:nth-child(2)`)
  - **Point-and-click picker (stretch goal):** the user clicks an example assignment and the app works out the selectors
- **Date parsing:** handles common formats ("Oct 31", "10/31/2026", "Due Friday").
- **Change detection:** only new or changed rows create or update items.
- **Be polite:** respect `robots.txt`, check at most hourly by default, and send a clear user agent.

### 4.4 Generic iCal (.ics) Feeds

Many tools (Blackboard, Moodle, Outlook, Notion) can export an `.ics` URL. One generic iCal importer covers all of them.

### 4.5 Deduplication and Updates

- Each imported item is matched by `source` and `sourceRef.externalId`.
- On re-sync:
  - If the external item changed (new due date, new title), update the local item **unless** `userEdited` is true. In that case, flag the conflict and let the user choose which version to keep.
  - If the external item was deleted, mark the local item "removed at source" instead of deleting it silently.
- Completing an imported item locally never deletes it, and later syncs never un-complete it.

---

## 5. Recurring / Scheduled To-Dos

Users can create rules that generate to-dos automatically.

### 5.1 Rule Types

| Type             | Example                                                  |
|------------------|----------------------------------------------------------|
| Weekly on days   | "Discussion post" every **Mon & Wed** at 11:59 PM        |
| Every N days     | "Water plants" every **3 days**                          |
| Every N weeks    | "Lab report" every **2 weeks** on Thursday               |
| Monthly          | "Pay rent" on the **1st** of each month                  |
| Custom (RRULE)   | Advanced users can write an iCal RRULE string            |

### 5.2 Rule Fields

- `title`, `description`, `course`, `tags`, `priority`: copied onto each generated item
- `startDate`, optional `endDate` or occurrence count (e.g. "until Dec 12", the end of the semester)
- `dueTime`: time of day each item is due
- `leadTime`: how far ahead of the due date the item appears (its assigned date). Example: show up 2 days before it's due
- `generateAhead`: how many future occurrences to create at once (default: next 1–2)
- **Skip dates:** exclude holidays or breaks (e.g. Thanksgiving week)

### 5.3 Behavior

- Each occurrence is its own to-do, so completing one doesn't affect the others.
- Editing a rule offers: **this occurrence only**, **this and future**, or **all**.
- Pausing a rule stops it from generating new items but keeps the existing ones.

---

## 6. Views

All views share the same **filters** (course, tag, source, status, priority) and a **search** box. Items are colored by course.

### 6.1 Checklist View

A simple list you can scan quickly.

- Each row: checkbox, title, course color chip, due date (relative, e.g. "Tomorrow" or "in 3 days"), and source icon.
- **Grouping options:** by due date (Overdue / Today / Tomorrow / This Week / Later / No Date), by course, or no grouping.
- **Sorting:** due date, priority, assigned date, title.
- Overdue items are highlighted, and completed items can be hidden or collapsed at the bottom.
- Drag and drop to reorder manually within a group.

```
OVERDUE
 [ ] Problem Set 3            CS 101    ● 2 days ago
TODAY
 [ ] Read Ch. 5               HIST 210  ● 11:59 PM
 [x] Discussion post          ENG 102   ● 5:00 PM
THIS WEEK
 [ ] Lab report               CHEM 110  ● Thu
```

### 6.2 Due-Date Calendar View

A standard month or week calendar where each to-do shows up **on the day it's due**.

- **Month / week / day** toggle.
- Each day cell lists its items as colored chips. If there are too many, show "+N more" and expand on click.
- Click an item to open its details, or click an empty day to add a to-do due that day.
- Drag an item to another day to change its due date.
- Today is highlighted, and past days with unfinished items show a warning marker.

```
        Mon        Tue        Wed        Thu        Fri
      ┌──────────┬──────────┬──────────┬──────────┬──────────┐
      │ 27       │ 28       │ 29       │ 30       │ 31       │
      │ ■ PS 4   │          │ ■ Disc.  │ ■ Lab    │ ■ Essay  │
      │          │          │ ■ Quiz 2 │          │          │
      └──────────┴──────────┴──────────┴──────────┴──────────┘
```

### 6.3 Timeline Calendar View (Assigned → Due Bars)

A Gantt-style calendar where each to-do is a **horizontal bar from its assigned date to its due date**. This shows how long you have for each task and where tasks overlap.

- **Week / 2-week / month** zoom levels, with horizontal scrolling through time.
- One bar per to-do, colored by course. Bars are stacked into rows so they don't overlap.
- Optional **swimlanes** by course.
- Bar styling:
  - Solid for in progress or upcoming, faded or striped once completed
  - Red outline (or red end cap) if overdue
  - A "today" line drawn vertically across the timeline
- Items with no assigned date start at their creation date. Items with no due date are listed in a side panel instead of drawn as bars.
- Drag either end of a bar to change the assigned or due date, or drag the whole bar to shift both.
- Hover a bar for details, click it to open the editor.

```
              Oct 26   27   28   29   30   31   Nov 1
              │    │    │    │    │    │    │
Essay  (ENG)  ████████████████████████████▌        ← due Fri
PS 4   (CS)        ██████▌
Lab    (CHEM)                ████████████▌
Quiz 2 (CS)                  ███▌
                        ▲ today
```

---

## 7. Notifications and Reminders (Recommended)

- Configurable reminders before the due date (e.g. 1 day and 1 hour before).
- A daily digest: "You have 3 things due today, 1 overdue."
- Delivered as browser notifications, with optional email or mobile push later.

---

## 8. Settings

- Time zone and week start day (Sunday or Monday)
- Default view on launch
- Default reminder times
- Semester start and end dates (used as defaults for recurrence end dates and skip dates)
- Manage integrations (§4) and recurrence rules (§5)
- Data export/import (JSON, CSV, .ics)

---

## 9. Suggested Architecture

| Layer           | Suggestion                                                         |
|-----------------|--------------------------------------------------------------------|
| App type        | Web app, installable as a PWA                                      |
| Frontend        | React + Tailwind CSS                                               |
| Calendar UI     | FullCalendar (month/week grid) and a timeline/Gantt lib (e.g. `vis-timeline`, `frappe-gantt`) or a custom SVG/CSS grid |
| State           | Zustand                                                            |
| Storage (local) | IndexedDB (via Dexie). Local-first: no login in the MVP            |
| Backend         | Serverless functions (e.g. Vercel / Netlify / Cloudflare) used only as a proxy: OAuth token exchange and fetching Canvas, iCal, and website data (avoids CORS limits). No scheduled jobs and no server storage |
| Database        | None for the MVP. Add one (PostgreSQL or Firestore) with accounts and cloud sync in a later phase |
| Scheduling      | Runs on the client: source syncs and recurrence generation happen at launch and on timers while the app is open |
| Dates           | `date-fns` or `luxon` for time zones; `rrule` for recurrence       |
| Parsing         | `ical.js` for .ics feeds; `cheerio` for HTML scraping; `chrono-node` for natural-language dates |

> **Note:** Canvas tokens and Google OAuth refresh tokens are sensitive. Store them encrypted on the server side, or in the browser's secure storage, and never in plain localStorage.

---

## 10. Roadmap

### Phase 1: MVP
- [ ] To-do data model and local storage
- [ ] Manual add / edit / delete / complete
- [ ] Assigned and due dates
- [ ] Checklist view with grouping and sorting
- [ ] Due-date calendar view (month/week)

### Phase 2: Scheduling
- [ ] Recurrence rules (weekly days, every N days/weeks, monthly)
- [ ] Skip dates and end dates
- [ ] Timeline (assigned → due bars) view

### Phase 3: Integrations
- [ ] Generic iCal feed import
- [ ] Canvas import (personal access token)
- [ ] Google Calendar import (OAuth)
- [ ] Deduplication and conflict handling

### Phase 4: Polish
- [ ] Class website scraper with selector rules
- [ ] Reminders and notifications
- [ ] Natural-language quick add
- [ ] Drag to reschedule in both calendar views
- [ ] Export/import

### Phase 5: Stretch
- [ ] Point-and-click website selector picker
- [ ] Two-way Google Calendar sync
- [ ] Accounts and cloud sync across devices
- [ ] PWA push notifications
- [ ] Auto-complete from Canvas submission status
- [ ] Time estimates and a "suggested work plan" that spreads work across the assigned → due window

---

## 11. Decisions

| Question | Decision |
|----------|----------|
| What kind of app? | **Web app / PWA.** Class-website scraping runs through serverless functions to get around CORS. |
| Single user or accounts? | **Local-first.** The MVP stores data in IndexedDB with no login. Accounts and cloud sync come in a later phase. |
| Can imported items be edited? | **Fully editable**, with conflict handling as in §4.5. |
| Frontend stack? | **React + Zustand + Tailwind CSS.** |
| Backend? | **Serverless functions as a stateless proxy** for OAuth and cross-origin fetching. |
| When do syncs run? | **Only while the app is open:** at launch, then on each source's interval. Nothing syncs in the background while the app is closed. |
| Is the assigned date required? | **Optional.** The timeline view falls back to the creation date. |
| Imported dates with no time? | Due at **11:59 PM** that day, in the user's time zone. |
| Who is it for? | **Any student**, not just one person, so it must work with any Canvas instance and be easy to set up. |
