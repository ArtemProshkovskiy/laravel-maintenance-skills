# Using `composer-dependency-triage`

A short guide: what the skill does, how to install it, and how to run it on your Laravel project.

---

## 1. What it does

`composer outdated` and `composer audit` show you **facts**: what's outdated, what's vulnerable, what's abandoned. They don't tell you **what to do, or in what order**.

This skill is the decision layer on top of them. It:

- reads the real Composer output — never invents a version or a CVE;
- tags every direct dependency: 🔴 vulnerable · 🟠 abandoned · 🟡 big update · 🟢 safe update · ✅ fine;
- **finds a live replacement for dead packages** — even when Composer just says *"No replacement was suggested"*;
- for a vulnerability, finds the **safest fix**: the lowest version that fixes it, drop-in if possible;
- checks that a big update is actually compatible with your PHP / Laravel;
- emits a three-lane plan — **✅ now / ⚠️ carefully / 🛑 defer** — with the exact commands.

> **Advise-only.** The skill changes nothing: it doesn't touch `composer.json`, the lockfile, or your code. You run every command yourself.

---

## 2. Requirements

| Requirement | Why |
|---|---|
| A PHP project with a `composer.json` | the entry point; without it the skill stops |
| `composer` on your PATH | runs `composer outdated` / `audit` / `why` |
| Composer **2.4+** | needed for `composer audit` (vulnerability data) |
| A populated `composer.lock` *(recommended)* | exact installed versions |

Tuned for **Laravel**, but works on any Composer-managed PHP project.

---

## 3. Install

One folder in `.claude/skills/` and it just works:

```bash
npx degit ArtemProshkovskiy/laravel-maintenance-skills/skills/composer-dependency-triage \
  .claude/skills/composer-dependency-triage
```

Want it in every project — install into `~/.claude/skills/...`. Want both skills at once:

```text
/plugin marketplace add ArtemProshkovskiy/laravel-maintenance-skills
/plugin install laravel-maintenance@laravel-maintenance-skills
```

---

## 4. Run it

Open Claude Code in your project folder (the one with `composer.json`) and ask in plain words:

- "audit my composer dependencies"
- "what should I update?"
- "are any of my packages abandoned?"
- "is it safe to bump this package?"
- "check my dependencies before upgrading Laravel"

Nothing to configure — the skill finds `composer.json`, runs the read-only commands, and assembles the report.

---

## 5. What you get — the report

Clean Markdown, straight into the chat:

```markdown
## Summary
Audited 19 direct dependencies — 🔴 3 vulnerable, 🟠 3 abandoned, 🟡 9 major, 🟢 3 safe.

## 🔴 Security — Do now
| Package             | Installed | Advisory      | Fixed in | Action                                  |
|---------------------|-----------|---------------|----------|-----------------------------------------|
| acme/crypto         | 2.0.30    | 4 advisories  | 2.0.54   | drop-in within 2.x — do NOT jump to 3.x |

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
| **✅ Do now** | vulnerabilities with a verified drop-in + small bumps | run the commands, run your tests |
| **⚠️ Do carefully** | abandoned swaps, majors that need code edits | read the changelog / upgrade guide first |
| **🛑 Defer** | risky majors blocked on a Laravel/PHP upgrade | handle in a dedicated pass |

---

## 6. How it keeps your app from breaking

- it never edits anything — you run every command;
- it advises previewing with `composer update <pkg> --with-dependencies --dry-run` first, so you see what else moves;
- it tells you to work on a branch, apply one lane at a time, commit the lockfile between steps, and run your tests — so a bad bump is easy to revert;
- for a security hole it prefers a verified drop-in patch over a disruptive major migration whenever one exists.

---

## 7. Saving the report (optional)

By default the report stays in the chat — nothing is written to disk. After printing it, the skill **may offer** to save it to `storage/logs/dependency-triage-<date>.md` (git-ignored in Laravel). You can give it a different path. Without your confirmation, no file is created.

---

## 8. What it does NOT do

- ❌ modify `composer.json`, `composer.lock`, or any code;
- ❌ run `composer require/update/remove` — you run those;
- ❌ invent versions or CVEs — only what the real tools returned;
- ❌ tell you to update a transitive dependency directly — it explains the chain via `composer why` and points you at the direct parent;
- ❌ recommend a major your PHP/Laravel can't satisfy — it marks it "blocked on upgrade".

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

**In short:** install the skill → open Claude Code in your project → ask "audit my composer dependencies" → get a *now / carefully / defer* plan and run the commands yourself.
