# Trucking Logistics Platform — From Bespoke System to Multi-Tenant SaaS

A case study on a production system I built and operate, and on the work of turning it into a
multi-tenant product. Source code is not included — this documents the architecture, the
engineering decisions, and what each one cost.

The system runs the full job lifecycle for a road-freight operator: instruction creation,
leg assignment, invoicing, payment allocation, payroll, and month-end statements. It has been
live and in daily use by controllers, finance clerks, and directors. The second half of this
document covers the conversion of that single-customer system into a subscription product
serving several operators from one deployment.

---

## Contents

1. [System overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Tech stack](#3-tech-stack)
4. [Part I — the single-tenant system](#4-part-i--the-single-tenant-system)
5. [Part II — the multi-tenant conversion](#5-part-ii--the-multi-tenant-conversion)
6. [Financial correctness: the parts that mattered most](#6-financial-correctness-the-parts-that-mattered-most)
7. [What I would do differently](#7-what-i-would-do-differently)

---

## 1. System Overview

The interesting parts of this system are not the CRUD screens. They are the financial
calculations that have to be reproducible after the fact: revenue attributed correctly across
multi-leg jobs, payroll that resolves against historical rates, and client statements that are
true point-in-time snapshots rather than accumulated ledgers. Most of the design decisions below
exist to make those calculations correct and auditable.

The domain is genuinely relational — 31 tables in the original schema, joined along a spine of
instruction → legs → trucks → invoice → payment → statement. Analytics is CTE-heavy SQL over
that spine. Compliance documents (driver licences, roadworthy certificates, fuel slips) live in
S3. Month-end statement generation is fired on a schedule from outside the application process.

The multi-tenant version keeps all of that and adds a tenancy key, a plan catalogue, feature
gating, usage accounting, and an operator-facing admin plane — growing the schema to 35 tables,
29 of which are tenant-scoped.

---

## 2. Architecture

![Single-tenant architecture: a React SPA behind Nginx calls an Express API whose route modules sit below a single global auth guard; the model layer issues raw SQL to PostgreSQL and mints presigned S3 URLs for compliance documents, while EventBridge fires a monthly Lambda that calls an authenticated endpoint to generate all client statements in one transaction.](docs/architecture-single-tenant.svg)

**Authenticated request path.** The SPA sends `Authorization: Bearer <jwt>` on every call. Nginx
forwards to Express. A single global `verifyToken` guard mounted in the central router validates
the token and populates the request with the user's identity, role, and tenant key. The model
layer then executes a parameterised query. Document access returns a time-limited S3 presigned
URL minted server-side — AWS credentials never reach the browser.

**Scheduled statement generation.** EventBridge fires a Lambda on the 1st of each month. The
Lambda calls `POST /api/statements/generate` with a shared-secret bearer token, authenticated
ahead of the normal JWT path. The Lambda is a trigger, not an implementation — the generation
logic stays inside the API next to the models it depends on.

The multi-tenant version of this diagram — the middleware chain, the admin plane, and the
per-tenant month-end — is in [Part II](#5-part-ii--the-multi-tenant-conversion).

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend framework | React 19 | Concurrent rendering; stable ecosystem for a data-heavy internal tool |
| Routing | React Router v7 | Nested route layout matching the role-scoped page hierarchy |
| Styling | Tailwind CSS v4 | Utility-first; avoids a growing bespoke CSS surface |
| Animation | Framer Motion | Declarative transitions for modals and page entries |
| Charts | Recharts + Chart.js | Recharts for composable data viz; Chart.js for types Recharts doesn't cover |
| PDF generation | jsPDF + autotable + html2pdf | Client-side tax invoices and wage slips, no server-side PDF service |
| Excel export | ExcelJS | Structured `.xlsx` export for financial reports |
| HTTP client | Axios | Interceptors attach the auth token globally |
| Backend framework | Express.js | Minimal and well-understood; a structured REST API doesn't need GraphQL |
| Authentication | Passport.js (LocalStrategy) + JWT | Passport for interactive login; JWT for stateless SPA calls |
| Password hashing | bcrypt | Industry-standard adaptive hashing |
| Security headers | Helmet | Sensible defaults in one line |
| Rate limiting | express-rate-limit | Applied to auth routes specifically — brute-force protection where it matters |
| Request validation | Zod | Schemas on the financially material writes; coerces browser-sent string numbers |
| Database | PostgreSQL | JSONB, arrays, window functions, and exact `NUMERIC` for money |
| Database access | node-postgres, raw SQL | Multi-CTE and lateral-join analytics are easier to reason about in SQL |
| File storage | AWS S3 (af-south-1) | Durable storage for compliance documents; presigned URLs keep credentials server-side |
| Upload middleware | multer / multer-s3 | Streams multipart uploads straight to S3 without buffering to disk |
| Scheduled jobs | AWS Lambda + EventBridge | Decouples the monthly run from application uptime |
| Deployment | GitHub Actions → Elastic Beanstalk | Push-to-deploy per branch; client and server ship as one versioned artifact |
| Reverse proxy | Nginx | TLS termination and the raised body limit needed for document uploads |
| Module system | ESM | Consistent module syntax across client and server |

---

## 4. Part I — The Single-Tenant System

### PostgreSQL as system of record, not a document store

The rest of the stack is AWS, so DynamoDB was the path of least resistance. It was the wrong fit
on four counts. The domain is inherently relational and the join graph is not knowable in advance
— an instruction has legs, legs have trucks and drivers, an instruction produces an invoice,
invoices are partly settled by payments, payments roll into statements. Analytics is ad-hoc and
aggregate-heavy: revenue attribution divides an instruction total by a leg count and a per-leg
truck count computed in sibling CTEs, which is a join problem with no join primitive available in
DynamoDB. Money needs multi-row transactions spanning an unbounded client list, which exceeds
DynamoDB's transaction limits. And financial correctness needs exact decimals — currency is
`NUMERIC(12,2)` with the driver's type parsers overridden so values never silently become
floats.

The scale argument that normally favours a key-value store doesn't apply: this is tens of users
per operator at a volume a single small RDS instance absorbs without effort. The cost is real
though — a relational database is a vertical-scaling bottleneck and a single point of failure,
the connection pool is capped at 20, and horizontal scaling would need pgBouncer or a read
replica first.

### Raw SQL throughout, no ORM

The analytics layer needs multi-CTE queries, lateral joins to explode embedded JSON arrays, and
window functions to resolve container rates. These are natural in SQL and tend to become
unwieldy in an ORM — at which point you reach for the raw-query escape hatch anyway and lose most
of the benefit. So the project skips the abstraction and writes the queries directly.

The honest cost: no compile-time type safety on query results, and — more consequentially — no
migration runner came along for free. I let the query-layer decision bleed into schema
management, where it doesn't belong. More on that in the final section.

### Embedded JSON for payment line items

A payment in this domain frequently allocates one sum across several invoice dates in a single
transaction. That maps cleanly onto one payment row with an embedded array of applications: the
array is the natural write unit, and reading a payment in full needs no join.

The trade-off surfaces on the analytics side, where aggregating across large payment volumes
requires lateral joins that cost more than indexed foreign-key lookups, and where individual line
items cannot be independently indexed or constrained by the database. For the write and
single-read patterns that dominate here, that was the right side of the trade; for a
reporting-heavy workload it would not be.

### Statement aging recomputed live, not carried forward

Each monthly client statement is built by querying outstanding instructions and ad-hoc charges as
they stand at generation time and aging each item against the generation date — rather than
taking last month's closing balance and adjusting it.

Carry-forward is cheaper, but it compounds any data error month over month and makes backdated
corrections genuinely hard to reconcile. Recomputing from live outstanding items means every
statement is a true point-in-time snapshot of what is actually owed. The price is that generation
is more expensive, and that an incorrectly set payment status propagates into every statement
until it's fixed — but a wrong number traceable to current data beats a wrong number inherited
from an opaque running total.

### Statement generation triggered externally, not by in-process cron

The original implementation was `node-cron` inside the Express process. That dependency is gone,
for three reasons that only show up in production:

- **An in-process timer misses the run if the process isn't up at that instant.** Generation fires
  once a month. A deploy, a crash, or an instance replacement at that moment silently skips it,
  and nobody notices until a client asks where their statement is.
- **It breaks the moment there's more than one instance.** Every instance fires its own timer, so
  scaling to two means generating every statement twice. The schedule was coupled to the
  deployment topology.
- **Retries and failure visibility come free from the scheduler.** EventBridge retries on failure
  and surfaces the run in CloudWatch without any of that being built into the app.

The Lambda is a trigger, not an implementation — the generation logic stays next to the models and
the pool it depends on, and the same endpoint backs the manual regeneration path in the finance
UI, so the scheduled and manual routes exercise identical code rather than drifting apart.
Re-firing is safe: an existing statement for the period is updated in place rather than
duplicated.

The costs are equally concrete. The schedule and the Lambda live outside the repo, so the
deployment is no longer fully described by the codebase. The shared secret is symmetric — anyone
holding it can trigger arbitrary statement regeneration, and rotating it requires a coordinated
change on both sides. And local development can't exercise the scheduled path end to end.

### Auth enforced by one global guard, with public routes deliberately above it

Per-route middleware fails open: a new route added without it is public, and nothing catches
that. Mounting the guard once in the central router after a small public block inverts the
default — a newly added route module is authenticated unless someone deliberately edits above the
guard line. There are four exceptions (auth, landing stats, health check, and the statement
trigger with its own secret check), each annotated in place.

The trade-off is that ordering in the router file is now load-bearing. Moving a mount across the
guard silently changes the security posture of every route in that module, and nothing in the
type system or the tests catches it. The frontend mirrors this with route guards, but those are a
UX affordance — the server guard is the enforcement point.

### Documents read through short-lived presigned URLs

Compliance documents are personal and regulatory records, so a public bucket is out regardless of
key entropy. Proxying every document byte through Express would work but consumes request
capacity on an instance sized for JSON traffic. Presigned URLs let the browser fetch directly from
S3 while the authorisation decision stays server-side: the API checks the user's role, then mints
a time-limited URL.

The sharp edge is that a presigned URL is a bearer token for that object until it expires. If a
user forwards one, the recipient needs no login, and access to the object itself isn't audited —
only the issuance is visible to the application.

### PDFs generated in the browser, not on the server

Tax invoices, wage slips, and financial exports are produced interactively, one at a time, by a
user looking at the data on screen. Rendering client-side builds the document from the exact view
state already loaded — no second serialisation path that can drift from what the user saw — and
keeps headless Chrome's memory footprint and cold-start latency off an instance that is also
serving the API.

The consequence is structural: documents can't be produced without a browser session. That rules
out emailing a statement PDF from a scheduled job, which is precisely why statement generation
writes rows to the database rather than producing files. Bulk generation — every wage slip for a
month in one operation — isn't possible either.

---

## 5. Part II — The Multi-Tenant Conversion

Turning a system built for one customer into one serving several is mostly not a feature problem.
It is a problem of adding an invariant to code that was written without it, and then having no
mechanism that enforces it.

![Multi-tenant architecture: the same Express pipeline gains a middleware chain — the auth guard populates the tenant key and plan state from the JWT, a read-only guard rejects writes from suspended tenants, plan and feature middleware gate access, and usage checks warn without blocking; the model layer filters every query by tenant against a shared schema of 29 tenant-scoped and 6 global tables, a cross-tenant super-admin plane assigns plans and suspends accounts, and the scheduled month-end loops tenants with one transaction each.](docs/architecture-multi-tenant.svg)

### Shared schema with a discriminator column, not a database per tenant

Tenancy is a `company_reg_num` column carried on every business table — 29 of the 35 tables in the
SaaS schema. The six that aren't tenant-scoped are deliberately global: the plan catalogue, the
feature-gate table, the role list, shipment types, supplier expense types, and the tax-deduction
bracket table. Those are reference data; duplicating them per tenant would mean a tax-bracket
update becomes an N-tenant migration.

Database-per-tenant and schema-per-tenant were the alternatives. Both give isolation the shared
model has to earn in application code, and both were rejected for the same reason: at the target
scale of a handful of operators, they multiply the operational surface — migrations to run N
times, connection pools to size across N databases, backups and restores per tenant — to solve an
isolation problem that a single well-tested predicate also solves. The shared column also
composes with the existing analytics: a scoping predicate is just another `WHERE` clause
alongside the joins and aggregates, where a per-tenant database would have fragmented every
cross-cutting query.

**What this actually costs, stated plainly:** isolation is an application-level invariant, not a
database-level one. Every model function takes the tenant key and every query filters on it; the
controller layer passes `req.user.company_reg_num` down from the verified JWT. There is no row-level
security, so a single model function written without the predicate is a cross-tenant data leak
that no constraint will catch. The discipline is documented as a project rule and the audit
covered every model, but "we checked carefully" is a weaker guarantee than "the database refuses."
That gap is the single most important thing to know about this design, and it is the first item in
the final section.

### Two independent gating axes: plan rank and feature key

Access control ended up needing two orthogonal mechanisms, because two different questions were
being asked.

`requirePlan('professional')` answers *is this tenant's plan at least this tier?* — a numeric rank
comparison across four ordered plans. `requireFeature('analytics')` answers *does this tenant's
plan include this specific capability?* — a lookup against a plan-to-feature mapping table.

Rank alone can't express a capability that a lower tier has and a higher one doesn't, and feature
keys alone force you to enumerate every feature for every plan at every call site. Having both
means the common case (higher tier includes everything below) is a one-word comparison, while the
catalogue stays editable in the database rather than in deployed code — adding a feature to a plan
is a row, not a release.

A third mechanism sits alongside these: role creation is gated by plan. Each employee role maps to
a minimum tier, checked both when an employee is created and again at login — so downgrading a
plan doesn't leave previously created accounts with access their tier no longer includes.

### Subscription state travels in the JWT

The token carries the tenant key, role, plan tier, subscription status, and trial expiry. Gating
middleware therefore reads plan state from the verified token instead of hitting the database on
every request — which matters when the check runs on essentially every authenticated call.

The trade-off is staleness, and it is not cosmetic. A token is valid for 24 hours, so a plan
upgrade, a downgrade, or a suspension applied by an administrator does not reach a user who is
already logged in until their token is reissued. For upgrades this is merely annoying. For
suspension it is a correctness hole, and it is why the suspension middleware doesn't simply trust
the token: it treats a non-suspended status in the token as a fast path, and falls through to a
live database read otherwise. That narrows the window rather than closing it. Closing it properly
means either short-lived tokens with refresh, or a revocation check on state-changing requests —
both of which trade the thing the design was buying in the first place.

### Usage limits warn and bill; they don't block

Each plan carries a seat cap and a vehicle cap. When a tenant is at or over the cap, the
middleware attaches a warning describing the overage charge and calls `next()` — creation
proceeds.

This was a deliberate product decision, not an incomplete implementation. The user hitting the cap
is a controller trying to get a truck onto a job today; blocking them converts a billing
conversation into an operational outage for a customer who has already demonstrated they want to
pay for more. Recording the overage and surfacing it in the UI and in the monthly usage snapshot
gets the commercial outcome without the outage. The same reasoning drives the middleware failing
open on error: a database hiccup in a *billing* check should never block *operational* work.

The obvious cost is that a tenant can run well over their plan indefinitely, and the system relies
on the billing process — not the software — to collect. Fail-open also means a persistent fault in
the usage query silently stops overage accounting altogether, and nothing surfaces that except
noticing the numbers look wrong.

### Suspension is read-only, not lockout

A suspended tenant can still read everything. Mutating requests are rejected with a 402 and a
message carrying the reason; `GET` always passes through.

A logistics operator's data is their operational record — invoices they need for tax, statements
their clients are querying, compliance documents. Locking them out of *reading* it over a billing
dispute is disproportionate and, in the case of statutory records, arguably worse than
disproportionate. Read-only applies real commercial pressure — they cannot run the business
forward — without holding their history hostage. It also makes reactivation instant and lossless.

### Onboarding is admin-approved, not self-serve

Registration creates a company in a pending state. A super-admin then assigns a plan, records
whether the setup fee is paid, sets the billing anchor day, and optionally starts a trial. Every
one of those actions writes to an append-only billing event log with the acting admin's identity.
Per-tenant limit overrides exist so a negotiated deal doesn't require a new plan row.

There is no payment gateway. That is a deliberate scope decision rather than an oversight: these
are B2B contracts with setup fees, negotiated terms, and invoicing — the sales motion already
involves a human, and building self-serve card payments would have added PCI surface and a
subscription-lifecycle state machine to serve a flow that doesn't exist yet. The cost is that
onboarding doesn't scale past the operator's own attention, and that plan state is only as
accurate as the admin keeping it current.

### Login became a policy decision point

Login now resolves a post-login destination from four inputs: role, plan tier, subscription
status, and trial expiry. Suspended, cancelled, pending-activation, and trial-expired tenants each
land on a dedicated screen explaining their state rather than on a dashboard that silently fails
to load. Lite-tier tenants land on a reduced dashboard built for the features their plan actually
includes, instead of a full dashboard where most cards are upgrade prompts.

The routing rule lives in one exported function used by both the login redirect and the route
guards, so the client can't disagree with itself about where a given subscription state belongs.

### Multi-tenancy reached the scheduled job too

The month-end run was the one place where the tenancy change had a genuine architectural
consequence rather than a mechanical one. The original ran every client in a single transaction
and rolled the whole thing back on any error — correct when all those clients belong to one
operator.

Under multi-tenancy that same behaviour means one tenant's malformed data cancels every other
tenant's month-end. So the handler enumerates active companies and runs a generation pass per
tenant, each in its own transaction, collecting per-company results. Atomicity is preserved where
it means something — within a tenant — and abandoned where it never did.

### Test coverage where the blast radius is largest

The SaaS work added Jest coverage over the pieces where a silent failure is most expensive: the
plan and subscription models, the gating middleware, and the admin controller. That is deliberately
narrow. It is the part of the system where a bug is either a revenue leak or a cross-tenant
disclosure, and it is also the part with the least domain complexity — which makes it cheap to
test. The financial calculation core remains covered by the frontend component and hook tests plus
manual verification against known-good months, which is the weakest part of the testing story and
an honest gap.

---

## 6. Financial Correctness: The Parts That Mattered Most

**Proportional revenue attribution across multi-leg jobs.** Per-truck and per-subcontractor
turnover splits each instruction's total across the legs it spans and the trucks that ran each
leg. Counting distinct legs per instruction and distinct trucks per leg, then dividing
accordingly, distributes revenue without double-counting when several trucks share one leg or one
instruction spans several legs. It resolves in a single pass over the leg-to-instruction join
rather than in application code stitching together multiple round trips.

**Point-in-time payroll against temporal history.** Wage calculations resolve an employee's base
salary and deductions from history tables as of the last day of the target month, selecting the
most recent values effective on or before that cutoff. A wage slip generated for a past month
therefore stays correct after a pay change, because it computes against the rates in force at the
time rather than today's values.

**Driver rates versioned with effective-date ranges.** Transport rates change, but historical
invoices must remain reproducible at whatever rate applied when the leg actually ran. Rather than
keeping only the current rate — silently corrupting past calculations on every change — or
reaching for full event sourcing, the rate table gained effective-from and optional effective-to
columns, with all pre-existing rates backdated to the start of recorded history so nothing broke.
A composite index keeps the date-range lookup efficient. The sharp edge: nothing in the database
stops two overlapping ranges existing for the same route.

**Invoice date anchored to service delivery, not record creation.** An invoice's date is the
earliest date on which the job's first leg actually ran, not the timestamp the invoice was
created. This anchors the document to when the service was delivered, which is what matters for
tax-period reporting and aging analysis. A small detail that prevents a whole category of
period-boundary disputes.

**Two user tables behind one login flow.** Authentication checks an owner/staff table and an
employee table in turn, tagging the session with which table the account came from; session
restore branches on that tag. This lets two account types with genuinely different schemas share
one login flow, while preserving the freedom to keep them separate or merge them later without
touching the auth middleware. A pragmatic accommodation of two pre-existing data shapes rather
than a forced unification.

---

## 7. What I Would Do Differently

**Enforce tenant isolation in the database, not in the query layer.** This is the most important
item on the list. Shared-schema tenancy was the right call, but its central invariant — every read
and write scoped to one tenant — lives entirely in application discipline. PostgreSQL row-level
security can enforce exactly this: set the tenant key as a session variable at connection
checkout, define a policy per table, and a query written without the predicate returns nothing
instead of returning someone else's data. The reason it wasn't done that way is ordering — the
tenancy column was retrofitted table by table across a live schema, and RLS on a partially
migrated table is a foot-gun. But retrofitting *after* the migration was possible and I didn't do
it. Application-enforced isolation means correctness depends on every future write path being
disciplined forever; a policy makes the failure mode "no rows" instead of "wrong tenant's rows."

**Enforce rate-range integrity in the database too.** Same class of mistake, smaller blast radius.
The effective-date versioning is sound, but "no two overlapping rate periods for the same route"
lives in application code. A single careless insert produces a silently ambiguous rate lookup that
surfaces later as a quietly wrong invoice. An exclusion constraint over the route and date range
turns a possible silent data-integrity bug into an impossible one. Trusting the application to
hold an invariant the database could guarantee was the wrong default — twice.

**Adopt a real migration runner.** Committing to raw SQL was right for the query layer, but I let
that decision bleed into schema management, where it doesn't belong. Migrations are hand-applied
`.sql` files with no applied-state tracking and no down-paths. The SaaS work made the cost visible:
the branch ended up with two migration directories using overlapping numbering, so "which
migrations has this database had?" became a question you answer by inspecting the database rather
than by reading the repo. The query layer and the schema-change workflow are separable concerns
and I conflated them. A lightweight runner would have cost almost nothing up front.

**Treat token-signing material as persistent infrastructure config from day one.** The original
implementation generated the JWT signing secret from random bytes at process startup, which meant
every deploy or crash invalidated every active session. It has since been fixed — the secret is
now required from the environment in production, with the server refusing to start without it, and
an ephemeral dev fallback that warns loudly. Worth recording *why* it survived as long as it did:
nothing about it shows up in normal testing. Sessions only break across a restart, and in
development you restart so often you stop noticing. That's exactly the class of bug that hides
until production.

**Design the plan-state cache with revocation in mind.** Putting subscription state in the JWT was
the right performance call and the wrong durability call, and the suspension fast-path is a patch
over the gap rather than a fix. If I were doing it again I'd either keep tokens short-lived with a
refresh endpoint that re-reads plan state, or accept one cached lookup on state-changing requests.
The current design makes an administrative action take up to a day to fully land, which is fine
for an upgrade and not fine for a suspension.

**Cover the financial core with tests before covering the billing layer.** The SaaS tests went to
the newest, least complex code. The revenue attribution, aging, and payroll calculations — the
parts where a bug is a wrong number on a document a client acts on — are still verified mostly by
comparison against known-good months. Those calculations are pure functions over query results and
would be entirely testable with fixture data. I tested what was easy to test rather than what was
expensive to get wrong.
