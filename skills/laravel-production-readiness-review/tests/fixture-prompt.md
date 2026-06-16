# Acceptance test — `laravel-production-readiness-review`

A reproducible acceptance test for the skill. Two parts:

1. **Fixture prompt** — feed it to Claude Code inside a *separate* throwaway Laravel
   sandbox. It builds `.env` / config / routes / code that deliberately exercise every
   check (A–G) **and** several false-positive traps the skill must NOT flag.
2. **Oracle** — run the skill against the sandbox and grade its report row by row.

> The skill reads the **real** `.env`, `config/`, `php artisan route:list --json`, and git
> state, and cites real anchors — it never invents them. So the fixture must be a
> **bootable** Laravel app. Requires a Laravel install
> (`composer create-project laravel/laravel sandbox`), PHP on PATH, and `git init`.

---

## 1. Fixture prompt (paste into the sandbox project)

```text
You are inside an empty throwaway Laravel project (sandbox). Your job is NOT to run the
production-readiness review skill — it is to BUILD A FIXTURE: .env / config / routes / code
that cover every check of laravel-production-readiness-review, including false-positive
traps. The app must BOOT and `php artisan route:list --json` must really return the routes.

Set APP_ENV=production in .env (so env-dependent checks are live, not gated).

1. Check A — env() outside config/ (must be flagged 🔴):
   - app/Services/PaymentGateway.php: a method that reads `env('STRIPE_SECRET')`.
   - app/Http/Middleware/FeatureFlag.php: `if (env('BETA', false)) { ... }`.
   TRAP (must NOT be flagged): in config/services.php add
   'demo' => ['key' => env('DEMO_KEY')] — env() inside config/ is legitimate.

2. Check B — closure route (must be flagged; 🔴 if a deploy script runs route:cache):
   - routes/web.php: Route::get('/status', fn () => response()->json(['ok' => true]));
   Also add a normal controller route as a ✅ (Route::get('/home', [HomeController::class,'index'])).

3. Check C — .env production values:
   - APP_DEBUG=true                 (🔴)
   - APP_URL=http://localhost       (🟡)
   - QUEUE_CONNECTION=sync          (🟡)
   - MAIL_MAILER=log                (🟡)
   TRAP ✅ (do NOT flag): CACHE_STORE=redis, SESSION_DRIVER=redis.

4. Check D — always-on:
   - leave APP_KEY EMPTY (APP_KEY=)                                  (🔴)
   - DO NOT add .env to .gitignore, then `git add .env && git commit` (🔴 tracked)
   - .env.example: put a REAL-looking value STRIPE_SECRET=sk_live_51AbCdEf...  (🔴)
   TRAP ✅: other .env.example keys are placeholders (APP_KEY=, DB_PASSWORD=) — not flagged.

5. Check E — drift:
   - add PULSE_DB_CONNECTION=pgsql to .env but NOT to .env.example       (🔵)

6. Check F — debug leftovers:
   - app/Http/Controllers/ReportController.php: a `dump($data)` on an action  (🔵/verify)
   TRAP (must NOT be flagged): a line containing `var_dump` only inside a comment,
   and a string literal "env(" inside a comment — neither is a finding.

7. Check G — dev tooling in require (optional, 🟡):
   - in composer.json, put "laravel/telescope" under "require" (not "require-dev").

8. Env-gating trap (separate quick check, optional):
   Note that if APP_ENV were `local`, checks in (3) must downgrade to 🟡 "if prod" — you
   can verify this by flipping APP_ENV=local and re-running later.

Do NOT leave comments in the generated code that reveal the expected verdict (no
`// 🔴`, `// TRAP`, `// flag this`). The skill must derive findings structurally from the
code itself; answer-key comments make the run dishonest. Code should look like a normal app.

At the end:
- run `php artisan route:list --json` and confirm /status and /home are listed;
- print a checklist "anchor → expected class" as YOUR notes (SEPARATE from the project
  files, not inside them).
Do NOT build the review report — the skill does that on the next step.
```

---

## 2. Oracle — grade the skill's report against this

Run the skill in the sandbox ("is this safe to deploy?") and check each row.

| Expect in the report | Verifies | Step |
|---|---|---|
| **`env('STRIPE_SECRET')`** (PaymentGateway) and **`env('BETA')`** (FeatureFlag) → 🔴, cited `file:line`, fix = move to config + `config(...)` | flagship A; silent-null logic | 2 |
| `env('DEMO_KEY')` **inside config/services.php** → NOT flagged | env()-in-config carve-out | 2 / A |
| **closure route `/status`** → flagged (🔴 if deploy runs route:cache, else 🟡), fix = controller | B; closure serialization | 3 |
| `/home` controller route → NOT flagged | no false positive on a normal route | 3 |
| **`APP_DEBUG=true`** → 🔴 (leaks stack traces) | C; env-dependent on prod | 4 |
| **`APP_URL=http://localhost`**, **`QUEUE=sync`**, **`MAIL_MAILER=log`** → 🟡 each | C value checks | 4 |
| `CACHE_STORE=redis` / `SESSION_DRIVER=redis` → NOT flagged | no false positive on safe values | 4 |
| **empty `APP_KEY`** → 🔴 | D; env-independent | 5 |
| **`.env` tracked in git** → 🔴 (via `git ls-files` / `git check-ignore`); fix names `git rm --cached` + **rotate secrets** | D; secret exposure | 5 |
| **real value in `.env.example`** (`sk_live_…`) → 🔴 (or 🟡 "verify"); placeholder keys NOT flagged | D; placeholder heuristic | 5 |
| **`PULSE_DB_CONNECTION` drift** → 🔵 | E | 6 |
| **`dump($data)`** in ReportController → 🔵 (verify on live path) | F | 6 |
| `var_dump`/`env(` appearing **only in comments** → NOT flagged | grep-discipline / context re-read | 6 |
| **`laravel/telescope` in `require`** → 🟡 prod-exposure (framed as exposure, not dep hygiene) | G | 7 |
| Report shows a 🔴/🟡/🔵/✅ layout, every finding with an anchor + exact fix | output discipline | 8 |
| Report states clean ≠ "safe to deploy"; names Boundaries (server env, web-server config out of scope) | honesty / not-a-gate | confidence |
| Nothing written to disk without asking | guardrails | 9 |

A **pass** = both flagship `env()` calls and the empty `APP_KEY` / tracked `.env` caught
with anchors; every trap left alone (env() in config, redis values, `/home`, comment-only
`var_dump`/`env(`); env-dependent values phrased correctly; and an honest "not safe-to-deploy"
caveat. **A single false positive on a trap is a fail** — crying wolf is the worst outcome.

---

## 3. Env-gating check (optional)

Flip `APP_ENV=local` in the fixture `.env` and re-run. Expect: the env-dependent values
(`APP_DEBUG=true`, `QUEUE=sync`, `MAIL=log`, `APP_URL=localhost`) are now reported **🟡 "if
these are production values"**, while the env-independent ones (`env()` outside config, empty
`APP_KEY`, tracked `.env`, drift, debug leftovers) **still fire** — confirming the gate.

---

## 4. Fallback-path check (optional)

Temporarily break the boot (rename `.env` so `route:list` fails) and re-run. Expect: the
skill reports that `php artisan route:list` failed, **falls back to parsing `routes/*.php`**,
says the inventory is best-effort, drops confidence one level, and skips the `.env`-value
checks (no `.env`) — rather than inventing routes or claiming a clean pass.
