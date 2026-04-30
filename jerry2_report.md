# Evaluation Report — Jerry (Revision 2)
**Project:** JerryEvaluation2  
**Stack:** Django 5.0.6 + DRF + django-cors-headers (SQLite / Supabase optional) · Next.js (JSX, not TypeScript) · Tailwind CSS  
**Evaluator note:** This is a re-submission. V1 scored 54/100 (D). The core failure in V1 was that the frontend was entirely localStorage-based mock data — no API calls were ever made. This report evaluates V2 against the same rubric.

---

## Manual Testing Findings (Pre-Review)

| Finding | Severity |
|---|---|
| No README anywhere in the project | Critical — setup impossible without guessing |
| `npm install` fails on macOS with `lightningcss-win32-x64-msvc` platform error | Critical setup blocker |
| `pip install` fails with `psycopg2-binary` build error on macOS | Critical setup blocker |
| `ModuleNotFoundError: No module named 'rest_framework_simplejwt.urls'` | Critical — backend won't start |
| UI is clean and close to Figma | Positive |
| Dashboard Quick Actions do nothing | Moderate UX |
| Filter button does nothing | Minor UX |
| "View ticket" buttons in tickets list have no onClick | Significant — no detail view accessible |
| "New Customer" button has no onClick | Significant UX gap |
| Create ticket modal works | Positive |
| No backend errors once running | Positive |

---

## What Improved from V1

V1's central problem: the frontend never called the backend. The entire state lived in `localStorage` via `useTicketStore`. V2 resolves this.

| Feature | V1 | V2 |
|---|---|---|
| Frontend calls real API | No — localStorage only | Yes — all three pages |
| Dashboard data | Mock numbers from hook | Real `api.dashboard.get()` |
| Tickets list | localStorage mock | Real `api.tickets.list()` |
| Customers list | N/A | Real `api.customers.list()` with pagination |
| Create ticket | Fake in-memory via `useTicketStore` | Real `api.tickets.create()` in modal |
| Sidebar icons | Assumed icon library or letters | Proper inline SVGs |
| New ticket modal | N/A (unconfirmed) | Fully built: form state, error handling, keyboard dismiss |

---

## Code Review

### Backend

**`backend/requirements.txt`**  
```
Django==5.0.6
djangorestframework==3.15.2
django-cors-headers==4.4.0
dj-database-url==2.2.0
python-decouple==3.8
supabase==2.5.3
djangorestframework-simplejwt==5.3.1
```

SQLite is the default via `dj-database-url` — unlike Ebuka, the backend can run without a Postgres instance. The Supabase package is present for optional storage/auth use, not for the main DB. JWT auth is configured in DRF settings. These are all reasonable choices — but two critical issues undermine the entire setup.

**Critical: `core/urls.py` — broken import**

```python
path('api/auth/', include('rest_framework_simplejwt.urls')),
```

`rest_framework_simplejwt.urls` does not exist in simplejwt 5.x — this module was removed. The correct approach is to import `TokenObtainPairView` and `TokenRefreshView` directly. As written, the backend raises `ModuleNotFoundError` on startup and cannot be started at all.

**`backend/dashboard/views.py` — hardcoded comment count**

```python
'comments': 10,
```

The dashboard response hardcodes `'comments': 10` instead of querying the actual comment count. Every user will see "10 comments" on the dashboard regardless of reality.

**`backend/tickets/views.py`**  
`TicketViewSet(ModelViewSet)` with `select_related('customer')`, search on title/description/status/priority, ordering on created_at/priority/status/progress. ModelViewSet gives list, retrieve, create, update, partial_update, destroy for free. Clean and functional.

**`backend/tickets/serializers.py`**  
`TicketSerializer` uses `SerializerMethodFields` for computed values: `ticket_number`, `priority_tone`, `status_tone`, `progress_tone`. These derive display-layer values from the model — a reasonable pattern that avoids frontend computation. `ticket_number` is `f"TICKET #{1000 + self.pk}"` — a property-based ID, simple but lacks the race-condition resilience of Peculiar's `public_id` approach.

**`backend/customers/models.py`**  
`Customer` + `Channel` as a separate model with FK and `unique_together(customer, name)`. Multi-channel attribution is a sound design for a CRM — better than Peculiar's `JSONField(default=list)` in terms of query-ability, though more verbose.

**`backend/supabase_client.py`**  
Lazy-initialized Supabase client, only active if `SUPABASE_URL` is set. The client is intended for auth/storage/realtime, not for the main database. However, none of the views actually use it — it's an unused scaffold. No credentials committed (unlike Ebuka).

---

### Frontend

**`package.json` — Windows platform package as explicit dependency**

