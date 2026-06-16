# composer-dependency-triage

**Tells you what to do about your Composer dependencies in Laravel.** Reads your project, changes nothing.

> Composer shows you what's outdated. This tells you **what to do about it**.

Ask *"audit my composer dependencies"* and get a plan: what to fix now, what to do carefully, what to defer. For dead packages it finds a maintained replacement — the case Composer usually stays silent on (*"No replacement was suggested"*).

---

## Install

One folder in `.claude/skills/` and it just works:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/composer-dependency-triage \
  .claude/skills/composer-dependency-triage
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

Open Claude Code in a project with `composer.json` and ask in plain words:

- *"audit my composer dependencies"*
- *"what should I update?"*
- *"are any of my packages abandoned?"*
- *"is it safe to bump this package?"*
- *"check my dependencies before upgrading Laravel"*

Nothing to configure.

---

## What you get

- **A color for every direct dependency:** 🔴 vulnerable · 🟠 abandoned · 🟡 big update · 🟢 safe update · ✅ fine.
- **A replacement for dead packages** — with a note: drop-in or an API change.
- **The safest fix for a vulnerability** — the lowest version that fixes it, no needless big upgrade.
- **A PHP/Laravel compatibility check** before any big update.
- **A three-lane plan** (now / carefully / defer) with ready commands. → [example report](examples/report.md)

It uses only the real `composer outdated` and `composer audit` output — it never makes up versions or CVEs. It is not a security scanner: a clean audit means *"nothing found"*, not *"guaranteed safe"*.

---

## What it does NOT do

- ❌ doesn't edit `composer.json`, the lockfile, or code — you run the commands;
- ❌ doesn't invent versions, CVEs, or replacements;
- ❌ doesn't run `composer update` — it only advises;
- ❌ doesn't replace a security scanner.

---

## Learn more

- [`USAGE.md`](USAGE.md) — full walkthrough
- [`SKILL.md`](SKILL.md) — the skill itself
- [`examples/report.md`](examples/report.md) — reference output

## License

[MIT](../../LICENSE) © Artem Proshkovskyi · part of the
[Laravel Maintenance Skills](https://github.com/ArtemProshkovskiy/laravel-maintenance-skills) library.
