# Evaluation Report — Peculiar (Revision 2)
**Project:** PeculiarEvaluation2  
**Stack:** Django 6.0.3 + DRF + django-cors-headers (SQLite) · Next.js 16 + React 19 · TypeScript · Tailwind CSS 4  
**Evaluator note:** This is a re-submission. V1 scored 64/100 (B). The central issue in V1 was that no API call was ever made from the frontend — all data was mock. This report evaluates V2 against the same rubric and notes all changes from V1.

---

## Manual Testing Findings (Pre-Review)

| Finding | Severity |
|---|---|
| Log matches, UI close to Figma | Positive |
| Filter buttons on dashboard/tickets do nothing | Moderate UX |
| Dashboard Quick Action buttons ("Create New Ticket", "Create New Customer", "Create New User") do nothing | Moderate UX |
| Notification bell is cosmetic | Cosmetic |
| Sidebar is functional (mobile slide-out works) | Positive |
| Create ticket → works, customer created in DB | Positive |
| Adding tags works | Positive |
| seed_demo_data command works | Positive |
| No backend errors | Positive |
| "api calls to save and update are broken" | Moderate — see analysis below |

---

## What Improved from V1

V1's central problem: the frontend never called the backend. Every screen used mock data from `lib/data.ts`. V2 resolves this completely.

**The single biggest change: `lib/data.ts` is now a real API client.**

| Feature | V1 | V2 |
|---|---|---|
| Frontend calls real API | No — mock snapshots | Yes — all screens |
| Initial page data | Mock `seededTickets` etc. | SSR from Django |
| Create ticket | Fakes local state | Real POST, persists |
| Create customer | Fakes local state | Real POST, persists |
| Update ticket status | Not possible | PATCH endpoint added |
| Add comment | Fakes local state | Real POST to `/tickets/{id}/comments/` |
| Dashboard metrics | Mock numbers | Real aggregation from Django |
| Monthly completion chart | Mock array | Real `ExtractMonth` + `Count` query |
| PATCH endpoint | ❌ 405 | ✅ `TicketDetailView.patch()` added |
| Seed data customers | All named "Ogechi Arinze" | 10 unique names |
| Error state (API down) | No — would blank out | `LoadErrorState` component shows message + action |

---

## Code Review

### Backend

**models.py**  
Unchanged and still the strongest model design in the cohort. `TicketWorkflowState` (`to_do`, `in_progress`, `overdue`, `done`) as a concept separate from `TicketStatus` (`open`, `in_progress`, `resolved`, `closed`) is the right domain distinction. `Customer.channels` as a `JSONField(default=list)` for multi-channel attribution, `Customer.tag` for segmentation, `Ticket.progress` as an explicit integer — all reflect a real CRM data model.

**views.py**  
The new `TicketDetailView.patch()` handler is correct:
```python
def patch(self, request, public_id: str):
    ticket = get_object_or_404(ticket_queryset(), public_id=public_id)
    serializer = TicketUpdateSerializer(ticket, data=request.data, partial=True)
    serializer.is_valid(raise_exception=True)
    serializer.save()
    refreshed_ticket = ticket_queryset().get(pk=ticket.pk)
    return Response(TicketSerializer(refreshed_ticket).data)
```

Re-fetching with `ticket_queryset().get(pk=ticket.pk)` after save ensures the response has fresh `select_related`/`prefetch_related` data. Smart.

`CustomerListCreateView` handles `?search=` and `?page=` / `?pageSize=` query params via the `paginate_customers()` service. `page_size = max(1, min(..., 50))` caps the page size — good defensive input handling.

**serializers.py**  
`TicketCreateSerializer` has a clean `get_or_create`-style customer lookup: matches by email, creates if not found. This means creating a ticket for an existing customer email reuses the customer record. This is the right behavior for a support tool where a customer's email is their identity.

`TicketUpdateSerializer` accepts `workflowState` (camelCase, source=`workflow_state`) correctly with `partial=True` support.

**One issue: `status` and `workflowState` can drift out of sync.** The frontend's `updateTicketStatus()` sends only `{ status: "resolved" }` via PATCH. The serializer accepts it and updates only `status`. But `workflow_state` stays at whatever value it was — possibly `to_do` or `in_progress`. There's no hook that auto-updates `workflow_state` when `status` changes. The `TicketsOverview` counts (Not Started / In Progress / Completed) are computed from `workflow_state`, so a ticket marked `status=resolved` won't move to the "Completed" bucket unless `workflow_state=done` is also patched separately. The frontend never sends `workflowState` — so the overview counts and statuses silently diverge after any status update.

