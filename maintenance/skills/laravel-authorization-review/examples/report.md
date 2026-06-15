# Laravel Authorization Review — Example Report

> Reference output for an illustrative **Laravel 11 / Sanctum API + web** project.
> Route names, controllers, and line numbers are **fictional**, used only to show the
> report shape and the kind of judgment to apply. A real run derives every route from
> `php artisan route:list --json` and cites a real controller `file:line` — and never
> invents a route, policy, or method.

---

## Summary

Reviewed **18 application routes** (skipped 6 framework/vendor routes). Project:
`laravel/framework` 11.x, auth via `auth:sanctum` (API) + session (web).

| Class | Count |
|-------|-------|
| 🔴 Exposed endpoint | 2 |
| 🔴 Authorization disabled | 1 |
| 🟡 Likely IDOR | 3 |
| 🟡 Unscoped query / data leak | 2 |
| 🔵 Hardening | 2 |
| ✅ Covered | 8 |

**Headline:** Two high-confidence holes to fix today — `DELETE /api/invoices/{invoice}`
has no authorization at all, and `UpdateProfileRequest::authorize()` returns `true` on
a route nothing else protects. Three likely IDORs rely on route-model binding without
an ownership check — verify the model is user-owned and add `authorize()`.

> **This is not a clean bill of health.** A finding-free route only means "no broken
> authorization in the four layers I checked" — it does not cover business-logic
> authorization, Livewire/Filament actions, or indirect references (see Boundaries in
> SKILL.md). Not a pentest.

---

## Coverage map

Every reviewed route, by layer. ✓ present · ✗ missing · n/a not applicable.
`auth` = authenticated · `authz` = action authorized · `scoped` = object scoped to
actor · `policy` = a policy exists for the model.

| Route | Action | auth | authz | scoped | policy | Verdict |
|-------|--------|:----:|:-----:|:------:|:------:|---------|
| `GET /api/invoices` | `InvoiceController@index` | ✓ | ✓ | ✗ | ✓ | 🟡 unscoped list |
| `GET /api/invoices/{invoice}` | `InvoiceController@show` | ✓ | ✗ | ✗ | ✓ | 🟡 likely IDOR |
| `DELETE /api/invoices/{invoice}` | `InvoiceController@destroy` | ✓ | ✗ | ✗ | ✓ | 🔴 exposed |
| `PUT /api/profile` | `ProfileController@update` | ✓ | ✗ | n/a | n/a | 🔴 authz disabled |
| `GET /api/orders/{order}` | `OrderController@show` | ✓ | ✓ | ✓ | ✓ | ✅ covered |
| `POST /api/teams/{team}/members` | `MemberController@store` | ✓ | ✓ (`can:`) | ✓ | ✓ | ✅ covered |
| `GET /admin/users` | `Admin\UserController@index` | ✓ | ✓ (`role:admin`) | n/a | ✓ | ✅ covered (admin) |
| `GET /api/users/{user}/exports` | `ExportController@show` | ✓ | ✓ | ✗ | ✓ | 🟡 likely IDOR |
| `POST /login` | `AuthController@store` | n/a | n/a | n/a | n/a | ✅ public-by-design |
| `POST /webhooks/stripe` | `StripeWebhook@handle` | n/a (sig) | n/a | n/a | n/a | ✅ signature-verified |
| … 8 more covered routes … | | ✓ | ✓ | ✓/n/a | ✓ | ✅ |

---

## 🔴 Verify & fix now (High confidence)

### 1. `DELETE /api/invoices/{invoice}` — no authorization (exposed)

```
route:list → DELETE api/invoices/{invoice} → App\Http\Controllers\InvoiceController@destroy
middleware: api, auth:sanctum            ← authenticated, but NOTHING authorizes the action
```
```php
// app/Http/Controllers/InvoiceController.php:88
public function destroy(Invoice $invoice)          // implicit binding resolves ANY invoice id
{
    $invoice->delete();                            // no authorize(), no owner scope
    return response()->noContent();
}
```
**Why it's a hole:** any authenticated user can delete *any* invoice by guessing/iterating
the id — route-model binding resolves the record but does not authorize it. `Invoice`
is user-owned (`invoices.user_id`, `User::invoices()` — confirmed in
`app/Models/Invoice.php:14`), so this is a confirmed object-level authorization break.

**Fix sketch** (you apply it):
```php
public function destroy(Invoice $invoice)
{
    $this->authorize('delete', $invoice);          // add this — needs InvoicePolicy::delete()
    $invoice->delete();
    return response()->noContent();
}
```
`InvoicePolicy` exists (`app/Policies/InvoicePolicy.php`) but has no `delete` method —
add `return $user->id === $invoice->user_id;`.

