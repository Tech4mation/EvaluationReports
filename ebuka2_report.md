# Evaluation Report — Ebuka (Revision 2)
**Project:** EbukaEvaluation2  
**Stack:** Django 6.0.3 + DRF + django-cors-headers (Supabase PostgreSQL) · Next.js · TypeScript · Tailwind CSS  
**Evaluator note:** This is a re-submission. V1 scored 64/100 (B). The core failure in V1 was an incomplete frontend — only the tickets list was wired to the API; dashboard was missing, ticket detail was a placeholder, new ticket form was static HTML, customers page didn't exist. This report evaluates V2 against the same rubric.

---

## Manual Testing Findings (Pre-Review)

| Finding | Severity |
|---|---|
| UI looks off but serviceable | Minor visual |
| No notification bell | Cosmetic |
| Sort and Filter are cosmetic UI pills, no handler | Moderate UX |
| Dashboard Quick Actions navigate to list pages, not create | Moderate UX |
| Sidebar nav icons are single letters ("H", "T", "C"), not icons | Aesthetic |
| Can't create customers — no button or modal on customers page | Significant UX gap |
| No backend errors | Positive |
| seed_data command works | Positive |
| "api calls to save and update are broken" | See analysis below |

---

## What Improved from V1

V1 was essentially backend complete / frontend incomplete. Here is what V2 added:

| View | V1 | V2 |
|---|---|---|
| **Dashboard** | `redirect("/tickets")` — not implemented | Real: stat cards, live chart, 5s refresh |
| **New Ticket Modal** | Static `<div>` placeholders, no inputs | Real: customer select, form, `createTicket()` |
| **Ticket Detail** | Hardcoded placeholder text | Real: all fields, customer context, timeline |
| **Comments** | Not implemented | Real: `createComment()`, appends to thread |
| **Customers page** | Sidebar link was `href="#"` — page didn't exist | Real: `fetchCustomers()`, search, table |
| **Root-level scaffold** | Leftover `django-admin startproject` files | Cleaned up in V2 |
| **`backend-contract.ts`** | Not present | Added: explicit type contract file |

---

## Code Review

### Backend

The backend is largely unchanged from V1, which was already strong. Key patterns maintained:
- `responses.py` — `success_response()` / `error_response()` / `api_exception_handler()` — uniform JSON envelope `{ success, message, data/errors }`
- `TicketListSerializer` / `TicketDetailSerializer` split — list omits comments, detail includes them
- `TicketWriteSerializer` with `PrimaryKeyRelatedField(source="customer", queryset=Customer.objects.all())` — correct ForeignKey write pattern
- `select_related("customer")` on list queryset, `prefetch_related("comments")` on detail queryset — N+1 prevention
- `DashboardStatsView` with `values("status").annotate(count=Count("id"))` aggregation

`Comment` model still has no `author_name` field — comments are anonymous (only `content` + `created_at`). This is unchanged from V1.

**Critical infrastructure issue: real Supabase credentials committed to `.env`**

```env
DATABASE_URL=postgresql://postgres.usulhifsccrvobmtdakw:jozyToby-24001@aws-1-eu-central-2.pooler.supabase.com:5432/postgres
```

This is a live connection string with a real password stored in version control. The `.env` itself is listed in `.gitignore` but the file exists in the repo and was evaluated with live credentials. This is a security issue — the password should be rotated.

The backend also **still requires PostgreSQL**. No SQLite fallback exists. `parse_database_url()` explicitly raises `RuntimeError("DATABASE_URL must be a PostgreSQL URL.")` for anything that isn't postgres. This means any evaluator without the committed credentials (or their own Supabase project) cannot run the backend at all. This was flagged in V1 and is unchanged.

**`seed_data.py`** creates 3 customers and 3 tickets with 4 comments. Small but functional.

---

### Frontend

**`lib/backend-contract.ts` — standout addition**  
A separate file that explicitly names the backend's API types (`BackendTicket`, `BackendComment`, `BackendDashboardStats`, etc.) and lists all routes in a comment block. Then `lib/types.ts` re-exports these as clean names. This is the clearest API contract documentation in the cohort — it makes the frontend's dependency on the backend explicit and in one place.

