# Using `composer-dependency-triage`

A plain, self-contained guide: what the skill does, how to install it in Claude
Code, and how to run it on your Laravel project.

---

## 1. What it does (in one minute)

`composer outdated` and `composer audit` **show you facts**: what is out of date,
what is vulnerable, what is abandoned. They do not tell you **what to do about it,
or in what order**.

This skill is the **decision layer** on top of those commands. It:

- reads the real Composer output — it never invents versions or CVEs;
- sorts every direct dependency into 🔴 vulnerable / 🟠 abandoned / 🟡 major /
  🟢 safe / ✅ healthy;
- **recommends a maintained replacement for abandoned packages** — even when
  Composer just prints *"No replacement was suggested"*;
- for a vulnerability, finds the **safest verified fix**: it reads the advisory
  range literally, picks the lowest fixed version, prefers a drop-in patch within
  the current major, and confirms that version exists before recommending it;
- checks that a major bump actually fits your current PHP / Laravel version;
- emits a ready-to-act plan in three lanes — **✅ Do now / ⚠️ Do carefully /
  🛑 Defer** — with the exact commands.

> **Advise-only.** The skill changes nothing in your project: it does not touch
> `composer.json`, the lockfile, or your code. You run every command yourself.

---

## 2. Requirements

| Requirement | Why |
|---|---|
| A PHP project with a `composer.json` | the entry point; without it the skill stops |
| `composer` on your PATH | the skill runs `composer outdated` / `audit` / `why` |
| Composer **2.4+** | needed for `composer audit` (vulnerability data) |
| A populated `composer.lock` *(recommended)* | exact installed versions |

Tuned for **Laravel**, but works on any Composer-managed PHP project.

---

## 3. Installing in Claude Code

This skill ships in the `laravel-maintenance` plugin. Add the marketplace and
install the plugin once — after that the skill activates on its own based on what
you ask:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```

---

## 4. Running it on your project

Open Claude Code **inside your Laravel project folder** and ask in plain language
— the skill recognizes the request and kicks in:

- "audit my composer dependencies"
- "what should I update?"
- "are any of my packages abandoned?"
- "is it safe to bump this package?"
- "check my dependencies before upgrading Laravel"

Nothing to configure — Claude Code finds `composer.json`, runs the right read-only
commands, and assembles the report.

---

## 5. What you get — the report

A clean Markdown report, straight into the chat:

```markdown
## Summary
Audited 19 direct dependencies — 🔴 3 vulnerable, 🟠 3 abandoned, 🟡 9 major, 🟢 3 safe.

## 🔴 Security — Do now
| Package             | Installed | Advisory                   | Fixed in | Action                                  |
|---------------------|-----------|----------------------------|----------|-----------------------------------------|
| phpseclib/phpseclib | 2.0.30    | 7 advisories (mostly high) | 2.0.54   | drop-in within 2.x — do NOT jump to 3.x |

## 🟠 Abandoned — Do carefully
laravelcollective/html → spatie/laravel-html  (API change — audit your Blade forms)

## Action plan
✅ Do now        — verified drop-in security patches + safe bumps
⚠️ Do carefully  — swap abandoned packages; plan the Laravel upgrade
🛑 Defer         — framework-coupled majors, sequenced into the upgrade
```

### How to read the three lanes

| Lane | What it is | What to do |
|---|---|---|
| **✅ Do now** | vulnerabilities fixed by a verified drop-in + safe small bumps | run the commands, run your tests |
| **⚠️ Do carefully** | abandoned swaps, majors needing edits | read the changelog / upgrade guide first |
| **🛑 Defer** | risky majors, blocked on a Laravel/PHP upgrade | handle in a dedicated pass |

---

## 6. How it keeps your app from breaking

The skill bakes safety into every recommendation:

- it never edits anything — you stay in control of each command;
- it advises previewing with `composer update <pkg> --with-dependencies --dry-run`
  before any real update, so you see exactly what else moves;
- it tells you to work on a branch, apply one lane at a time, commit the lockfile
  between steps, and run your test suite after each — so a bad bump is isolated
  and revertible;
- for a security hole it prefers a verified drop-in patch over a disruptive major
  migration whenever one exists.

---

## 7. Saving the report (optional)

By default the report stays in the chat — nothing is written to disk. After
printing it, the skill **may offer** to save it. If you agree, it writes to
`storage/logs/dependency-triage-<YYYY-MM-DD>.md` (already git-ignored in Laravel).
You can give it a different path. Without your confirmation, no file is created.

---

## 8. Boundaries — what it does NOT do

- ❌ modify `composer.json`, `composer.lock`, or any code;
- ❌ run `composer require/update/remove` — you run those;
- ❌ invent versions or CVEs — only what the real tools returned;
- ❌ tell you to update a transitive dependency directly — it explains the chain
  via `composer why` and points you at the direct parent;
- ❌ recommend a major your PHP/Laravel version can't satisfy — it marks it
  "blocked on upgrade".

---

## 9. Common situations

| Situation | What the skill does |
|---|---|
| No `composer.json` | says it's not a Composer project and stops |
| Several `composer.json` (monorepo) | asks which package to triage |
| Composer < 2.4 (no `audit`) | runs `outdated` only, says vulnerability data is unavailable |
| No network | warns the data may be incomplete and continues from local data |
| Everything fine | says so plainly — "all current, none abandoned, audit clean" |

---

**In short:** install the plugin → open Claude Code in your project → ask "audit
my composer dependencies" → get a *Do now / Do carefully / Defer* plan and run the
suggested commands yourself.
