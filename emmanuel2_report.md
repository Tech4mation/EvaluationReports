# Evaluation Report — Emmanuel (Revision 2)
**Project:** EmmanuelEvaluation2 (Ticket Matrix)  
**Stack:** Django 5.1.3 + DRF + django-filter + django-cors-headers (SQLite default / Supabase Postgres) · Next.js 15 (App Router, TypeScript) · SWR · Tailwind CSS  
**Evaluator note:** This is a re-submission. V1 scored 63/100 (B). V1's two critical failures were: (1) the entire project was pushed as a single commit, and (2) the frontend design used a completely different visual language (dark navy, indigo) rather than the Nativetalk Figma. This report evaluates V2 against the same rubric. **Late submission note:** Emmanuel did not submit with the rest of the cohort. A 4-point late submission deduction is applied to the final score and itemised below.

---

## Manual Testing Findings (Pre-Review)

| Finding | Severity |
|---|---|
| UI is the closest to Figma in the cohort | Positive |
| Zero setup friction — SQLite default, no `.env` needed | Positive |
| Create customer: works, tags work, duplicate email rejected | Positive |
| Create ticket: works, comments work | Positive |
| Dashboard Quick Actions open modals | Positive |
| New ticket modal doesn't check for / select existing customers | Moderate UX gap |
| Completion chart may appear empty without seed data | Minor |
| Filter button is cosmetic | Minor UX |
| Notification bell is cosmetic | Cosmetic |
| Logo branding says "Ticket Matrix", not Nativetalk logo | Cosmetic |

---

## What Improved from V1

V1 had two critical failures. V2 addresses both directly and goes significantly further.

| V1 Issue | V1 | V2 |
|---|---|---|
| **Single commit** | 1 commit for entire project | 18 atomic commits, all conventional format |
| **Wrong design** | Dark navy sidebar, indigo primary | Nativetalk light theme, green primary |
| `CustomerViewSet` read-only | `ReadOnlyModelViewSet` — POST 405 | `ModelViewSet` — full CRUD |
| `/customers` page missing | 404 on sidebar link | Full page: table, search, pagination, New Customer modal |
| `category` free-text | No choices — inconsistent with Figma | `Category` as TextChoices (technical/billing/account/feature_request/general) |
| `ALLOWED_HOSTS = ["*"]` | Insecure wildcard | Env-configurable with safe default |
| No seed data | No demo data command | `seed_demo` management command |
| No mobile nav | Fixed sidebar — broke on small screens | Mobile hamburger drawer below `lg` |
| No Docker | Local setup only | Full docker-compose (backend + frontend) |

**New in V2 (not present in any form in V1):**
- UUID primary keys on all models
- `Customer.tag` (VIP/Frequent/New) as `TextChoices` with model-level validation
- `Customer.channels` as `JSONField` with serializer-level validation against an allowed set
- `Customer.company`, `Customer.avatar_url`
- `Ticket.reference` auto-generated as `TKT-{8hex}` on first save
- `ON_HOLD` status tier
- `TicketListSerializer` vs `TicketDetailSerializer` split — list returns `comments_count` (integer), detail returns full comment objects — prevents payload bloat
- `TicketStatsView` with `TruncMonth` for server-side 12-bucket monthly completion bars
- SWR for client-side data fetching with `mutate()` on writes
- Zero border-radius design rule enforced in both `tailwind.config.ts` and `globals.css`
- Postman collection with auto-saving of `customer_id` / `ticket_id`
- Deployment guide (`docs/deploy/README.md`) for Vercel + Render + Supabase

---

## Code Review

### Backend

**`apps/customers/models.py`**

```python
class Customer(models.Model):
    class Tag(models.TextChoices):
        NONE = "", "—"
        VIP = "vip", "VIP Customer"
        FREQUENT = "frequent", "Frequent Buyer"
        NEW = "new", "New"

    CHANNEL_CHOICES = ("instagram", "facebook", "whatsapp", "email", "sms")

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    tag = models.CharField(max_length=16, choices=Tag.choices, default=Tag.NONE, blank=True)
    channels = models.JSONField(default=list, blank=True)
```

UUID PKs throughout — correct for a multi-tenant system where numeric IDs would be guessable. `Tag` as `TextChoices` gives you both DB validation and clean display labels. `channels` as a `JSONField` with `CHANNEL_CHOICES` tuple on the class is the right trade-off: queryable in SQL when needed, flexible for multi-select. The `CustomerSerializer.validate_channels()` method enforces the allowed set at the serializer level with a clear error message listing allowed values.

