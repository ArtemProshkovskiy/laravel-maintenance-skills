# Using `laravel-authorization-review`

A plain, self-contained guide: what the skill does, how to install it in Claude Code,
and how to run it on your Laravel project.

---

## 1. What it does (in one minute)

Static scanners (Larastan, Psalm, Semgrep, taint analysis) trace where untrusted
**input** flows. They cannot tell you whether `Invoice::find($id)` *should* have been
scoped to the logged-in user — that's a question about **intent**, and it's exactly the
bug class behind **IDOR / broken object-level authorization (BOLA)**, #1 in the OWASP
API Security Top 10.

This skill is the **judgment layer** for that gap. For every HTTP route it walks the
authorization chain and reports where a layer that should exist is missing:

- **Authentication** — is the route behind `auth` / `auth:sanctum`?
- **Authorization** — does *something* check the action? (`can:` middleware,
  `$this->authorize()`, `authorizeResource()`, a real FormRequest `authorize()`, a gate,
  or an RBAC package's `role:`/`permission:` middleware)
- **Object scoping (the IDOR core)** — is the record fetched *through the actor*
  (`$user->invoices()->findOrFail($id)`) or authorized against the bound model — rather
  than resolved by raw id via route-model binding?
- **API Resource output** — does the response leak owned/internal fields (`user_id`,
  `email`, `is_admin`, tokens) or return records not scoped to the actor?

It then produces a **coverage map** (every route × every layer) and a prioritized list:
🔴 fix now · 🟡 review · 🔵 hardening · ✅ covered — each finding carrying its
**confidence** and an **evidence chain** (route → controller `file:line` → missing layer)
you can verify in seconds.

> **Advise-only.** The skill changes nothing: it does not edit routes, controllers,
> policies, or any code. It hands you findings and fix sketches; you apply them.

### Why it can be trusted (and not hallucinate)

It's anchored to **`php artisan route:list --json`** — the deterministic inventory of
every endpoint and its middleware, Laravel's equivalent of `composer audit`'s JSON.
Every finding traces to a real route in that list and a cited `file:line`. No anchor →
no finding.

---

## 2. Requirements

| Requirement | Why |
|---|---|
| A Laravel project (`artisan` present) | the entry point; without it the skill stops |
| The app can boot enough to run `php artisan route:list` | the ground-truth route inventory |
| Source for `app/`, `routes/`, `app/Policies/` readable | to walk the chain |

If the app can't boot (missing `.env`, DB required at boot), the skill **falls back** to
parsing `routes/*.php` statically and tells you the inventory is best-effort.

---

## 3. Installing in Claude Code

This skill ships in the `laravel-maintenance` plugin. Add the marketplace and install
the plugin once — after that the skill activates on its own based on what you ask:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```

---

## 4. Running it on your project

Open Claude Code **inside your Laravel project folder** and ask in plain language:

- "review my authorization"
- "find IDOR in this app"
- "which routes have no auth?"
- "is this endpoint scoped to the owner?"
- "do my controllers check policies?"
- "audit access control before launch"

**Reviewing a PR / a single feature?** Say so — "review authorization on the routes in
this PR" — and the skill intersects the route inventory with your changed files instead
of scanning everything. This is the cheap, recurring use: catch a new endpoint that
shipped without a policy.

---

## 5. What you get — the report

A clean Markdown report, straight into the chat. See the full reference in
[`examples/report.md`](examples/report.md). Shape:

```markdown
## Summary
Reviewed 18 routes — 🔴 3 exposed, 🟡 5 review, 🔵 2 hardening, ✅ 8 covered.

## Coverage map
| Route                          | auth | authz | scoped | policy | Verdict        |
|--------------------------------|:----:|:-----:|:------:|:------:|----------------|
| DELETE /api/invoices/{invoice} |  ✓   |   ✗   |   ✗    |   ✓    | 🔴 exposed     |
| GET /api/orders/{order}        |  ✓   |   ✓   |   ✓    |   ✓    | ✅ covered     |

## 🔴 Verify & fix now
DELETE /api/invoices/{invoice} → InvoiceController@destroy:88
  → implicit binding resolves ANY invoice; no authorize(), no owner scope → IDOR
  → fix: $this->authorize('delete', $invoice)
```

### How to read the lanes

| Lane | What it is | What to do |
|---|---|---|
| 🔴 **Verify & fix now** | high-confidence structural holes (no auth/authz/scope on owned data) | confirm with the cited line, add the check, add an authorization test |
| 🟡 **Review** | likely IDOR / data leak that depends on whether a model is owned or a field is sensitive | decide the intent (owned vs. team-shared vs. public), then scope/authorize |
| 🔵 **Hardening** | defense-in-depth (missing policy, mass-assignment of ownership keys) | tidy when convenient |
| ✅ **Covered** | all applicable layers present | nothing — shown so you can see what was checked |

---

## 6. How it keeps you from chasing ghosts

False positives are what kill a security tool's credibility, so the skill is built to
suppress them:

- it recognizes **all the legitimate ways** to authorize in Laravel (`can:`,
  `authorizeResource`, FormRequests, gates, spatie/laravel-permission) before claiming a
  layer is missing — see [`references/auth-patterns.md`](references/auth-patterns.md);
- it knows **public-by-design** routes (login, register, password reset, signature-
  verified webhooks, public read-only pages) and won't flag them for missing auth — see
  [`references/public-by-design.md`](references/public-by-design.md);
- when it can't prove a model is user-owned, the finding is **Medium/Low "verify"**, not
  a confirmed hole — and it states the assumption to check;
- every finding carries an **evidence chain** so you can confirm or dismiss it in seconds.

---

## 7. Saving the report (optional)

By default the report stays in the chat — nothing is written to disk. After printing it,
the skill **may offer** to save to
`storage/logs/authorization-review-<YYYY-MM-DD>.md` (git-ignored in Laravel). You can
give it a different path. Without your confirmation, no file is created.

---

## 8. Boundaries — what it does NOT do

- ❌ edit routes, controllers, policies, or any code — you apply every fix;
- ❌ invent a route, policy, or method — only what `route:list` and your files contain;
- ❌ cover **Livewire / Filament / Nova** action authorization — those live outside the
  `route:list` inventory and use their own models (a future version may add them);
- ❌ judge **business-logic** authorization (approval states, time-based rules) — flagged
  for a human;
- ❌ catch IDOR via **indirect references** not visible in code, or infra/gateway authz;
- ❌ replace a pentest. A finding-free report means "nothing broken in the layers
  checked", **not** "secure".

---

## 9. Common situations

| Situation | What the skill does |
|---|---|
| Not a Laravel project | says so and stops |
| App won't boot (`route:list` fails) | falls back to parsing `routes/*.php`, says the inventory is best-effort, drops confidence |
| Custom guard / RBAC package | treats `role:`/`permission:`/custom-guard middleware as valid authorization |
| Public route (login, webhook) | recognized as public-by-design; not flagged for missing auth |
| Admin panel querying all users | unscoped queries there are expected, not flagged as IDOR |
| Everything covered | says so plainly — "every reviewed route satisfies its layers" (still not a security guarantee) |

---

**In short:** install the plugin → open Claude Code in your Laravel project → ask
"review my authorization" → get a coverage map + 🔴/🟡/🔵 findings with evidence, and
fix them yourself.
