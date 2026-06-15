# composer-dependency-triage

**A Claude Code skill that turns `composer outdated` + `composer audit` into a prioritized, advise-only action plan** — and recommends maintained **replacements for abandoned packages**, the case where Composer itself usually just prints *"No replacement was suggested"*. Advise-only: it reads your project, never edits it.

> Composer tells you what's outdated. This tells you what to **do** about it.

Point it at any Laravel project and ask *"audit my composer dependencies"*. It reads your
`composer.json` / `composer.lock` and the real tool JSON as ground truth (never invents a version
or a CVE), focuses on your **direct** dependencies, checks PHP/Laravel compatibility before
recommending any major bump, and emits a **Do-now / Do-carefully / Defer** plan with exact commands.

---

## Install just this skill

You don't need a marketplace or the rest of the library — drop this one folder into your
project's `.claude/skills/` and it activates on its own:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/maintenance/skills/composer-dependency-triage \
  .claude/skills/composer-dependency-triage
```

Prefer it available in **every** project? Install it globally instead:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/maintenance/skills/composer-dependency-triage \
  ~/.claude/skills/composer-dependency-triage
```

No `npx`? Clone and copy the folder by hand:

```bash
git clone https://github.com/ArtemProshkovskiy/laravel-maintenance-skills
cp -r laravel-maintenance-skills/maintenance/skills/composer-dependency-triage .claude/skills/
```

<details>
<summary>Want the whole suite (this + laravel-authorization-review) at once?</summary>

This skill ships in the `laravel-maintenance` Claude Code plugin. Add the marketplace once and
get every skill in the library:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```
</details>

---

## Use it

Open Claude Code **inside your Laravel project** (the folder with `composer.json`) and ask in plain
language — the skill recognizes the request and runs itself:

- *"audit my composer dependencies"*
- *"what should I update?"*
- *"are any of my packages abandoned?"*
- *"is it safe to bump this package?"*
- *"check my dependencies before upgrading Laravel"*

Nothing to configure. It finds `composer.json`, runs the read-only tools, and assembles the report.

---

## What you get

- **Reads real tool output as ground truth** — never invents a version number or a CVE.
- **Classifies every direct dependency**: 🔴 vulnerable · 🟠 abandoned · 🟡 major update · 🟢 safe update · ✅ healthy.
- **Recommends maintained replacements for abandoned packages** — even when Composer just prints
  *"No replacement was suggested"* — with the migration shape (drop-in vs. API change).
- **Finds the safest fix for a vulnerability** — reads each advisory range literally, computes the
  lowest fixed version, prefers a drop-in patch within your current major, and verifies that
  version exists, so a CVE never pushes you into a needless major migration.
- **Checks PHP/Laravel compatibility** before suggesting a major, routing framework-coupled bumps
  into the upgrade path instead of recommending them blindly.
- **Scores breakage risk** and emits a three-lane plan (Do-now / Do-carefully / Defer) with exact
  commands. See a full [example report](examples/report.md).

It is **not a security product**: vulnerability facts come from `composer audit` (the PHP Security
Advisories database), not a CVE database of its own. A clean audit means *"nothing found"*, not
*"guaranteed secure"*.

---

## Requirements

| Requirement | Why |
|---|---|
| A Laravel/PHP project with `composer.json` | the entry point; without it the skill stops |
| Composer available (`composer outdated`, `composer audit`) | the ground-truth dependency + advisory data |
| Readable `composer.json` / `composer.lock` | to classify direct dependencies and pinned versions |

---

## Boundaries — what it does NOT do

- ❌ edit `composer.json`, the lockfile, or any code — you run every command yourself;
- ❌ invent a version number, a CVE, or a replacement — only what the real tools + registries report;
- ❌ run `composer update` / install anything — it's advise-only and read-only;
- ❌ replace a security scanner. It relays `composer audit` facts with judgment, not a CVE database of its own.

---

## Learn more

- [`USAGE.md`](USAGE.md) — full walkthrough
- [`SKILL.md`](SKILL.md) — the skill itself (the instructions Claude Code loads)
- [`examples/report.md`](examples/report.md) — reference output

## License

[MIT](../../../LICENSE) © Artem Proshkovskyi · part of the
[Laravel Maintenance Skills](https://github.com/ArtemProshkovskiy/laravel-maintenance-skills) library.
