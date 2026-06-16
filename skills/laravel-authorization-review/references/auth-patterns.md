# Laravel Authorization Patterns — Recognize a Real Check

The single biggest source of **false positives** in an authorization review is not
recognizing a legitimate check. Laravel offers many equally-valid ways to authorize a
request; a missing `$this->authorize()` is **not** a finding if one of the others
covers the route. **Before flagging any layer as missing, confirm none of these
patterns satisfies it.**

> All of these are *hard facts you can cite* (a middleware string, a method call, a
> file). Use them to clear a route, not to assert one — absence of all of them on an
> owned-data route is what makes a finding.

---

## Layer 1 — Authentication (is there a logged-in actor?)

Present when the route's `middleware` (from `route:list`) includes any of:

| Middleware | Meaning |
|---|---|
| `auth` | default web guard (session) |
| `auth:sanctum` | SPA / API token auth (most modern APIs) |
| `auth:api`, `auth:<guard>` | a specific guard |
| `auth.basic` | HTTP basic |
| custom guard middleware | project-defined; read the class to confirm it authenticates |

Notes:
- Route-group middleware **is already merged** into each route's list in `route:list`
  — you don't need to chase the group definition.
- `web` / `api` groups alone are **not** authentication — they add session/CSRF or
  throttling, not a login requirement. Don't credit `web` as auth.
- `verified` (email-verified) and `2fa` are *additional* gates, not a substitute for `auth`.

---

## Layer 2 — Authorization (is THIS actor allowed THIS action?)

Any **one** of these satisfies authorization for the route:

### a) Route-level `can:` middleware
```php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('can:update,post');
```
The `can:` middleware resolves the policy ability against the route-model-bound
`{post}` — this **also covers object scoping** (Layer 3) for that model.

