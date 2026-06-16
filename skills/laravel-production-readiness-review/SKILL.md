---
name: laravel-production-readiness-review
description: >-
  Laravel production-readiness reviewer. Audits the "works locally, breaks/leaks in
  production" gap before deploy: env() called outside config/ (returns null after
  config:cache), closure routes that make route:cache fail, unsafe .env values
  (APP_DEBUG=true, empty APP_KEY, sync queue, APP_URL=localhost), secret exposure
  (.env tracked in git, real values in .env.example), .env drift, and leftover debug
  calls. Anchored to real .env / config / route:list output, advise-only. Use when the
  user wants a pre-deploy check or hits a prod-only bug — e.g. "is this safe to deploy",
  "production readiness check", "env is null after deploy", "config:cache broke my app",
  "closure route route:cache fails", "why does it work locally but not in production".
  Advise-only: never edits files.
license: MIT
compatible_agents:
  - Claude Code
  - Cursor
  - Codex
tags:
  - laravel
  - php
  - deployment
  - production
  - config
  - env
  - security
compatibility: >-
  Requires a Laravel project (artisan on PATH). Uses one read-only command —
  `php artisan route:list --json` — plus read-only git (`git ls-files`,
  `git check-ignore`) and static reading of `.env`, `.env.example`, `config/`, and code.
  Falls back to static `routes/*.php` parsing when the app can't boot. Never edits files.
metadata:
  author: "Artem Proshkovskyi"
  version: "0.1.0"
  category: "deployment / advisor"
allowed-tools: Bash(php artisan route:list:*) Bash(git ls-files:*) Bash(git check-ignore:*) Read Glob Grep
---

# Laravel Production Readiness Review

> **It works on your machine. This finds why it will break — or leak — in production.**

## Context

This skill audits the one bug class that local development structurally hides: the gap
between **dev** (no caches, `APP_DEBUG=true`, single box) and **prod** (config/route
caches compiled, debug off, multi-server). Code that runs perfectly locally can break
**silently** the moment a deploy runs `php artisan config:cache` / `route:cache`, or leak
data the moment it runs with `APP_DEBUG=true`. None of it shows up in local testing.

The flagship case is **`env()` called outside `config/`**: after `config:cache`, the
framework stops reading `.env`, so that call returns `null` — no error, just wrong
behaviour. **No existing tool audits for this** (Telescope, Larastan, Enlightn look
elsewhere). That is the gap this skill owns.

The reason it's trustworthy and not a guesser is its **ground-truth anchor**: every finding
traces to a real fact — a cited `file:line`, a real `.env` key, a real `route:list` entry,
a real git state. **No anchor → no finding.**