**services.py**  
`build_dashboard_snapshot()` is well-implemented. Monthly completion uses `ExtractMonth` + `Count(filter=Q(...))` — this is the efficient single-query approach rather than Python-side looping. `build_last_updated_label()` aggregates `Max` across tickets, customers, and comments to surface the most recent activity — a thoughtful detail.

`_next_code()` scans all existing IDs via a regex to find the highest numeric suffix and returns the next value. **Race condition risk**: under concurrent creates, two requests could both scan, both get the same highest value, and both try to insert the same `public_id`. `unique=True` on the field would catch this with a `IntegrityError` at the DB level, but no retry logic exists. For a SQLite single-user demo this is acceptable.

**settings.py**  
`CORS_ALLOWED_ORIGINS = ["http://127.0.0.1:3000", "http://localhost:3000"]` — specific origins, correct. `TIME_ZONE = "Africa/Lagos"` — locale-appropriate, consistent with target product.

---

### Frontend

**lib/data.ts — the core change**  
V1's mock functions are replaced with real `fetch` calls. `getApiBaseUrl()` reads `NEXT_PUBLIC_API_BASE_URL` (client-visible env var) and defaults to `http://127.0.0.1:8000/api`. This is the right approach for a browser → Django architecture.

`extractErrorMessage()` is the most robust error extraction in the cohort. It recursively traverses the error payload — string, array, object with `detail`, nested object values — and returns the first message found. This handles DRF's varied error shapes (`{"detail": "..."}`, `{"field": ["error"]}`, `["message"]`) without fragile assumptions.

The custom `ApiError` class with a `status` code allows `findTicket()` to distinguish 404 (return null, triggers `notFound()`) from 500 (propagates error, shows `LoadErrorState`). Proper error discrimination.

