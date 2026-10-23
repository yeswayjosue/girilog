# Supabase → MongoDB Migration Plan (GiriLog)

Status: Draft — for review before execution
Scope decisions locked in: full replacement of Supabase (data + auth), a new backend API is in scope, hard cutover is acceptable (single-user data today), target datastore is MongoDB Atlas.

## 1. Why this is bigger than "swap the database"

Today GiriLog is a pure frontend SPA. The browser talks directly to Supabase using the anon key, and Postgres Row Level Security (`auth.uid() = user_id`) is the only thing enforcing that a user can only see their own rows. Supabase Auth also handles signup, login, password reset, and session tokens.

MongoDB has no equivalent to RLS and no built-in auth service. Exposing a MongoDB connection string or API directly to a browser is not safe. So this migration has three tracks, not one:

1. **Data**: 4 Postgres tables → MongoDB collections.
2. **Access control**: Postgres RLS → enforced in application code, in a new backend.
3. **Auth**: Supabase Auth → a replacement auth system, since we're not keeping Supabase around at all.

## 2. Recommendation: auth approach

You asked for a recommendation here, so here's the tradeoff and the default I'd go with.

| Option | Pros | Cons |
|---|---|---|
| **Custom auth (JWT/session, bcrypt, stored in MongoDB)** | No external dependency; fully in your stack; simple, since GiriLog only uses email/password today (the Google/Apple OAuth buttons are commented out and unused) | You own security-sensitive code (password reset flows, token handling) |
| Clerk / Auth0 (hosted) | Less code to maintain, handles OAuth/MFA if you ever want it | New vendor dependency — trades "off Supabase" for "on a different platform"; overkill for one-user email/password today |
| Keep Supabase Auth, move only data | Least work now | Doesn't achieve full decoupling from Supabase, and you'd still need a backend to check Supabase sessions before hitting Mongo — most of the complexity without the payoff |

**Recommendation: custom auth**, since you're building a backend anyway and the app's auth surface is small (email/password + reset). Concretely:

- `users` collection in MongoDB: `email`, `passwordHash` (bcrypt), `createdAt`.
- Login issues a short-lived JWT (or a session cookie — cookie is simpler to secure against XSS since it can be httpOnly); store it client-side and send on every API request.
- Every API route reads the authenticated user id from the verified token and filters every query by it — this is the code that replaces RLS. It needs to be applied consistently (a shared middleware, not copy-pasted per route).
- If you later want Google/Apple login or MFA, that's a good moment to reconsider Clerk — but not needed for this migration.

## 3. Target architecture

```
Browser (React/Vite SPA)
   │  fetch() to your own API (replaces supabase-js calls)
   ▼
New API server (Node/Express or Fastify)
   │  - auth middleware (verifies JWT/session, attaches userId)
   │  - route handlers mirror current BaseService methods
   │  - enforces { userId } filter on every query (replaces RLS)
   ▼
MongoDB Atlas
```

Every current `src/services/*.ts` file (`ClientService`, `InvoiceService`, `LineItemService`, `SettingsService`) extends a shared `BaseService` that does `getAll / getById / create / update / delete`, always scoped to `user_id`. That maps cleanly to REST or RPC-style endpoints — the API layer is mostly a straight port of that logic, just moved server-side instead of relying on RLS.

Open item to decide during build (not blocking this plan): where the API server itself is hosted (Render/Railway/Fly/a small VPS/etc.) and whether you use Mongoose (schema validation, easier to reason about) or the native MongoDB driver.

## 4. Data model: Postgres → MongoDB

Current schema (from `supabase_schema.sql`): `girilog_clients`, `girilog_invoices`, `girilog_line_items` (FK to invoices), `girilog_settings` — all with a `user_id` column and RLS tied to `auth.uid()`.

Proposed MongoDB collections:

**`users`** (new — replaces `auth.users`)
- `_id` (ObjectId)
- `email`, `passwordHash`, `createdAt`

**`settings`** (was `girilog_settings`)
- `_id`, `userId` (ref → users), `businessName`, `businessEmail`, `businessAddress`, `businessPhone`, `logoUrl`, `invoicePrefix`, `defaultTaxRate`, `currency`, `annualRevenueGoal`, `onboardingDismissed`
- 1:1 with user — consider embedding directly into the `users` document instead of a separate collection, since it's always fetched together and never queried independently. Either works; embedding removes one round trip.

**`clients`** (was `girilog_clients`)
- `_id`, `userId`, `name`, `company`, `email`, `phone`, `address`, `shortCode`, `logoUrl`, `taxEnabled`, `defaultTaxRate`, `defaultHourlyRate`, `showDate`, `showProject`, `projects` (array — was already JSONB, maps 1:1), `createdAt`, `updatedAt`

