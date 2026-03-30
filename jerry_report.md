# Evaluation Report — Jerry

**Project:** Nativetalk CRM Ticketing Module
**Evaluator:** Senior Engineering Review
**Date:** 2026-03-30

---

## Overview

Jerry's submission shows a clear two-tier approach: the existing Figma screens (Dashboard, Customers) were exported directly as static code using **Figma Make** (a Figma-to-React plugin), while the pages requiring extrapolation (Tickets list, Ticket Detail, New Ticket Modal) were built from scratch with real API integration. The backend is clean and fully functional. However, the framework used is **Vite + React Router, not Next.js** as specified, and the two Figma-exported pages show mock data — they are not connected to the backend at all.

---

## Critical Finding: Framework

The brief explicitly requires **Next.js (React)**. Jerry used **React + Vite + React Router v7**.

This is not a styling deviation — it is a different framework entirely. There is no SSR, no App Router, no `next/navigation`, no `next/link`. The project runs as a standard Vite SPA. This directly fails the tech stack requirement.

---

## Setup & Testing Log

| # | Issue | Severity |
|---|---|---|
| 1 | **Framework is Vite + React, not Next.js** | **Critical — spec violation** |
| 2 | API base URL hardcoded as `http://localhost:8000/api` — no `.env` / env variable | Moderate |
| 3 | `CORS_ALLOW_ALL_ORIGINS = True` in settings — overrides the allowed origins list, allows any origin | Moderate |
| 4 | `dist/` build artifacts committed to the repo | Minor |
| 5 | One commit: `"chore: empty code changes, no modifications made"` — meaningless commit | Minor |
| 6 | `assigned_to` dropdown has hardcoded names (John Smith, Sarah Johnson, Mike Davis) — not pulled from API | Minor |
| 7 | Email auto-generates if left blank (`name@example.com`) — creates junk data silently | Minor |

**Setup is clean:** Django 4.2.16 + SQLite, standard `pip install -r requirements.txt && python manage.py migrate`. No friction.

---

## API Endpoint Test Results

Base URL: `http://localhost:8000/api`

| Endpoint | Method | Result | Notes |
|---|---|---|---|
| `/customers/` | GET | ✅ Pass | List only — read-only view |
| `/customers/` | POST | ❌ 405 | `CustomerListView` is `ListAPIView` — no create |
| `/tickets/` | GET | ✅ Pass | Paginated response (`results`, `count`) |
| `/tickets/` | POST | ✅ Pass | Inline customer `get_or_create` by email |
| `/tickets/{id}/` | GET | ✅ Pass | Full detail with nested customer + comments |
| `/tickets/{id}/` | PATCH | ✅ Pass | Partial update (all fields) |
| `/tickets/{id}/` | DELETE | ✅ Pass | Destroy |
| `/tickets/{id}/comments/` | POST | ✅ Pass | Accepts `author` + `text` (note: not `content`) |
| `/dashboard/stats/` | GET | ✅ Pass | Returns totals, completion_rate, by_status, by_priority, total_customers |

**8/9 pass. Customer creation blocked.**

### Pagination Note
`REST_FRAMEWORK` has `DEFAULT_PAGINATION_CLASS` enabled with `PAGE_SIZE: 50`. The API returns `{"results": [...], "count": N}`. The frontend's `normalizeTicketsResponse()` handles both paginated and raw array responses — a good defensive pattern.

---

## Frontend Review — What Actually Works vs What the Code Suggests

### The Figma Make Pattern

Jerry used **Figma Make** — a Figma plugin that exports designs directly as React/TypeScript components — for the Dashboard and Customers pages. The `src/imports/` folder contains these exports: `AdminDashboard.tsx`, `CustomersTable.tsx`, and associated SVG path data files.

The `ATTRIBUTIONS.md` confirms this: *"This Figma Make file includes components from shadcn/ui..."*

### Broken Layout — Root Cause of Blank Tickets Page

When actually run, the **Tickets page is completely blank**. The root cause is in `Layout.tsx`:

```tsx
// Layout root
<div className="bg-white relative size-full">  // size-full = w-full h-full
```

