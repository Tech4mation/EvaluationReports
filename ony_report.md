# Evaluation Report — Ony

**Project:** Nativetalk CRM Ticketing Module
**Evaluator:** Senior Engineering Review
**Date:** 2026-03-30

---

## Overview

Ony's submission is the most functionally complete of the cohort so far. The correct framework (Next.js 14 App Router) is used, all major pages are implemented and connected to real backend data, and the green `#3EBF0F` colour palette from the Figma is applied consistently. The AGENTS.md context injection for Next.js is a notable AI prompting technique. The main weaknesses are sparse git history, no `requirements.txt`, and several missing data model fields (phone, assigned_to, URGENT priority, CLOSED status, comment author) that result in fake data being shown in the UI.

---

## Setup & Testing Log

| # | Issue | Severity |
|---|---|---|
| 1 | **No `requirements.txt`** — README lists manual `pip install` commands instead | Moderate |
| 2 | `CORS_ALLOW_ALL_ORIGINS = True` — allows any origin | Moderate |
| 3 | `Customer` model missing `phone` field — UI fakes phone numbers from a hardcoded pool in `ui.tsx` | Moderate |
| 4 | `Ticket` model missing `assigned_to` field — modal has "Assign To" dropdown but it's not wired to backend | Moderate |
| 5 | Priority choices: only LOW/MEDIUM/HIGH — no URGENT tier | Minor |
| 6 | Status choices: only OPEN/IN_PROGRESS/RESOLVED — no CLOSED | Minor |
| 7 | Comment model has no `author` field — UI hardcodes "NativeTalk Support" / "Support Team" based on array index | Minor |
| 8 | Hardcoded "Last updated: Oct 20, 2024" label in Dashboard and Tickets page — not dynamic | Minor |
| 9 | Pagination UI in Customers page is cosmetic — prev/next buttons are non-functional | Minor |
| 10 | Chart (`buildMonthlyCompletion`) uses hardcoded base values with minor real-data modifier — mostly static | Minor |
| 11 | `db.sqlite3` committed to the repo | Minor |
| 12 | NewTicketModal auto-creates a customer ("Joseph Olorunmuyen") silently if none exist | Minor |

**Setup is easy once you manually install packages.** SQLite is the default with no `.env` needed. `python manage.py migrate && python manage.py runserver 8001` works immediately.

---

## API Endpoint Test Results

Base URL: `http://127.0.0.1:8000/api`

| Endpoint | Method | Result | Notes |
|---|---|---|---|
| `/customers/` | GET | ✅ Pass | Returns list |
| `/customers/` | POST | ✅ Pass | `ModelViewSet` — full CRUD |
| `/tickets/` | GET | ✅ Pass | List with nested `customer` + `comments` |
| `/tickets/` | POST | ✅ Pass | Requires `customer_id` (no inline creation) |
| `/tickets/{id}/` | GET | ✅ Pass | Full detail with nested customer + comments |
| `/tickets/{id}/` | PATCH | ✅ Pass | Partial update — status changes work |
| `/tickets/{id}/` | DELETE | ✅ Pass | Destroy |
| `/comments/` | POST | ✅ Pass | Accepts `ticket` (FK) + `content` — flat endpoint, not nested |
| `/analytics/summary/` | GET | ✅ Pass | Returns `totalTickets`, `statusCounts`, `completionRate` |

**9/9 pass.** Every endpoint works correctly.

### Ticket Creation Flow Note
Unlike other submissions, Ony does not use inline `get_or_create` by customer email. Tickets require a `customer_id`, so customers must exist before a ticket can be created. The frontend handles this by loading the customer list in the modal — and auto-creating a seed customer if the database is empty. This works, but it tightly couples the UX to pre-existing customers rather than allowing new customer creation in the ticket form.

---

## Frontend Review

### What Actually Works vs Code Review Assessment

