# Laravel Maintenance Skills for Claude Code

**An open, growing library of advise-only [Claude Code](https://claude.com/claude-code) skills for maintaining Laravel projects** — dependency hygiene, update triage, and upgrade readiness. Add the library once; pick the skill you need.

> ### Composer tells you what's outdated. These skills tell you what to do about it.

🌐 **[Browse the skills on the project site →](https://ArtemProshkovskiy.github.io/laravel-maintenance-skills/)** · by [Artem Proshkovskyi](https://github.com/ArtemProshkovskiy)

---

## The problem

Every Laravel project drifts. Packages fall behind, advisories pile up, a few
dependencies quietly get abandoned, and a framework upgrade keeps getting pushed
to "later". The tools you already have only get you halfway:

- `composer outdated` lists current-vs-latest versions — but no priority, no risk, no plan.
- `composer audit` lists known advisories — but doesn't tell you what to do, or in what order.

What's missing is the **judgment layer**: the part that turns that raw output into
a prioritized, human-readable **action plan** — what to fix now, what to approach
carefully, what to defer — with the exact commands and the reasoning behind each call.

That's what these skills add to Claude Code.

---

## What a skill is here

A skill is a folder Claude Code loads **on demand** the moment your request
matches it — no setup, no flags. You just ask in plain language ("audit my
composer dependencies") and the right skill activates, runs the real tools, and
writes the report.

Every skill in this repo is **advise-only**. It reads your project and runs
**read-only** commands (`composer outdated`, `composer audit`, `composer why`),
then hands you recommendations. It **never** edits your `composer.json`,
lockfile, or code — you run every command yourself, after reviewing it.

---

## Skills in this library

| Skill | What it does | Status |
|-------|--------------|--------|
| [`composer-dependency-triage`](maintenance/skills/composer-dependency-triage/) | Turns `composer outdated` + `composer audit` output into a **Do-now / Do-carefully / Defer** plan, recommends **maintained replacements** for abandoned packages, and checks PHP/Laravel compatibility before any major bump. | ✅ Available |
| `upgrade-orchestrator` | Plan and stage a Laravel framework upgrade end to end. | 🔜 Planned |

The suite is built to grow — each new skill is a self-contained, advise-only
folder under [`maintenance/skills/`](maintenance/skills/).

---

## Featured: `composer-dependency-triage`

A Laravel-native dependency triage advisor. Point it at any Laravel project and it:

- **Reads real tool output as ground truth** — never invents a version number or a CVE.
- **Classifies every direct dependency**: 🔴 vulnerable · 🟠 abandoned · 🟡 major update · 🟢 safe update · ✅ healthy.
- **Recommends maintained replacements for abandoned packages** — even when Composer just prints *"No replacement was suggested"* — with the migration shape (drop-in vs. API change).
- **Finds the safest fix for a vulnerability**: it reads each advisory range literally, computes the lowest fixed version, prefers a drop-in patch within the current major, and **verifies that version actually exists** before recommending it — so you don't get pushed into a needless major migration.
- **Checks PHP/Laravel compatibility** before suggesting a major, and routes framework-coupled bumps into the upgrade path instead of recommending them blindly.
- **Scores breakage risk** and emits a three-lane plan with exact commands, so an update can't silently take your app down.

### Why this beats running the tools by hand

| | What it gives you | What it leaves out |
|---|---|---|
| `composer outdated` | current vs. latest versions, the abandoned flag | priority, risk, effort, a plan |
| `composer audit` | known advisories for installed versions | what to do, in what order, which fix is safe |
| **This skill** | **all of the above read as ground truth, plus** classification, replacement advice for abandoned packages, the *safest* verified fix version, compatibility checks, breakage-risk scoring, and a Do-now / Do-carefully / Defer plan with commands | it's an advisor, not a scanner — it never edits your project and relies on `composer audit` for vulnerability facts |

It is **not** a security product: vulnerability facts come from `composer audit`
(the PHP Security Advisories database), not from a CVE database of its own. A
clean audit means "nothing found", not "guaranteed secure".

---

## Example report (excerpt)

```markdown
## Summary
Audited 19 direct dependencies — 🔴 3 vulnerable, 🟠 3 abandoned, 🟡 9 major, 🟢 3 safe.
Headline: two of three security holes are drop-in patches; the third needs the Laravel 9→10 upgrade.

## 🔴 Security — Do now
| Package             | Installed | Advisory                    | Fixed in | Action                                  |
|---------------------|-----------|-----------------------------|----------|-----------------------------------------|
| phpseclib/phpseclib | 2.0.30    | 7 advisories (mostly high)  | 2.0.54   | drop-in within 2.x — do NOT jump to 3.x |
| guzzlehttp/guzzle   | 7.4.0     | 5 high (CVE-2022-3109x …)   | 7.4.5    | drop-in within major 7                  |

## 🟠 Abandoned — Do carefully
laravelcollective/html → spatie/laravel-html  (API change — audit your Blade forms)

## Action plan
✅ Do now        — verified drop-in security patches + safe minor/patch bumps
⚠️ Do carefully  — swap abandoned packages; plan the Laravel 9 → 10 upgrade
🛑 Defer         — framework-coupled majors, sequenced into the upgrade
```

See the full reference output in
[`maintenance/skills/composer-dependency-triage/examples/report.md`](maintenance/skills/composer-dependency-triage/examples/report.md).

---

## Install (Claude Code)

This repo is a Claude Code **plugin marketplace**. Add the marketplace, then
install the plugin — that gives you every skill in the suite at once.

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```

---

## Using it

Open Claude Code **inside your Laravel project** (the folder with `composer.json`)
and ask in plain language — the skill recognizes the request and activates itself:

- *"audit my composer dependencies"*
- *"what should I update?"*
- *"are any of my packages abandoned?"*
- *"is it safe to bump this package?"*
- *"check my dependencies before upgrading Laravel"*

Nothing to configure. Claude Code finds `composer.json`, runs the right read-only
commands, and assembles the report. For a full walkthrough see
[`USAGE.md`](maintenance/skills/composer-dependency-triage/USAGE.md).

---

## Library layout

```
laravel-maintenance-skills/
├── .claude-plugin/marketplace.json     # registers the plugin (the library)
├── docs/                               # the catalog website (GitHub Pages)
│   ├── index.html                      #   library home + searchable catalog
│   ├── skills/<skill>.html             #   one detail page per skill (SEO-indexed)
│   ├── assets/{style.css,og.svg}
│   ├── sitemap.xml · robots.txt
└── maintenance/                        # the plugin
    ├── .claude-plugin/plugin.json
    └── skills/
        └── <skill-name>/               # one self-contained, advise-only skill
            ├── SKILL.md                #   required — the skill itself
            ├── USAGE.md                #   how to run it
            ├── examples/report.md      #   reference output
            ├── references/             #   on-demand deep material
            └── tests/                  #   acceptance fixture + oracle
```

## Adding a skill to the library

The library is built to grow. To add one:

1. Create `maintenance/skills/<skill-name>/` with a `SKILL.md` (and ideally
   `USAGE.md` + an `examples/` report). Keep it **advise-only**.
2. Add a row to the **Skills in this library** table above.
3. Surface it on the catalog site: add a `<a class="scard">` card to
   `docs/index.html`, a detail page `docs/skills/<skill-name>.html` (copy an
   existing page's SEO head), and a `<url>` entry in `docs/sitemap.xml`.

Each skill stays self-contained, so it can also be read and installed on its own.

## License

[MIT](LICENSE) © Artem Proshkovskyi