`size-full` means 100% of parent. Since no parent in the chain sets an explicit height, this collapses to 0px height. The content inside is:

```tsx
// TicketsPage content area — inside Layout
<div className="absolute bottom-0 left-[272px] right-0 top-[72px] overflow-y-auto">
```

An `absolute` positioned child inside a zero-height `relative` container: **invisible**. The Dashboard (`w-screen h-screen`) and Customers page (`w-screen h-screen`) bypass Layout entirely, which is why they display. The moment a page uses `Layout`, it breaks.

Additionally, `Layout`'s top header is hardcoded `w-[1168px]` with `right-0 absolute` — it's anchored to the right edge and fixed at 1168px wide, misaligning on any viewport that isn't exactly 1440px wide. This causes the notification icon and module title to be visually off-position.

### The New Ticket Modal — Does Not Open

The Dashboard "Create New Ticket" button triggers this code:

```tsx
const textElement = target.closest('[data-name="Text"]');
if (textElement?.textContent?.includes("Create New Ticket")) {
  setIsModalOpen(true);
}
```

**This never fires.** The Figma-exported button's click target is a deeply nested SVG/div hierarchy. `.closest('[data-name="Text"]')` does not find the right element, so the modal never opens. Since the Tickets page is blank, the modal can't be opened from there either. The modal exists in code but is **completely unreachable** through the UI.

### Actual Behaviour vs Code Review Assessment

| View | Code exists | API code written | Actually works in browser |
|---|---|---|---|
| **Dashboard** | ✅ (Figma export) | ❌ Static mock data | ✅ Renders (mock only) |
| **Customers** | ✅ (Figma export) | ❌ Static mock data | ✅ Renders (mock only) |
| **Tickets List** | ✅ Custom built | ✅ API calls written | ❌ **Blank — layout collapses** |
| **New Ticket Modal** | ✅ Custom built | ✅ API calls written | ❌ **Unreachable — trigger broken** |
| **Ticket Detail** | ✅ Custom built | ✅ API calls written | ❌ **Unreachable — Tickets page blank** |
| **Comments** | ✅ Custom built | ✅ API calls written | ❌ **Unreachable** |

The Ticket Detail page uses its own `min-h-screen` wrapper (not `Layout`) and would technically render if navigated to directly by URL — but cannot be reached organically through the UI since the Tickets page is blank.

### Design Fidelity

Despite nothing working, Jerry has the **best design fidelity** in terms of visual language. The green `#3ebf0f` primary colour, Encode Sans font, badge styles, and card shapes match the Figma precisely. The Ticket Detail source code maintains this design language consistently. However, fidelity is only meaningful if the UI is actually visible and usable — and it is not.

---

## Git History

**20 commits** — the best history in the cohort.

```
93f091e feat: enhance TicketsPage with ticket loading, error handling, and improved UI components
95574d2 Add initial HTML structure for the Native Talk application
6522f9d removed guides
aabfca5 fix: remove left and right padding in index.html
87c46b1 fix: update Menu component styles for consistent appearance
9997612 fix: update binary files for main application and database   ← vague
32c79a7 fix: update CustomersPage and DashboardPage styles
...
2491148 feat: implement useSidebarNav hook for sidebar navigation handling
180cb4e Add single-file Django backend for Nativetalk Ticketing system
4c73916 initial commit: set up Django project structure with models, views, serializers
d5f43df chore: empty code changes, no modifications made   ← useless
e25d0e4 updated ui   ← vague
```

Most commits are descriptive and atomic. Several are vague ("fix: update binary files", "updated ui", "chore: empty code changes") but the overall history is useful and traceable.

---

## Grading

### Rubric Breakdown (25 pts each, 100 total)

---

### 1. Speed vs. Accuracy — 10 / 25

Did the candidate deliver a functional MVP within the time limit without sacrificing structural integrity?

**Assessment:**
In practice, only two pages render: Dashboard and Customers — both showing Figma mock data, not real backend data. The Tickets page is blank due to a layout height collapse bug. The New Ticket modal is unreachable. The Ticket Detail and Comments exist in code but cannot be accessed through the UI. The backend is well-built and the API code in the frontend is written correctly — but none of it is reachable by a user running the app. Wrong framework (Vite not Next.js) compounds the spec violations.

