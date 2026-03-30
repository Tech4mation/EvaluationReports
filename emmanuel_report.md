# Evaluation Report — Emmanuel

**Project:** Nativetalk CRM Ticketing Module
**Evaluator:** Senior Engineering Review
**Date:** 2026-03-30

---

## Overview

Emmanuel submitted a clean, well-integrated full-stack ticketing system. Unlike Ebuka's submission where the frontend was largely placeholder, Emmanuel's frontend is genuinely functional end-to-end — the dashboard, ticket list, new ticket modal, ticket detail view, and comments all work and are connected to the backend. The backend is clean, readable, and correctly configured. The single major weakness is the git commit history: the entire project was pushed in one commit, directly failing the git workflow requirement in the brief.

---

## Setup & Testing Log

### Environment

- **Backend:** Django 5.0.6, requires Python ≥3.10. SQLite default — **no setup friction**. `cd backend && pip install -r requirements.txt && python manage.py migrate` works cleanly.
- **Frontend:** Next.js 15, standard `npm install && npm run dev`. No issues.
- **CORS:** Correctly configured for `localhost:3000`.
- **No `.env` needed** — frontend defaults to `http://localhost:8000` if `NEXT_PUBLIC_API_URL` is unset.

### Issues Encountered

| # | Issue | Severity |
|---|---|---|
| 1 | No `NEXT_PUBLIC_API_URL` documentation in frontend README | Minor — defaults work, but undocumented |
| 2 | `ALLOWED_HOSTS = ["*"]` in settings — insecure for production | Minor for evaluation context |
| 3 | `CustomerViewSet` is `ReadOnlyModelViewSet` — no POST/DELETE on `/api/customers/` | Moderate — can't create customers independently via API |
| 4 | Single git commit for entire project | **Critical** — directly fails the git workflow requirement |

---

## API Endpoint Test Results

Base URL: `http://localhost:8000/api`

| Endpoint | Method | Result | Notes |
|---|---|---|---|
| `/customers/` | GET | ✅ Pass | Returns customer list ordered by name |
| `/customers/` | POST | ❌ **405 Method Not Allowed** | CustomerViewSet is read-only — no customer creation via API |
| `/tickets/` | GET | ✅ Pass | Returns tickets with nested customer + display fields |
| `/tickets/` | POST | ✅ Pass | **Smart:** accepts `customer_name` + `customer_email` inline — does `get_or_create` on customer |
| `/tickets/{id}/` | GET | ✅ Pass | Returns full ticket with nested comments array |
| `/tickets/{id}/` | PATCH | ✅ Pass | Partial update (status, priority, category) |
| `/tickets/{id}/` | DELETE | ✅ Pass | Standard destroy |
| `/tickets/{id}/comments/` | POST | ✅ Pass | Accepts `content` + optional `created_by` |
| `/dashboard/metrics/` | GET | ✅ Pass | Most detailed dashboard response of all submissions |

**8/9 endpoints pass. 1 failure: customer creation blocked by read-only viewset.**

### Notable: Dashboard Response

Emmanuel's dashboard endpoint is the richest of all submissions — returns:
```json
{
  "total_tickets": 10,
  "open_tickets": 3,
  "in_progress_tickets": 4,
  "resolved_tickets": 2,
  "closed_tickets": 1,
  "completion_rate": 30.0,
  "tickets_by_status": [{ "status": "Open", "count": 3, "fill": "#6366F1" }, ...],
  "tickets_by_priority": [{ "priority": "high", "count": 5 }, ...],
  "tickets_by_category": [{ "category": "Bug", "count": 4 }, ...]
}
```
Includes chart-ready fill colors embedded in the response, priority breakdown, and top-5 category breakdown. The frontend consumes all of it.

### Notable: Ticket Creation — Inline Customer

The `TicketCreateSerializer` uses Django's `get_or_create` on the Customer model by email. This means the New Ticket form doesn't need a pre-existing customer ID — you submit customer details inline with the ticket:

```json
POST /api/tickets/
{
  "customer_name": "Jane Doe",
  "customer_email": "jane@example.com",
  "customer_phone": "+2348012345678",
  "customer_company": "Acme Corp",
  "subject": "Login issue",
  "description": "Cannot log in after password reset.",
  "priority": "high",
  "category": "Technical",
  "status": "open"
}
```
This is a thoughtful architectural decision — simpler UX, avoids a two-step flow. However, it means if a customer already exists, their name/phone/company will not be updated on repeated submissions (only the email is matched).