### 2. `PUT /api/profile` — authorization explicitly disabled

```
route:list → PUT api/profile → App\Http\Controllers\ProfileController@update
form request: App\Http\Requests\UpdateProfileRequest
```
```php
// app/Http/Requests/UpdateProfileRequest.php:12
public function authorize(): bool
{
    return true;                                   // authorization turned off
}
```
The controller relies solely on this FormRequest, and nothing else (no `can:`, no
inline `authorize()`) gates the route. On a profile-update endpoint that accepts a
`user_id` in its payload, `return true` lets a user write to another account. **High
confidence** because it's the sole gate and the request is state-changing.

**Fix sketch:** scope to the authenticated user (a profile update should target
`$request->user()`, not an arbitrary id), or return a real check:
`return $this->user()->id === (int) $this->input('user_id');`

---

## 🟡 Review — needs your judgment (Medium confidence)

| # | Route | Finding | Assumption to confirm |
|---|-------|---------|------------------------|
| 3 | `GET /api/invoices/{invoice}` | `show(Invoice $invoice)` returns the bound model with no `authorize('view', …)` | `Invoice` is user-owned → add `authorize`/scope. If invoices are intentionally shared within a team, scope to the team instead. |
| 4 | `GET /api/users/{user}/exports` | `ExportController@show` loads the export by id, not through `$user->exports()` | the `{user}` segment isn't enforced against the bound export → scope the binding or check ownership. |
| 5 | `GET /api/invoices` (index) | `Invoice::paginate()` returns **all** invoices, not `$request->user()->invoices()` | unless an admin-only route, scope the list to the actor. |

```php
// Finding #3 — app/Http/Controllers/InvoiceController.php:61
public function show(Invoice $invoice)             // no authorize('view', $invoice)
{
    return new InvoiceResource($invoice);
}
// Fix: $this->authorize('view', $invoice);  OR  $request->user()->invoices()->findOrFail($invoice->id)
```

> These are **Medium** because each depends on whether the model is truly user-owned vs.
> team-shared. The fix differs accordingly — that's a call for you, not the tool.

### Data-leak (API Resource) — Medium/Low

```php
// app/Http/Resources/UserResource.php:18  (returned by GET /api/teams/{team}/members)
return [
    'id'    => $this->id,
    'name'  => $this->name,
    'email' => $this->email,          // ← exposes every team member's email
    'is_admin' => $this->is_admin,    // ← exposes internal role flag
];
```
**Verify:** is exposing `email`/`is_admin` to all team members intended? If not, guard
with `'email' => $this->when($request->user()->can('view', $this->resource), $this->email)`.

---

## 🔵 Hardening (Low — defense-in-depth)

- **`Comment` has no policy.** `CommentController` does inline `if ($comment->user_id !== auth()->id()) abort(403)` in three methods. It works, but scattered checks rot — extract a `CommentPolicy` so authorization is auditable in one place.
- **Mass-assignment of ownership key.** `Post::create($request->all())` with `user_id` in `$fillable` (`app/Models/Post.php:9`) lets a caller set `user_id`. Even though the row is otherwise scoped, drop `user_id` from `$fillable` or set it explicitly from `auth()->id()`.

---

## ✅ Covered (summarized from the map)

8 routes satisfy all applicable layers — e.g. `GET /api/orders/{order}` authorizes via
`OrderPolicy@view` and scopes through `$request->user()->orders()`; team-member creation
uses `can:create,team` middleware (object-scoped); admin routes are gated by
`role:admin`. `POST /login` and the Stripe webhook are public-by-design (the webhook
verifies its signature). See the coverage map above; not repeated here.

---

## Notes & caveats

- **Hard facts** — the route inventory, middleware, and which files/methods/policies
  exist — come from `php artisan route:list --json` and the filesystem. **Judgment** —
  whether a model is "owned", whether a field is sensitive, the fix sketches — is
  AI-derived and marked with the assumption to confirm.
- **Confidence travels with every finding.** 🔴 = structural (missing layer, cited
  route + line). 🟡 = depends on an ownership/sensitivity assumption stated inline.
- **A finding-free route is not "secure"** — only "no broken authorization in the four
  checked layers". Out of scope: business-logic authz, Livewire/Filament actions,
  indirect references, infra/gateway authz. **Not a pentest.**
- **Advice only** — no files were modified. Every fix sketch is for you to apply and
  review, on a branch, with your authorization tests run after.