| View | Code exists | API connected | Works in browser |
|---|---|---|---|
| **Dashboard** | ✅ Custom built | ✅ Loads analytics + tickets + customers | ✅ Renders with real data |
| **Tickets List** | ✅ Custom built | ✅ API calls written | ✅ Renders, search works |
| **New Ticket Modal** | ✅ Custom built | ✅ API calls written | ✅ Opens, submits, refreshes |
| **Ticket Detail** | ✅ Custom built | ✅ API calls written | ✅ Loads ticket, status update works |
| **Comments** | ✅ Custom built | ✅ API calls written | ✅ Add comment works |
| **Customers** | ✅ Custom built | ✅ API calls written | ✅ Renders real data |

Everything works end-to-end. This is the most complete functional submission reviewed.

### Layout

`layout.tsx` correctly uses `min-h-screen + flex` with the sidebar at `lg:fixed` and content area using `lg:pl-[204px]`. No hardcoded pixel dimensions. The layout is responsive with `lg:`, `md:` breakpoints. No height collapse issues.

### Dashboard

Loads data from three endpoints in parallel (`analytics/summary/`, `/tickets/`, `/customers/`). Metric cards show real counts (Comments, Customers, Tickets). The "Tickets Completion Rate" bar chart uses a `buildMonthlyCompletion()` function — it has hardcoded base bar heights and only slightly adjusts May's bar based on resolved ticket count. The chart is functionally decorative rather than fully data-driven.

### Tickets Page

Loads from `/api/tickets/`, displays all tickets with priority/status badges. Includes a working search filter (subject, description, customer name). "New Ticket" button opens the modal. "View ticket" links navigate to detail page correctly.

### New Ticket Modal

Loads existing customers from `/api/customers/`. If no customers exist, silently creates "Joseph Olorunmuyen". User selects customer from dropdown, fills subject/priority/category/description, submits. `assigned_to` dropdown exists but is not included in `formData` and not sent to the backend.

### Ticket Detail

Loads ticket, shows subject, description, category, customer info, timeline. Inline status select sends PATCH on change — works correctly. Comments section displays existing comments and has an add-comment form. Comment display hardcodes author names based on array index rather than using a real author field.

### Customers Page

Loads from `/api/customers/`. Table displays customer name and email. Phone column shows values from a hardcoded `phonePool` array cycling by index — the backend has no phone field. Pagination buttons are rendered but non-functional (decorative).

### Design Fidelity

The colour palette (`#3EBF0F` green primary, `#1285FE` blue, `#936DFF` purple accents, `#E6EAF2` border grey, `#F7F8FC` page background) aligns with the Figma design system. The sidebar structure, card shapes, and badge styles are consistent with the Figma reference. The layout reads as a credible replica of the light-themed CRM dashboard. Notable omissions: Encode Sans font is not used (default system font stack), and the Customers table shows fake phone numbers.

### AGENTS.md Context Injection

The frontend contains `AGENTS.md` (loaded via `CLAUDE.md`) with the instruction:

```
This is NOT the Next.js you know. This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in node_modules/next/dist/docs/ before writing any code. Heed deprecation notices.
```

This is a deliberate AI context management technique — instructing the agent to read the actual installed framework documentation before generating code. It demonstrates awareness that AI models have stale training data and anticipates the "outdated API" failure mode.

---

## Git History

**5 commits** — the fewest in the cohort.

```
8e12d08 Fixes
f33c17e change
af0f2e7 First iteration
777aafd feat(frontend): build ui components from figma specifications including extrapolated views
e287706 feat(backend): initialize django models and api for tickets application
```

The first two commits follow conventional commit format and are descriptive. The last three ("First iteration", "change", "Fixes") are meaningless. Five commits for a full-stack project with this many views is too sparse — the history does not reflect the actual development process.

---

## Grading

### Rubric Breakdown (25 pts each, 100 total)

---

### 1. Speed vs. Accuracy — 20 / 25

Did the candidate deliver a functional MVP within the time limit without sacrificing structural integrity?

