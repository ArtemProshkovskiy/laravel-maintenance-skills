# Acceptance test — `composer-dependency-triage`

A reproducible, real-data acceptance test for the skill. It has two parts:

1. **Fixture prompt** — feed it to Claude Code inside a *separate* throwaway
   Laravel sandbox. It builds a deliberately messy `composer.json` + a real
   `composer.lock` that exercises every classification branch of the skill.
2. **Oracle** — run the skill against that sandbox and grade its report row by row.

> The skill reads **real** `composer outdated` / `composer audit` output and never
> invents facts, so the fixture is built from **real packages with real known
> states** (phpseclib 2.0.30, `fzaninotto/faker`, …). Requires PHP 7.4/8.0,
> Composer **2.4+**, and network.

---

## 1. Fixture prompt (paste into the sandbox project)

```text
Ты внутри пустого тестового Laravel-проекта (sandbox). Задача — НЕ запускать
никаких triage/audit-скиллов, а подготовить ФИКСТУРУ: умышленно «грязный» набор
зависимостей, покрывающий все ветки классификации скилла composer-dependency-triage.
Это реальные пакеты с реальными известными состояниями — ничего выдумывать не нужно.

1. Создай (или перезапиши) composer.json с такими ПРЯМЫМИ зависимостями:

require:
  "php":                      "^7.4|^8.0"
  "laravel/framework":        "^8.0"      // мажор 9–12, framework-coupled → 🛑 Defer/blocked
  "laravel/sanctum":          "^2.0"      // framework-coupled мажор → 🛑 Defer
  "guzzlehttp/guzzle":        "6.5.5"     // 🔴 уязвим, фикс drop-in в 6.x; ТОЧНЫЙ ПИН → require, не update
  "phpseclib/phpseclib":      "2.0.30"    // 🔴 уязвим, drop-in в 2.x (НЕ 3.x); ТОЧНЫЙ ПИН
  "swiftmailer/swiftmailer":  "^6.2"      // 🟠 abandoned, framework-pulled → уйдёт с Laravel 9
  "fzaninotto/faker":         "^1.9"      // 🟠 abandoned + "No replacement was suggested" → fakerphp/faker
  "laravelcollective/html":   "^6.0"      // 🟠 abandoned → Composer сам предложит spatie/laravel-html
  "ramsey/uuid":              "4.2.0"     // 🟢 безопасный patch в 4.x
  + один пакет на ТЕКУЩЕЙ последней стабильной версии (проверь Packagist) → ✅ Healthy
    (напр. vlucas/phpdotenv)

require-dev:
  "phpunit/phpunit":          "^9.0"      // тест-мажор под PHP 8.1+ скрыт platform-пином → 🧰 Dev
  "mockery/mockery":          "^1.4"      // 🟢 безопасный bump в dev

2. Добавь "minimum-stability":"stable", "prefer-stable":true и
   "config": { "platform": { "php": "7.4.33" } }  // намеренно скрывает мажоры под PHP 8.1+

3. Запусти: composer update --no-interaction --no-plugins  → реальный composer.lock.
   Если резолвер не сходится — минимально подгоняй версии, СОХРАНЯЯ намерение каждой
   строки. НЕ добавляй fakerphp/faker (перебьёт кейс abandoned fzaninotto).

4. Меняй только composer.json и сгенерированный lock. Никаких правок чужого кода.

5. В конце выведи чек-лист «пакет → ожидаемый класс» и подтверди, что:
   - composer.lock сгенерирован;
   - composer audit --format=json --no-interaction --no-plugins реально возвращает advisories.
Сам triage-отчёт не строй — это сделает скилл на следующем шаге.
```

### Optional enhancement — cover the pure-transitive-vuln branch

The fixture above exercises `composer why` via swiftmailer (direct **and**
framework-pulled), but not a *pure* transitive vulnerability (step 2: "act on the
direct parent, never bump the transitive package directly"). To cover it, add a
direct package pinned to an **old** version whose **transitive** dependency carries
a CVE the parent must fix — then **verify with `composer audit` that the advisory
lands on the transitive package, not the direct one** before relying on it (real
advisory data drifts; don't assume). If audit doesn't surface a transitive-only
advisory, pick a different parent rather than faking it.

---

## 2. Oracle — grade the skill's report against this

Run the skill in the sandbox ("audit my composer dependencies") and check each row.
Every line maps to a step in `SKILL.md`.

| Expect in the report | Verifies | Step |
|---|---|---|
| **phpseclib 2.0.30** → fix **inside 2.x at 2.0.54**, explicit "don't jump to 3.x" | reads `<=2.0.53` ⇒ `2.0.54`; prefers drop-in (the headline trap) | 4 |
| **guzzle 6.5.5** → fix **6.5.8** inside 6.x, 7.x deferred | per-major fix floor, no needless major | 4 |
| **guzzle & phpseclib are EXACT-pinned** → remediation is `composer require "<pkg>:<floor>"`, **not** `composer update` (which is a no-op) | exact-pin awareness; constraint change first | 4/8 |
| Pin remediation **preserves pinning style** (exact→exact at floor; loosening only offered with tradeoff) | respects intentional pins | 4 |
| **laravel/framework** → 3 of 4 CVE fixed by 8.83.x; CVE-2025-27515 needs Laravel 10+ → 🛑 Defer | per-CVE floors; routes the unfixable one to upgrade | 4/6 |
| Floors phrased as "fixed in ≥X", not the raw `<Y` affected range | clear remediation phrasing | 4 |
| **fzaninotto/faker** → **fakerphp/faker**, "verify before adopting" | "No replacement was suggested" gap-fill (core value) | 5 |
| **laravelcollective/html → spatie**, **swiftmailer → symfony/mailer** | leads with Composer's own suggestion | 5 |
| **swiftmailer** routed to **Laravel 9 upgrade** (framework-pulled via `why`), not a manual swap | transitive/framework-coupled judgment | 2/6 |
| **laravel/framework / sanctum** → 🔒 blocked-on-upgrade (PHP/Laravel can't satisfy major) | compatibility gate | 6 |
| **platform pin `php 7.4.33`** noted as the reason PHP-8.1+ majors are hidden | robustness/transparency | robustness |
| Every direct dep listed (sanctum, phpdotenv not dropped); **no package duplicated across sections** | "never drop a direct dep"; no double-listing | 3/9 |
| Pure-transitive advisory (if enhancement used) → explain chain, bump the **direct parent** | never update a transitive dep directly | 2 |
| Final: three lanes **✅ Do now / ⚠️ Do carefully / 🛑 Defer**; nothing written to disk without asking | plan shape + guardrails | 8/10 |

A pass = every hard fact traces to real tool output, both exact-pinned CVEs are
remediated via `composer require` (not a no-op `update`), and no needless major is
pushed for a security fix.
