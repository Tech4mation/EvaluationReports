# Evaluation Report — Ony (Revision 2)
**Project:** OnyEvaluation2  
**Stack:** Django 6.0.3 + DRF + django-cors-headers (SQLite default / optional Supabase) · Next.js · TypeScript · Tailwind CSS · Axios · Lucide  
**Evaluator note:** This is a re-submission. V1 scored 70/100 (A). V1 was the most complete submission in the cohort at that time — all pages functional, all API calls wired. The main weaknesses were a missing `requirements.txt`, `CORS_ALLOW_ALL_ORIGINS = True`, and several schema gaps (no phone, assigned_to, URGENT priority, CLOSED status, comment author) that forced fake data in the UI. This report evaluates V2 against the same rubric.

---

## Manual Testing Findings (Pre-Review)

| Finding | Severity |
|---|---|
| UI is very close to Figma — best design fidelity in the cohort | Positive |
| README is well done — setup works without friction | Positive |
| SQLite default — no .env needed for local dev | Positive |
| Dashboard Quick Actions open create modals | Positive |
| Create ticket works, create customer works | Positive |
| View ticket, ticket detail, status update, comments all work | Positive |
| Customers page search works, pagination works | Positive |
| Filter button is cosmetic | Minor UX |
| Notification bell is cosmetic | Cosmetic |

---

## What Improved from V1

V1 had a complete working product but with several model-level gaps that forced the frontend to fake data. V2 addresses every one of them.

| V1 Issue | V1 | V2 |
|---|---|---|
| `requirements.txt` missing | Not present — README had manual pip commands | Present ✅ |
| `CORS_ALLOW_ALL_ORIGINS = True` | Open to all origins | Specific list: localhost:3000, 127.0.0.1:3000 ✅ |
| `Customer.phone` missing | Faked from hardcoded pool in `ui.tsx` | Real field: `phone = CharField(max_length=32, blank=True)` ✅ |
| `Ticket.assigned_to` missing | Dropdown present but not sent to backend | Real field + wired in form ✅ |
| No URGENT priority | Only LOW/MEDIUM/HIGH | `('URGENT', 'Urgent')` added ✅ |
| No CLOSED status | Only OPEN/IN_PROGRESS/RESOLVED | `('CLOSED', 'Closed')` added ✅ |
| `Comment.author` missing | Hardcoded "NativeTalk Support" / alternating names | `author = CharField(default='NativeTalk Support')` ✅ |
| "Last updated" hardcoded | "Oct 20, 2024" static string | `getLastUpdatedLabel(tickets)` computes from real data ✅ |
| Pagination cosmetic | Prev/next buttons non-functional | Client-side pagination with `useMemo`, functional controls ✅ |
| Chart mostly static | Hardcoded base heights | `buildMonthlyCompletion()` now uses real ticket data ✅ |
| No `NewCustomerModal` | Could only create customers via ticket modal | Standalone modal in customers page and dashboard ✅ |
| Dashboard Quick Actions navigated to pages | No modal triggers | "Create New Ticket" + "Create New Customer" now open modals ✅ |
| `db.sqlite3` committed | In repo | Not committed ✅ |

---

## Code Review

### Backend

**`models.py`**

All V1 schema gaps are resolved in a single migration (`0002_comment_author_customer_phone_ticket_assigned_to_and_more.py`). The model is now complete:

```python
class Customer(models.Model):
    name = models.CharField(max_length=255)
    email = models.EmailField(unique=True)
    phone = models.CharField(max_length=32, blank=True)

class Ticket(models.Model):
    PRIORITY_CHOICES = [('LOW',..),('MEDIUM',..),('HIGH',..),('URGENT','Urgent')]
    STATUS_CHOICES = [('OPEN',..),('IN_PROGRESS',..),('RESOLVED',..),('CLOSED','Closed')]
    ...
    assigned_to = models.CharField(max_length=255, blank=True)

class Comment(models.Model):
    author = models.CharField(max_length=255, default='NativeTalk Support')
```

All four Figma-spec tiers are now represented. The `email = EmailField(unique=True)` on Customer enforces identity uniqueness at the DB level.

**`views.py`**

`TicketViewSet` with `select_related('customer').prefetch_related('comments')` — correct N+1 prevention. The `analytics_summary` view builds a `normalized_status_counts` dict initialized from `STATUS_CHOICES` keys so zero-count statuses are always present in the response (no missing keys on the frontend). `completionRate` correctly counts both RESOLVED and CLOSED as completed.

**`serializers.py`**  
`TicketSerializer` uses `customer = CustomerSerializer(read_only=True)` + `customer_id = PrimaryKeyRelatedField(source='customer', write_only=True)` — the standard DRF read/write split. `comments = CommentSerializer(many=True, read_only=True)` is always included.