```json
"devDependencies": {
  "lightningcss-win32-x64-msvc": "^1.32.0",
  "tailwindcss": "latest"
}
```

`lightningcss-win32-x64-msvc` is a Windows-only binary committed as a direct devDependency — not just in the lockfile. Any `npm install` on macOS or Linux will fail trying to download this package. This is not a lockfile platform mismatch (which would be bad enough) — it is an explicit manifest entry that poisons the install on every non-Windows machine.

All dependency versions use `"latest"` (Next.js, React, Tailwind) — unpinned, meaning the build is not reproducible.

**`lib/api.js` — core improvement**

```js
const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api';
export const api = {
  dashboard: { get: () => request('/dashboard/') },
  customers: { list, get, create, update, delete },
  tickets: { list, get, create, update, delete }
};
```

Clean, correct structure. Trailing slashes on all paths. `request()` reads `NEXT_PUBLIC_API_URL` with a sensible default. The object-namespaced API shape is readable. Error handling propagates `fetch` errors upward — pages would need to catch them.

**`lib/tickets.js` + `lib/use-ticket-store.js` — V1 dead code still present**  
Both files from V1's localStorage architecture remain in the project. `use-ticket-store.js` uses `useEffectEvent` — an experimental React 19 API not in stable releases. Neither file is imported by any current page. They should have been deleted.

**`components/admin-shell.jsx`**  
Clean layout component: sidebar with proper inline SVG icons (Home, Tickets, Customers), active route highlighting with brand-green color + left indicator stripe, header with page title and notification bell. "Denise Adebayor / Admin" hardcoded in the sidebar footer — same pattern as Peculiar.

**`components/new-ticket-modal-trigger.jsx` — genuinely well built**  
This is the strongest component in the submission. The trigger button + modal pattern is correct: `useState(isOpen)`, `useRef` on the first input for auto-focus, body scroll lock via `document.body.style.overflow`, Escape key handler added/removed on open/close, form reset on close, `disabled={saving}` on submit, inline error banner. The form fields cover Customer Name, Subject (title), Priority, Status, Description — all wired via a controlled `change()` handler.

**Critical bug: `customerName` is collected but never sent**

```jsx
const handleSubmit = async (event) => {
  event.preventDefault();
  await api.tickets.create({
    title: form.title,
    description: form.description,
    priority: form.priority || "Medium",
    status: form.status || "Not Started",
    progress: 0,
    // customerName: form.customerName ← silently omitted
  });
```

The user fills in the "Customer Name" field but it is never included in the API call. The backend's `TicketSerializer` has a FK `customer` field — without it, either the creation fails or the ticket is created with no customer association. The customer name collection is entirely decorative.

**Status values mismatch:**  
The modal offers `["Not Started", "In Progress", "To do"]` as status options. The backend model defines its own status choices (previously read as the class lists). These do not necessarily align — "Not Started" may be invalid depending on what the backend accepts.

**`app/tickets/page.jsx` — no ticket detail route**

```jsx
<button type="button" className="...bg-[#eefbe8]...">
  View ticket
</button>
```

Every row in the tickets table has a "View ticket" button with no `onClick` and no `href`. There is no `app/tickets/[id]/page.jsx` — the route does not exist. Users can see the tickets list but cannot view, update, or comment on any ticket. The backend's `TicketViewSet` exposes `/api/tickets/{id}/` — it simply was never wired to a frontend page.

**`app/customers/page.jsx` — New Customer button has no onClick**

```jsx
<button type="button" className="...bg-[#3dc515]...">
  <PlusIcon /><span>New Customer</span>
</button>
```

The "New Customer" button is rendered but has no handler and no modal. Customers can only be created by POSTing to the backend directly.

**`app/page.jsx` (dashboard) — Quick Actions are dead**

```jsx
{quickActions.map(({ label, tone, icon: Icon }) => (
  <button key={label} type="button">  {/* no onClick */}
    ...
  </button>
))}
```

"Create New Ticket", "Create New Customer", "Create New User" — none navigate or open a modal. The filter button is also dead.

---

## Summary of Remaining Issues

| Issue | Impact |
|---|---|
| No README — zero setup instructions | Critical — project is not runnable without guessing |
| `lightningcss-win32-x64-msvc` in `package.json` devDependencies | Critical — `npm install` fails on macOS/Linux |
| `psycopg2-binary` build failure on macOS | Critical setup blocker |
| `rest_framework_simplejwt.urls` import breaks backend startup | Critical — backend won't start |
| No ticket detail page | Significant — no view/update/comment flow |
| "View ticket" buttons have no onClick | Significant |
| `customerName` collected but not sent to API | Significant — customer link always missing |
| "New Customer" button has no onClick | Significant UX gap |
| Dashboard Quick Actions and Filter are dead | Moderate UX |
| Dashboard hardcodes `'comments': 10` | Data accuracy |
| V1 dead code (`lib/tickets.js`, `lib/use-ticket-store.js`) still present | Cleanup |
| `useEffectEvent` in dead code (experimental React 19 API) | Low (unused) |
| Windows `venv/` committed with `.exe` files | Repo hygiene |
| All dependencies use `"latest"` (unpinned) | Reproducibility |
| "Denise Adebayor" hardcoded in sidebar | Cosmetic |

