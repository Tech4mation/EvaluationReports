# Evaluation Report — Ebuka

**Project:** Nativetalk CRM Ticketing Module
**Evaluator:** Senior Engineering Review
**Date:** 2026-03-30

---

## Overview

Ebuka submitted a full-stack ticketing system using Django REST Framework (backend) and Next.js (frontend). The backend is polished and production-oriented, with Supabase PostgreSQL integration and a clean API design. However, the frontend is substantially incomplete — only the tickets list is connected to the API; all other views were left as hardcoded placeholders.

---

## Setup & Testing Log

### Environment Issues Encountered

| # | Issue | Root Cause | Verdict |
|---|---|---|---|
| 1 | `pip install -r requirements.txt` failed initially | venv was created with system Python 3.9; Django 6.0.3 requires Python ≥3.12 | **Candidate's fault** — README does not state Python version requirement |
| 2 | `requirements.txt` not found at repo root | The actual backend lives in `backend/` subdirectory, but the README does not say `cd backend/` before running setup commands | **Candidate's fault** — README setup path is wrong |
| 3 | Root-level `manage.py` and `config/` folder present alongside `backend/` | Leftover `django-admin startproject` scaffold was never deleted | **Candidate's fault** — creates confusion about project entry point |
| 4 | `python manage.py migrate` failed (PostgreSQL connection refused) | Backend is hardcoded to PostgreSQL only — no SQLite fallback. Requires a live Supabase or local Postgres instance | **Candidate's fault** — spec allows SQLite; this increases reviewer setup friction significantly |

### What Was Successfully Tested

- Backend migrations ran successfully after configuring Supabase `DATABASE_URL`
- Seed data command (`python manage.py seed_data`) populated the database
- Backend server started without errors on `http://localhost:8000`
- Frontend installed and ran on `http://localhost:3000`
- All API endpoints tested via Postman (see below)

---

## API Endpoint Test Results

Base URL: `http://localhost:8000/api`

| Endpoint | Method | Result | Notes |
|---|---|---|---|
| `/customers/` | GET | ✅ Pass | Returns customer list with consistent response wrapper |
| `/customers/` | POST | ✅ Pass | Creates customer; validates unique email |
| `/tickets/` | GET | ✅ Pass | Returns tickets with nested customer object |
| `/tickets/` | POST | ✅ Pass | Creates ticket; requires valid customer ID |
| `/tickets/{id}/` | GET | ✅ Pass | Returns ticket detail with comments array |
| `/tickets/{id}/` | PATCH | ✅ Pass | Partial update works correctly |
| `/tickets/{id}/` | DELETE | ✅ Pass | Deletes ticket cleanly |
| `/tickets/{id}/comments/` | POST | ✅ Pass | Adds comment to ticket |
| `/dashboard/` | GET | ✅ Pass | Returns status counts, total, completion rate |

**Backend API: 9/9 endpoints functional. No failures.**

---

## Frontend Review

| View | Expected | Actual | Status |
|---|---|---|---|
| **Dashboard / Home** | Dashboard with metric cards and charts | `page.tsx` is a single line: `redirect("/tickets")`. No dashboard exists. | ❌ Not implemented |
| **Tickets List** | Live list of tickets from API | Fully connected to backend via `lib/tickets.ts`. Stats cards and table render real data. | ✅ Works |
| **New Ticket** | Interactive modal/form that creates a ticket | Page exists but all form fields are static `<div>` elements — not `<input>` tags. The "Create Ticket" button has no handler and no API call. | ❌ Not functional |
| **Ticket Detail** | Extrapolated detail view with ticket info and comments | Page exists but contains 100% hardcoded placeholder text: *"ready to be connected to the backend detail endpoint"* and *"Load the ticket details and comments here once the API route is ready."* Zero API integration. | ❌ Placeholder only |
| **Comments** | Add comment to a ticket | Not implemented anywhere in the frontend. | ❌ Not implemented |
| **Customers** | Customer list page | Sidebar link points to `href="#"` — a dead link. No customers page exists. | ❌ Not implemented |
| **Search / Filter** | Functional ticket filtering | Styled as UI controls but are static `<div>` elements with no interactivity. | ❌ Decorative only |

---

## What Works vs. What Doesn't

### ✅ Works Well
- Full backend API — all 9 endpoints functional, clean responses
- Supabase PostgreSQL integration with smart `DATABASE_URL` parsing
- CORS correctly configured
- Custom consistent API response wrapper (`success_response`)
- Custom DRF exception handler for uniform error responses
- `DATABASE_URL` validation that rejects placeholder values (prevents silent misconfiguration)
- Seed data management command
- Tickets list page — live data, correct API integration
- UI shell/layout — visually clean, matches Figma design language

### ❌ Does Not Work
- Home page (no dashboard — just redirects to tickets)
- Customers page (link is `href="#"`, page does not exist)
- Ticket detail page (hardcoded placeholder, not connected to API)
- New ticket form (static HTML, no inputs, no API call)
- Comments (not implemented in frontend)
- Search and filter (decorative divs only)
- Setup out-of-the-box (requires Python 3.12 and Supabase — not documented)