**Page architecture**  
All three workspace pages use `export const dynamic = "force-dynamic"` + `async function Page()` for SSR. The server fetch runs at request time (Django is at `http://127.0.0.1:8000/api` from the server's perspective). The fetched data is passed as `initialSnapshot` to client components. Client components handle mutations and optimistic state.

This is the correct Next.js pattern: server provides initial data, client handles interactivity.

**TicketsWorkspace**  
`handleCreateTicket()` calls `createTicket()` then `getTicketWorkspaceSnapshot()` to refresh. This is a full re-fetch after mutation — correct and consistent, avoids stale state.

**No ticket search or filter in the tickets list.** `TicketsList` renders all tickets from the snapshot without a search input, status filter, or any client-side filtering. There's no `useState` for a query in `TicketsWorkspace`. All the filter infrastructure from V1 (for customers) was not extended to tickets.

**TicketDetailsWorkspace**  
Status update via `handleStatusChange()` calls `updateTicketStatus()` (PATCH) and sets the returned `updatedTicket` in state. NOT optimistic — it waits for the API response before updating the UI. This is acceptable but feels slightly laggy on slow connections. No rollback needed since it's not optimistic.

`handleAddComment()` calls `addTicketComment()` which returns the full updated ticket (backend re-fetches after insert). `setTicket(updatedTicket)` correctly updates the comment thread without needing local state manipulation.

**AddCommentForm**  
Author name resets to empty on every page load — no `localStorage` persistence. Minor UX friction but not a bug.

**CustomersWorkspace**  
Uses `useDeferredValue` for the search input — defers heavy filter computation away from keystroke handlers. Real pagination with page/size controls. `handleCreateCustomer()` prepends the new customer to the list and resets to page 1. This is correct optimistic-style behavior for customer creation.

**dashboard/dashboard-screen.tsx — the main gap**  
Dashboard Quick Actions and Filter button are plain `<button>` elements with no `onClick`:

```tsx
<button>  // no onClick — filter button does nothing
  <FilterIcon />
  <span>Filter</span>
</button>

// Quick Actions
function QuickAction({ title, ... }) {
  return (
    <button /* no onClick */>
      {title}
    </button>
  );
}
```

"Create New User", "Create New Ticket", "Create New Customer" — none trigger navigation or open a modal. This is likely what the user refers to when they say "helper buttons don't work". These could link to `/tickets`, `/customers`, or trigger the modals directly.

**The "save and update broken" assessment**  
Reviewing the code: `createTicket()`, `createCustomer()`, `updateTicketStatus()`, `addTicketComment()` are all correctly wired. The "broken" user observation likely refers to the dashboard Quick Actions (visually present, zero functionality) and the filter button. The actual CRUD paths work.

**sidebar.tsx**  
Fully functional slide-out sidebar on mobile: overlay closes on link click, active route highlighted with brand color + left indicator stripe. "Denise Adebayo" / "Admin" still hardcoded in the footer.

---

## README Accuracy Issue

The root `README.md` "Current Status" section still reads:

> *"The frontend currently still uses local UI snapshot data in `frontend/src/lib` for most screens. Full frontend-to-backend runtime integration can be completed next without changing the overall screen structure."*

This is copy-pasted from V1 and is now incorrect. V2 has full API integration. The README also still omits the PATCH endpoint from the endpoints table. A reviewer reading the README would assume V2 is still a mock-only frontend.

---

## Summary of Remaining Issues

| Issue | Impact |
|---|---|
| Dashboard Quick Actions are dead `<button>` elements | Moderate UX |
| Dashboard Filter button is dead | Minor UX |
| No ticket search or filter in the tickets list | Moderate UX |
| `status` and `workflow_state` can drift after PATCH | Data consistency |
| README "Current Status" says frontend still uses mock data — outdated | Documentation bug |
| PATCH endpoint not listed in README endpoints table | Documentation gap |
| Author name not persisted in comment form | Minor UX |
| "Denise Adebayo" hardcoded in sidebar | Cosmetic |

---

## Comparison: V1 vs V2

| Dimension | V1 | V2 |
|---|---|---|
| API integration | No (mock data) | Yes — all screens |
| Ticket creates | Fake in-memory | Real, persists |
| Customer creates | Fake in-memory | Real, persists |
| Status updates | Not possible | PATCH works |
| Comment adds | Fake in-memory | Real, persists |
| Dashboard metrics | Mock numbers | Real from Django |
| PATCH endpoint | Missing | Present |
| Seed data names | All "Ogechi Arinze" | 10 unique people |
| LoadErrorState | None | Present |
| Ticket filter | Client-side mock filter | Not implemented |
| Dashboard actions | Dead buttons | Still dead buttons |
| README accuracy | Honest about gaps | Stale — says integration not done |

---

## Scoring

### 1. Speed vs Accuracy — 18 / 25

V2 resolves the most critical issue in V1: the frontend now talks to the backend. All primary flows work — create ticket, view tickets, view detail, update status, add comment, create customer, view customers. The backend endpoints are comprehensive and correctly designed.

Deductions: Dashboard Quick Actions and Filter are still dead buttons; no ticket search/filter; `workflow_state` drifts on status updates. These aren't broken API calls — the API itself works — but they represent meaningful gaps in the delivered UI.

### 2. Prompting & Context Management — 20 / 25

Very strong evidence of schema-first thinking: `lib/types.ts` mirrors the backend serializer output exactly (camelCase `workflowState`, `referenceCode`, `authorName`, `createdAt`). The `NewTicketInput` interface maps field-for-field to `TicketCreateSerializer`. `extractErrorMessage()` is the most robust error handler in the cohort — recursive, handles all DRF error shapes.

SSR + client mutation split is correct Next.js App Router usage. `useDeferredValue` for search, service layer in Django, `public_id` system for clean IDs — these are all deliberate architectural choices that reflect strong context management across backend and frontend agents.

Minor gap: the dashboard Quick Actions were never wired.

### 3. Architectural Oversight — 20 / 25

Best backend architecture in the cohort (service layer, specific CORS origins, `public_id` system, N+1 prevention, paginated customer API, `build_dashboard_snapshot()` with real ORM aggregation) is preserved and extended.

Frontend architecture is also strong: SSR + client split, `LoadErrorState` fallbacks, `useDeferredValue`, ~30 components organized by feature area, proper route groups, camelCase contract alignment.

Deductions: `status`/`workflow_state` drift is a real data consistency gap; three dead dashboard buttons; no ticket search/filter.

### 4. Documentation — 15 / 25

The README has good structure: tech stack, all GET endpoints listed, local setup instructions (Windows + macOS), seed commands, "Verified Commands" section. These are all positives from V1 retained.

However: the "Current Status" section is now actively wrong — it says the frontend still uses mock data, which is the opposite of the truth. A developer reading this README before running the app would be misled. The PATCH endpoint is still absent from the endpoint table. The frontend `README.md` just redirects to the root README (fine). No `.env.local.example`. Git history is minimal.

---

## Final Score

| Category | Score |
|---|---|
| Speed vs Accuracy | 18 / 25 |
| Prompting & Context Management | 20 / 25 |
| Architectural Oversight | 20 / 25 |
| Documentation | 15 / 25 |
| **Total** | **73 / 100** |
| **Grade** | **A** |

**Change from V1:** 64/100 (B) → 73/100 (A)

V2 completes the integration that V1 was explicitly missing. The backend was already the strongest in the cohort and that hasn't changed. The frontend is now a real app: data persists, status updates work, comments persist. The documentation regression (stale "Current Status" note) is the main miss — it should have been updated to reflect the completed integration.