---

## Comparison: V1 vs V2

| Dimension | V1 | V2 |
|---|---|---|
| Frontend API calls | None — localStorage only | Yes — all three pages |
| Dashboard data | Mock hook | Real Django API |
| Ticket create | localStorage mock | Real API (customerName dropped) |
| Ticket detail | Unknown/likely missing | No — route does not exist |
| Status update | Not possible | Not possible (no detail page) |
| Comments | localStorage mock | Not accessible (no detail page) |
| Customer create | N/A | Button exists, no handler |
| Customers list | N/A | Real API with pagination |
| Backend startup | Unknown | Broken (simplejwt.urls import) |
| README | None | None |
| Sidebar icons | Unknown | Proper SVGs |
| New ticket modal | Unknown | Functional (but drops customerName) |
| V1 dead code | Canonical | Still present, unused |

---

## Scoring

### 1. Speed vs Accuracy — 14 / 25

V2 wires up the dashboard, tickets list, and customers list to real API calls — the core failure of V1 is resolved. The new ticket modal is correctly built and calls `api.tickets.create()`. The customers list has real pagination.

Deductions: no ticket detail page means the entire view/update/comment workflow is inaccessible; the "View ticket" button exists but leads nowhere. `customerName` is collected in the create modal but silently dropped before the API call. "New Customer" has no handler. Three dead dashboard Quick Action buttons. The backend cannot start out of the box due to the broken JWT URL import. The breadth of functionality delivered is narrower than Ebuka V2 (which had a working detail page, comment creation, and dashboard) despite Ebuka's own bugs.

### 2. Prompting & Context Management — 14 / 25

`lib/api.js` shows correct API client structure: namespaced by resource, trailing slashes, environment variable for base URL, defaults that match the backend's dev port. The new ticket modal reflects good understanding of controlled-form patterns, accessibility (Escape key, focus management, ARIA), and error display.

Deductions: the ticket detail/update/comment integration was never prompted — a significant portion of the spec is absent. `customerName` was mapped to a UI field but never included in the API call, indicating the create flow was not validated end-to-end. Status option values in the modal don't necessarily match the backend's accepted values. The dashboard's hardcoded `'comments': 10` was never caught or corrected. Dead V1 files retained.

### 3. Architectural Oversight — 12 / 25

The backend `TicketViewSet(ModelViewSet)` pattern is clean and the `Channel` model for multi-channel attribution is a sound design choice. SQLite default avoids the PostgreSQL-only setup friction that Ebuka had. CORS is correctly configured.

Significant deductions: three independent setup-breaking issues on a macOS evaluator environment (Windows platform package in package.json, psycopg2-binary build failure, broken JWT URL import) means the project cannot be run at all without manual intervention. No ticket detail route. Customer name dropped in create. V1 localStorage files retained alongside the real API client — the repo has two competing data layers that would confuse any new developer. Windows `venv/` committed.

### 4. Documentation — 5 / 25

No README at any level — not at root, not in backend, not in frontend. The evaluator's first observation was "I don't even know how to start or setup his project." There are no setup instructions, no endpoint table, no seed data documentation, no `.env.example`, no explanation of the JWT setup or how to obtain tokens.

The 5 points acknowledge that the code is structurally readable — `admin-shell.jsx`, `api.js`, and the modal component are well-organized and the intent of each file is clear from its content alone. But the complete absence of external documentation is the worst documentation outcome in the cohort.

---

## Final Score

| Category | Score |
|---|---|
| Speed vs Accuracy | 14 / 25 |
| Prompting & Context Management | 14 / 25 |
| Architectural Oversight | 12 / 25 |
| Documentation | 5 / 25 |
| **Total** | **45 / 100** |
| **Grade** | **E** |

**Change from V1:** 54/100 (D) → 45/100 (E)

V2 resolves the central V1 issue — the frontend now talks to the backend. But the improvement is offset by a setup experience that is broken on three independent axes (none of which existed in V1, which at least ran cleanly via localStorage). The no-README problem persists from V1. No ticket detail page means the majority of the CRM workflow (view, update, comment) is inaccessible. The score regresses from V1 primarily because V1's all-localStorage approach ran without any setup errors, whereas V2 requires manual interventions just to start.