---

## Grading

### Rubric Breakdown (25 pts each, 100 total)

---

### 1. Speed vs. Accuracy — 16 / 25

Did the candidate deliver a functional MVP within the time limit without sacrificing structural integrity?

**Assessment:**
The backend is complete and production-quality. The tickets list page works end-to-end. However, 5 of 6 required views are either missing or non-functional: no dashboard, no customers page, no working create-ticket form, no ticket detail connected to data, and no comments. The spec explicitly required all of these. Shipping placeholder pages with comments like *"ready to be connected"* as a final submission represents a significant incompleteness in the deliverable.

**Positives:** Backend entirely functional. Real Supabase integration (bonus requirement). Tickets list connected to API.
**Negatives:** Ticket detail is a stub. New ticket form is static HTML. Customers page doesn't exist. Dashboard missing.

**Score: 16 / 25 (64%)**

---

### 2. Prompting & Context Management — 12 / 25

Did the candidate effectively pass the Django API schema to the Next.js agent? Did they feed the Figma design system to the AI to generate the missing UI components?

**Assessment:**
`lib/tickets.ts` shows genuine sophistication — flexible response normalization, support for multiple API response shapes, TypeScript types that match the backend schema — suggesting the Django schema was passed effectively to the frontend agent for the tickets feature. The visual shell matches the Figma design language well.

However, the **Extrapolation Challenge** (Ticket Detail + Comments — explicitly called out in the brief as the key test of AI prompting skill) was not completed. The detail page contains the AI's placeholder output, never refined into a working component. The create-ticket form fields are `<div>` elements rather than actual inputs, indicating the form generation prompt was either not executed or its output was not validated. The customers page was never prompted at all.

**Score: 12 / 25 (48%)**

---

### 3. Architectural Oversight — 17 / 25

Did the candidate catch common AI blind spots (CORS, mobile responsiveness, structural integrity)?

**Assessment:**
Several architectural decisions show real awareness:
- CORS correctly configured with environment-variable-driven allowed origins
- `DATABASE_URL` validation explicitly guards against common copy-paste errors
- Custom exception handler ensures consistent error response format
- Supabase integration is genuinely production-ready

However:
- **No SQLite fallback** — the backend requires a live PostgreSQL/Supabase instance. This is a setup blocker for any reviewer without credentials, and the spec explicitly lists SQLite as acceptable.
- **No mobile navigation** — the sidebar is `hidden` on mobile with no hamburger menu or mobile drawer; the app is inaccessible on small screens.
- **Placeholder content shipped as final output** — a critical AI blind spot: the AI generated scaffold comments (*"replace this with real data"*) that were never replaced.
- **Root-level scaffold not cleaned up** — leftover `django-admin startproject` files create structural ambiguity.

**Score: 17 / 25 (68%)**

---

### 4. Documentation — 19 / 25

Is the Git history clean? Can the project be spun up instantly using the README?

**Assessment:**
The README is genuinely good — it covers architecture, API endpoints with sample payloads, CORS config, seed data, and separate setup steps for both frontend and backend. This is clearly above average.

Git history has 4 commits with mostly descriptive messages:
- `feat: initialize frontend and backend project structure`
- `Refactor ticketing pages and components for improved UI and functionality` *(vague — doesn't say what changed or why)*
- `docs: add comprehensive README for CRM Ticketing MVP setup and API overview`
- `fix: add validation for DATABASE_URL to prevent using placeholder values`

Deductions:
- README does not state Python ≥3.12 is required (causes immediate setup failure on macOS default Python)
- README setup commands run from root, but `requirements.txt` and `manage.py` are inside `backend/` — the `cd backend/` step is missing
- One commit message is vague

**Score: 19 / 25 (76%)**

---

## Final Score

| Rubric Dimension | Score | Percentage |
|---|---|---|
| Speed vs. Accuracy | 16 / 25 | 64% |
| Prompting & Context Management | 12 / 25 | 48% |
| Architectural Oversight | 17 / 25 | 68% |
| Documentation | 19 / 25 | 76% |
| **Total** | **64 / 100** | **64%** |

### Grade: **B (64%)**

---

## Summary

Ebuka demonstrates solid backend engineering skill — the Django API is clean, well-structured, and genuinely production-ready with Supabase. The problem is the frontend: the majority of the required UI was left as placeholder scaffolding. The AI clearly generated shell components and routing structure, but the candidate did not complete the loop — connecting the generated pages to the API, replacing placeholder text with real data, or building the create/comment interactions.

The evaluation challenge explicitly tested whether a developer can use AI to **deliver a working MVP end-to-end**, not just a working backend with a skeleton frontend. On that measure, this submission is incomplete.

**Strongest area:** Backend architecture and API design
**Weakest area:** Prompting & context management — the extrapolation challenge was the core test and was not delivered