You are a **production-readiness advisor**. You **advise only** — you read files and run
read-only commands; you never edit anything. See [Guardrails](#guardrails) and
[Anti-patterns](#anti-patterns).

**Scope & tools.** Requires a Laravel project (`artisan` on PATH). Uses one read-only
command — `php artisan route:list --json` — plus read-only git and static reading of
`.env`, `.env.example`, `config/`, and code. Never modifies `.env`, config, code, or
`.gitignore`. The full rule catalogue lives in
[`references/prod-readiness-rules.md`](references/prod-readiness-rules.md).

---

## Rules

- **Anchor every finding to a real fact** — a cited `file:line`, a `.env` key, a
  `route:list` entry, or a git state. No anchor → no finding.
- **Gate env-dependent checks on `APP_ENV`.** If the file is `local`/`dev`, report
  value findings as 🟡 *"if these are production values"*, never 🔴. Env-independent
  checks (`env()` outside config, empty `APP_KEY`, secret exposure, drift, debug
  leftovers) fire in any environment.
- **Grep finds candidates; reading confirms them.** Match with word boundaries, then
  re-read each hit in context to drop comments, strings, and — critically — **method/static
  calls** (`$x->env(`, `Foo::env(`): only the bare `env()` helper is a finding, a method
  *named* `env` is not. Never report a raw grep count.
- **`env()` is only legitimate inside `config/`.** A call anywhere else is the flagship
  finding; a call inside `config/` is never flagged.
- **Classify by consequence, not by check type** — what *happens on prod* sets the lane.
- **Advise-only.** Emit the finding + the exact fix for the human to apply; never edit
  `.env`, config, code, or `.gitignore`.

---

## When to use

Activate when the user wants a pre-deploy / production-readiness check, or is debugging a
problem that only appears in production. Triggers: "is this safe to deploy", "production
readiness check", "env is null after deploy", "config:cache broke my app", "closure route
route:cache fails", "why does it work locally but not in production", "audit my .env before
launch", "did I leak any secrets".

---

## Method

Work the steps in order. **Step 1 is non-negotiable: every finding traces to a real fact —
a cited line, a `.env` key, a `route:list` entry — never to a guess.**

### 1. Confirm the project & set env gating

1. **Confirm it's a Laravel project.** Look for `artisan` + `app/`/`config/`. If absent,
   stop and say so.
2. **Read the Laravel version** from `composer.json` (`laravel/framework`) — `route:list
   --json` parsing and a few defaults vary by version.
3. **Read `APP_ENV` from `.env`.** This sets the gate: env-dependent checks (step 4) are
   🟡 "if prod" on `local`/`dev`; env-independent checks (steps 2, 5, 6) always fire.
   If there's no `.env`, say so — skip the `.env`-value checks, still run the code checks.

### 2. `config:cache` safety — `env()` outside `config/` (the flagship)

`Grep` `\benv\(` across `app/`, `routes/`, `resources/views/`, `bootstrap/`, `database/` —
**excluding `config/`** — and **re-read each hit in context**: drop comments, string
literals, and **method/static calls** (`->env(`, `::env(`) — only the bare `env()` helper
counts. Each confirmed call returns `null` after `php artisan config:cache`. Report 🔴 with the
cited `file:line` and the fix: move the value into a `config/*.php` key and read it via
`config('…')`. If a deploy script is present, check whether it runs `config:cache` and add
the note from [the catalogue](references/prod-readiness-rules.md).

### 3. `route:cache` safety — closure routes

From `php artisan route:list --json`, find entries whose `action` is the **literal string
`"Closure"`** (verified on Laravel 11/12). Each makes `php artisan route:cache` fail. Report
🟡 (→ 🔴 if a deploy script runs `route:cache`) with the route and the fix: move the closure
into a controller. **Exclude framework-registered closures** (`storage/{path}`, `up`,
`sanctum/csrf-cookie`, `_ignition/*`, `livewire/*`) and non-closures that look adjacent
(`Route::view` → `ViewController`, `Route::redirect` → `RedirectController`) — only
**application** closures in your own `routes/*.php` are in scope. If `route:list` can't run,
fall back to the static pattern in the catalogue (an inline `fn (`/`function (` handler) and
drop confidence one level.

### 4. `.env` production values (env-dependent — gated by step 1)

Parse `.env` (normalize first — strip `export `, whitespace around `=`, surrounding quotes,
and `# inline comments`; see catalogue) and apply [catalogue §C](references/prod-readiness-rules.md):
`APP_DEBUG=true` (🔴), `APP_URL=localhost` (🟡), `APP_ENV` not `production` (🟡),
`QUEUE_CONNECTION=sync` (🟡), `CACHE_STORE`/`CACHE_DRIVER`/`SESSION_DRIVER` = `file|array`
(🟡 — `CACHE_STORE` on Laravel 11+, `CACHE_DRIVER` on ≤10), `MAIL_MAILER=log` (🟡),
`LOG_LEVEL=debug` (🔵). On `local`/`dev` these are 🟡 "if these are production values".

### 5. Always-on checks (env-independent)

Apply [catalogue §D](references/prod-readiness-rules.md):

- **`APP_KEY` empty/missing** → 🔴.
- **`.env` exposed to git** → run `git check-ignore .env` (no match = not ignored) and
  `git ls-files --error-unmatch .env` (success = tracked). Either bad case → 🔴. If not a
  git repo, skip this and say so.
- **Real secrets in `.env.example`** → apply the placeholder heuristic; a real-looking
  value → 🔴 (mark **verify** if the heuristic is uncertain).

### 6. Drift + debug leftovers (env-independent)

- **Drift** ([§E](references/prod-readiness-rules.md)):
  keys in `.env` missing from `.env.example` → 🔵.
- **Debug leftovers** ([§F](references/prod-readiness-rules.md)):
  `\bdd\(`/`\bdump\(`/`\bray\(`/`\bvar_dump\(` in `app/`, `routes/`, `resources/views/` →
  🔵, cited; confirm in context.
- **Bonus** ([§G](references/prod-readiness-rules.md)):
  debug packages in `require` instead of `require-dev` → 🟡 prod-exposure.

### 7. Classify each finding by confidence

- **High** → deterministic fact (`env()` present/absent, `.env` tracked yes/no, empty
  `APP_KEY`) → may be 🔴.
- **Medium** → depends on an assumption (is this the prod `.env`? is this a real secret? is
  this `dd()` on a live path?) → phrase as "verify", state the assumption.
- **Low** → hygiene → 🔵.
- If you can't prove the environment is production, downgrade env-dependent findings to 🟡.

### 8. Output — verdict first, then the detail (optimize for fast reading)

The reader asked one question ("is this safe to deploy?"); answer it on line 1, then let
them drill down. Structure, in this order:

1. **Verdict line** — the headline answer, one line:
   `🔴 Not safe to deploy — N blocker(s)` · `🟡 Deploy with caution — N to review` ·
   `✅ No blockers found in the checks I ran` (never "safe to deploy" — see honesty).
2. **Summary table** — count per lane (🔴/🟡/🔵/✅) so the size is clear at a glance.
3. **The lanes**, each finding on a **consistent, scannable line**:
   `<anchor> — <what breaks on prod> → <exact fix>`. Put code blocks / detail only on 🔴
   items; keep 🟡 and 🔵 to one line each. Number the 🔴 findings.
4. **Fix in this order** — a short numbered list of the 🔴 items, so the reader knows what
   to do first without re-reading.
5. **✅ Covered** — a compact summary (one line per passed check), so they see what was
   verified, not just what failed.
6. **One honesty caveat** — clean ≠ "safe to deploy".

Keep it tight: a finding the reader can't act on in one read is too verbose. See
[`examples/report.md`](examples/report.md).

### 9. Saving the report (optional, on request only)

By default the report stays in the chat. You **may offer** to save it to
`storage/logs/production-readiness-<YYYY-MM-DD>.md` (git-ignored in Laravel). Only write
the file with the user's explicit confirmation; otherwise nothing is written to disk.

---

## Confidence & honesty

Separate the two in every report:

- **Hard facts** — `env()` locations, the route inventory, `.env` keys/values, git state.
  These come from the tools and the filesystem. Cite them.
- **Judgment** — whether a `.env` file is the production one, whether a value is a real
  secret, whether a `dd()` sits on a live path. These are AI-derived. Mark them and state
  the assumption.

> **A clean report means "no production-readiness problem found in the checks I ran" —
> NOT "safe to deploy."** It can't see runtime env vars set outside `.env`, server config,
> or anything in [Boundaries](#boundaries). Say this plainly.

---

## Robustness & fallbacks

State plainly what you could **not** check:

- **App won't boot / `route:list` fails** → fall back to static `routes/*.php` parsing +
  `Grep` for `Route::…`, drop confidence one level, and say the route inventory is
  best-effort (closures registered dynamically may be missed).
- **No `.env`** → skip the `.env`-value, secret-exposure, and drift checks; say why; still
  run the code checks (`env()` outside config, closures, debug leftovers).
- **Not a git repository** → skip the "tracked by git" half of the secret check; keep the
  `.gitignore` / `.env.example` halves.
- **Multiple env files** (`.env.production`, `.env.staging`) → review each, say which is
  which; don't assume a single `.env` is production.
- **Fully clean project** → say so plainly: "no production-readiness problems found in the
  checks I ran" — don't manufacture findings. (Still not a guarantee it's safe to deploy.)

---

## Boundaries

This skill audits the **dev↔prod gap** from the repository: `env()` usage, `config`/`route`
cache safety, `.env`/`.env.example`, git exposure, and debug leftovers. It does **NOT**
cover, and must say so when relevant:

- **Runtime/server environment** — env vars injected by the host, CI, or a secrets manager
  (not in `.env`), web-server config, file permissions, TLS. It reads the repo, not the box.
- **Whether your deploy actually runs the cache commands** — it infers from a deploy script
  if present, otherwise it advises conditionally.
- **Performance, dependency vulnerabilities, or authorization** — those are the other
  skills in this library.
- **Functional correctness** — it checks production *posture*, not whether your features work.

It is **not a deploy tool and not a security product** — it relays a structured reading of
your repository, with confidence levels, for a human to verify and apply.

---

## Examples

**The flagship — `env()` outside `config/`.** Works locally; returns `null` after the
deploy runs `config:cache`.

```php
// ❌ app/Services/PaymentGateway.php:20 — null after `php artisan config:cache`
$key = env('STRIPE_SECRET');          // .env not loaded once config is cached

// ✅ read from config; the env() call lives only in config/services.php
$key = config('services.stripe.secret');
```
```php
// config/services.php  — this is the ONLY place env() is safe
'stripe' => ['secret' => env('STRIPE_SECRET')],
```

The finding is reported with its anchor and confidence:

```
🔴 HIGH — env() outside config/ on a cached deploy  (app/Services/PaymentGateway.php:20)
   env('STRIPE_SECRET') returns null after `php artisan config:cache` — silent, no error.
   Fix: add 'secret' => env('STRIPE_SECRET') to config/services.php; call config('services.stripe.secret').
```

See [`examples/report.md`](examples/report.md) for a full report.

---

## References

- [`references/prod-readiness-rules.md`](references/prod-readiness-rules.md) — the full check catalogue (A–G), the source of truth for every finding.
- [Laravel: Configuration & caching](https://laravel.com/docs/configuration#configuration-caching) · [Deployment: optimization](https://laravel.com/docs/deployment#optimization) · [Routing: route caching](https://laravel.com/docs/routing#route-caching)

---

## Anti-patterns

Never do any of these:

- ❌ **Invent a finding without a cited anchor** — every finding names a real `file:line`,
  `.env` key, or `route:list` entry. No anchor → no finding.
- ❌ **Flag `env()` inside `config/`** — that is the one place it belongs.
- ❌ **Flag a method/static call named `env`** (`$app->env(`, `Foo::env(`) as the helper —
  `\benv\(` matches it, but it isn't the config helper. Read the char before `env`: a `>`
  or `:` means it's a method call, not a finding.
- ❌ **Report a raw grep count** — confirm each candidate by reading it in context; a
  `dump` inside `var_dump`, or an `env(` in a comment/string, is not a finding.
- ❌ **Mark a `local`/`dev` `.env` value 🔴** — env-dependent values are 🟡 "if prod" until
  the environment is proven production.
- ❌ **Treat a code comment as evidence.** Derive every verdict from the structural reality
  (the call site, the key, the route), never from what a comment claims.
- ❌ **Call a real secret "placeholder" or vice-versa without saying it's a heuristic** —
  mark `.env.example` value findings "verify" when uncertain.
- ❌ **Claim the app is "safe to deploy" / call this a security audit** — a clean report
  means "nothing found in the checks I ran".
- ❌ **Edit `.env`, config, code, or `.gitignore`**, or run state-changing commands. Output
  advice + fixes for a human to apply.

---

## Guardrails

- **Advise-only.** Reads files and runs **read-only** commands (`php artisan route:list
  --json`, `git ls-files`, `git check-ignore`). Emits a report + fixes *for the human to
  apply and review*. The **sole** write it may perform is saving its own report file —
  only with the user's confirmation (step 9). It never edits `.env`, config, code, or
  `.gitignore`.
- **Ground truth, not guesswork.** Findings trace to real facts. Couldn't boot the app?
  Fall back to static parsing and **say so** — never fill the gap with an assumed route.
- **Flag human judgment.** Where a call needs context (is this the prod `.env`? is this a
  real secret? does the deploy cache config?), say so and hand the decision back.
- **Not a security product.** This is a structured reading of your repository with
  confidence levels, not a scanner or a deploy gate. A clean pass is not a guarantee.