**`lib/api.ts`**  
`parseResponse<T>()` correctly unwraps the `{ success, message, data }` envelope and throws `ApiRequestError` with the message for non-success responses. `getApiBaseUrl()` throws if `NEXT_PUBLIC_API_BASE_URL` is not set (no default fallback — requires the `.env` file). Both dashboard and tickets pages use a 5-second `setInterval` for live refresh — a thoughtful touch.

**Dashboard (`app/page.tsx`)**  
Now a real page. `fetchDashboard()` loads stat cards (Complete, Customers, Tickets) with real values. The chart renders `tickets_by_status` values as proportional bars. Live refresh every 5 seconds. Quick Actions are present but navigate to list pages:

```tsx
<NavLinkCard href="/customers" title="Create New User" description="Open customers view" />
<NavLinkCard href="/tickets" title="Create New Ticket" description="Open tickets workspace" />
<NavLinkCard href="/customers" title="Create New Customer" description="Review customer list" />
```

None open a modal or trigger a creation flow. This is purely navigation, not action.

The "Sort / Filter" element is a styled `<div>` with two `<span>` text nodes and zero interaction. It looks like a control but does nothing.

**Tickets page (`app/tickets/page.tsx`)**  
Search is functional — `filteredTickets` filters by subject, customer name, and category. Stat cards show live data. The `NewTicketModal` is correctly wired — `onCreated` calls `loadTicketsPage(false)` to refresh the list after creation.

**New Ticket Modal — critical bug: duplicate orphaned description field**

The modal contains two `<label>` elements both titled "Description":

```tsx
// Line 165–170: ORPHANED — no value, no onChange, does not update form state
<div>
  <FieldLabel label="Description" error={getFieldError(fieldErrors, "description")} />
  <TextInput placeholder="Brief description of the issue" />
</div>

// Lines 172–200: Priority, Category, Status selects ...

// Line 215–220: REAL — wired to form.description
<div>
  <FieldLabel label="Description" error={getFieldError(fieldErrors, "description")} />
  <TextArea
    value={form.description}
    onChange={(event) => setForm((current) => ({ ...current, description: event.target.value }))}
  />
</div>
```

The first "Description" input has no `value` prop and no `onChange` — it is completely disconnected from form state. The submit button is gated on `form.description.trim()` being non-empty:

```ts
const canSubmit = useMemo(() => Boolean(form.customer_id && form.subject.trim() && form.description.trim()), [form]);
```

A user who fills in the first (orphaned) description box will find the "Create Ticket" button remains disabled. They must scroll past priority/category/status to find the real textarea below. **This is almost certainly what the user experienced as "save... broken".**

**Ticket Detail (`app/tickets/[id]/page.tsx`)**  
Fully functional — `fetchTicketDetail(id)` loads the ticket, customer fields, status/priority/category badges, timeline (created/updated), and the comments thread. `createComment()` posts to the API and appends the returned comment to local state. Clean.

However: there is **no way to update ticket status** from the detail page. Status is displayed via `StatusBadge` only — there is no select dropdown or edit control. "Update" is therefore entirely absent from the detail view. This is likely the second part of the user's "save and update are broken" observation.

**Customers page (`app/customers/page.tsx`)**  
Exists and is connected. Loads `fetchCustomers()`, shows a table with name/email/phone, client-side search. However, there is no "New Customer" button, no modal, no form. Customers can only be created via the backend directly. The page header has `actions={<div className="text-sm...">...</div>}` — a placeholder `...` where a create button should be.

The `Channel` and `Tag` columns in the table are hardcoded decorative content:
```tsx
<span className="text-[11px] text-[#ff3c8e]">o</span>  // "o" for channel
<span className="inline-flex rounded-full bg-[#edf5ff]...">Request Buyer</span>  // hardcoded tag
```

**Sidebar (`components/crm-shell.tsx`)**  
Icons are single-character strings: `{ href: "/", label: "Home", icon: "H" }`. These are rendered as `<span>{item.icon}</span>` — just the letter "H", "T", "C" in the nav. No icon library was used. The mobile nav is a 3-column grid of text links — functional but visually bare.

---

## Analysis: "api calls to save and update are broken"

- **Save (create ticket):** The create flow works if the user finds the real description textarea. The orphaned first `TextInput` labeled "Description" keeps the submit button permanently disabled for users who fill it in. This is the breakage.
- **Update:** No ticket status update exists anywhere in the frontend. There is a PATCH endpoint on the backend, but no UI exposes it.

