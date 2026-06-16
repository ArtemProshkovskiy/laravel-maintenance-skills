# Public-by-Design Routes — Don't Flag These for "Missing Auth"

A security tool dies from false positives. Many routes are **meant** to be reachable
without authentication; flagging them as "exposed" trains the user to ignore the
report. Use this file as the filter before raising any "missing authentication"
finding.

> **Rule of thumb:** if a route is on this list (by name, URI shape, or purpose), do
> **not** flag it for missing `auth`. If it only *resembles* a public route but you
> can't confirm it, mark it **"assumed public — confirm"** at Low confidence — never
> assert a hole, and never silently drop it from the coverage map.

---

## Authentication & account entry points (public by necessity)

These must work for a logged-out user — that's their entire purpose:

| Purpose | Typical route names / URIs |
|---|---|
| Login | `login`, `POST /login` |
| Registration | `register`, `POST /register` |
| Password reset | `password.request`, `password.email`, `password.reset`, `password.update`, `/forgot-password`, `/reset-password` |
| Email verification (notice/link) | `verification.notice`, `verification.verify` (the link is **signed** — that *is* its auth) |
| OAuth / social callbacks | `/auth/{provider}/callback`, Socialite redirects |
| Logout | `logout` (typically behind `auth`, but harmless either way) |

Laravel Breeze/Jetstream/Fortify register exactly these — recognizing the starter-kit
route names is the fastest way to clear them.

---

## Public content (intentionally readable by anyone)

- Landing/home (`/`), marketing pages, `/about`, `/pricing`, `/contact`.
- Public blog/docs/product listings and their `show` pages — **read-only `GET`**.
- Sitemaps, RSS feeds, `robots.txt`, health checks (`/up`, `/health`).

⚠️ **Scope the exemption to reads.** A *public listing* `GET` is fine; a `POST`/`PUT`/
`DELETE` on the same resource is **not** public-by-design — review it normally. And a
"public" endpoint that returns **owned/per-user data** based on an ID in the URL is the
classic IDOR — public route ≠ public data.

---

## Webhooks (authenticated by signature, not by login)

Third-party webhooks are unauthenticated in the `auth` sense but should verify a
**signature**:

| Source | Verification you should expect |
|---|---|
| Stripe | `Stripe-Signature` header / Cashier's webhook middleware |
| GitHub | `X-Hub-Signature-256` HMAC |
| Paddle, Mailgun, Twilio, etc. | provider-specific signature middleware |

- A webhook route **with** real signature-verification middleware/logic → ✅ (note the
  mechanism). Do **not** flag it for missing `auth`.
- A webhook route with **no** signature verification at all → that **is** a finding
  (🟡/🔴): anyone can forge the call. This is the one webhook case worth raising.
- **The third state: middleware that's present but doesn't actually verify.** A
  `VerifyXSignature`-named middleware that only checks the header is *non-blank*
  (`abort_if(blank($request->header('X-Signature')), 403)`) and never computes the HMAC
  is **security theatre** — anyone sends any non-empty value and passes. Don't credit it
  as ✅ on the name alone: **read the middleware body.** If it doesn't compare a computed
  signature against the payload+secret, treat it as "no verification" → 🟡 (note it's
  latent if the handler is currently a no-op, active once it does real work).

---

## Token / signed-URL protected (auth without a session)

- **Signed URLs** (`signed`/`ValidateSignature` middleware, `URL::signedRoute`) — the
  signature is the authorization. Credit it — **but only for the parameters the signature
  actually covers.** A `signed` route protects the *exact URL* that was signed; if a
  sensitive id is appended **outside** the signed payload (or the route mixes a signed
  segment with an unsigned `{id}` the generator didn't include), that id is still
  tamperable → real IDOR despite the `signed` middleware. Find the **generator** of the
  URL (`URL::temporarySignedRoute(...)`, often in a Resource or service) and confirm the
  sensitive id is part of what was signed. Signed-but-the-id-isn't-in-the-signature is a
  finding, not a pass.
- **API tokens** via `auth:sanctum`/`auth:api` — that's Layer 1, already covered.
- **CSRF** (`web` group / `VerifyCsrfToken`) protects against forgery but is **not**
  authentication or authorization — don't count it as either.

---

## Framework / tooling routes (usually skip entirely)

Out of application scope unless the user asks — don't review or flag:
`telescope/*`, `horizon/*`, `_ignition/*`, `sanctum/csrf-cookie`, `livewire/*`
(internal message endpoint), `storage/*` (symlinked files), debugbar routes.

---

## What is NOT excused by this file

Being "public-facing" never excuses these — review them normally:

- Any **state-changing** route (`POST`/`PUT`/`PATCH`/`DELETE`), even on otherwise-public resources.
- Any route returning **per-user / owned data** keyed by an ID — public route, private data → IDOR.
- A webhook with **no signature check**.
- An "API is public" claim with no rate-limiting/abuse controls — note as hardening.

---

## Extending this file

Add entries for recurring public route patterns (a new starter kit's route names, a
common webhook provider). Every correct entry here is one fewer false positive — but
keep the **state-changing / owned-data** carve-outs intact; those are exactly where
real bugs hide behind a "it's public" assumption.
