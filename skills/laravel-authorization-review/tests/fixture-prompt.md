# Acceptance test — `laravel-authorization-review`

A reproducible acceptance test for the skill. Two parts:

1. **Fixture prompt** — feed it to Claude Code inside a *separate* throwaway Laravel
   sandbox. It builds routes + controllers + policies + FormRequests + resources that
   deliberately exercise every classification branch (and several false-positive traps
   the skill must NOT flag).
2. **Oracle** — run the skill against the sandbox and grade its report row by row.

> The skill reads the **real** `php artisan route:list --json` and cites real
> `file:line`s — it never invents routes. So the fixture must be a **bootable** Laravel
> app whose `route:list` actually returns these routes. Requires a Laravel install
> (`composer create-project laravel/laravel sandbox`) and PHP on PATH.

---

## 1. Fixture prompt (paste into the sandbox project)

```text
Ты внутри пустого тестового Laravel-проекта (sandbox). Задача — НЕ запускать
авторизационный ревью-скилл, а ПОДГОТОВИТЬ ФИКСТУРУ: набор роутов/контроллеров,
покрывающий все ветки классификации скилла laravel-authorization-review, включая
ловушки ложных срабатываний. Приложение должно БУТИТЬСЯ и `php artisan route:list
--json` должен реально возвращать эти роуты.

1. Модели (app/Models) с явными сигналами владения:
   - Invoice: миграция с invoice.user_id; belongsTo(User); в User добавь invoices().
   - Order:   order.user_id; belongsTo(User); User::orders().
   - Comment: comment.user_id; belongsTo(User).
   - Post:    post.user_id; $fillable ВКЛЮЧАЕТ 'user_id' (ловушка mass-assignment).

2. Policies (app/Policies):
   - OrderPolicy с методом view() → $user->id === $order->user_id.  (зарегистрируй/авто-дискавери)
   - InvoicePolicy СУЩЕСТВУЕТ, но БЕЗ метода delete() (намеренно неполный).
   - Comment — БЕЗ политики вовсе (ловушка hardening).

   ВАЖНО (Laravel 11/12): базовый `App\Http\Controllers\Controller` теперь пустой и НЕ
   содержит trait `AuthorizesRequests`. В любом контроллере, который вызывает
   `$this->authorize(...)` или `authorizeResource(...)`, добавь
   `use Illuminate\Foundation\Auth\Access\AuthorizesRequests;` (в классе) — иначе приложение
   не забутится и фикстура будет невалидной. Как альтернативу можно использовать
   `Gate::authorize(...)` (доступно без trait) — оба варианта валидны для скилла.

3. Контроллеры и роуты (routes/api.php под middleware ['api','auth:sanctum'], кроме public):

   🔴 EXPOSED — DELETE /api/invoices/{invoice} → InvoiceController@destroy:
       public function destroy(Invoice $invoice){ $invoice->delete(); return response()->noContent(); }
       // нет authorize(), нет скоупа; Invoice owned → подтверждённый IDOR

   🔴 AUTHZ DISABLED — PUT /api/profile → ProfileController@update,
       type-hint UpdateProfileRequest, где authorize(){ return true; }, и больше ничего не защищает.

   🟡 LIKELY IDOR — GET /api/invoices/{invoice} → InvoiceController@show:
       return new InvoiceResource($invoice);  // нет authorize('view'), нет скоупа

   🟡 UNSCOPED LIST — GET /api/invoices → InvoiceController@index:
       return InvoiceResource::collection(Invoice::paginate());  // не ->user()->invoices()

   🟡 DATA LEAK — UserResource отдаёт 'email' и 'is_admin' без when(); возвращается из
       GET /api/teams/{team}/members.

   ✅ COVERED (не флагать) — GET /api/orders/{order} → OrderController@show:
       $this->authorize('view',$order); return new OrderResource(
           $request->user()->orders()->findOrFail($order->id));  // authz + scope

   ✅ COVERED via can: (не флагать) — POST /api/teams/{team}/members с
       ->middleware('can:create,team').

   ✅ COVERED via authorizeResource (не флагать) — ResourcefulController с
       authorizeResource(Order::class,'order') в конструкторе; его resource-методы НЕ флагать.
       ВАЖНО, чтобы это был чистый ✅, а не случайная ловушка:
         (а) имя байндинга во втором аргументе ДОЛЖНО совпадать с параметром роута —
             если используешь Route::apiResource('catalog-orders', …), параметр будет
             {catalog_order}, поэтому пиши authorizeResource(Order::class,'catalog_order');
             либо назови ресурс 'orders' (тогда параметр {order} и 'order' совпадает);
         (б) у OrderPolicy ДОЛЖЕН быть метод viewAny() (для index), иначе Laravel по
             умолчанию вернёт 403 и index станет «verify», а не ✅.

   🟡 AUTHZRESOURCE MISWIRED (опциональная ловушка, если хочешь её проверить) —
       второй ResourcefulController, где имя байндинга НЕ совпадает с параметром роута
       и/или у политики нет viewAny → скилл должен пометить это «verify», не как чистый ✅.

   🔵 HARDENING — CommentController@update/destroy/show с инлайн
       if($comment->user_id!==auth()->id()) abort(403); — работает, но нет CommentPolicy.

   🔵 HARDENING — Post::create($request->all()) с user_id в $fillable.

4. Ловушки ЛОЖНЫХ срабатываний (скилл НЕ должен флагать «нет auth»):
   - routes/web.php: POST /login, POST /register, password.* (можешь взять Breeze-роуты
     или просто закладки-контроллеры) — public-by-design.
   - POST /webhooks/stripe → StripeWebhook@handle с middleware проверки подписи
     (заглушка VerifyStripeSignature) — НЕ флагать за отсутствие auth.
   - GET / (лендинг), GET /posts (публичный read-only листинг) — public-by-design.
   - admin: GET /admin/users → middleware ['auth','role:admin'] (можешь заглушить
     role middleware) с Order::all()-подобным глобальным запросом — НЕ флагать как IDOR.

5. Никаких правок чужого кода вне создания этих файлов.

   ⚠️ ЧЕСТНОСТЬ ТЕСТА: НЕ оставляй в сгенерированном коде комментарии, выдающие
   ожидаемый вердикт (никаких `// 🔴 EXPOSED`, `// IDOR`, `// TRAP`, `// public-by-design`
   и т.п.). Скилл обязан выводить находки СТРУКТУРНО из самого кода; подсказки-комментарии
   делают прогон нечестным и маскируют ложные срабатывания. Код должен выглядеть как
   обычное приложение — реалистично, без меток-ответов.

   В конце:
   - запусти `php artisan route:list --json` и подтверди, что роуты выше реально в списке;
   - выведи чек-лист «роут → ожидаемый класс» (это твоя памятка для проверки — ОТДЕЛЬНО
     от кода, не внутри файлов проекта).
   Сам ревью-отчёт НЕ строй — это сделает скилл на следующем шаге.