---

## Summary of Remaining Issues

| Issue | Impact |
|---|---|
| Duplicate orphaned "Description" field in new ticket modal | Critical UX — submit stays disabled |
| No ticket status update UI (PATCH endpoint exists but unused) | Significant functionality gap |
| No "Create Customer" button on customers page | Significant UX gap |
| Dashboard Quick Actions navigate to pages, not modals | Moderate UX |
| Sort / Filter are decorative text spans | Minor UX |
| Sidebar icons are single letters (H, T, C) | Aesthetic |
| Real Supabase credentials committed to `.env` | Security issue |
| PostgreSQL required — no SQLite fallback | Setup friction |
| `Channel` and `Tag` customer columns are hardcoded | Cosmetic |
| No comment author name | Minor feature gap |
| README hardcodes Windows path (`C:\Users\USER\Documents\...`) | Portability issue |

---

## Comparison: V1 vs V2

| Dimension | V1 | V2 |
|---|---|---|
| Dashboard | Missing (redirect to tickets) | Present, real data, live refresh |
| Ticket detail | Placeholder text | Fully connected |
| New ticket form | Static HTML divs | Real modal (with description bug) |
| Comments | Not implemented | Working API call |
| Customers page | `href="#"` dead link | Real page (no create button) |
| Ticket status update | No | No (PATCH unused) |
| Root scaffold leftover | Yes | Cleaned up |
| `backend-contract.ts` | No | Yes |
| Committed credentials | No | Yes (new issue) |
| Sidebar icons | N/A | Letter placeholders |

---

## Scoring

### 1. Speed vs Accuracy — 18 / 25

V2 completes what V1 left as stubs: dashboard, ticket detail, comment creation, customers page, and a functional ticket create modal. These were explicitly the failing areas in V1. All five screens now load real data from the backend.

Deductions: the orphaned description field is a functional bug that makes ticket creation confusing; no status update UI means the write path is only half-complete; customers can only be viewed, not created.

### 2. Prompting & Context Management — 17 / 25

`backend-contract.ts` is the standout — explicitly naming the backend types and routing in one file, then re-exporting them through `lib/types.ts`, shows careful API-to-frontend mapping. The custom `{ success, message, data }` envelope is correctly handled by `parseResponse<T>()`. The 5-second live refresh is a thoughtful product detail.

Deductions: the orphaned description field indicates the new ticket modal was generated and not validated end-to-end; Quick Actions were never upgraded from navigation to action triggers; status update was never prompted.

### 3. Architectural Oversight — 16 / 25

Strong patterns maintained from V1: custom exception handler, uniform response envelope, `DATABASE_URL` validation. `backend-contract.ts` is a clean contract boundary.

Significant deductions: real credentials committed to `.env` is a security issue that could expose the Supabase database; PostgreSQL-only requirement remains an unnecessary setup barrier (spec allows SQLite); no `CustomerCreate` form means customers can't be managed from the UI; sidebar text-letter icons show the shell component was never properly finished.

### 4. Documentation — 17 / 25

Comprehensive README: full API contract listing, response envelope docs, setup steps, PowerShell command examples for create/list operations, Frontend integration notes, troubleshooting section. Backend README also has sample commands.

Deductions: hardcoded Windows path (`cd C:\Users\USER\Documents\TECH4MATION\EbukaEvaluation2`) in every step — does not work on macOS/Linux; README file links use Windows absolute paths (broken cross-platform); committed credentials in `.env` (a governance/security issue separate from documentation).

---

## Final Score

| Category | Score |
|---|---|
| Speed vs Accuracy | 18 / 25 |
| Prompting & Context Management | 17 / 25 |
| Architectural Oversight | 16 / 25 |
| Documentation | 17 / 25 |
| **Total** | **68 / 100** |
| **Grade** | **B** |

**Change from V1:** 64/100 (B) → 68/100 (B)

V2 delivers a dramatically more complete product — all five screens are functional where V1 had only one. The backend remains strong. The improvement is real but the grade holds at B because the orphaned form field creates a broken create flow, status updates are absent, the PostgreSQL-only requirement and committed credentials are unchanged concerns, and the sidebar/customers-page gaps prevent this from reaching A range.
