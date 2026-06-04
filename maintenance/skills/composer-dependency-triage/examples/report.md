# Composer Dependency Triage — Example Report

> Reference output for an illustrative **Laravel 10 / PHP 8.1** project. Package
> names, versions, and advisories are **fictional**, used only to show the report
> shape and the kind of judgment to apply. A real run derives every hard fact from
> `composer outdated --direct --format=json` and `composer audit --format=json`
> against the actual project — and never invents versions or CVEs.

---

## Summary

Audited **15 direct dependencies** (10 runtime, 5 dev). Project: `laravel/framework`
10.48.2 on PHP 8.1.

| Class | Count |
|-------|-------|
| 🔴 Vulnerable | 2 (+1 transitive, see below) |
| 🟠 Abandoned | 1 |
| 🟡 Outdated — major | 3 |
| 🔒 Blocked on upgrade | 1 |
| 🟢 Outdated — minor/patch | 4 |
| ✅ Healthy | 5 |

**Headline:** Fix the auth advisory and the abandoned `faker` today — both are
low-effort. One vulnerability is *transitive* (you don't bump it directly). The
`league/flysystem` major is **blocked on a PHP upgrade**, and the `laravel/framework`
major belongs to a dedicated Laravel upgrade — both deferred.

---

## 🔴 Security — Do now

| Package | Type | Installed | Advisory | Fixed in | Action |
|---------|------|-----------|----------|----------|--------|
| `acme/jwt-auth` | direct | 1.4.2 | CVE-2024-00000 — signature bypass (high) | 1.4.5 | drop-in patch, same major → `composer update acme/jwt-auth --with-dependencies --dry-run` first |
| `acme/crypto` | direct (**exact pin** `=2.0.30`) | 2.0.30 | 4 advisories — `<=2.0.53` (high) | 2.0.54 | drop-in within 2.x, but the **exact pin blocks `update`** → use `composer require` (see below) |
| `evil/transitive-lib` | **transitive** | 2.0.1 | CVE-2024-11111 — RCE (critical) | 2.0.4 | **Don't update directly** — see chain below |

> **Fix-version sanity check (step 4).** Advisory floors are read literally:
> `<=X` ⇒ the fix is `X+1`, not "the branch is dead". `acme/jwt-auth` clears at
> **1.4.5** — a same-major drop-in, verified present via `composer show
> acme/jwt-auth --all` — so this stays a ✅ Do-now patch, **not** a major
> migration. Escalate to a new major only when the current major has no fixed
> release.

> **Exact-pin finding (step 4/8).** `acme/crypto` is constrained to `=2.0.30` in
> `composer.json`, so `composer update acme/crypto` is a **no-op** — it can't pass
> the pin. The fix clears at **2.0.54** (read `<=2.0.53` ⇒ `2.0.54`), verified via
> `composer show acme/crypto --all`. Remediate by **changing the constraint**, and
> preserve the pinning style — pin to the exact fixed version:
> `composer require "acme/crypto:2.0.54"`. For a **crypto-sensitive** package, staying
> exact is the safer default — loosening to `^2.0.54`/`~2.0.54` buys automatic patch
> currency but cedes review control over future releases (supply-chain tradeoff).

**Transitive finding — explain the chain, don't tell the user to bump it directly:**

```bash
composer why evil/transitive-lib
# acme/reporting 3.1.0 requires evil/transitive-lib (^2.0)
```

> `evil/transitive-lib` is pulled in by your direct dependency **`acme/reporting`**.
> You can't safely `update` it on its own. Options: bump `acme/reporting` (check if
> a release exists that requires the patched `^2.0.4`), or — only if forced — add a
> temporary constraint. **Human judgment required.** Facts: advisory from `composer
> audit`; chain from `composer why`.

---

## 🟠 Abandoned — Do carefully

