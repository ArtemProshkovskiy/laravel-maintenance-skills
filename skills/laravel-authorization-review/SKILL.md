---
name: laravel-authorization-review
description: >-
  Laravel-native authorization & IDOR reviewer. Walks the authorization chain of
  every HTTP route — middleware → authorize/policy/gate → query scoping → API
  Resource output — and reports broken object-level authorization (IDOR/BOLA), the
  exact class that taint/SAST scanners miss because it's about intent, not data
  flow. Anchors every finding to real `php artisan route:list --json` output plus a
  cited controller `file:line`, classifies by confidence (High/Medium/Low), and
  produces a per-route COVERAGE MAP plus a prioritized, advise-only plan. Use when
  the user wants to review authorization, find IDOR / broken access control,
  audit which endpoints are unprotected, check policies/gates coverage, or sanity-
  check a new endpoint in a PR — e.g. "review my authorization", "find IDOR in
  this app", "which routes have no auth", "is this endpoint scoped to the owner",
  "do my controllers check policies", "audit access control before launch".
  Advise-only: never edits code; reports findings with evidence for a human to fix.
license: MIT
compatible_agents:
  - Claude Code
  - Cursor
  - Codex
tags:
  - laravel
  - php
  - security
  - authorization
  - idor
  - bola
  - access-control
