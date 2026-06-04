# Laravel & PHP Compatibility Notes

Reference for **step 5 of the skill** — checking that a recommended major bump
actually fits the project's current Laravel and PHP versions before suggesting
it. The goal is to avoid recommending a bump that is really **blocked on a
framework/PHP upgrade**, and to route those findings to the upgrade path instead.

> ⚠️ Treat the version pairings below as **orientation, not ground truth**. Always
> confirm the candidate package's real requirements from its own
> `composer.json` / `composer outdated` output. Laravel's support windows shift;
> verify against the official Laravel release/support docs before asserting.

## How to use this file

1. From the project: read the `php` constraint and `laravel/framework` (or
   `illuminate/*`) version out of `composer.json` / `composer.lock` (skill step 1).
2. For each candidate **major** bump, read that version's `require` block (PHP +
   any `illuminate/*` / `laravel/framework` constraint).
3. Compare:
   - Candidate fits current PHP **and** Laravel → eligible (still rank by risk).
   - Candidate needs a **newer PHP or Laravel** → mark **🔒 blocked-on-upgrade**,
     do not recommend the bump; hand it to a Laravel upgrade pass.

## Laravel ↔ PHP baseline (verify against official docs)

| Laravel | Minimum PHP | Notes |
|---------|-------------|-------|
| 12.x | PHP 8.2+ | Current generation at time of writing — confirm live. |
| 11.x | PHP 8.2+ | Dropped older PHP versions. |
| 10.x | PHP 8.1+ | |
| 9.x | PHP 8.0+ | First to use Symfony Mailer instead of Swiftmailer. |
| 8.x and older | PHP 7.3+ | Legacy; many ecosystem packages have moved on. |

PHP constraints are the most common hidden blocker: a package's new major often
requires a PHP version the project hasn't moved to yet.

## First-party / framework-coupled packages

These track the Laravel major and should be bumped **as part of a framework
upgrade**, not in isolation. A new major here almost always means "upgrade Laravel
first":

- `laravel/framework`
- `laravel/sanctum`, `laravel/passport`, `laravel/horizon`, `laravel/telescope`,
  `laravel/scout`, `laravel/cashier`, `laravel/socialite`, `laravel/octane`
- `livewire/livewire` (loosely coupled — check its supported Laravel range)
- Major testing tools that move with the stack: `phpunit/phpunit`, `pestphp/pest`,
  `nunomaduro/collision`, `orchestra/testbench` (testbench tracks the framework major directly)

When you see a major available for any of these, default to **🔒 blocked-on-upgrade
/ Defer** and note that it belongs in a coordinated Laravel upgrade.

## Signals that a bump is blocked on an upgrade

- Candidate `require` lists a PHP version above the project's.
- Candidate `require` lists `illuminate/*` or `laravel/framework` above the
  project's current major.
- The package's UPGRADE guide says "requires Laravel N+".
- `composer update --dry-run <pkg>` reports an unresolvable dependency conflict.

In all of these: **stop recommending the bump**, mark it blocked-on-upgrade, and
point the user at a future upgrade skill / Laravel upgrade guide.
