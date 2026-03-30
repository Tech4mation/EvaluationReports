# Evaluation Report — Varol

**Project:** Nativetalk CRM Ticketing Module
**Evaluator:** Senior Engineering Review
**Date:** 2026-03-30

---

## Overview

Varol's submission is fully integrated end-to-end — all four screens load real backend data, ticket creation and comment posting call the API, and data persists on refresh. The standout architectural move is a Next.js catch-all proxy route (`app/api/[...path]/route.ts`) that forwards all frontend requests to Django server-to-server, completely eliminating browser CORS constraints without installing `django-cors-headers`. This is the most novel approach in the cohort. The weakness is sparse process: a single git commit for the entire project, no `requirements.txt`, a non-functional search input on the tickets page, and a minimal UI that does not closely replicate the Figma design.

---

## Setup & Testing Log

| # | Issue | Severity |
|---|---|---|
| 1 | **No `requirements.txt`** — README provides manual `pip install "django==4.2.7" djangorestframework` | Moderate |
| 2 | **Only 1 git commit** — entire project in one commit, no development history | Major |
| 3 | Tickets page search input is **not wired** — text can be typed but rows are never filtered | Moderate |
| 4 | Dashboard "Quick Actions" buttons (Create New User, Create New Ticket, Create New Customer) are non-functional | Moderate |
| 5 | "Last updated: Oct 20, 2024" hardcoded in tickets page — not dynamic | Minor |
| 6 | "Due in 6h 45m" SLA label on ticket detail is hardcoded | Minor |
| 7 | Comment model has no `author` field — display hardcodes "Support Team" for all comments | Minor |
| 8 | No inline customer creation in ticket modal — user must create a customer first, then create a ticket | Minor |
| 9 | No `PATCH /tickets/{id}/` — ticket status cannot be updated via API | Minor |
| 10 | `db.sqlite3` committed to the repo | Minor |

**Setup works but requires manual step:** `pip install "django==4.2.7" djangorestframework && python manage.py migrate`. No friction beyond the missing `requirements.txt`.

---

## API Endpoint Test Results

Base URL: `http://127.0.0.1:8000/api`

| Endpoint | Method | Result | Notes |
|---|---|---|---|
| `/customers/` | GET | ✅ Pass | Full list |
| `/customers/` | POST | ✅ Pass | `ModelViewSet` — full CRUD |
| `/tickets/` | GET | ✅ Pass | List with `customer_name` via `source` annotation |
| `/tickets/` | POST | ✅ Pass | Requires `customer` (FK id) |
| `/tickets/{id}/` | GET | ✅ Pass | Uses `TicketDetailSerializer` — includes nested comments |
| `/tickets/{id}/comments/` | POST | ✅ Pass | DRF `@action` — returns comment |
| `/metrics/tickets/` | GET | ✅ Pass | `total_tickets`, `completed_tickets`, `completion_rate`, `grouped_by_status` |
| `/tickets/{id}/` | PATCH | ❌ 405 | `ModelViewSet` but no partial update implemented for status |

**7/8 pass.** All core operations work. No status update endpoint.

### Architectural Highlight: Next.js API Proxy

Rather than configuring CORS on Django, Varol built a catch-all Next.js route at `app/api/[...path]/route.ts`:

```ts
const DJANGO_API_BASE = process.env.DJANGO_API_BASE || "http://127.0.0.1:8000/api";

async function proxy(request: NextRequest, path: string[]) {
  const targetUrl = `${DJANGO_API_BASE}/${path.join("/")}${request.nextUrl.search}`;
  // ...forwards method, headers, body to Django
}

export async function GET(...) { return proxy(request, path); }
export async function POST(...) { return proxy(request, path); }
// ...PATCH, PUT, DELETE also handled
```

The frontend's `lib/api.ts` calls `/api/tickets/` (a Next.js route), which server-to-server proxies to `http://127.0.0.1:8000/api/tickets/`. The browser never contacts Django directly — no CORS headers are needed at all. This is the only submission to solve CORS at the architecture level rather than configuring a middleware.

Additionally, `DJANGO_API_BASE` is read from `process.env`, making the backend URL configurable via `.env.local` — the README documents this.

---

## Frontend Review

### What Actually Works vs What the Code Suggests

| View | Code exists | API connected | Works in browser |
|---|---|---|---|
| **Dashboard** | ✅ Built | ✅ Real API (server component) | ✅ Renders with real data |
| **Tickets List** | ✅ Built | ✅ API calls written | ✅ Loads real tickets |
| **New Ticket Modal** | ✅ Built | ✅ API call on submit | ✅ Creates ticket, updates list |
| **Ticket Detail** | ✅ Built | ✅ Real API (server component) | ✅ Loads real ticket + comments |
| **Comments** | ✅ Built | ✅ API call on submit | ✅ Posts comment, appends to list |
| **Customers** | ✅ Built | ✅ API calls written | ✅ Loads real customers, creates inline |

Everything is integrated. Data persists on refresh.

### Dashboard

A Next.js server component that fetches from `DJANGO_API_BASE` at render time. Shows completion rate, customer count, ticket count. Includes graceful error state if the backend is down. Bar chart is data-driven from `grouped_by_status`. "Quick Actions" section has three buttons that render but have no `onClick` handlers — dead UI.

### Tickets Page

Client component. Loads tickets and customers in parallel on mount. Status summary cards count from real data. "New Ticket" button opens a modal where the user selects a customer from the loaded list, fills in subject/priority/category/description, and submits — calls `createTicket()` and updates the list on success. Includes an error state with retry.