---

## Frontend Review

| View | Status | Notes |
|---|---|---|
| **Dashboard** | ✅ Fully functional | Live metric cards (Total, Open, In Progress, Resolved), Donut completion chart, Priority bar chart, Recent tickets list with View links — all from real API data |
| **Tickets List** | ✅ Fully functional | Loads from API, table with status/priority badges, "New Ticket" button triggers modal |
| **New Ticket Modal** | ✅ Fully functional | Real form with all inputs — customer name, email, phone, company, subject, description, priority, status, category. Calls `POST /api/tickets/`, closes on success and prepends new ticket to list |
| **Ticket Detail** | ✅ Fully functional | Loads ticket from API, shows description, inline status change dropdown (calls PATCH), customer info sidebar (name, email, phone, company), full comment thread |
| **Comments** | ✅ Fully functional | Comment list with author avatars, relative timestamps, add comment form with author name field and textarea, posts to API |
| **Customers page** | ❌ Dead link | Sidebar has `/customers` link but no page exists — 404 |
| **Reports / Settings** | ❌ Dead links | Present in sidebar nav but no pages exist |
| **Mobile nav** | ⚠️ Partial | Sidebar is fixed-width with no mobile collapse. App is functional on medium screens but sidebar would overflow on small viewports |

### API Integration Quality

The `lib/api.ts` is clean — a single generic `request<T>` wrapper handles all calls. TypeScript types in `types/index.ts` precisely match the backend serializer output (including `priority_display`, `status_display`, nested `customer`, `comments` array). This level of type alignment strongly suggests the Django schema was fed to the frontend agent.

---

## Code Quality Observations

**Backend strengths:**
- Well-structured serializer separation: `TicketListSerializer` vs `TicketDetailSerializer` vs `TicketCreateSerializer` vs `TicketUpdateSerializer` — each purpose-built
- `CommentCreateSerializer` keeps the write schema separate from the read schema
- `select_related("customer").prefetch_related("comments")` on the queryset — no N+1 queries
- Dashboard aggregation uses a single DB query with `.annotate(count=Count("id"))` — efficient
- `created_by` field on Comment enables basic authorship tracking

**Backend weaknesses:**
- `category` is a free-text field (no choices) — inconsistent with the Figma design which shows category as a dropdown
- `CustomerViewSet` read-only is intentional but limits API flexibility
- No seed data command (unlike Ebuka and others)

**Frontend strengths:**
- Status can be changed directly from the Ticket Detail sidebar via dropdown — calls `PATCH` in real time
- Comment form shows author initials as an avatar (generated from `created_by` name)
- Relative timestamps ("2h ago", "just now") on comments
- Error states are user-friendly with retry buttons
- Loading spinners on every async operation

---

## Git History

```
5b9e181 feat: full-stack Nativetalk CRM ticketing MVP
```

**1 commit.** The entire frontend and backend — all models, views, serializers, components, API client, types, charts — was squashed into a single commit. This directly fails Phase 3 of the brief which explicitly required:

> *"Descriptive Commits: We will review your commit history. Ensure your commits are descriptive and atomic."*

There is no way to trace the development process, see what was built first, or assess how context was passed between the frontend and backend phases. This is the most significant gap in the submission.

---

## Grading

### Rubric Breakdown (25 pts each, 100 total)

---

### 1. Speed vs. Accuracy — 17 / 25

Did the candidate deliver a functional MVP within the time limit without sacrificing structural integrity?

**Assessment:**
All core features are functional — dashboard, ticket list, ticket detail, comments, new ticket form. The backend is cleanly structured with proper serializer separation and no N+1 queries. The inline customer `get_or_create` shows genuine product thinking.

However, "Accuracy" explicitly includes design fidelity. The brief states: *"Replicate the Provided Designs... as closely as possible to the design system (colors, layout, typography)."* Emmanuel's UI deviates fundamentally from the Figma on every visual dimension: wrong sidebar colour (dark navy vs white), wrong primary color (indigo vs green), wrong logo, wrong metric card content, wrong chart type (donut/priority bar vs monthly completion bar), and the Quick Actions section is missing entirely. This is not a minor styling difference — it is a completely different design language.