**Score: 10 / 25 (40%)**

---

### 2. Prompting & Context Management — 12 / 25

Did the candidate pass the Django schema to the Next.js agent? Did they feed the Figma design system to the AI to generate the missing UI components?

**Assessment:**
The TypeScript types in `lib/api.ts` precisely match the backend schema — `priority: "low" | "medium" | "high" | "urgent"`, `category: "technical" | "billing" | "general" | "feature"`, `comment_count`, `comments` array in detail. This shows the API schema was fed to the frontend agent.

For the Figma design: Jerry used Figma Make to export existing screens. This technically "feeds" the Figma to a tool, but the output is static and non-functional. The extrapolation challenge (Ticket Detail + Comments) was attempted well in code — the two-column layout is polished and the API calls are correctly written — but a critical AI blind spot was missed: the `Layout.tsx` height issue means none of it ever renders. Feeding Figma to a tool and getting pixel-perfect static output, then failing to wire it to data or verify it runs, is a context management failure.

**Score: 12 / 25 (48%)**

---

### 3. Architectural Oversight — 13 / 25

Did the candidate catch common AI blind spots (CORS, mobile responsiveness, structural integrity)?

**Assessment:**
The most critical architectural oversight is the **layout collapse** — using `size-full` in `Layout.tsx` without ensuring the parent chain has a defined height. This is a classic AI code generation blind spot: the AI generates layout code that looks correct in isolation but silently fails at runtime. A basic smoke test (opening the browser) would have caught this immediately.

Additional issues:
- **`CORS_ALLOW_ALL_ORIGINS = True`** — overrides the origins list, allows any domain
- **Framework mismatch** — Vite, not Next.js
- **Hardcoded API base URL** — no env variable
- **`w-[1440px] h-[1024px]`** on Ticket Detail — hardcoded fixed viewport size
- **`w-[1168px]` hardcoded header** — breaks on any viewport ≠ 1440px, causing visual misalignment
- **`dist/` committed** — build artifacts in repo
- **Modal trigger text-content match** — fragile and broken in practice

Positives: SQLite, paginated response handled in both layers, backend structure is clean.

**Score: 13 / 25 (52%)**

---

### 4. Documentation — 19 / 25

Is the Git history clean? Can the project be spun up instantly using the README?

**Assessment:**
The README is the most detailed of the cohort — covers architecture, Figma design specifications, API reference table, data models, design system colours, prerequisites, full backend/frontend setup steps, and testing flow. A developer could set this up with no additional questions.

The 20-commit history gives the best insight into development process. Most commits are descriptive. Deductions for 3 vague/empty commits and for committing the `dist/` folder. Using `pnpm` (not `npm`) is noted in the README but could trip up developers without it installed.

**Score: 19 / 25 (76%)**

---

## Final Score

| Rubric Dimension | Score | Percentage |
|---|---|---|
| Speed vs. Accuracy | 10 / 25 | 40% |
| Prompting & Context Management | 12 / 25 | 48% |
| Architectural Oversight | 13 / 25 | 52% |
| Documentation | 19 / 25 | 76% |
| **Total** | **54 / 100** | **54%** |

### Grade: **D (54%)**

---

## Summary

Jerry demonstrates strong design instinct and the best understanding of how to use Figma as a source of truth — Figma Make export gives him pixel-perfect screens faster than anyone else. The custom-built Ticket Detail is polished and fully integrated. The backend is clean and properly documented with 20 commits.

The critical shortcomings are: wrong framework (Vite not Next.js), the Dashboard and Customers pages are static Figma mockups with no API connection, and `CORS_ALLOW_ALL_ORIGINS = True` is a basic security oversight. The Ticket Detail also has a hardcoded fixed-size layout that breaks on any non-1440px screen.

The Figma Make approach surfaces an interesting tradeoff: using a design export tool gets you visual fidelity instantly but the output is not wired to data. A stronger submission would have used the export as a visual reference and then built real components against the API.

**Strongest area:** Documentation and design fidelity
**Weakest area:** Architecture — wrong framework, static dashboard, CORS misconfiguration, fixed viewport layout
