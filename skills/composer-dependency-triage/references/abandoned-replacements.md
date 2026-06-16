# Abandoned → Maintained Replacements (Laravel-focused)

A curated map of commonly-abandoned PHP/Laravel packages to their maintained,
community-standard successors. **The skill consults this when `composer outdated`
flags a package abandoned and Composer prints "No replacement was suggested."**

> ⚠️ **All entries are judgment, not facts.** Always tell the user to **verify on
> Packagist** that the replacement is current and actively maintained before
> adopting. Ecosystems move; this file is a starting point, not a guarantee.
> Confirm the live `abandoned` flag and latest release date from real tool output.

## How to use this file

1. Take the abandoned package name from `composer outdated --direct --format=json`.
2. Look it up below. Lead with Packagist's own suggested replacement if it gave one.
3. Report: **replacement package**, **migration shape** (drop-in vs. API change),
   and a **"verify before adopting"** note.
4. If a package is **not** in this table, recommend searching Packagist for the
   most-installed actively-maintained alternative and say it's unverified.

## Map

| Abandoned package | Recommended replacement | Migration shape | Notes |
|-------------------|-------------------------|-----------------|-------|
| `fzaninotto/faker` | `fakerphp/faker` | Near drop-in | Community fork after the original was archived. Same namespace `Faker\`; update the require, code usually unchanged. The canonical example of "no replacement suggested" by Composer. |
| `swiftmailer/swiftmailer` | `symfony/mailer` | API change | Swiftmailer EOL. Laravel 9+ already uses Symfony Mailer; relevant for legacy apps/direct usage. |
| `guzzle/guzzle` (legacy `guzzle/guzzle`) | `guzzlehttp/guzzle` | API change | Old pre-4.x package; modern Guzzle lives under `guzzlehttp/`. |
| `doctrine/cache` | `symfony/cache` or PSR-6/16 cache | API change | `doctrine/cache` deprecated; migrate to a PSR cache. |
| `kylekatarnls/update-helper` | (remove) | Removal | Transitively pulled by old Carbon; usually disappears once Carbon is current. Check with `composer why`. |
| `laravelcollective/html` | `spatie/laravel-html` or framework forms | API change | LaravelCollective HTML is community-archived; Spatie's package is the common successor. Verify against your Blade usage. |
| `barryvdh/laravel-cors` | `fruitcake/php-cors` / framework CORS | API change | Laravel ships first-party CORS handling on modern versions; the old package is abandoned. |
| `predis/predis` (when unmaintained periods occur) | `phpredis` (ext) or current `predis/predis` | Config change | Predis maintenance has lapsed at times — verify current status before switching; `phpredis` is the C extension alternative. |

## When there is no good replacement

Some abandoned packages have **no** maintained successor (niche or one-off
utilities). In that case, do **not** invent one. Advise the user to either:

- vendor/inline the small piece of functionality they actually use, or
- evaluate whether the dependency is still needed at all, or
- search Packagist and assess candidates by install count, recent releases, and
  open-issue health — and clearly mark the choice as unverified.

## Extending this file

Add a row when you encounter a recurring abandoned package. Keep entries
Laravel-relevant, cite the migration shape, and never assert a replacement as
"safe" without the verify-before-adopting caveat.