Additional deductions: `/customers` page doesn't exist despite sidebar link, `POST /customers/` returns 405, category is free-text.

**Score: 17 / 25 (68%)**

---

### 2. Prompting & Context Management — 13 / 25

Did the candidate pass the Django schema to the Next.js agent? Did they feed the Figma design system to the AI to generate the missing UI components?

**Assessment:**
The schema passing to the frontend agent was done well — TypeScript types in `types/index.ts` precisely mirror the Django serializer output including `priority_display`, `status_display`, nested `customer`, and `comments`. The Ticket Detail extrapolation (two-column layout, customer sidebar, inline status editing) shows good product thinking for the missing views.

However, the Figma design system was clearly not passed to the AI. The brief evaluates this explicitly: *"Did you successfully feed the Figma design system to the AI to generate the missing UI components?"* The answer is no. Every visual design decision diverges from the Figma:

| Element | Figma | Emmanuel |
|---|---|---|
| Sidebar background | White / light | Dark navy |
| Primary colour | Green `#56c228` | Indigo `#6366F1` |
| Logo | "native talk" green pill | Headphone icon |
| Nav items | Home, Tickets, Customers | Dashboard, Tickets, Customers, Reports, Settings |
| Dashboard metric cards | Comments, Customers, Tickets | Total, Open, In Progress, Resolved |
| Chart | Monthly completion bar (Jan–Dec) | Donut + priority bar |
| Quick Actions | Present | Absent |

This represents a complete failure to use the Figma design system as context, which is a core criterion of the evaluation.

**Score: 13 / 25 (52%)**

---

### 3. Architectural Oversight — 19 / 25

Did the candidate catch common AI blind spots (CORS, mobile responsiveness, structural integrity)?

**Assessment:**
CORS is excellently configured — explicitly lists allowed methods and headers (not just origins), which is more thorough than most submissions. SQLite default means zero setup friction for any reviewer. The `select_related`/`prefetch_related` usage shows awareness of database performance. Error states and loading spinners are present on all async UI.

Deductions: `ALLOWED_HOSTS = ["*"]` is a basic security oversight. The sidebar has no mobile collapse — the app breaks on narrow screens. No Supabase integration (missed bonus). `CustomerViewSet` being read-only limits the API surface without documentation explaining why.

**Score: 19 / 25 (76%)**

---

### 4. Documentation — 14 / 25

Is the Git history clean? Can the project be spun up instantly using the README?

**Assessment:**
The README is genuinely excellent — covers project structure, all features, backend setup step-by-step, frontend setup, API reference table, Supabase config guidance, and a "Testing the full flow" section. A developer could spin this up without any additional guidance.

However, the git history completely undermines this category. A single squash commit means there is no development history to review. The brief is explicit that commit history is evaluated — *"We will review your commit history."* This is worth significant deduction: atomic, descriptive commits are a core deliverable of Phase 3, not a bonus.

**Score: 14 / 25 (56%)**

---

## Final Score

| Rubric Dimension | Score | Percentage |
|---|---|---|
| Speed vs. Accuracy | 17 / 25 | 68% |
| Prompting & Context Management | 13 / 25 | 52% |
| Architectural Oversight | 19 / 25 | 76% |
| Documentation | 14 / 25 | 56% |
| **Total** | **63 / 100** | **63%** |

### Grade: **B (63%)**

---

## Summary

Emmanuel's submission is deceptively impressive at first glance — every view is built, the API is fully integrated, and the backend is cleanly designed. The inline customer `get_or_create` pattern, the real-time status editing, and the TypeScript type alignment all demonstrate genuine engineering skill.

But two critical requirements from the brief were missed entirely. First, the Figma design system was not used — Emmanuel delivered a custom dark-themed UI when the spec required replicating a specific light green-accented design. Second, the entire project was submitted as one commit when the brief explicitly evaluated atomic, descriptive commits.

These are not minor gaps. They are two of the four evaluated criteria. A functionally complete app in the wrong design with no commit history scores significantly lower than it would otherwise deserve.

**Strongest area:** Architectural Oversight — clean backend, CORS, SQLite, no N+1 queries
**Weakest areas:** Figma design fidelity (not replicated), Git history (single commit)
