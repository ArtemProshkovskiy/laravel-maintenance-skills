# Laravel Production Readiness Review — Example Report

> Reference output for an illustrative **Laravel 12** project being prepared for deploy.
> File names, line numbers, and `.env` values are **fictional**, used only to show the
> report shape. A real run derives every finding from the project's real `.env` / config /
> `route:list` / git state and cites a real anchor — it never invents one.

---

## 🔴 Not safe to deploy — 4 blockers

`laravel/framework` 12.x · `.env` `APP_ENV=production` · git repo present · deploy runs
`config:cache` + `route:cache`.

| Lane | Count |
|------|-------|
| 🔴 Fix now (breaks/leaks on prod) | 4 |
| 🟡 Review | 3 |
| 🔵 Hardening | 2 |
| ✅ Covered | 5 |

> **Not a "safe to deploy" stamp.** A clean check only means "no problem in what I audited" —
> it can't see server-injected env vars, web-server config, or file permissions (see Boundaries).

---

## 🔴 Fix now

**1. `env()` outside `config/` → `null` after `config:cache`** *(deploy runs `config:cache`, so this is live)*
- `app/Services/PaymentGateway.php:20` — `env('STRIPE_SECRET')` → null in prod → charges fail silently
- `app/Http/Middleware/FeatureFlag.php:14` — `env('BETA_FEATURES', false)` → always falls back
- **Fix:** move each value into a `config/*.php` key, read via `config('…')`:
  ```php
  // config/services.php
  'stripe' => ['secret' => env('STRIPE_SECRET')],
  // app/Services/PaymentGateway.php:20
  $key = config('services.stripe.secret');
  ```

**2. `.env:3` — `APP_DEBUG=true` on a production `.env`** → Ignition leaks stack traces, env, and source to any visitor on an error → **Fix:** `APP_DEBUG=false`.

**3. `.env` committed to git** (`git ls-files` = tracked, `git check-ignore` = not ignored) → every secret is in history → **Fix:** add `.env` to `.gitignore`, `git rm --cached .env`, **rotate** `APP_KEY` / DB / Stripe / mail.

**4. `.env.example:18` — real Stripe key in the template** (`sk_live_…`) → **Fix:** replace with `STRIPE_SECRET=` and rotate.

---

## 🟡 Review — your judgment

- `routes/web.php:42` — closure route `/status` → `route:cache` fails on deploy → move to a controller.
- `.env:21` — `QUEUE_CONNECTION=sync` → "queued" jobs run inline; switch to `redis`/`database` if you have async work.
- `composer.json:24` — `laravel/telescope` in `require` → ships to prod & exposes `/telescope`; move to `require-dev`.

---

## 🔵 Hardening

- `.env` drift — `PULSE_DB_CONNECTION`, `SCOUT_DRIVER` are in `.env` but missing from `.env.example` → add as placeholders.
- `.env:9` — `LOG_LEVEL=debug` on prod → verbose logs may persist sensitive payloads; use `warning`/`error`.

---

## Fix in this order

1. `APP_DEBUG=false` (one line, stops the leak now).
2. Untrack `.env` + rotate the four secrets.
3. Move the 2 `env()` calls into `config/` (or your deploy breaks on next `config:cache`).
4. Replace the real key in `.env.example`.
5. Then the 🟡 items before the next release.

---

## ✅ Covered

- `APP_KEY` set · `APP_URL=https://app.example.com` (not localhost).
- `CACHE_STORE=redis`, `SESSION_DRIVER=redis` — multi-server safe · `MAIL_MAILER=ses`.
- `env()` everywhere else is **inside `config/`** (12 sites) — legitimate.

---

*Hard facts (env() locations, `.env` keys, git state, the deploy workflow) come from the
tools and filesystem; judgment (is this the prod `.env`? a real secret?) is marked for you.
No files were modified — apply each fix on a branch. A clean check is not "safe to deploy".*