**Search input is not wired:** The input field renders and accepts text, but the displayed `rows` are never filtered by it. The `query` state is never used.

### Ticket Detail

Server component that fetches the ticket at render time. Passes the ticket to a `TicketDetails` client component that handles comment posting in state. Adding a comment calls `addTicketComment()` and appends to the displayed list on success. "Due in 6h 45m" SLA is hardcoded.

### Customers Page

Customer creation is done inline — name/email/phone inputs sit directly in the page header alongside the search bar, with a "+ New Customer" button. Functional but the UX is unconventional compared to a modal. New customers call `createCustomer()` and prepend to the list.

### Component Structure

Only 3 components total (`tickets-page.tsx`, `ticket-details.tsx`, `app-shell.tsx`). Most of the UI is inlined directly in the page files. This works but limits reusability and makes pages dense. No shared design tokens — hex colors are hardcoded throughout (`#45c01a`, `#44c91f`, `#2789f4`, etc.).

### Design Fidelity

The colour palette (`#44c91f` / `#45c01a` green, blue, purple accents, light card layout) is directionally aligned with the Figma reference but not pixel-precise. The overall UI is clean and readable but minimal — no logo, no Encode Sans font, no badge shapes matching the Figma. Of the fully-integrated submissions, this has the weakest visual match to the Figma design system.

---

## Git History

**1 commit** — the worst in the cohort by a large margin.

```
cd99ff6 Initial clean commit without build artifacts
```

The entire project — backend models, serializers, views, frontend pages, components, API client, proxy route — is in a single commit. There is no development history whatsoever.

---

## Grading

### Rubric Breakdown (25 pts each, 100 total)

---

### 1. Speed vs. Accuracy — 18 / 25

Did the candidate deliver a functional MVP within the time limit without sacrificing structural integrity?

**Assessment:**
All four screens work and display real backend data. Ticket creation, comment posting, and customer creation all call the API and persist on refresh. The correct framework (Next.js) is used. Error and loading states are handled. This is one of the more complete integrations in the cohort.

Deductions: search input on tickets page is non-functional, Quick Action buttons are dead, no status update capability, customer creation requires a pre-existing customer before a ticket can be created, comment author not tracked.

**Score: 18 / 25 (72%)**

---

### 2. Prompting & Context Management — 14 / 25

Did the candidate pass the Django schema to the Next.js agent? Did they feed the Figma design system to the AI?

**Assessment:**
The TypeScript types in `lib/api.ts` accurately mirror the backend schema: `not_started`/`in_progress`/`completed` status values, `customer` (FK int) as the write field, `customer_name` as a read annotation, `grouped_by_status` structure. Schema alignment is solid.

The proxy pattern (`app/api/[...path]/route.ts`) demonstrates architectural awareness of the frontend/backend relationship. `DJANGO_API_BASE` as an env var shows deployment-awareness.

Deductions: minimal component decomposition suggests the frontend was generated with limited design context. No design system tokens — all colours hardcoded. Figma design was not visibly translated into the UI. Search wired in the UI but never connected in state — a gap not caught before submission.

**Score: 14 / 25 (56%)**

---

### 3. Architectural Oversight — 16 / 25

Did the candidate catch common AI blind spots (CORS, mobile responsiveness, structural integrity)?

**Assessment:**
The Next.js proxy is the strongest architectural oversight move in the cohort — CORS is solved by design rather than configuration. `DJANGO_API_BASE` env var makes the backend URL configurable. `select_related("customer").prefetch_related("comments")` on the ticket queryset avoids N+1 queries. Separate serializers for list vs detail (`TicketListSerializer` / `TicketDetailSerializer`) is clean. Error states with retry buttons are present.

Deductions: no `requirements.txt`, ticket search input renders but never filters, no PATCH endpoint so ticket status is immutable, hardcoded SLA and date labels, only 3 frontend components for the entire application.

**Score: 16 / 25 (64%)**

---

### 4. Documentation — 7 / 25

Is the Git history clean? Can the project be spun up instantly using the README?

**Assessment:**
The README covers setup steps clearly for both macOS/Linux and Windows, lists all API endpoints, explains the proxy pattern, documents `DJANGO_API_BASE` configuration, and includes a troubleshooting section. A developer can follow it without friction (minus having to manually type the pip install command).

The git history is the worst in the cohort: a single commit for the entire project. There is no way to trace any aspect of the development process, understand what changed when, or see how the backend and frontend were built incrementally. A 1-commit history for a full-stack multi-page application with a proxy layer is a fundamental documentation failure.

**Score: 7 / 25 (28%)**

---

## Final Score

| Rubric Dimension | Score | Percentage |
|---|---|---|
| Speed vs. Accuracy | 18 / 25 | 72% |
| Prompting & Context Management | 14 / 25 | 56% |
| Architectural Oversight | 16 / 25 | 64% |
| Documentation | 7 / 25 | 28% |
| **Total** | **55 / 100** | **55%** |

### Grade: **D (55%)**

---

## Summary

Varol delivers a fully integrated submission where every screen loads real data and every action persists to the backend — one of only two candidates (alongside Ony) to achieve this. The Next.js API proxy pattern is the most architecturally inventive approach in the cohort, solving CORS at the design level rather than via middleware configuration.

The submission is pulled down significantly by a single-commit history (no development visibility), no `requirements.txt`, a non-functional search input, and a UI that does not closely follow the Figma design system.

**Strongest area:** Speed vs. Accuracy — full API integration, data persists, correct framework
**Weakest area:** Documentation — one commit for the entire project
