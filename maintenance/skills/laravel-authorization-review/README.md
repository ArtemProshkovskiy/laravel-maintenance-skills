# laravel-authorization-review

**A Claude Code skill that finds IDOR / broken object-level authorization (BOLA) in Laravel — the bug class static scanners structurally miss.** Advise-only: it reads your code, never edits it.

> SAST tells you where data flows. This tells you where it flows to the **wrong user**.

Point it at any Laravel project and ask *"review my authorization"*. It walks every route's
authorization chain — middleware → policy/gate → Eloquent query scoping → API Resource output —
anchored to real `php artisan route:list --json`, and returns a **per-route coverage map** plus
confidence-rated findings, each with a cited controller `file:line`.

---

## Install just this skill

You don't need a marketplace or the rest of the library — drop this one folder into your
project's `.claude/skills/` and it activates on its own:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/maintenance/skills/laravel-authorization-review \
  .claude/skills/laravel-authorization-review
```

Prefer it available in **every** project? Install it globally instead:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/maintenance/skills/laravel-authorization-review \
  ~/.claude/skills/laravel-authorization-review
```

No `npx`? Clone and copy the folder by hand:

```bash
git clone https://github.com/ArtemProshkovskiy/laravel-maintenance-skills
cp -r laravel-maintenance-skills/maintenance/skills/laravel-authorization-review .claude/skills/
```

<details>
<summary>Want the whole suite (this + composer-dependency-triage) at once?</summary>

This skill ships in the `laravel-maintenance` Claude Code plugin. Add the marketplace once and
get every skill in the library:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```
</details>

---

## Use it

Open Claude Code **inside your Laravel project** (the folder with `artisan`) and ask in plain
language — the skill recognizes the request and runs itself:

- *"review my authorization"* / *"find IDOR in this app"*
- *"which routes have no auth?"*
- *"is this endpoint scoped to the owner?"*
- *"do my controllers check policies?"*
- *"review authorization on the routes in this PR"* — it intersects the route inventory with your
  changed files, so you sanity-check new endpoints before merge instead of scanning everything.

Nothing to configure. It finds `artisan`, runs `php artisan route:list --json`, reads your
controllers/policies/requests/resources, and assembles the report.

---

## What you get

A coverage map (every in-scope route × every layer) and a prioritized list:

| Lane | What it is |
|---|---|
| 🔴 **Verify & fix now** | high-confidence structural holes — no auth/authz/scope on owned data |
| 🟡 **Review** | likely IDOR / data leak that depends on whether a model is owned or a field is sensitive |
| 🔵 **Hardening** | defense-in-depth — missing policy, mass-assignment of ownership keys |
| ✅ **Covered** | all applicable layers present — shown so you can see what was checked |

Each finding carries its **confidence** (High / Medium / Low) and an **evidence chain**
(`route → Controller@method:line → missing layer`) you can verify in seconds. See a full
[example report](examples/report.md).

### Why it can be trusted (and won't hallucinate)

Every finding traces to a real route in `php artisan route:list --json` and a cited `file:line`.
No anchor → no finding. It recognizes **all** the legitimate ways Laravel authorizes (`can:`
middleware, `Gate::authorize`, `authorizeResource`, FormRequests, gates, spatie/laravel-permission)
and knows public-by-design routes (login, register, signature-verified webhooks), so it doesn't
cry wolf. Up to date with **Laravel 11 & 12** conventions.

---

## Requirements

| Requirement | Why |
|---|---|
| A Laravel project (`artisan` present) | the entry point; without it the skill stops |
| The app boots enough to run `php artisan route:list` | the ground-truth route inventory |
| Readable `app/`, `routes/`, `app/Policies/` | to walk the authorization chain |

If the app can't boot (missing extension, DB-at-boot, throwing provider), the skill **falls back**
to parsing `routes/*.php` statically and tells you the inventory is best-effort (lower confidence).

---

## Boundaries — what it does NOT do

- ❌ edit routes, controllers, policies, or any code — you apply every fix;
- ❌ invent a route, policy, or method — only what `route:list` and your files contain;
- ❌ cover **Livewire / Filament / Nova** action authorization — those live outside the
  `route:list` anchor and use their own models (a future version may add them);
- ❌ judge **business-logic** authorization (approval states, time-based rules) — flagged for a human;
- ❌ catch IDOR via indirect references not visible in code, or infra/gateway authz;
- ❌ replace a pentest. A finding-free report means *"nothing broken in the layers checked"*, **not** *"secure"*.

---

## Learn more

- [`USAGE.md`](USAGE.md) — full walkthrough
- [`SKILL.md`](SKILL.md) — the skill itself (the instructions Claude Code loads)
- [`examples/report.md`](examples/report.md) — reference output
- [`references/`](references/) — the authorization-pattern & public-by-design catalogues it reasons from

## License

[MIT](../../../LICENSE) © Artem Proshkovskyi · part of the
[Laravel Maintenance Skills](https://github.com/ArtemProshkovskiy/laravel-maintenance-skills) library.