**`apps/tickets/models.py`**

`Ticket.reference` auto-generates as `TKT-{uuid.uuid4().hex[:8].upper()}` in `save()` — an 8-hex-char reference (e.g. `TKT-1A2B3C4D`) that's URL-friendly, human-readable, and collision-resistant for a demo. Five statuses: `open`, `in_progress`, `on_hold`, `resolved`, `closed` — more complete than most submissions. `Category` as TextChoices (not free-text as in V1). `progress` as `PositiveSmallIntegerField(default=0)` for 0–100. DB indexes on `status`, `priority`, `category`.

**`apps/tickets/views.py`**

`TicketViewSet` with `select_related("customer").prefetch_related("comments")`. `get_serializer_class()` dispatches to `TicketListSerializer` for list, `TicketCreateSerializer` for writes, `TicketDetailSerializer` for retrieve — clean separation. The `@action(detail=True, methods=["get","post"], url_path="comments")` nested comment endpoint is correct DRF router idiom.

`TicketStatsView` is the strongest dashboard aggregation in the cohort:

```python
monthly_qs = (
    qs.annotate(month=TruncMonth("created_at"))
    .values("month")
    .annotate(
        total=Count("id"),
        done=Count("id", filter=Q(status=Ticket.Status.RESOLVED) | Q(status=Ticket.Status.CLOSED)),
    )
)
```

Single ORM query using `TruncMonth` + conditional `Count(filter=...)`. Produces 12 server-side buckets with real completion percentages per month. Fallback to 0.0 for empty months. Returns `monthly_completion` array ready for the chart.

**`apps/tickets/serializers.py`**

`TicketCreateSerializer.validate()` explicitly checks that either `customer_id` or both `customer_name + customer_email` are provided — correct server-side validation. `create()` does `get_or_create` by email. `TicketListSerializer` uses `comments_count = serializers.IntegerField(source="comments.count", read_only=True)` — correct. The list doesn't pull in all comment objects, just the count.

**`settings.py`**

`CorsMiddleware` is the very first middleware entry — correct. `CORS_ALLOWED_ORIGINS` + `CSRF_TRUSTED_ORIGINS` both read from env with the same default list. `ALLOWED_HOSTS` from env (not `["*"]`). `DATABASE_URL` branch: if set, uses `dj_database_url.parse(..., ssl_require=True)` with 600s pool; if unset, falls back to SQLite. `whitenoise` for static serving. Pagination at `PAGE_SIZE=20` globally.

---

### Frontend

**`lib/api.ts` — most comprehensive typed API client in the cohort**

All types and the fetch client in one file. Union types for every enum-like field:

```ts
export type TicketStatus = "open" | "in_progress" | "on_hold" | "resolved" | "closed";
export type TicketPriority = "low" | "medium" | "high" | "urgent";
export type TicketCategory = "technical" | "billing" | "account" | "feature_request" | "general";
export type CustomerTag = "" | "vip" | "frequent" | "new";
export type CustomerChannel = "instagram" | "facebook" | "whatsapp" | "email" | "sms";
```

`TicketListItem` vs `TicketDetail extends TicketListItem` interface split mirrors the serializer split — list has `comments_count: number`, detail has `comments: Comment[]`. `Paginated<T>` generic type for any paginated endpoint. `NewTicketPayload` and `NewCustomerPayload` are explicit write shapes, separate from the read types. `request<T>()` handles 204 no-content correctly (`return undefined as T`), surfaces JSON error detail to the caller.

SWR usage throughout: `useSWR("/tickets/stats/", api.fetcher)` — consistent key + fetcher pattern. After mutations, `mutate()` is called with a key filter function to invalidate all matching cache entries:

```ts
await Promise.all([
  mutate((key) => typeof key === "string" && key.startsWith("/tickets/")),
  mutate("/tickets/stats/"),
  mutate((key) => typeof key === "string" && key.startsWith("/customers/")),
]);
```

This is the most correct SWR invalidation pattern in the cohort — it covers all ticket list variants regardless of query string.