> **⚠️ How it actually renders in `route:list --json` (don't match on the literal `can:`).**
> The JSON inventory expands middleware **aliases to their resolved class strings**, so
> `can:update,post` appears as
> `Illuminate\Auth\Middleware\Authorize:update,post`, **not** `can:update,post`. If you
> grep the middleware array for the literal `can:`, you will miss it and false-flag a
> properly-authorized route as "no authz." Match the ability form regardless of skin:
> a middleware entry of `can:<ability>[,<param>]` **or**
> `Illuminate\Auth\Middleware\Authorize:<ability>[,<param>]` is the same Layer-2 check.
> (The human-readable `php artisan route:list` table may still print the `can:` alias —
> but step 1 of the skill uses `--json`, where it's the FQCN.) Likewise `auth:sanctum`
> renders as `Illuminate\Auth\Middleware\Authenticate:sanctum` — **but the `:guard` suffix
> is frequently ABSENT.** A bare `auth` (or a group `->middleware('auth')`) renders as just
> `Illuminate\Auth\Middleware\Authenticate` with **no colon**, the default guard being implied
> by `config/auth.php`. **Match `Authenticate` with OR without a `:guard` suffix as Layer-1
> auth** — keying on `Authenticate:` (with colon) will mis-classify every default-guard route
> as unauthenticated (a mass false positive on real apps). Resolve the actual guard from the
> `defaults.guard` in `config/auth.php` when you need it. Custom alias middleware
> (e.g. `role:admin`) renders as **its own class** (`App\Http\Middleware\EnsureRole:admin`)
> — see §1 and §2f: read that class to confirm it truly authenticates/authorizes.

### b) `Gate::authorize()` / `$this->authorize()` in the controller method
```php
public function update(Request $request, Post $post)
{
    Gate::authorize('update', $post);         // framework-default form — throws 403 if denied; also Layer 3
    // or, when the controller uses the AuthorizesRequests trait:
    $this->authorize('update', $post);
}
```
**Version note (important to avoid both false positives and false negatives):** since
Laravel 11 the base `App\Http\Controllers\Controller` is an empty abstract class and
**does not** include the `AuthorizesRequests` trait, so `$this->authorize()` /
`$this->authorizeForUser()` are available **only if the controller (or its base) adds
`use AuthorizesRequests;`**. The framework-default equivalent that always works is
`Gate::authorize('update', $post)`. On Laravel ≤ 10 the trait was wired into the base
controller, so `$this->authorize()` is the common form there. Treat both as valid Layer-2
checks; if you see `$this->authorize()` with no trait imported in an 11+ app, that's a
real bug (fatal error), not authorization.

Equivalents: `$request->user()->can('update', $post)` guarded with an explicit
`abort_if`/`abort_unless`, `Gate::forUser($user)->authorize(...)`.

### c) `authorizeResource()` in the controller constructor
```php
public function __construct()
{
    $this->authorizeResource(Post::class, 'post');   // maps resource methods → policy abilities
}
```
This wires every resourceful method (`index`/`show`/`store`/`update`/`destroy`) to the
matching `PostPolicy` ability automatically. **A controller with this in the
constructor is authorized on all its resource methods** — do not flag them for a
missing inline `authorize()`. (Confirm the methods are the standard resource names; a
custom method is *not* auto-covered.) `authorizeResource()` is also a method of the
`AuthorizesRequests` trait, so the same version caveat as (b) applies — in Laravel 11+
the controller (or its base) must `use AuthorizesRequests;` for it to exist. The
equivalent without the trait is the route-level `can:` middleware (a) or
`Route::resource(...)->middleware(...)`.

> **Two `authorizeResource` failures that pass review unless you check for them:**
>
> 1. **The binding name must match the route parameter.** `authorizeResource(Order::class, 'order')`
>    hands the bound model to the policy via the route parameter named `order`. But
>    `Route::apiResource('catalog-orders', …)` derives the parameter from the URI slug —
>    so it's `{catalog_order}`, not `{order}` (confirm in `route:list`). When the names
>    don't match, the policy is called **without** the model instance, and per-object
>    authorization silently weakens. Verify the second argument equals the actual route
>    parameter; the fix is `authorizeResource(Order::class, 'catalog_order')`.
> 2. **The mapped ability must exist.** `index → viewAny`, `show → view`, `store → create`,
>    `update → update`, `destroy → delete`. If the policy is **missing** the mapped method
>    (e.g. no `viewAny` for `index`), Laravel **denies by default** (unless a `before()` or a
>    Gate allows it) — so the visible symptom is a 403/availability bug, not a leak. But the
>    controller's query may still be unscoped (`Model::all()`), which becomes a leak the moment
>    someone adds a permissive `viewAny`/`before`. Flag it "verify ability exists" **and** check
>    the index query is scoped to the actor regardless.

### d) FormRequest with a real `authorize()`
```php
class UpdatePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('post'));   // real check
    }
}
```
Valid **only if `authorize()` contains a real check**. `return true;` does **not**
authorize — see the trap below. The controller method must actually type-hint this
FormRequest for it to apply.

### e) Gate checks
`Gate::allows('x')` / `Gate::denies('x')` / `Gate::any([...])` guarded with an
`abort`/early return. Gate abilities are defined in a service provider
(`Gate::define(...)`) or via policies — a definition you may need to read to judge intent.

### f) Package-based permission middleware
Common and legitimate — treat as valid authorization:
- **spatie/laravel-permission**: `role:admin`, `permission:edit posts`,
  `role_or_permission:...` middleware.
- Other RBAC packages with their own middleware. Read the package's convention; don't
  flag a `role:`/`permission:` middleware as "no authz".

---

## Layer 3 — Object scoping (the IDOR core: the RIGHT record for THIS actor)

Authorization (Layer 2) often *also* scopes the object (b, c, and `can:` all bind to a
specific model). Beyond those, object scoping is satisfied when the record is fetched
**through the actor**:

```php
$post = $request->user()->posts()->findOrFail($id);     // can only ever find own posts
auth()->user()->orders()->where('id', $id)->firstOrFail();
```
or via **scoped route binding**:
```php
Route::scopeBindings();                          // child must belong to parent
Route::resource('users.posts', PostController::class)->scoped();
// {post} is resolved through $user->posts() — cannot reach another user's post
```
or a **global scope / tenancy** that constrains every query on the model
(e.g. a `TenantScope`, a `BelongsToTeam` trait, stancl/tenancy). If the model has a
global scope enforcing ownership, an apparently-unscoped `Model::find($id)` may be safe
— note it and verify.

> **⚠️ Scoping often hides one or two layers below the controller — follow the call
> before you flag.** In a mature app the controller line is frequently a clean-looking
> `repository->getOne($id)` or `repository->getByAlbum($album, $user)` with **no visible
> `authorize()`** — textbook IDOR bait. The actual ownership filter can live in:
> - a **Repository** method (`AlbumRepository::getOne` → a query builder call), or
> - a **custom Eloquent Builder** (`AlbumBuilder::accessible()` doing `whereBelongsTo($user)`
>   / `where('user_id', $user->id)`), reached only via `Model::query()` returning that builder, or
> - a **`$user` argument threaded into the repository** (`getByArtist($artist, $user)`) that
>   scopes the inner query.
>
> A bare `findOrFail($id)` that goes through such a layer **404s for a foreign row** and is
> *safe*. Before reporting "unscoped find = IDOR", **open the repository/builder it calls**
> and confirm whether ownership is applied there. Only assert the finding if the scope is
> absent at *every* layer in the chain. This is the single biggest false-positive trap on
> real codebases — the toy examples above put scoping at the controller line; real apps don't.

**NOT object scoping (the IDOR trap):**
```php
public function update(Post $post)   // implicit binding resolves ANY post by id
{
    $post->update($request->validated());   // no authorize(), no owner scope → IDOR
}
$post = Post::findOrFail($request->id);      // same: resolves any id
```

### Ownership signals (to judge "should this be scoped?")
A model is **owned** (so an unscoped fetch is a finding) when you can cite any of:
- a `user_id` / `team_id` / `tenant_id` / `owner_id` column (check the migration/`$fillable`),
- a `belongsTo(User::class)` / `belongsTo(Team::class)` relationship,
- the User model has a `hasMany` to it (`$user->posts()`),
- a tenancy global scope.

No ownership signal → the finding is **Medium/Low "verify"**, not a confirmed IDOR.

---

## Layer 4 — Policy exists

A policy is present when **any** of these holds (check them in this order):
- **Auto-discovered**: `App\Policies\{Model}Policy` exists for `App\Models\{Model}`.
  Laravel resolves policies by convention — it looks in a `Policies` directory at or
  above the model's directory (e.g. `app/Models/Policies` then `app/Policies`) for a
  class named `{Model}Policy`. This is the default and needs **no** registration.
- **`#[UsePolicy]` attribute** on the model (Laravel 11+): `#[UsePolicy(OrderPolicy::class)]`
  above the model class binds the policy explicitly — credit it.
- **Manually registered** with `Gate::policy(Model::class, ModelPolicy::class)`.
  - **Laravel 11+**: this lives in `AppServiceProvider::boot()` — there is **no**
    `app/Providers/AuthServiceProvider.php` in the default skeleton anymore. Don't
    expect a `$policies` array; grep for `Gate::policy(` / `#[UsePolicy` instead.
  - **Laravel ≤ 10**: registered in the `AuthServiceProvider::$policies` array (or
    `Gate::policy(...)`). Older apps upgraded in place may still ship an
    `AuthServiceProvider`, so check for it too.
- **Custom discovery**: `Gate::guessPolicyNamesUsing(...)` in a provider overrides the
  naming convention — if present, the auto-discovery path above may differ; read it.

A managed model (created/updated/deleted by a controller) with **no policy and no
inline gate** is a 🔵 hardening finding — the app relies on scattered ad-hoc checks
instead of one auditable policy. Not an exposure by itself, but it's why exposures slip in.

---

## The high-signal trap: `authorize()` returning `true`

```php
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;       // authorization explicitly disabled
    }
}
```
`return true;` is the FormRequest default and is **correct** when authorization lives
elsewhere (a `can:` middleware, `authorizeResource`, an inline `authorize()`) **or**
the route is genuinely public. It is a **🔴 finding only** when it's the *sole* gate on
a state-changing/owned-data route — then authorization has been turned off. Always
check the other layers before escalating; phrase a borderline case as "verify intent".

---

## Extending this file

Add a pattern when you meet a legitimate authorization mechanism this skill might
otherwise flag (a new RBAC package's middleware, a custom guard convention, a tenancy
library). The goal is **recall of valid checks** — every pattern added here is one
fewer false positive.