compatibility: >-
  Requires a Laravel project (artisan on PATH). Uses `php artisan route:list --json`
  as the ground-truth route inventory; falls back to static parsing of routes/*.php
  when the app can't boot. Covers HTTP routes, controllers, policies, gates,
  FormRequests, Eloquent query scoping, and API Resource output. Does NOT cover
  Livewire/Filament/Nova action authorization (no route:list anchor — see Boundaries).
metadata:
  author: "Artem Proshkovskyi"
  version: "0.1.0"
  category: "security / advisor"
allowed-tools: Bash(php artisan route:list:*) Read Glob Grep
---

# Laravel Authorization Review

> **SAST tells you where data flows. This tells you where it flows to the wrong user.**

This skill is a **judgment layer** for the one security category automated scanners
structurally cannot do: **broken object-level authorization (IDOR / BOLA)** — #1 in
the OWASP API Security Top 10. Taint scanners trace untrusted *input*; they cannot
decide whether `Order::find($id)` *should* have been scoped to the current user.
That is a question about **intent**, and it needs reasoning across middleware,
controller, policy, and query — exactly what an LLM can do and grep cannot.

The reason this skill is trustworthy and not a hallucination engine is its **ground
truth anchor**: `php artisan route:list --json` is the deterministic inventory of
every endpoint and its middleware — Laravel's equivalent of `composer audit`'s JSON.
**Every finding traces to a real route in that list and a cited `file:line`.** If you
cannot point to both, you do not report it.

You are an **authorization-review advisor**. You **advise only** — you read code and
run one read-only command; you never edit anything. See [Guardrails](#guardrails)
and [Anti-patterns](#anti-patterns).

---

## When to use

Activate when the user wants to review authorization / access control, hunt for IDOR
(broken object-level authorization), find unprotected endpoints, check policy/gate
coverage, or sanity-check a new endpoint before merge. Triggers: "review my
authorization", "find IDOR", "which routes have no auth", "is this endpoint scoped to
the owner", "do my controllers check policies", "audit access control before launch",
"review this PR's new routes".

---

## Method

Work the steps in order. **Step 1 is non-negotiable: every finding traces to the real
route inventory and a cited controller line, never to a guess about what the code
"probably" does.**

### 1. Ground truth from `route:list` (never from imagination)

1. **Confirm it's a Laravel project.** Look for `artisan` at the repo root and
   `app/`/`routes/`. If absent, stop and say this isn't a Laravel project.
2. **Get the route inventory — the anchor:**
   ```bash
   php artisan route:list --json
   ```
   This returns, per route: HTTP `method`, `uri`, `name`, the full `middleware` list,
   and `action` (`App\Http\Controllers\X@method` or a closure). This is the spine —
   the set of endpoints you reason about, and the source of truth for *what middleware
   actually applies* (route group middleware is already merged in).
   > **`--json` renders middleware as resolved class strings, not aliases.** `can:update,post`
   > appears as `Illuminate\Auth\Middleware\Authorize:update,post`; `auth:sanctum` as
   > `Illuminate\Auth\Middleware\Authenticate:sanctum`; a custom `role:admin` as its own
   > class (`App\Http\Middleware\EnsureRole:admin`). **Never decide "no authz" by grepping
   > for the literal `can:`** — match the `Authorize:`/class form too, and read any custom
   > middleware class to confirm what it enforces. See [`references/auth-patterns.md`](references/auth-patterns.md) §2a.
3. **Scope the review.** By default review **application routes** — skip framework/
   vendor/`telescope`/`horizon`/`_ignition` routes unless asked. (`route:list --json`
   already merges `GET|HEAD` into one entry — there are no separate `HEAD` rows to
   dedupe.) An **application closure** (an inline `GET /` or `GET /api/user`) counts as
   in-scope; framework-registered closures (`storage/*`, `up`, `sanctum/csrf-cookie`)
   do not. If the user is reviewing a **PR/diff**, intersect the route list with the
   changed files (run on the diff: only routes whose controller/route file changed).
4. **Fallback when the app won't boot:** parse `routes/*.php` statically + `Grep` for
   `Route::`/middleware, and **say so explicitly** — the inventory is now best-effort and
   findings drop one confidence level. Never silently substitute a guess for the real list.
   > Modern **Laravel 11/12 boots `route:list` on defaults even with no `.env`** — a
   > missing `.env` is *not* a reliable failure trigger. Real boot failures come from a
   > **missing PHP extension** (e.g. an image library throwing `extension … must be
   > installed` at boot), a service provider that **hits the DB/cache at boot** before
   > it's migrated, or a throwing provider. If it's a missing extension you can enable,
   > prefer fixing the boot (`php -d extension=gd artisan route:list --json`) over the
   > lossy static fallback. Also: route files aren't always `web.php`/`api.php` — glob
   > **all** of `routes/*.php` (apps split into `api.base.php`, `admin.php`, etc.).

> **Rule:** every reported route exists in `route:list` (or the parsed route files),
> and every finding cites the controller `file:line` and the relevant code. Can't
> show both? Don't report it — and never invent a route, policy, or method name.

### 2. The authorization chain — four layers per route

**On a large app, establish the convention first, then audit by exception.** Before
walking 100+ routes one by one, spend the first few minutes finding *how this app
authorizes*: does the base controller add `AuthorizesRequests`? Are policies
auto-discovered or registered? Where does object scoping live — inline, or in a
**repository / custom Eloquent Builder** (see auth-patterns §3)? Is auth applied per-route
or by a route-group `->middleware('auth')`? Once you know the house style, most routes are
a fast conformance check against it, and the findings are the **deviations** — the one
controller that takes a raw bound model when every sibling scopes through the repository.
This is the difference between auditing a toy and a real codebase; do it before the
per-route pass, not instead of it.

For each in-scope route, walk the chain and record, factually, which layers are
present. A route is exposed when a layer that *should* exist is missing.

| Layer | Present when (fact you can cite) | Where to look |
|-------|----------------------------------|---------------|
| **1 · Authentication** | `auth`, `auth:sanctum`, `auth:api`, or a custom guard in the route's `middleware` | `route:list` |
| **2 · Authorization** | `can:` middleware (renders in `--json` as `Illuminate\Auth\Middleware\Authorize:<ability>` — see note) **OR** `Gate::authorize`/`allows`/`denies` (framework-default form) / `$this->authorize()` (needs `AuthorizesRequests` trait — see note) / `$user->can()`/`cannot()` in the method **OR** `authorizeResource()` in the controller constructor **OR** a FormRequest whose `authorize()` returns a *real* check | `route:list` + controller/request files |
| **3 · Object scoping (IDOR core)** | the acted-on record is fetched **scoped to the actor** (`$request->user()->posts()->findOrFail(...)`, scoped route binding) **OR** authorized against the bound model (`Gate::authorize('update', $post)` / `authorize('update', $post)`) | controller method |
| **4 · Policy exists** | an `App\Policies\{Model}Policy` is auto-discovered, bound via the `#[UsePolicy]` attribute, or registered with `Gate::policy(...)` (in `AppServiceProvider::boot()` on Laravel 11+, or `AuthServiceProvider::$policies` on ≤10) | filesystem + provider |

> **Laravel-version note.** Since **Laravel 11** the base `Controller` is empty and does
> **not** include `AuthorizesRequests`, so `$this->authorize()` / `authorizeResource()`
> exist only when the controller adds `use AuthorizesRequests;`; the always-available
> form is `Gate::authorize(...)`. Likewise `AuthServiceProvider` is gone from the 11+
> skeleton — policies are registered in `AppServiceProvider::boot()` via `Gate::policy()`,
> via the `#[UsePolicy]` attribute, or auto-discovered. Recognize **all** of these as
> valid; see [`references/auth-patterns.md`](references/auth-patterns.md).

Read [`references/auth-patterns.md`](references/auth-patterns.md) for the full
catalogue of how each layer can legitimately be satisfied — so you recognize a real
check and don't flag a pattern you simply didn't know about.

### 3. The IDOR core — object-level authorization (the flagship)

This is where the value is. The trap people fall into:

- **Implicit route-model binding resolves, it does NOT authorize.** A method signed
  `public function update(Post $post)` will happily load *anyone's* post by ID.
  Without `authorize('update', $post)` or owner-scoping, that's IDOR. This is the
  single most common Laravel access-control bug — treat it as the headline check.
- **`Model::find($request->id)` / `findOrFail($id)`** then mutate/return, with no
  policy call and no `where('user_id', ...)`/relationship scoping → 🔴/🟡 depending on
  how clearly the model is user-owned (step 4).
- **Actor-selected target from request input (IDOR on write — the worst case).** When
  the record to *write* is chosen by an id taken from the **request body/query** rather
  than from the actor — `User::findOrFail($request->input('user_id'))->update(...)`,
  `Order::find($request->order_id)->delete()` — and nothing checks ownership, that's a
  **confirmed cross-account write: High confidence even without route-model binding**,
  because the attacker names the victim directly. This is strictly worse than a missing
  `authorize()` on a bound model — do **not** soften it to "verify". A FormRequest whose
  `authorize()` returns `true` in front of this is the headline exposure, not a footnote.
- **Scoped bindings matter:** `Route::scopeBindings()` or a nested resource that scopes
  the child to the parent (`users.posts` resolving `$post` through `$user->posts()`)
  *is* object scoping — credit it, don't flag it.
- **Mass-assignment adjacency:** `$model->update($request->all())` where `$fillable`
  includes an ownership/role key (`user_id`, `team_id`, `is_admin`) is privilege
  escalation even when the row was correctly scoped — note it as a related finding.

To judge "should this be scoped?", look for ownership signals on the model:
`user_id`/`team_id`/`tenant_id` columns, `belongsTo(User::class)`, a global scope, or
the user's relationship methods. **If you can't establish the model is owned, the
finding is Medium/Low and phrased as "verify", not asserted as a hole** (step 5).

> **Ownership can be conditional on edition / config / feature flag.** In some apps the
> *same* model is shared-by-design in one mode and user-owned in another (e.g. a policy
> that `return true`s for the community edition but row-scopes under a paid license, or
> a `single_user_mode`/multi-tenant toggle). When you see ownership gated on a runtime
> flag, **say which mode the finding applies to** ("cross-user read **only under the Plus
> license**; intended in community mode") rather than asserting or dismissing it flatly.
> Treat "exposed only under configuration X" as a valid, first-class finding qualifier.

### 4. API Resource output — the second leak surface

An endpoint can authorize the *action* correctly and still **leak data in the
response**. Distinguish two leak types and label which one you found — a route can have
one, both, or neither: **field-level** (the resource exposes sensitive fields) and
**row-level** (the endpoint returns rows not scoped to the actor). For routes that
return a `JsonResource`/`ResourceCollection` (or raw model/`->toArray()`):

- Flag resources that expose **ownership/internal fields** — `user_id`, `email`,
  `*_token`, `is_admin`, password hashes, other users' PII — without a per-field
  guard (`when()`, `$this->mergeWhen()`, conditional on `$request->user()`).
- Flag a **collection endpoint that returns records not scoped to the actor**
  (`PostResource::collection(Post::all())`) — that's IDOR at the list level.
- Note `Model::all()`/unscoped `get()` feeding a resource on an owned model.

Confidence is Medium here (field sensitivity is a judgment) — cite the resource
`file:line` and the exact field.

### 5. False-positive control (first-class, not a footnote)

A security tool dies from false positives. Before flagging "no auth"/"no authz",
apply these filters — and when in doubt, **lower the confidence, don't drop or
inflate the finding**:

- **Public-by-design routes** — login, register, `password/*`, email verification,
  the home/landing pages, public content listings, and signature-verified **webhooks**
  (Stripe/GitHub etc., often behind a `verifyWebhookSignature`-style middleware) are
  *meant* to be unauthenticated. Consult
  [`references/public-by-design.md`](references/public-by-design.md). Don't flag these
  for "missing auth"; if a route only *looks* public, mark it **"assumed public —
  confirm"**, don't assert a hole.
- **Authorization can live in layers you must actually check** — a missing
  `$this->authorize()` is *not* a finding if a `can:` middleware, `authorizeResource()`,
  or a FormRequest already covers it. Verify all four layers before concluding a gap.
- **`FormRequest::authorize()` returning `true`** is only a 🔴 finding when the route
  is otherwise unprotected *and* touches owned data. On a clearly public or
  separately-authorized route it's intentional — note as "verify intent", not a hole.
- **Admin/global contexts** — an admin panel route behind an `admin`/role middleware
  legitimately queries across all users. Unscoped queries there are expected; don't
  flag them as IDOR.

### 6. Classify each finding by confidence + evidence

One row per finding, most severe wins. **Confidence is mandatory** and travels with an
evidence chain the human can verify in seconds.

| Class | Signal | Confidence |
|-------|--------|------------|
| 🔴 **Exposed endpoint** | state-changing (POST/PUT/PATCH/DELETE) or owned-data route with **no auth AND no authz AND no scoping** | High — structural |
| 🔴 **Authorization disabled** | `FormRequest::authorize(){ return true; }` on an otherwise-unprotected owned-data route | High |
| 🔴 **Cross-account write (IDOR on write)** | the record to mutate is selected from request **input** (`User::find($request->input('user_id'))->update()`) with no ownership check, on an authenticated route | High |
| 🟡 **Likely IDOR** | route-model binding / `find($id)` on a likely-owned model with no `authorize()` and no owner scoping | Medium |
| 🟡 **Unscoped query / data leak** | `Model::all()`/unscoped query, or an API Resource exposing owned/internal fields | Medium / Low |
| 🔵 **Hardening** | no policy for a managed model, ad-hoc inline check instead of a policy, mass-assignment of ownership keys | Low — defense-in-depth |
| ✅ **Covered** | all applicable layers present | — |

> **Confidence rule.** *High* = you can point at the missing layer structurally
> (route + method, no check anywhere in the chain). *Medium* = the gap depends on
> whether the model is owned / the field is sensitive — state the assumption.
> *Low* = stylistic/defense-in-depth. Never present Medium/Low as a confirmed
> vulnerability.
>
> **Structurally broken but currently fails-closed.** Some bugs deny by default *today*
> yet hide a latent leak — e.g. `authorizeResource` mapping `index → viewAny` when the
> policy has no `viewAny` (Laravel denies, so the endpoint 403s) while the controller's
> query is `Model::all()` (unscoped). Lane the **active** defect by its structural severity
> (a mis-bound `authorizeResource` is a real 🔴/🟡 authz break), and **separately note** the
> latent unscoped query — "fails closed now, leaks the moment a permissive `viewAny`/`before`
> is added." Don't downgrade a structural break to ✅ just because the current symptom is a 403.

### 7. Output — coverage map + prioritized lanes

Produce the report using [`examples/report.md`](examples/report.md) as the template.

1. **Summary** — counts per class + the headline (the 1–2 things to fix today).
2. **Coverage map** — a table of **every in-scope route** → `auth ✓ · authz ✓ ·
   scoped ✓ · policy ✓` (✓ / ✗ / n/a). This is the analog of "list every dependency,
   even the healthy ones": it shows what was checked, surfaces both the ✅ and the
   holes, and makes the gaps scannable. Never silently omit a route you reviewed.
   > The `policy` column is **defense-in-depth, not a pass/fail gate.** When `authz` and
   > `scoped` are satisfied *inline* (an `abort_unless($x->user_id === auth()->id())` or a
   > `can:` check) the route is **covered** even with `policy ✗` — mark it ✗ and lane it 🔵
   > (extract a policy), not 🔴. A `✗` here means "no centralized policy", not "unprotected".
3. **Findings by lane**, each row carrying its evidence chain
   (`route → Controller@method:line → missing layer` + snippet) and confidence:
   - 🔴 **Verify & fix now** — high-confidence exposures.
   - 🟡 **Review (needs judgment)** — likely IDOR / leaks; state the assumption to check.
   - 🔵 **Hardening** — defense-in-depth.
   - ✅ **Covered** — summarized, cross-referenced from the map (don't repeat per route).
4. For each 🔴/🟡: a **fix sketch** in Laravel idiom (add `authorize()`, scope the
   query through the relationship, write the policy) — as *advice for the human to
   apply*, not an edit.

List each route **once**, in its most-severe lane; the coverage map carries the rest.

### 8. Saving the report (optional, on request only)

By default, **output to the conversation only** — write nothing to disk. After
presenting it, you **may offer** to save, proposing:

```
storage/logs/authorization-review-<YYYY-MM-DD>.md
```

Write the file **only if the user agrees**. `storage/logs/` is Laravel-native and
git-ignored by default. If it doesn't exist, **ask where to save** rather than
guessing. Date the filename. This report is the **only** file the skill may ever
write, and only when asked — it never touches routes, controllers, policies, or any
source code. See [Guardrails](#guardrails).

---

## Confidence & honesty

Separate the two in every report:

- **Hard facts** — the route inventory, its middleware, which files/methods/policies
  exist. These come from `route:list` and the filesystem. Cite them.
- **Judgment** — whether a model is "owned", whether a field is sensitive, whether a
  query "should" be scoped, the fix sketch. These are AI-derived. Mark them and state
  the assumption.

> **A clean report means "no broken authorization found in the layers I checked" —
> NOT "this app is secure."** This skill does not cover business-logic authorization,
> indirect references it can't see in code, or the surfaces listed in
> [Boundaries](#boundaries). Say this plainly; never imply a clean pass is a pentest.

---

## Robustness & fallbacks

State plainly what you could **not** check:

- **App won't boot / `route:list` fails** → fall back to static `routes/*.php` parsing
  + `Grep`, drop confidence one level, and say the inventory is incomplete (closures,
  dynamically-registered routes, and merged group middleware may be missed).
- **Closures instead of controllers** → you can still see route middleware; read the
  closure body inline for the authz/scoping check.
- **Route exists but its action method doesn't** → an `apiResource`/`resource` can
  register `show`/`update`/`destroy` for a controller that only implements `index`. That
  route 500s at runtime (`BadMethodCallException`) — it's a **robustness bug, not an authz
  exposure**. Note it as 🔵 if useful, but don't lane it 🔴/🟡 or count it as unprotected.
- **Custom auth/authz** (custom guards, gate definitions in a provider, package-based
  permissions like spatie/laravel-permission `role:`/`permission:` middleware) → treat
  these as valid authorization layers; check `references/auth-patterns.md` and don't
  flag a pattern you merely didn't recognize.
- **Multi-guard / multiple route files** (api.php, web.php, admin.php) → review each;
  note which guard applies where.
- **Fully covered app** → say so plainly: "every reviewed route satisfies its
  applicable layers" — don't manufacture findings. (Still not a guarantee of security.)

## Boundaries

This skill reviews **HTTP authorization**: routes, controllers, policies, gates,
FormRequests, Eloquent query scoping, and API Resource output. It does **NOT** cover,
and must say so when relevant:

- **Livewire / Filament / Nova / Inertia action authorization** — these are callable
  outside the `route:list` inventory and use their own authorization models; analyzing
  them without that anchor would produce false positives. Out of scope for this version.
- **Business-logic authorization** (multi-step workflows, approval states, "can a
  manager refund after 30 days") — needs domain knowledge; flag for human/pentest.
- **IDOR via indirect references not visible in code** (predictable IDs leaked
  elsewhere, signed-URL misuse) — out of reach of static reading.
- **AuthN strength** (password policy, MFA, token lifetime) and **infra/gateway authz**.

It is **not a pentest and not a security product** — it relays a structured reading of
your code, with confidence levels, for a human to verify and fix.

## Anti-patterns

Never do any of these:

- ❌ **Invent a route, policy, controller, or method** — every finding cites a real
  `route:list` entry and a real `file:line`. No anchor → no finding.
- ❌ **Treat a code comment as evidence.** A comment (`// safe`, `// TODO: add auth`,
  even `// IDOR here`) is not a fact — cite the structural reality (the missing call, the
  unscoped query, the route + line). Code comments can lie; `route:list` and the call
  graph do not. Derive every verdict from the code itself, never from what a comment claims.
- ❌ **Flag "missing authorize()" without checking all four layers** — a `can:`
  middleware, `authorizeResource()`, or FormRequest may already cover it.
- ❌ **Flag a public-by-design route** (login/register/webhook/landing) for missing
  auth — consult `references/public-by-design.md`; mark "assumed public — confirm" if unsure.
- ❌ **Assert IDOR on a model you haven't shown is owned** — if ownership is unproven,
  it's Medium/Low "verify", not a confirmed hole.
- ❌ **Flag unscoped queries inside an admin/role-gated context** as IDOR — that's expected.
- ❌ **Present Medium/Low confidence as a confirmed vulnerability** — confidence travels
  with every finding.
- ❌ **Claim the app is "secure" / call this a pentest** — a clean report means "nothing
  found in the layers checked".
- ❌ **Drop a reviewed route from the coverage map** — show every route you checked,
  including the ✅ ones.
- ❌ **Edit routes, controllers, policies, or any source**, or run state-changing
  commands. Output advice + fix sketches for a human to apply.

## Guardrails

- **Advise-only.** Reads files and runs one **read-only** command (`php artisan
  route:list --json`). Emits a report + fix sketches *for the human to apply and
  review*. The **sole** write it may perform is saving its own report file — only with
  the user's confirmation (step 8). It never edits routes, controllers, policies, or code.
- **Ground truth, not guesswork.** Findings trace to the route inventory and cited
  code. Couldn't boot the app? Fall back to static parsing and **say so** — never fill
  the gap with an assumed route.
- **Flag human judgment.** Where a call needs domain knowledge (is this resource
  meant to be public? is this field sensitive? is this break acceptable?), say so and
  hand the decision back.
- **Not a security product.** This is a structured code reading with confidence
  levels, not a CVE scanner or a pentest. A clean pass is not a guarantee.
