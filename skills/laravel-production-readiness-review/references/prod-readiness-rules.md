# Production-Readiness Rules — the check catalogue

The source of truth for every finding. Each rule is **IF `<fact>` → ON PROD `<consequence>`
→ severity + fix**. Every finding the skill reports must trace to one of these and to a
real `file:line` / `.env` key / `route:list` entry. **No anchor → no finding.**

Three cross-cutting rules apply to all checks:

- **Env gating.** Read `APP_ENV` from `.env`. *Env-dependent* checks only matter on
  production — if `APP_ENV` is `local`/`dev`/`testing`, report them 🟡 *"if these are
  production values"*, never 🔴. *Env-independent* checks fire in any environment.
- **Grep is a candidate finder, not a verdict.** Match with word boundaries
  (`\benv\(`, `\bdd\(`, `\bdump\(`, `\bray\(`, `\bvar_dump\(`) then **re-read each hit in
  context** to drop comments, strings, method/static calls (`->env(`, `::env(`), and
  unrelated tokens (`getenv()`, `var_dump` ≠ `dump`). A raw grep count is never reported.
- **Normalize `.env` lines before reading a value.** A real `.env` is not strict
  `KEY=value`: handle an optional `export ` prefix, **whitespace around `=`**
  (`APP_DEBUG = true`), surrounding single/double quotes (`APP_URL='http://localhost'`),
  and trailing `# inline comments`. Skip blank lines and full-line `#` comments. So
  `export QUEUE_CONNECTION=sync` → key `QUEUE_CONNECTION`, value `sync`. A literal
  `^KEY=value` match misses these and is wrong.

---

## A. `env()` outside `config/` — the flagship  *(env-independent)*

**Why it breaks.** `php artisan config:cache` compiles all config into one cached file and
**stops loading `.env` at runtime**. After that, any `env('FOO')` call **outside** the
`config/` directory returns its default — usually `null`. Inside `config/` it's fine,
because those values were already baked in at cache time. So the bug is invisible until
the deploy runs `config:cache`, then a feature silently misbehaves with no error.

| | |
|---|---|
| **Detect** | `\benv\(` in `app/`, `routes/`, `resources/views/`, `bootstrap/`, `database/` — **not** `config/`; confirm each hit in context |
| **Severity** | 🔴 High — deterministic **and silent** |
| **Fix** | move the value into a `config/x.php` key, read it via `config('x.key')`; the `env()` call now lives only in `config/` |
| **Deploy signal** | if a deploy script exists, note whether it runs `config:cache`. Default stays 🔴; downgrade to 🟡 + *"latent until config:cache"* **only** if you confirm the deploy never caches config |
| **Exclude** | only the bare `env()` **helper** counts. NOT a finding: `env()` inside `config/`; `env(` in a comment or string literal; a **method/static call** `$x->env(` / `Foo::env(` (a method *named* `env`, not the helper — `\benv\(` matches it because `>`/`:` is a word boundary, so you MUST drop it by reading the char before `env`); a method definition `function env(`. The boundary already excludes `getenv(` (no boundary inside `getenv`). |

---

## B. Closure routes vs `route:cache`  *(env-independent)*

**Why it breaks.** `php artisan route:cache` serializes the route table. A closure (inline
`fn`/`function`) **cannot be serialized** — the command aborts with
`LogicException: Unable to prepare route [...] for serialization. Uses Closure.` The whole
deploy step fails. Unlike A this is **loud** — you find out immediately.

| | |
|---|---|
| **Detect (primary)** | `route:list --json` entries whose `action` is the **literal string `"Closure"`** (verified on Laravel 11/12). A controller renders as `"App\…\Controller@method"`; `Route::view` as `"Illuminate\Routing\ViewController"`; `Route::redirect` as `"…\RedirectController"` — **none of those are closures**, never flag them |
| **Exclude (framework closures)** | a stock app registers closures you must **NOT** flag — `storage/{path}` (GET+PUT), `up` (Laravel 11+ health check), `sanctum/csrf-cookie`, `_ignition/*`, `livewire/*`. Only **application** closures (defined in your own `routes/*.php`) are in scope |
| **Detect (static fallback)** | when `route:list` can't run: in `routes/*.php`, a `Route::<verb>(...)` whose handler (2nd arg) is an inline `fn (`/`fn(` or `function (`/`function(` — not an array `[Controller::class, 'method']`, a `'Controller@method'` string, `Route::view(...)`, or `Route::redirect(...)`. Confirm by reading the route line |
| **Severity** | 🟡 default → 🔴 if a deploy script runs `route:cache` |
| **Fix** | move the closure into a controller method and reference `[Controller::class, 'method']` |
| **Note** | the A/B severity asymmetry is intentional — A is a silent data bug (worse), B is a build failure caught at deploy time |

---

## C. Production `.env` values  *(env-dependent — gated)*

Each is wrong **on production**; on `local`/`dev` they are normal → report 🟡 "if prod".

