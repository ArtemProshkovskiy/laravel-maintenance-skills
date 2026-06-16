# Laravel Maintenance Skills

**A library of advise-only [Claude Code](https://claude.com/claude-code) skills that keep a Laravel project healthy.** They read your project and change nothing — they only advise. You don't install the whole library; grab just the one skill you need (one folder, no marketplace required).

> ### Your tools tell you what's outdated and where data flows. These skills tell you **what to do about it.**

🌐 **[Browse the skills on the project site →](https://ArtemProshkovskiy.github.io/laravel-maintenance-skills/)** · by [Artem Proshkovskyi](https://github.com/ArtemProshkovskiy)

---


## What a skill is here

A skill is a folder Claude Code loads **on demand** the moment your request matches it — no setup, no flags. You ask in plain language ("audit my composer dependencies") and the right skill activates, runs the real tools, and writes the report.

Every skill here is **advise-only**. It reads your project and runs **read-only** commands (`composer outdated`, `composer audit`, `php artisan route:list`), then hands you recommendations. It **never** edits your `composer.json`, lockfile, routes, policies, or code — you run every command yourself.

---

## The skills

Each skill is its own folder. Install the one you need and read the details in its README.

### [composer-dependency-triage](skills/composer-dependency-triage/README.md)

What to do about your Composer dependencies. Turns `composer outdated` + `composer audit` into a now / carefully / defer plan, finds live replacements for dead packages, and picks the safest fix for a vulnerability. → [README](skills/composer-dependency-triage/README.md)

### [laravel-authorization-review](skills/laravel-authorization-review/README.md)

Finds IDOR / broken object-level authorization (BOLA) — the bug class static scanners miss. Checks the full authorization chain on every route and shows what's covered and what's not. → [README](skills/laravel-authorization-review/README.md)

---

## Install

Each skill installs as one folder — no marketplace needed. Swap in the skill you want:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/<skill-name> \
  .claude/skills/<skill-name>
```

Want it in every project? Install into `~/.claude/skills/...`. No `npx`? Clone the repo and copy the skill folder into `.claude/skills/`.

<details>
<summary>With Laravel Boost</summary>

```bash
php artisan boost:add-skill ArtemProshkovskiy/laravel-maintenance-skills
```
Boost lists both skills and lets you pick which to add. They're also in the [Laravel Skills directory](https://skills.laravel.cloud/).
</details>

<details>
<summary>Want every skill at once (plugin marketplace)</summary>

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```
</details>

---

## Using it

Open Claude Code inside your Laravel project and ask in plain language — the right skill activates itself:

- *"audit my composer dependencies"* / *"what should I update?"*
- *"are any of my packages abandoned?"* / *"is it safe to bump this package?"*
- *"review my authorization"* / *"find IDOR in this app"*
- *"which routes have no auth?"* / *"review authorization on this PR's new routes"*

Nothing to configure.

---

## Repository layout

```
laravel-maintenance-skills/
├── .claude-plugin/
│   ├── marketplace.json             # registers the plugin (the library)
│   └── plugin.json                  # the Claude Code plugin (root)
├── docs/                            # the catalog website (GitHub Pages)
└── skills/                          # discovered by Claude Code AND skills.laravel.cloud
    └── <skill-name>/                # one self-contained, advise-only skill
        ├── SKILL.md                 #   the skill itself (required; YAML frontmatter)
        ├── README.md                #   the skill's presentation
        ├── USAGE.md                 #   how to run it
        ├── examples/report.md       #   reference output
        ├── references/              #   on-demand material
        └── tests/                   #   acceptance fixture
```

The single root `skills/` directory serves both ecosystems: Claude Code loads it as
the plugin's skills, and the [Laravel Skills directory](https://skills.laravel.cloud/)
imports it via `php artisan boost:add-skill ArtemProshkovskiy/laravel-maintenance-skills`.
Each `SKILL.md` carries the `name`, `description`, `compatible_agents`, and `tags`
frontmatter the directory requires.

## Adding a skill

1. Create `skills/<name>/` with a `SKILL.md` (and ideally `README.md` + `USAGE.md` + `examples/`). Keep it advise-only.
2. Add a block for it under **The skills** above.
3. Surface it on the site: a card in `docs/index.html`, a page `docs/skills/<name>.html`, and a `<url>` in `docs/sitemap.xml`.

## License

[MIT](LICENSE) © Artem Proshkovskyi