| Package | Composer says | Recommended replacement | Migration |
|---------|---------------|-------------------------|-----------|
| `fzaninotto/faker` | abandoned — *"No replacement was suggested"* | **`fakerphp/faker`** *(verify before adopting)* | Near drop-in: same `Faker\` namespace; swap the require, code usually unchanged. |

```bash
# Human runs this, on a branch, with tests:
composer remove --dev fzaninotto/faker
composer require --dev fakerphp/faker
```

> This is the skill's core value-add: Composer flagged it abandoned with **no
> suggestion**; the maintained community fork is `fakerphp/faker`. Replacement is
> **judgment, not a fact** — confirm it's current on Packagist and that your
> factories/seeders compile.

---

## 🟢 Safe updates — Do now

| Package | Installed | Latest | Bump | Effort |
|---------|-----------|--------|------|--------|
| `guzzlehttp/guzzle` | 7.8.0 | 7.8.2 | patch | drop-in |
| `nesbot/carbon` | 2.71.0 | 2.72.1 | minor | drop-in, skim changelog |
| `spatie/laravel-permission` | 6.1.0 | 6.3.0 | minor | drop-in |
| `league/csv` | 9.10.0 | 9.11.0 | minor | drop-in |

```bash
# Human runs — safe currency wins, on a branch:
composer update guzzlehttp/guzzle nesbot/carbon spatie/laravel-permission league/csv
```

---

## 🟡 / 🔒 Major updates — Defer / blocked

| Package | Installed | Latest | Jump | Compatibility | Verdict |
|---------|-----------|--------|------|---------------|---------|
| `laravel/framework` | 10.48.2 | 11.x | major | needs PHP 8.2+ (project on 8.1) and is framework-coupled | **🔒 Defer** — dedicated Laravel upgrade |
| `league/flysystem` | 3.15.0 | 4.x | major | **latest requires PHP 8.2** — project on 8.1 | **🔒 Blocked on upgrade** — bump PHP first |
| `phpunit/phpunit` | 10.5.0 | 11.x | major | compatible with PHP 8.1 | **⚠️ Do carefully** (dev) |
| `pestphp/pest` | 2.34.0 | 3.x | major | tracks PHPUnit major | **⚠️ Do carefully** (dev) — do alongside PHPUnit |

> `league/flysystem` 4.x is **not** recommendable today: its latest requires PHP
> 8.2 and the project runs 8.1 (see `references/laravel-compatibility.md`). Marked
> blocked-on-upgrade and handed to the upgrade path — *not* recommended blindly.
> `laravel/framework` 11 is both PHP-blocked and framework-coupled → upgrade pass.

---

## 🧰 Dev dependencies (grouped separately)

Covered above: `fzaninotto/faker` (abandoned), `phpunit/phpunit` + `pestphp/pest`
(majors, do carefully together). `laravel/pint` and `barryvdh/laravel-debugbar`
are ✅ healthy.

---

## Action plan

### ✅ Do now
```bash
# Preview first — confirm the resolver succeeds and see what else moves:
composer update acme/jwt-auth --with-dependencies --dry-run
# Security patch (direct, drop-in) + safe minor/patch currency, on a branch, tests green:
composer update acme/jwt-auth
composer update guzzlehttp/guzzle nesbot/carbon spatie/laravel-permission league/csv

# acme/crypto is EXACT-pinned (=2.0.30) — `update` won't move it; change the constraint:
composer require "acme/crypto:2.0.54" --dry-run        # preview the resolve
composer require "acme/crypto:2.0.54"                  # rewrites pin + updates, drop-in within 2.x
```

### ⚠️ Do carefully (read first, branch, tests green)
```bash
# Replace abandoned faker (no Composer suggestion → fakerphp/faker)
composer remove --dev fzaninotto/faker
composer require --dev fakerphp/faker

# Dev test stack majors — plan PHPUnit + Pest together, read changelogs
composer require --dev "phpunit/phpunit:^11.0" "pestphp/pest:^3.0"

# Transitive RCE — act on the DIRECT parent, not the transitive package
composer why evil/transitive-lib       # confirm the chain
# then bump acme/reporting if a fixed release exists
```

### 🛑 Defer / blocked
```text
league/flysystem 3 → 4   🔒 requires PHP 8.2 (project on 8.1) — upgrade PHP first
laravel/framework 10 → 11 🔒 PHP 8.2 + framework-coupled — dedicated Laravel upgrade pass
```

---

## Notes & caveats

- **Hard facts** (versions, CVEs, abandoned flags, the `why` chain) come from
  `composer outdated`, `composer audit`, and `composer why`. **Judgment**
  (replacements, effort, risk) is AI-derived — replacements marked *verify before adopting*.
- Triage covers **direct** dependencies; the one transitive advisory is explained
  via its chain, not recommended for direct update.
- A clean `composer audit` would mean "audit found nothing", **not** "guaranteed secure".
- **Won't-break-the-app discipline:** preview every bump with `--with-dependencies
  --dry-run`, apply one lane at a time on a branch, commit the lockfile between
  steps, and run the test suite after each — so a bad update is isolated and
  revertible.
- This is **advice only** — no files were modified. Every command is for you to run and review.
- Could not check: *(example)* none — both tools ran. If `audit` had been
  unavailable (Composer < 2.4 / offline), that gap would be stated here explicitly.