**`invoices`** (was `girilog_invoices` + `girilog_line_items` merged)
- `_id`, `userId`, `invoiceNumber`, `clientId` (ref → clients, nullable), `clientName`, `clientEmail`, `clientAddress`, `status`, `issueDate`, `dueDate`, `subtotal`, `taxRate`, `taxAmount`, `discountRate`, `total`, `currency`, `notes`, `createdAt`, `updatedAt`
- `lineItems: [ { description, quantity, unitPrice, amount, date, project } ]` — **embedded array, not a separate collection.**

  Why: line items have no independent existence — they're always created, read, and deleted together with their parent invoice (the current schema even cascades deletes). That's the textbook case for embedding in MongoDB instead of a join. It also removes an entire collection and the need to enforce `user_id` on it separately.

  Tradeoff to flag: if you ever want cross-invoice line-item reporting (e.g. "all billed items in Q3 across every invoice"), embedding makes that a bit more work (aggregation over nested arrays instead of a flat query). Given current usage, embedding is still the right call — but worth a deliberate yes/no before you commit, since it's the one real schema judgment call in this migration.

**Field-level notes:**

- **Money fields** (`subtotal`, `taxAmount`, `total`, `unitPrice`, `amount`, etc.): Postgres `NUMERIC` is exact; MongoDB's default number type is a 64-bit float, which can introduce rounding error on currency math. Use `Decimal128` (Mongoose supports this natively) or store money as integer minor units (cents) — don't store raw JS floats. This is worth deciding before the transform script is written, since it determines the conversion logic.
- **IDs**: Postgres `BIGSERIAL` → MongoDB `ObjectId` for `_id`. You'll need an old-id → new-ObjectId mapping table during migration to rewrite foreign keys (`client_id` on invoices, `user_id` everywhere).
- **Timestamps**: `TIMESTAMPTZ` → native `Date` (ISODate), straightforward.
- **`auth.uid()` defaults**: these were Postgres-side conveniences; the new API sets `userId` explicitly from the authenticated session instead.

## 5. Step-by-step migration procedure

### Step 1 — Build the new backend first, against empty data
Stand up the API server, MongoDB Atlas cluster, auth (signup/login/reset), and CRUD routes for all four resources. Test it end-to-end with dummy data before touching real data. This de-risks the migration — by the time you touch your real invoices, the target system is already proven to work.

### Step 2 — Export from Supabase
For each table, export as JSON (cleaner to transform than CSV). Two ways to do this, either is fine at this data volume:
- Supabase SQL editor: `SELECT jsonb_agg(t) FROM girilog_clients t;` (repeat per table) and save the result.
- Or `pg_dump`/`psql` against the Supabase connection string if you want a full relational backup as a safety net too (recommended regardless — see Step 6).

Also export `auth.users` (id, email, created_at) — but **not** usable password hashes. Supabase's hashing isn't portable to a bcrypt scheme you'd own. Flagged explicitly in Step 7.

### Step 3 — Transform
Write a one-off Node script that:
1. Reads the exported JSON per table.
2. Builds the old-id → new-ObjectId map (users first, then clients, then invoices).
3. Merges each invoice's `girilog_line_items` rows into that invoice's `lineItems` array.
4. Converts money fields to `Decimal128`/cents per the decision in Section 4.
5. Converts timestamp strings to `Date`.
6. Rewrites `user_id` / `client_id` references using the id map.
7. Writes one JSON file per target collection, shaped exactly like the new schema.

### Step 4 — Import into Atlas
Use a small Node script with `insertMany()` (not raw `mongoimport`) — you need it anyway to handle `Decimal128` and `Date` conversion correctly, and it gives you a place to log/catch per-document errors instead of failing silently.

### Step 5 — Verify
- Document count per collection == row count per table (accounting for line items now living inside invoices).
- Sum of `total` across all invoices matches pre-migration sum (catches money conversion bugs).
- Every invoice's `clientId` (if set) resolves to a real client document — no orphans.
- Spot-check a handful of full invoices (with line items) side by side, old vs new.

### Step 6 — Cutover
Since this is just your own data and you're fine with some migration pain:
1. Take a final `pg_dump` backup of Supabase, store it somewhere outside Supabase.
2. Run Steps 2–5.
3. Point the frontend's API base URL at the new backend (env var swap) and redeploy.
4. Smoke-test: log in, view dashboard, open an existing invoice, create a new one, edit a client.
5. Keep the Supabase project intact (paused, not deleted) for a couple of weeks as a rollback safety net.

### Step 7 — Auth / password migration (the one real gotcha)
Supabase Auth password hashes can't be reused in a different auth system. Since it's just your account, the simplest path is: create your MongoDB `users` record during Step 4 with no password set (or a random placeholder), then do a normal "reset password" flow against the new system once it's live. Not worth building a "verify against Supabase then migrate hash" flow for one user.

### Step 8 — Decommission
After you're confident the new system is solid (a couple of weeks of normal use is plenty for a single-user app), delete the Supabase project. Keep the final `pg_dump` backup archived somewhere regardless.

## 6. Risk checklist

- Full `pg_dump` backup taken *before* any export/transform work starts.
- Money-field conversion tested on a few sample rows before running it over everything.
- Supabase project paused, not deleted, until the new system has been used for real for a couple of weeks.
- Line-item embedding decision (Section 4) made deliberately, not by default.

## 7. Open questions to settle before/during build

1. Confirm: embed `lineItems` in `invoices`, or keep as a separate collection? (Recommendation: embed, per Section 4.)
2. Money storage: `Decimal128` vs. integer cents? (Either is fine; pick one before writing the transform script.)
3. Where does the new API server get hosted?
4. Mongoose vs. native MongoDB driver for the backend?

None of these block starting Step 1 — they're implementation details the plan above already anticipates either way.