```

---

## 2. Oracle — grade the skill's report against this

Run the skill in the sandbox ("review my authorization") and check each row. Every line
maps to a step in `SKILL.md`.

| Expect in the report | Verifies | Step |
|---|---|---|
| **DELETE /api/invoices/{invoice}** → 🔴 exposed, High; cites `InvoiceController@destroy` line; notes Invoice is owned (`user_id`/`User::invoices()`) | object-scoping core; implicit-binding-≠-authorization | 3/6 |
| Fix sketch names **`authorize('delete', $invoice)`** and that **InvoicePolicy lacks `delete()`** | reads policy presence + missing method | 2/3 |
| **PUT /api/profile** → 🔴 authz disabled, High; cites `UpdateProfileRequest::authorize(){return true}` as the sole gate | `return true` trap, only when nothing else protects | 2 |
| **GET /api/invoices/{invoice}** → 🟡 likely IDOR, Medium; states the owned-vs-team assumption | confidence on ownership; "verify" phrasing | 3/6 |
| **GET /api/invoices** (index) → 🟡 unscoped list (not `->user()->invoices()`) | unscoped collection = list-level IDOR | 3/4 |
| **UserResource** email/is_admin leak → 🟡/Low, cites resource `file:line` + fields | API Resource output surface | 4 |
| **GET /api/orders/{order}** → ✅ covered, NOT flagged (has authorize + owner scope) | recognizes a real check (no false positive) | 2/3 |
| **POST /api/teams/{team}/members** → ✅ covered via `can:` middleware, NOT flagged | `can:` counts as authz + scope | 2 |
| **ResourcefulController** resource methods → ✅, NOT flagged for missing inline `authorize()` | `authorizeResource()` recognized | 2 |
| If the `authorizeResource` **binding name ≠ route parameter** (e.g. `'order'` vs `{catalog_order}`) → flagged **"verify"**, not a clean ✅ | param-name silent failure | 2 (auth-patterns c) |
| If the mapped policy **ability is missing** (e.g. no `viewAny` for `index`) → noted as default-deny "verify ability exists" + index query still checked for scoping | missing-ability default-deny | 2 (auth-patterns c) |
| **PUT /api/profile** writing `User::find($request->input('user_id'))` → recognized as 🔴 **cross-account write / IDOR-on-write**, High (worse than plain "authz disabled") | actor-selected target from input | 3/6 |
| **CommentController** → 🔵 hardening (no CommentPolicy; inline checks), NOT a 🔴 | inline check works → not exposed, but hardening | 4 |
| **Post::create($request->all())** with `user_id` fillable → 🔵 mass-assignment of ownership key | mass-assignment adjacency | 3 |
| **login/register/password.*** → NOT flagged for missing auth (public-by-design) | false-positive control | 5 |
| **POST /webhooks/stripe** with signature middleware → ✅, NOT flagged | webhook = signature auth | 5 |
| **GET / and GET /posts** (read-only public) → NOT flagged | public read-only carve-out | 5 |
| **GET /admin/users** unscoped query behind `role:admin` → NOT flagged as IDOR | admin-context carve-out | 5 |
| **Coverage map** lists EVERY reviewed route (covered ones too); no route double-listed across lanes | coverage-map discipline | 7 |
| Report states a clean route ≠ "secure"; names Boundaries (Livewire/Filament, business logic out of scope) | honesty / not-a-pentest | confidence |
| Nothing written to disk without asking | guardrails | 8 |

A **pass** = both 🔴s caught with cited lines, all four ✅/non-flag traps left alone (no
false positives), Medium findings phrased as "verify" with the stated assumption, and the
coverage map shows every reviewed route. A single false positive on a public-by-design or
already-authorized route is a **fail** — for a security tool, crying wolf is the worst
outcome.

---

## 3. Fallback-path check (optional)

Temporarily break the boot (rename `.env`) and re-run. Expect: the skill reports that
`php artisan route:list` failed, **falls back to parsing `routes/*.php`**, says the
inventory is best-effort, and **drops confidence one level** — rather than inventing
routes or claiming a clean pass.