One note: every ticket list response includes all comments for all tickets. With `prefetch_related`, this is 2 SQL queries (not N+1), but the JSON payload grows proportionally with comment volume. This is fine for a demo but would require a separate comments-only endpoint or sparse fields at scale.

**`settings.py`**  
`CORS_ALLOWED_ORIGINS` reads from env with a specific-origins default. `ALLOWED_HOSTS` uses a `split_csv()` helper for env-configurable parsing. `dj_database_url.config(default=sqlite:///.../db.sqlite3)` — clean SQLite fallback.

---

### Frontend

**`lib/types.ts`**  
Fully updated to match the new model. All four priority values, all four status values. `Comment` type now includes `author`. `assigned_to` on `Ticket`. The type contract is accurate — no phantom fields, no missing fields.

**`lib/api.ts`**  
Axios instance with `baseURL: process.env.NEXT_PUBLIC_API_BASE_URL ?? 'http://127.0.0.1:8000/api'`. Clean default. All pages import from this central instance.

**`lib/ui.tsx`**  
Display-layer utilities extracted from pages. `getTicketProgress()` derives a progress percentage from status/priority — no stored field, but the derivation logic is consistent and the values aren't arbitrary (CLOSED=100, RESOLVED=88, IN_PROGRESS=60). `buildMonthlyCompletion()` now computes from real ticket `created_at` and `status` data — the V1 static chart is gone. `getLastUpdatedLabel()` reduces across `updated_at` timestamps to find the most recent activity.

One remaining cosmetic gap: `getCustomerProfile()` returns `'VIP Customer'` or `'Frequent Buyer'` alternating by list index. The model has no `tag` field — this is pure decoration. Same for the channels column (always shows all three icons for every customer/ticket). These are display-only fabrications, not functional gaps.

**`app/page.tsx` (Dashboard)**  
Three parallel `Promise.all` fetches: analytics, tickets, customers. Stat cards display real counts. Monthly completion chart renders from `buildMonthlyCompletion(tickets)`. Quick Actions:
- "Create New User" → `<Link href="/customers">` (navigation — reasonable)
- "Create New Ticket" → `onClick={() => setOpenModal(true)}` ✅
- "Create New Customer" → `onClick={() => setOpenCustomerModal(true)}` ✅

After modal close, `fetchDashboard()` re-fetches all three endpoints. State is fresh.

**`app/tickets/page.tsx` — ticket search is absent (regression from V1)**

V1 had a working client-side search on the tickets list that filtered by subject, description, and customer name. V2's tickets page has no search input — `tickets.map()` renders all results directly without filtering. The search was preserved on the customers page but removed from tickets. This is a step backward from V1.

**`app/tickets/[id]/page.tsx` (Ticket Detail)**  
`updateStatus()` calls PATCH then re-fetches via GET to confirm server state — two round trips per update, but correct. `handleAddComment()` POSTs to `/comments/` then re-fetches the ticket to refresh the thread. The comment author is hardcoded to `'NativeTalk Support'` in the POST body — there is no input for the user to set their name. The `Comment.author` field exists in the model but the UI never uses it as a user-editable field.

**`components/NewTicketModal.tsx`**  
On mount, loads customers from `/customers/`. If customers exist, shows a `<select>` pre-populated with the first customer. If no customers exist, shows an inline create form (name, email, phone) — the ticket submission then creates the customer first, then creates the ticket using the new customer ID. This is the most user-friendly first-run experience in the cohort for this edge case.

All form fields including `assigned_to` are now included in the POST payload via spread (`...formData`). This was the V1 dangling-dropdown issue — now resolved.

**`components/NewCustomerModal.tsx`**  
Simple, correct: controlled form for name/email/phone, calls `/customers/` POST, then fires `onCreated()` callback (which refreshes the parent list) and `onClose()`.

**`components/Sidebar.tsx`**  
Lucide icons (House, Ticket, Users) — real icon components. `usePathname()` for active route detection. Left indicator stripe on active item. "Denise Adebayor / Admin" hardcoded in footer — same cosmetic issue as Peculiar.

**`frontend/AGENTS.md` — context injection note**

The AGENTS.md (loaded via CLAUDE.md) instructs the AI agent:

```
This is NOT the Next.js you know. This version has breaking changes — APIs, conventions,
and file structure may all differ from your training data. Read the relevant guide in
`node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
```

This was flagged positively in V1 as a deliberate meta-prompting technique — instructing the agent to read installed documentation rather than rely on training data. It persists unchanged in V2. Worth noting that this instruction asks the agent to read from `node_modules/next/dist/docs/`, which is a real path inside the Next.js package. Whether this actually changes AI behavior or just adds noise is debatable, but the intent is clear: prevent outdated API usage.

---

## Summary of Remaining Issues

| Issue | Impact |
|---|---|
| Ticket search removed from V2 (present in V1) | Moderate — regression |
| Comment author hardcoded — no input for user's name | Minor feature gap |
| Channels and Tag columns are decorative (no model fields) | Cosmetic |
| `getTicketProgress` is derived, not stored — can't be set explicitly | Minor fidelity gap |
| `assigned_to` uses hardcoded names (Tonia Wells, James Carter, Denise Adebayo) | Minor |
| Comments always eager-loaded in ticket list response | Scalability concern |
| No seed/demo data command | Setup convenience |
| "Denise Adebayor" hardcoded in sidebar | Cosmetic |
| Filter button and notification bell are decorative | Minor UX |

---

## Comparison: V1 vs V2

| Dimension | V1 | V2 |
|---|---|---|
| All pages functional | Yes | Yes |
| Model fields complete | No (missing phone, assigned_to, URGENT, CLOSED, author) | Yes |
| `requirements.txt` | Missing | Present |
| CORS | `ALLOW_ALL = True` | Specific origins |
| Pagination | Cosmetic | Functional |
| Chart | Static/hardcoded | Real data |
| Last updated label | Hardcoded "Oct 20, 2024" | Dynamic from tickets |
| Quick Actions open modals | No — navigated to pages | Yes for two of three |
| New Customer modal | Not standalone | Present |
| Ticket search | Present | Removed (regression) |
| `db.sqlite3` committed | Yes | No |

---

## Scoring

### 1. Speed vs Accuracy — 22 / 25

V2 is functionally complete across all views. Every V1 model gap is resolved. Dashboard Quick Actions trigger modals. Create ticket, create customer, update status, add comment — all work end-to-end. The frontend is visually the closest to Figma in the cohort.

Deductions: ticket search was present in V1 and is absent in V2 — this is a regression in a commonly expected CRM feature. Channel and Tag columns are still decorative display fabrications rather than real data.

### 2. Prompting & Context Management — 22 / 25

Every schema gap from V1 (phone, assigned_to, URGENT, CLOSED, author) is now in the model and reflected accurately in `lib/types.ts`. The modal's inline customer-creation flow for the first-run empty-database case shows careful handling of an edge case in the spec. The AGENTS.md context injection technique is maintained.

Deductions: comment author is never a user-fillable field despite the model supporting it; `getTicketProgress` hardcodes value tiers rather than using a stored progress field; ticket search was not regenerated when V2 was built.

### 3. Architectural Oversight — 20 / 25

All V1 structural issues resolved: CORS locked down, `requirements.txt` present, model complete, migration clean, pagination functional, chart data-driven. `split_csv()` helper for env parsing is clean. `prefetch_related` + `select_related` on ticket queryset correctly prevents N+1.

Deductions: no user-facing error handling anywhere — every `catch` block calls `console.error` and silently fails (the user sees nothing if an API call fails). Comments are always included in the ticket list payload, which grows unboundedly; a `?detail=true` or separate comments endpoint would improve this. No seed data management command. Hardcoded assignee names in the modal.

### 4. Documentation — 21 / 25

The README addresses every V1 documentation gap: `requirements.txt` is present and `pip install -r requirements.txt` is now the setup command. Both `backend/.env.example` and `frontend/.env.example` are present. The README explains the SQLite default, the optional Supabase path, and the frontend API override — the three setup scenarios a reviewer might need. Endpoint table is accurate. Setup confirmed working without friction.

Deductions: no seed/demo data command (other submissions include `seed_demo_data` to pre-populate the DB for review). AGENTS.md is present but is an AI-directed instruction not a human-readable doc. Minor: did not check V2 git history but V1 had sparse, vague commits.

---

## Final Score

| Category | Score |
|---|---|
| Speed vs Accuracy | 22 / 25 |
| Prompting & Context Management | 22 / 25 |
| Architectural Oversight | 20 / 25 |
| Documentation | 21 / 25 |
| **Total** | **85 / 100** |
| **Grade** | **A** |

**Change from V1:** 70/100 (A) → 85/100 (A)

V2 is a thorough revision. Every flagged issue from V1 is addressed: the schema is complete, CORS is correct, `requirements.txt` is present, pagination works, the chart is data-driven, and modals now function where V1 had navigation links. The only meaningful regression is the absence of ticket search, which was present in V1. This remains the strongest product in the cohort — all core workflows function, the design fidelity is the closest to Figma, and the setup is the smoothest.
