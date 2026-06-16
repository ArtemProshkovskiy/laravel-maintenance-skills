# Using `laravel-production-readiness-review`

A short guide: what the skill does, how to install it, and how to run it on your Laravel project.

---

## 1. What it does

Local development hides a whole class of bugs. Code runs fine on your machine — no caches,
`APP_DEBUG=true`, one server — then **breaks or leaks the moment it hits production** and the
deploy runs `php artisan config:cache` / `route:cache`.

This skill audits that **dev↔prod gap** before you deploy. It:

- finds **`env()` called outside `config/`** — which returns `null` after `config:cache`,
  silently (the flagship case no other tool checks);
- finds **closure routes** that make `route:cache` fail;
- flags unsafe **`.env` values** (`APP_DEBUG=true`, empty `APP_KEY`, `QUEUE=sync`,
  `APP_URL=localhost`, …);
- catches **leaked secrets** (`.env` tracked in git, real values in `.env.example`);
- catches **`.env` drift** and leftover **debug calls** (`dd`, `dump`, `ray`, `var_dump`).

> **Advise-only.** The skill changes nothing: it never edits `.env`, config, code, or
> `.gitignore`. You apply every fix.

### Why it can be trusted

Every finding traces to a **real fact** — a cited `file:line`, a `.env` key, a `route:list`
entry, or a git state. No anchor, no finding. And it reads `APP_ENV` first, so it won't flag
your local `.env` values as production problems.

---

## 2. Requirements

| Requirement | Why |
|---|---|
| A Laravel project (`artisan` present) | the entry point; without it the skill stops |
| `php artisan route:list` runnable | to find closure routes (static fallback if it can't boot) |
| Readable `.env`, `.env.example`, `config/`, code | the facts it audits |
| A git repo *(recommended)* | to check whether `.env` is tracked / ignored |

---

## 3. Install

One folder in `.claude/skills/` and it just works:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/laravel-production-readiness-review \
  .claude/skills/laravel-production-readiness-review
```

Want it in every project — install into `~/.claude/skills/...`. Want all skills at once:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```

---

## 4. Run it

Open Claude Code in your project folder (the one with `artisan`) and ask in plain words:

- "is this safe to deploy?"
- "production readiness check"
- "env is null after deploy" / "config:cache broke my app"
- "did I leak any secrets?"
- "why does it work locally but not in production?"

Nothing to configure — the skill reads `.env` / config / `route:list` / git and assembles the report.

---

## 5. What you get — the report

Clean Markdown, straight into the chat. Full example in [`examples/report.md`](examples/report.md). Shape:

```markdown
## Summary
Project: Laravel 12 · APP_ENV=production — 🔴 4 break/leak, 🟡 3 review, 🔵 3 hardening, ✅ 5 covered.

## 🔴 Fix now
app/Services/PaymentGateway.php:20 → env('STRIPE_SECRET') returns null after config:cache
  → fix: config('services.stripe.secret') (move env() into config/services.php)
.env:3 → APP_DEBUG=true on a production .env → leaks stack traces → set APP_DEBUG=false
```

### How to read the lanes

| Lane | What it is | What to do |
|---|---|---|
| 🔴 **Fix now** | breaks or leaks on prod right now | confirm with the cited anchor, apply the fix |
| 🟡 **Review** | value/code wrong for prod, or depends on an assumption | decide (is this prod? is this a real secret?), then fix |
| 🔵 **Hardening** | drift, debug leftovers, log level | tidy when convenient |
| ✅ **Covered** | checks that passed | nothing — shown so you see what was verified |

---

## 6. How it avoids false positives

- **Env gating.** It reads `APP_ENV`; on a `local`/`dev` file the value checks become 🟡
  "if these are production values" instead of 🔴.
- **`env()` inside `config/` is never flagged** — that's the one place it belongs.
- **Grep finds candidates, reading confirms them** — a `dump` inside `var_dump`, or an
  `env(` in a comment, is dropped.
- **Uncertain calls are "verify", not asserted** — e.g. a real-looking value in
  `.env.example` is flagged with the assumption to check.

---

## 7. Saving the report (optional)

By default the report stays in the chat — nothing is written to disk. After printing it, the
skill **may offer** to save it to `storage/logs/production-readiness-<date>.md` (git-ignored
in Laravel). You can give it a different path. Without your confirmation, no file is created.

---

## 8. What it does NOT do

- ❌ edit `.env`, config, code, or `.gitignore` — you apply every fix;
- ❌ see runtime env vars set outside `.env` (real server, CI, secrets manager);
- ❌ check web-server config, file permissions, or TLS — it reads the repo, not the box;
- ❌ cover performance, dependency vulnerabilities, or authorization — those are the other
  skills in this library;
- ❌ replace a deploy pipeline or a pentest. A clean report means "nothing found in the
  checks I ran", **not** "safe to deploy".

---

## 9. Common situations

| Situation | What the skill does |
|---|---|
| Not a Laravel project | says so and stops |
| App won't boot (`route:list` fails) | falls back to parsing `routes/*.php`, says best-effort, drops confidence |
| No `.env` | skips the `.env`-value/secret/drift checks, says why, still runs code checks |
| Not a git repo | skips the "tracked by git" check, keeps the `.gitignore` / `.env.example` checks |
| `APP_ENV=local` | env-dependent values reported as 🟡 "if these are prod values", not 🔴 |
| Everything clean | says so plainly — "no problems in the checks I ran" (still not "safe to deploy") |

---

**In short:** install the skill → open Claude Code in your Laravel project → ask "is this safe
to deploy?" → get a 🔴/🟡/🔵 plan with `file:line` and the exact fix, and apply it yourself.