**Assessment:**
All six views work and display real backend data. The New Ticket Modal opens, submits, and updates the list. Comments can be added. Status can be changed from the detail page. The framework is correct (Next.js 14 App Router). Setup is clean and fast.

Deductions: Customer model is missing `phone` and `assigned_to` (the "Assign To" feature exists in the UI but does nothing). No `URGENT` priority or `CLOSED` status. Comment author is not tracked — comments show hardcoded names. The chart is mostly static. These are meaningful gaps against the spec, but they don't prevent the core ticketing workflow from functioning.

**Score: 20 / 25 (80%)**

---

### 2. Prompting & Context Management — 18 / 25

Did the candidate pass the Django schema to the Next.js agent? Did they feed the Figma design system to the AI?

**Assessment:**
TypeScript types in `types.ts` accurately reflect the backend schema: uppercase priority values (`'LOW' | 'MEDIUM' | 'HIGH'`), uppercase status values (`'OPEN' | 'IN_PROGRESS' | 'RESOLVED'`), `customer_id` as the write field. Schema alignment is consistent.

The AGENTS.md is the standout prompting technique: instructing the AI to read the installed Next.js documentation before writing any code. This shows meta-awareness of LLM training data limitations — a sophisticated context management move.

Deductions: The backend schema is missing several Figma-spec fields (phone, assigned_to, URGENT, CLOSED) — suggesting the Figma was not fully passed to the backend agent. The chart is mostly hardcoded, suggesting the analytics design requirement wasn't fully translated into real data.

**Score: 18 / 25 (72%)**

---

### 3. Architectural Oversight — 17 / 25

Did the candidate catch common AI blind spots (CORS, mobile responsiveness, structural integrity)?

**Assessment:**
Strong: `CorsMiddleware` is correctly placed in `MIDDLEWARE` before `CommonMiddleware`. Layout uses proper flexbox with responsive breakpoints — no height collapse. SQLite default with `dj_database_url.config(default=...)` is clean.

Issues: `CORS_ALLOW_ALL_ORIGINS = True` — unnecessary when a specific origin list would suffice. No `requirements.txt` — new developers must manually read the README and transcribe a pip install command, which is fragile. Fake phone data silently masks a missing model field. Pagination UI is cosmetic without backend pagination support. `assigned_to` is dangling — visible in UI but sends no data.

**Score: 17 / 25 (68%)**

---

### 4. Documentation — 10 / 25

Is the Git history clean? Can the project be spun up instantly using the README?

**Assessment:**
The README covers architecture, tech stack, features, and backend/frontend setup steps clearly. A developer can spin it up by following the instructions — albeit by manually transcribing a pip install command rather than running `pip install -r requirements.txt`.

The git history is the weakest in the cohort: 5 commits total, 3 of which are meaningless ("Fixes", "change", "First iteration"). The history provides almost no insight into the development process. A 5-commit history for a full-stack multi-page application is a significant shortfall.

**Score: 10 / 25 (40%)**

---

## Final Score

| Rubric Dimension | Score | Percentage |
|---|---|---|
| Speed vs. Accuracy | 20 / 25 | 80% |
| Prompting & Context Management | 18 / 25 | 72% |
| Architectural Oversight | 17 / 25 | 68% |
| Documentation | 10 / 25 | 40% |
| **Total** | **65 / 100** | **65%** |

### Grade: **C+ (65%)**

---

## Summary

Ony's submission is the most functionally complete reviewed: all pages render with real data, all API endpoints pass, navigation works, the modal creates tickets, and comments can be added — on the correct framework (Next.js). The AGENTS.md context injection technique is the most deliberate AI prompting move seen in the cohort.

The score is held back by very poor documentation (5 commits, 3 vague, no requirements.txt) and a data model that omits several Figma-spec fields, leading to fake phone numbers, a non-functional "Assign To" dropdown, and nameless comments.

**Strongest area:** Speed vs. Accuracy — the most complete working implementation
**Weakest area:** Documentation — fewest commits of the cohort, no requirements.txt
