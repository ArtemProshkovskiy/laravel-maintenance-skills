# laravel-authorization-review

**Finds IDOR / broken object-level authorization (BOLA) in Laravel** — the bug class static scanners miss. Reads your code, changes nothing.

> SAST shows you where data flows. This shows you where it flows to the **wrong user**.

Ask *"review my authorization"* and for every route it checks the whole chain: middleware → policy/gate → Eloquent query → API Resource output. It's anchored to the real `php artisan route:list --json`. You get a coverage map and findings, each with a `file:line`.

---

## Install

One folder in `.claude/skills/` and it just works:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/laravel-authorization-review \
  .claude/skills/laravel-authorization-review
```

Want it in every project? Install it globally — use `~/.claude/skills/...` instead.

<details>
<summary>Want both skills at once</summary>

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```
</details>

---

## Use it

Open Claude Code in a project with `artisan` and ask in plain words:

- *"review my authorization"* / *"find IDOR in this app"*
- *"which routes have no auth?"*
- *"is this endpoint scoped to the owner?"*
- *"do my controllers check policies?"*
- *"review authorization on the routes in this PR"* — it uses only the changed files.

Nothing to configure.

---

## What you get

A coverage map (every route × every layer) and a prioritized list:

| Lane | What it is |
|---|---|
| 🔴 **Fix now** | a clear hole — no auth / authz / owner check on owned data |
| 🟡 **Review** | likely IDOR / data leak — depends on the model or field |
| 🔵 **Hardening** | extra safety — missing policy, owner keys open to mass-assignment |
| ✅ **Covered** | all layers present — shown so you see what was checked |

Each finding carries its **confidence** (High / Medium / Low) and an **evidence chain** (`route → Controller@method:line → missing layer`). → [example report](examples/report.md)

Every finding ties to a real route and a `file:line` — no anchor, no finding. It knows every valid way to authorize in Laravel (`can:`, `Gate::authorize`, `authorizeResource`, FormRequest, spatie/laravel-permission) and which routes are public on purpose (login, register, webhooks), so it won't cry wolf. Up to date with **Laravel 11 & 12**.

---

## What it does NOT do

- ❌ doesn't edit routes, controllers, policies, or code — you apply the fixes;
- ❌ doesn't invent routes, policies, or methods;
- ❌ doesn't cover **Livewire / Filament / Nova** — those live outside `route:list`;
- ❌ doesn't judge business-logic authorization (states, time-based rules) — flagged for a human;
- ❌ doesn't replace a pentest. A finding-free report means *"nothing broken in the layers checked"*, not *"secure"*.

If the app can't boot, the skill falls back to parsing `routes/*.php` and tells you the inventory is best-effort.

---

## Learn more

- [`USAGE.md`](USAGE.md) — full walkthrough
- [`SKILL.md`](SKILL.md) — the skill itself
- [`examples/report.md`](examples/report.md) — reference output
- [`references/`](references/) — the authorization-pattern & public-by-design catalogues

## License

[MIT](../../LICENSE) © Artem Proshkovskyi · part of the
[Laravel Maintenance Skills](https://github.com/ArtemProshkovskiy/laravel-maintenance-skills) library.