**Dashboard (`app/(dashboard)/dashboard/page.tsx`)**  
Three parallel SWR fetches: stats, customers, recent tickets. Metric cards for Comments (derived from recent tickets' `comments_count`), Customers (from `customers?.count`), Tickets (from `stats?.total`). All real data. The Quick Actions component wires all three buttons:

```tsx
{ label: "Create New Customer", onClick: () => setShowCustomerModal(true) }
{ label: "Create New Ticket",   onClick: () => setShowTicketModal(true)  }
{ label: "View All Tickets",    onClick: () => router.push("/tickets")   }
```

`CompletionChart` renders real `stats.monthly_completion` data — pure SVG bars (no chart library), with the peak month highlighted in brand green. When no data is seeded, all 12 buckets show at minimum height (`Math.max(ratio * 100, 2)%`) — the chart renders but looks visually empty. The user observed "not sure the task conversion rate analytics is working" — this is a seed-data UX issue, not a code defect.

**New Ticket Modal — existing customer lookup gap**  
The modal collects `customer_name` but no `customer_email`, synthesizing one:

```ts
const synthesizedEmail =
  form.customer_email?.trim() ||
  `${form.customer_name.trim().toLowerCase().replace(/\s+/g, ".")}@nativetalk.test`;
```

If the user enters "Joseph Olorunmeyan" twice, they'll get `joseph.olorunmeyan@nativetalk.test` both times — `get_or_create` will match and reuse the customer. But if the name has a typo or the user enters just "Joseph", a new `joseph@nativetalk.test` customer is created. There is no dropdown to select an existing customer, and the email synthesis is invisible to the user. The user's observation ("the customer column is not checking for existing customer") is correct. A customer select populated from the API — as Ony's modal does — would be the right fix.

**New Customer Modal**  
Full-featured: name, email, phone, company, tag (select), channels (multi-toggle with visual state). `validate_channels()` server-side and `toggleChannel()` client-side. SWR `mutate()` on success invalidates all customer queries. Error surfaced to user in UI.

**Customers page**  
Server-side search: `useSWR("/customers/?search=...", api.fetcher)` — the search input updates the SWR key, triggering a backend filter via Django's `search_fields`. This is the only submission using server-side search correctly. The table has full responsive behavior: card list below `md`, table at `md+`. Real phone, tag, and channels shown from model data.

Pagination: DRF sends `count`, `next`, `previous` in the response but the frontend pagination buttons are cosmetic — prev/next are always `disabled`. The `count` and `results.length` are displayed correctly, but page navigation is not implemented.

**Completion chart**  
The `CompletionChart` component bars are proportional to `bucket.rate / 100` — a 75% completion month shows a 75% height bar. The peak month (highest rate) is highlighted green; others gray. All 12 months are always rendered. This is a correct pure-SVG implementation without any chart library dependency.

**Design**  
The commit `feat(frontend): adopt nativetalk light theme, palette, sidebar, primitives` restructured the design. The zero-border-radius rule is enforced at two levels (tailwind reset + `*` global rule) which matches the Figma's sharp-cornered aesthetic. The branding name "Ticket Matrix" is used throughout (README, watermarks, page metadata) instead of "Nativetalk" — the logo is different from the spec.

---

## Summary of Remaining Issues

| Issue | Impact |
|---|---|
| New ticket modal doesn't look up existing customers (synthesizes email) | Moderate UX |
| Customers pagination UI is cosmetic (prev/next always disabled) | Minor UX |
| Completion chart appears empty without seed data | UX/setup |
| `progress` field exists on model but no UI control to set it | Minor feature gap |
| `ON_HOLD` status has no option in the detail page status select | Minor UI gap |
| Filter button is cosmetic | Minor UX |
| Logo/branding is "Ticket Matrix", not Nativetalk logo | Cosmetic/identity |
| Notification bell is cosmetic | Cosmetic |
| **Late submission** | -4 pt deduction |

---

## Comparison: V1 vs V2

| Dimension | V1 | V2 |
|---|---|---|
| Git history | 1 commit | 18 descriptive commits |
| Design language | Dark navy, indigo — wrong | Nativetalk light theme ✅ |
| Customer CRUD | Read-only ViewSet | Full ModelViewSet |
| Customers page | 404 | Full: search, table, modal, tags, channels |
| Customer model | name, email only | + phone, company, tag, channels, avatar_url, UUID PK |
| Ticket model | No reference, free-text category | reference code, TextChoices category, UUID PK, progress, ON_HOLD |
| Dashboard chart | Donut + priority bars (no Figma match) | Monthly completion bars (Figma match) |
| Chart data | Backend aggregation, but wrong chart type | `TruncMonth` monthly completion — correct |
| Mobile nav | None | Hamburger drawer below `lg` |
| Seed data | None | `seed_demo` management command |
| Docker | None | Full docker-compose |
| Postman | None | Full collection with chained variables |
| Deployment docs | None | Vercel + Render guide |
| Error states | User-visible | User-visible (maintained) |

---

## Scoring

### 1. Speed vs Accuracy — 21 / 25

V2 delivers the Nativetalk light theme, all four core pages, working Quick Actions, a full customers page with real tag/channel data, and the monthly completion chart from real backend data. The backend model is the richest in the cohort. All core CRM flows work end-to-end.

Deductions: the new ticket modal doesn't resolve existing customers from the database — it always creates via `get_or_create` using a synthesized email, which can create duplicate customer records when the same person submits without an email. Customers pagination is cosmetic. The completion chart appears empty unless seed data is run, which is not communicated in the UI.

### 2. Prompting & Context Management — 22 / 25

`lib/api.ts` is the strongest frontend type contract in the cohort — every backend enum has a union type, every serializer has a corresponding interface, list and detail shapes are explicitly split. SWR key invalidation after mutations covers all affected endpoints. The `CompletionChart` consumes the backend's `monthly_completion` array without transformation. `validate_channels()` server-side and `toggleChannel()` client-side are in sync.

Deductions: the email synthesis workaround in the new ticket modal (`name@nativetalk.test`) is a notable context gap — the existing customer lookup pattern (select from `/customers/` in the modal) was not prompted. `progress` field is on the model and in the types, but no form field was built to set it.

### 3. Architectural Oversight — 22 / 25

`CorsMiddleware` first, `CSRF_TRUSTED_ORIGINS` mirrors CORS list, `ALLOWED_HOSTS` from env, `whitenoise` for statics, `ssl_require=True` on Postgres connection, SQLite fallback, `psycopg[binary]` (v3), `gunicorn` for production, Docker. `TicketListSerializer`/`TicketDetailSerializer` split prevents comment payload bloat on the list endpoint. `TruncMonth` server-side aggregation (single query). Mobile nav drawer, customers card/table responsive layout. Error states visible in UI.

Deductions: frontend customers pagination is cosmetic despite the backend returning `next`/`previous`. `ON_HOLD` status has no option in the ticket detail status selector. The `progress` field has no write UI.

### 4. Documentation — 23 / 25

18 commits following `feat:`, `chore:`, `docs:` conventional format, each isolating a single concern. README covers Docker, non-Docker, API table, Postman guide, deployment guide, and a `Plan compliance` table mapping every brief requirement to a file and commit. `seed_demo` and `db_check` management commands are documented and functional. `.env.example` files for both layers. Postman collection auto-saves IDs between requests.

Deductions: "Ticket Matrix" branding (README title, file watermarks, page metadata) rather than Nativetalk. The plan-compliance table references a GitHub URL, which evaluators may not have access to.

---

## Merit Score (Before Late Penalty)

| Category | Score |
|---|---|
| Speed vs Accuracy | 21 / 25 |
| Prompting & Context Management | 22 / 25 |
| Architectural Oversight | 22 / 25 |
| Documentation | 23 / 25 |
| **Subtotal** | **88 / 100** |

On merit alone, this is the strongest submission in the cohort (Ony's merit score is 85/100).

### Late Submission Adjustment: −16 points

Emmanuel submitted 4 days after the cohort deadline. Penalty: **4 points × 4 days = 16 points**, deducted from the merit subtotal.

---

## Final Score

| Category | Score |
|---|---|
| Speed vs Accuracy | 21 / 25 |
| Prompting & Context Management | 22 / 25 |
| Architectural Oversight | 22 / 25 |
| Documentation | 23 / 25 |
| **Merit Subtotal** | **88 / 100** |
| **Late penalty (4 days × 4 pts)** | **−16** |
| **Total** | **72 / 100** |
| **Grade** | **A** |

**Change from V1:** 63/100 (B) → 72/100 (A)

V2 is a near-complete rebuild and the strongest technical submission in the cohort on merit. Every V1 failure is addressed: 18 descriptive commits replace the single squash, the design matches the Nativetalk light theme, the customers page is fully built, and the customer viewset is writable. The backend model is the most complete — UUID PKs, reference codes, tag/channels on Customer, TextChoices categories, a `TruncMonth` stats endpoint, Docker, and a Postman collection. The only remaining workflow gap is the new ticket modal not looking up existing customers. The late submission deduction brings the final score below Ony (85), reflecting the timing disadvantage borne by the rest of the cohort.