| Key / value | On prod | Severity |
|---|---|---|
| `APP_DEBUG=true` | Ignition error page leaks stack traces, env vars, and source to anyone who triggers an error | 🔴 |
| `APP_URL=localhost` / `127.0.0.1` / `http://…:8000` | breaks absolute links in mail, password-reset, and signed URLs | 🟡 |
| `APP_ENV` not `production` | framework/packages relax prod-only behaviour | 🟡 |
| `QUEUE_CONNECTION=sync` | "queued" jobs run inline on the request — no async, request blocks | 🟡 |
| `CACHE_STORE`/`SESSION_DRIVER` = `array` | data is lost between requests | 🟡 |
| `CACHE_STORE`/`SESSION_DRIVER` = `file` | works on one box, breaks across multi-server / load balancer | 🟡 |

> **Version note:** the cache key was renamed in Laravel 11 — it's **`CACHE_STORE`** on
> Laravel 11+ and **`CACHE_DRIVER`** on Laravel ≤10. Match **either** name (use the version
> from `composer.json` to know which is canonical, but accept both). `SESSION_DRIVER` is
> unchanged. `cookie` is a valid session driver (not flagged); only `array`/`file` are.
| `MAIL_MAILER=log` | emails go to the log file — users receive nothing | 🟡 |
| `LOG_LEVEL=debug` | verbose logs, may persist sensitive payloads | 🔵 |

**Fix:** set the prod-appropriate value in the server's real `.env` (you do this; the skill
never edits `.env`).

---

## D. Always-on checks  *(env-independent — bite in any environment)*

| Fact | Consequence | Severity |
|---|---|---|
| `APP_KEY` empty / missing | encryption, encrypted cookies, and sessions throw `MissingAppKeyException` everywhere | 🔴 |
| `.env` **not** matched by `.gitignore` (`git check-ignore .env` fails) **or** tracked by git (`git ls-files --error-unmatch .env` succeeds) | real secrets committed to history | 🔴 |
| `.env.example` contains **real** (non-placeholder) secret values | secrets leak via the public template | 🔴 |

**Secret-key scope first, then placeholder heuristic.** Only consider keys whose **name**
implies a secret — `*_KEY`, `*_SECRET`, `*_PASSWORD`, `*_PASS`, `*_TOKEN`, `*_DSN`,
`*_CLIENT_SECRET`, `*_API_KEY`, plus `APP_KEY`/`DB_PASSWORD`. A non-secret key like
`APP_NAME=Acme`, `APP_ENV=local`, or `APP_URL=…` is **never** a secret finding, whatever its
value. For an in-scope secret key, the value is a **placeholder** (not flagged) if empty or
matching `your-…`, `xxx`, `changeme`, `null`, `example`, `base64:` with no payload, obvious
dummies; if it looks like a **real** credential (e.g. `sk_live_…`, a long random string, a
real DSN) → flag, marking **verify** (Medium) since the heuristic can be wrong.

**Fix:** `.gitignore` `.env`; if already tracked, `git rm --cached .env` **and rotate the
leaked secrets**; replace real values in `.env.example` with placeholders.

---

## E. `.env` ↔ `.env.example` drift  *(env-independent)*

| Fact | Consequence | Severity |
|---|---|---|
| key present in `.env` but **missing** from `.env.example` | the next developer / a fresh deploy is missing config and the app misbehaves on setup | 🔵 |

**Fix:** add the missing keys (with placeholder values) to `.env.example`. Do **not** copy
the real values across (that would re-create check D).

---

## F. Debug leftovers  *(env-independent)*

| Fact | Consequence | Severity |
|---|---|---|
| `\bdd\(` / `\bdump\(` / `\bray\(` / `\bvar_dump\(` in `app/`, `routes/`, `resources/views/` | `dd()` halts the request mid-flight; `dump()`/`var_dump()` injects output / leaks data | 🔵 (mark **verify** if on a clearly live code path — a stray `dd()` in a controller action is closer to 🟡) |

**Fix:** remove the call. Confirm each hit in context — `dump` inside a word (`var_dump`),
or inside a comment/string, is not a finding.

---

## G. Bonus — dev tooling shipped to production  *(optional)*

| Fact | Consequence | Severity |
|---|---|---|
| a debug/dev package in `composer.json` `require` instead of `require-dev` — `barryvdh/laravel-debugbar`, `laravel/telescope`, `laravel/pail`, `fakerphp/faker`, `nunomaduro/collision` | ships dev tooling to prod; **Telescope/Debugbar can expose request data, queries, and env** at their routes | 🟡 |

**Framed as prod-exposure, not dependency hygiene** — this does not overlap
`composer-dependency-triage`, which judges versions/abandonment, not `require` placement.

**Fix:** move the package to `require-dev` (`composer remove X && composer require --dev X`).

---

## Severity legend

- 🔴 **Fix now** — breaks or leaks on production right now.
- 🟡 **Review** — value/code wrong for prod, or depends on an assumption (env, code path).
- 🔵 **Hardening** — hygiene; drift, log level, stray debug calls.
- ✅ **Covered** — the check passed; shown so the user sees what was verified.
