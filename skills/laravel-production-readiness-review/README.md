# laravel-production-readiness-review

**Catches the "works locally, breaks/leaks in production" bugs in Laravel — before you deploy.** Reads your project, changes nothing.

> It works on your machine. This finds why it will break — or leak — in production.

Ask *"is this safe to deploy?"* and it audits the gap between dev and prod: `env()` used where `config:cache` will null it out, closure routes that break `route:cache`, unsafe `.env` values, leaked secrets, and debug leftovers. Anchored to your real `.env` / config / `route:list` / git. You get a plan with the exact fix for each finding.

---

## Install

One folder in `.claude/skills/` and it just works:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/laravel-production-readiness-review \
  .claude/skills/laravel-production-readiness-review
```

Want it in every project? Install it globally — use `~/.claude/skills/...` instead.

<details>
<summary>Want all skills at once</summary>

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```
</details>

---

## Use it

Open Claude Code in a project with `artisan` and ask in plain words:

- *"is this safe to deploy?"* / *"production readiness check"*
- *"env is null after deploy"* / *"config:cache broke my app"*
- *"did I leak any secrets?"*
- *"why does it work locally but not in production?"*

Nothing to configure.

---

## What it checks

- **`env()` outside `config/`** *(the flagship)* — returns `null` after `config:cache`, silently. **No other tool audits this.**
- **Closure routes** — break `php artisan route:cache`.
- **Unsafe `.env` values** — `APP_DEBUG=true`, empty `APP_KEY`, `QUEUE=sync`, `APP_URL=localhost`, and more.
- **Secret exposure** — `.env` tracked in git, real values in `.env.example`.
- **`.env` drift** — keys missing from `.env.example`.
- **Debug leftovers** — `dd()`, `dump()`, `ray()`, `var_dump()`.

Output: a summary + four lanes (🔴 fix now · 🟡 review · 🔵 hardening · ✅ covered), each finding as `file:line → problem → exact fix`. → [example report](examples/report.md)

It reads `APP_ENV` first: on a `local`/`dev` file the value checks become 🟡 *"if these are production values"*, so it won't cry wolf on your dev setup.

---

## What it does NOT do

- ❌ doesn't edit `.env`, config, code, or `.gitignore` — you apply the fixes;
- ❌ doesn't see server-injected env vars, web-server config, or file permissions — it reads the repo;
- ❌ doesn't replace a deploy pipeline or a security audit. "Clean" = *"nothing found in what I checked"*, not *"safe to deploy"*.

---

## Learn more

- [`USAGE.md`](USAGE.md) — full walkthrough
- [`SKILL.md`](SKILL.md) — the skill itself
- [`examples/report.md`](examples/report.md) — reference output
- [`references/prod-readiness-rules.md`](references/prod-readiness-rules.md) — the check catalogue it reasons from

## License

[MIT](../../LICENSE) © Artem Proshkovskyi · part of the
[Laravel Maintenance Skills](https://github.com/ArtemProshkovskiy/laravel-maintenance-skills) library.
