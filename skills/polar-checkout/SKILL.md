---
name: polar-checkout
description: Add Polar subscription checkout to an existing authenticated app when asked to "add Polar checkout", "sell a subscription", or "create a Polar checkout session". Use for environment setup, reusable product creation, authenticated checkout, a success destination, and sandbox verification in the user's own app.
---

## When to use

Add subscription checkout to the user's existing app while preserving its auth,
database, and UI. Examples target Next.js 15 App Router, TypeScript, Better Auth,
and Drizzle/PostgreSQL; the snippets were tested together in a working app.

Work from the user's app, not from a template. Before editing, read its
`package.json` and lockfile, auth setup and server-side session lookup, database
client, schema and migration setup, `.env.example` / `.env.local` key names (never
print secret values), existing pages and routes, and any README. File paths below
(such as `src/lib/polar.ts`) are suggestions relative to the app root: reuse
existing helpers and follow the app's structure and import aliases. For anything
not shown here, use the installed SDK types and the official Polar docs listed at
the end; never invent request fields.

## Prerequisites

- `@polar-sh/sdk` requires Node >=22 (SDK `engines`). Set:
  add `"engines": { "node": ">=22" }` to `package.json` and `.nvmrc` containing `22`.

- Existing users with stable IDs and email addresses; server-side session lookup.
- Polar sandbox organization and organization token with product/checkout access.
- PostgreSQL for entitlement state and webhook receipts; a reachable webhook URL
  for actual payment confirmation.
- Inspect installed package exports/types before choosing APIs. The snippets were
  tested with SDK **1.0.2**, the versioned entry `@polar-sh/sdk/2026-04`, and
  webhook endpoint API version **2026-04**. That SDK's root entry only exports
  error classes (no `new Polar`), and it has no `@polar-sh/sdk/webhooks` entry.
  Polar's docs may show a newer version (such as `2026-10`); whichever you pick,
  use the same version for the SDK import and the webhook endpoint.
- Use a supported Node version: SDK 1.0.2 declares >=22; the optional Next.js
  adapter 1.1.0 declares >=24. Reuse the app's package manager and lockfile. The
  scripts below run with `tsx`, and the signed webhook test uses
  `standardwebhooks`; add both as dev dependencies if missing.

## Procedure

1. **Configure environment and a lazy Polar client.**
   Add these billing entries to `.env.local` (and, without real values, to
   `.env.example`); keep the app's existing database and auth configuration.
   Keep tokens and signing secrets server-only (no `NEXT_PUBLIC_` prefix):

```dotenv
APP_URL=http://localhost:3000
POLAR_SERVER=sandbox
POLAR_ACCESS_TOKEN=
POLAR_PRO_PRODUCT_ID=
POLAR_WEBHOOK_SECRET=test_secret_local
# TEST_CHECKOUT_EMAIL=shiptopaid-test@gmail.com
```
   `APP_URL` has no trailing slash. The local signing secret is for synthetic
   tests; use the endpoint's actual secret for remote delivery. Sandbox and
   production have separate organizations, tokens, products, and webhook secrets.
   Switch them together when deploying.
   Add a small env helper and a lazy Polar client (e.g. `src/lib/env.ts` and
   `src/lib/polar.ts`), or reuse equivalents the app already has:

```ts
export function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}
```

```ts
import { createPolar } from '@polar-sh/sdk/2026-04';
import { requiredEnv } from './env';

export function getPolar() {
  const server = process.env.POLAR_SERVER ?? 'sandbox';
  if (server !== 'sandbox' && server !== 'production')
    throw new Error('POLAR_SERVER must be sandbox or production');
  return createPolar({
    environment: server,
    accessToken: requiredEnv('POLAR_ACCESS_TOKEN'),
  },);
}
```
   Read env inside functions. Eager module-level env reads can break `next build`
   without credentials. Check the app's own auth and DB modules too: if they read
   env or open a connection at import time, create the instance on first use
   instead (e.g. a `getAuth()` / `getDb()` function that caches it). create-next-app
   ignores `.env*`, including `.env.example`;
   add `!.env.example` to the target app's ignore rules when including its template.

2. **Create the subscription product idempotently.**
   Add a re-runnable product script (e.g. `scripts/create-products.ts`):

```ts
import { getPolar } from '../src/lib/polar';

async function main() {
  const polar = getPolar();
  for await (const product of polar.products.iterList({ is_archived: false })) {
    if (product.name === 'Pro Monthly') {
      print(product.id);
      return;
    }
  }
  const product = await polar.products.create({
    name: 'Pro Monthly',
    recurring_interval: 'month',
    prices: [{
      amount_type: 'fixed',
      price_amount: 900,
      price_currency: 'usd',
    },],
  },);
  print(product.id);
}

function print(id: string) {
  console.log(`Product ID: ${id}\nPOLAR_PRO_PRODUCT_ID=${id}`);
}

main().catch(() => {
  console.error('Product setup failed. Check Polar configuration and token permissions.');
  process.exitCode = 1;
},);
```
   Run sequentially: reuse by name prevents duplicates on rerun, not concurrent
   races. Existing pricing is not modified; inspect reused price/currency/interval.
   Adapt the name and price deliberately for the app's subscription.
   Add these npm scripts (the test scripts are written in step 7 and in the
   polar-webhooks skill):

```json
{
  "polar:products": "node --env-file=.env.local --import tsx scripts/create-products.ts",
  "test:checkout": "node --env-file=.env.local --import tsx scripts/test-checkout.ts",
  "test:webhook": "node --env-file=.env.local --import tsx scripts/test-webhook.ts"
}
```
   Run `npm run polar:products` with sandbox credentials. Paste its printed
   `POLAR_PRO_PRODUCT_ID=...` into `.env.local`; rerun to confirm the same ID.

3. **Create checkout using the server-authenticated user.**
   Add a billing helper (e.g. `src/lib/billing.ts`):

```ts
import { getPolar } from './polar';
import { requiredEnv } from './env';

export function createCheckoutForUser(user: {
  id: string;
  email: string;
},) {
  return getPolar().checkouts.create({
    products: [requiredEnv('POLAR_PRO_PRODUCT_ID')],
    external_customer_id: user.id,
    customer_email: user.email,
    success_url: `${requiredEnv('APP_URL')}/dashboard?checkout_id={CHECKOUT_ID}`,
    metadata: { userId: user.id },
  },);
}
```
   SDK payloads use snake_case: `external_customer_id`, `customer_email`,
   `success_url`, `product_id`, `current_period_end`. Use product IDs in `products`.
   Preserve the `{CHECKOUT_ID}` placeholder in the success URL.
   Passing `external_customer_id = user.id` links users without a separate mapping
   table. Webhooks read `subscription.customer.external_id`, falling back to
   `subscription.metadata.userId`. Never trust browser-supplied identity/email.

4. **Require auth in `/api/checkout`.**
   Reuse the app's session guard. If it has none, a Better Auth example (here
   `getAuth()` returns the app's lazily created Better Auth instance; adapt the
   import to however the app exports it):

```ts
import { headers } from 'next/headers';
import { redirect } from 'next/navigation';
import { getAuth } from './auth';

export async function requireSession() {
  const requestHeaders = await headers();
  const session = await getAuth().api.getSession({ headers: requestHeaders });
  if (!session) redirect('/sign-in');
  return session;
}
```
   Create the checkout route (e.g. `src/app/api/checkout/route.ts`):

```ts
import { redirect } from 'next/navigation';
import { requireSession } from '@/lib/session';
import { createCheckoutForUser } from '@/lib/billing';

export const runtime = 'nodejs';

export async function GET() {
  const {
    user
  } = await requireSession();
  const checkout = await createCheckoutForUser(user);
  redirect(checkout.url);
}
```
   Link the upgrade control to `/api/checkout`. Mark DB-using routes/pages
   `runtime = 'nodejs'`. Keep `redirect()` outside catches that swallow its
   control-flow exception; show useful checkout errors without exposing secrets.
   Optional alternative, verified in installed `@polar-sh/nextjs/dist/index.d.ts`:
   `Checkout({ accessToken, successUrl, environment })` returns a request handler.
   Adapter config uses camelCase; SDK payloads use snake_case. Adapter 1.1.0 uses
   SDK 2026-10 internally. Prefer the direct implementation for server-owned
   identity/product selection. If adapting the helper, authenticate first,
   override identity/product query parameters server-side, initialize lazily,
   and inspect its installed types/runtime and `dist/index.js` for compatibility.

5. **Provide a success page that waits for verified payment state.**
   Use an existing signed-in page as the success destination (the examples use
   `/dashboard`). In Next.js 15 a server page receives `searchParams` as a Promise,
   e.g. `searchParams: Promise<{ upgrade?: string; checkout_id?: string;
   billing?: string }>`. The page authenticates, reads persisted entitlement, awaits
   `searchParams` into `params`, and shows a pending notice:

```tsx
      {params.checkout_id && (
        <p role="status" className="mt-6 rounded-lg bg-indigo-50 p-4">
          Thanks! Your access updates when payment confirmation arrives. Refresh if
          your subscription is still pending.
        </p>
      )}
```
   A query parameter only displays the notice. Grant access from verified webhook
   state; allow refresh while confirmation is pending. If choosing another success
   path, update `success_url` consistently and preserve these semantics.

6. **Persist webhook receipts and subscription entitlement.**
   Follow the polar-webhooks skill for this step: it has the `entitlements` and
   `webhook_events` tables (linked to the app's existing user table;
   `webhook_events.id` is a primary key holding `webhook-id`), the entitlement
   helpers, `applySubscriptionEvent`, and the full webhook route. Generate and
   review a migration that only adds billing tables; never recreate the app's
   existing auth tables. Configure endpoint API version 2026-04 and the six events
   the handler accepts: `subscription.created`, `.updated`, `.active`, `.canceled`,
   `.revoked`, `.uncanceled`.
   The critical sequence inside the route (it also needs its imports and the
   Node runtime):

```ts
  const body = await request.text();
  let event;
  try {
    event = await webhooks.validateEvent(
      body,
      Object.fromEntries(request.headers),
      requiredEnv('POLAR_WEBHOOK_SECRET'),
    );
  } catch (error) {
    if (error instanceof webhooks.PolarWebhookVerificationError)
      return new Response('Invalid signature', { status: 403 });
    if (error instanceof webhooks.PolarWebhookError)
      return new Response('Invalid event', { status: 400 });
    throw error;
  }
  const id = request.headers.get('webhook-id')!;
  // Atomic receipt + update: a failed update rolls back the receipt for retries.
  await db.transaction(async (tx) => {
    if (await recordWebhookEvent(id, event.type, tx))
      await applySubscriptionEvent(event, tx);
  },);
  return new Response(null, { status: 202 });
```
   Never call `request.json()` before verification. Receipt insertion uses
   `.onConflictDoNothing().returning(...)`; update entitlement only for a new ID.
   Commit receipt and update together so failures roll back and remain retryable.
   Respond 202 promptly after durable work; do not launch unawaited updates.
   Version distinction: older guidance uses `validateEvent`/`WebhookVerificationError`
   from `@polar-sh/sdk/webhooks` and base64-encodes secrets before Standard Webhooks.
   Installed 1.0.2 instead uses `webhooks` from `@polar-sh/sdk/2026-04`, Web Crypto
   HMAC verification, and `PolarWebhookVerificationError`. It accepts raw UTF-8 and
   legacy base64 secrets. It checks signatures/types without Zod-parsing payloads;
   preserve consumed-field validation in `applySubscriptionEvent`.
   Local signed tests still base64-encode the secret for Standard Webhooks, which
   decodes it to the signing bytes. In the signed test (polar-webhooks step 5):

```ts
  const signer = new Webhook(
    Buffer.from(requiredEnv('POLAR_WEBHOOK_SECRET')).toString('base64'),
  );
```
   Do not replace the endpoint secret with this encoded test value.
   Entitlement is status in `['active', 'trialing']`. SDK `Subscription` types
   include `cancel_at_period_end` and document that canceled subscriptions may
   remain active with this flag. Do not revoke from `subscription.canceled` alone:
   retain access while status is active until period end/revocation updates it.
   Enforce entitlement on paid APIs/server actions, too. This design stores one
   subscription per user, allowlists `POLAR_PRO_PRODUCT_IDS` (fallback
   `POLAR_PRO_PRODUCT_ID`), and ignores other products with 202. Preserve its
   `modified_at` (fallback `created_at`) persistence and incoming >= stored guard.
   Multiple subscriptions still need explicit reconciliation.

7. **Add and run the checkout smoke test.**
   Add `scripts/test-checkout.ts`. It creates a checkout for a synthetic user with
   the billing helper (`randomUUID` from `node:crypto`) and prints the URL:

```ts
  const checkout = await createCheckoutForUser({
    id: `checkout-test-${randomUUID()}`,
    email: process.env.TEST_CHECKOUT_EMAIL || 'shiptopaid-test@gmail.com',
  },);
  console.log(checkout.url);
```
   Wrap `main()` in a sanitized error handler: print only the error message and
   numeric `statusCode`, with every environment value and any `Bearer ...` token
   replaced by `[redacted]`, set `process.exitCode = 1`, and never log the raw
   error object (it can contain credentials or customer data).
   Use a deliverable domain, preferably an address you control. Polar rejects
   `@example.com` checkout emails with 422 “domain does not accept email”. Local
   webhook fixture emails do not contact Polar. `npm run test:checkout` prints a
   real sandbox URL, not a payment result. Its synthetic ID is not a database user;
   complete payment as a real signed-in user for entitlement foreign keys to work.
   Other stacks: retain server-owned identity, the checkout API call, raw-body
   signature verification, atomic deduplication, and a pending success UI; replace
   Next.js session/redirect/runtime conventions with framework equivalents.

## Verification checklist

- [ ] An unrelated product subscription returns 202 and grants no Pro access;
  test `POLAR_PRO_PRODUCT_IDS` (comma-separated) and fallback `POLAR_PRO_PRODUCT_ID`.
- [ ] Revoke, then send a stale active event with a distinct webhook ID: still 202,
  access stays revoked. Persist `modified_at` (fallback `created_at`); only update
  when incoming >= stored. Test equal timestamps and the created-at fallback.

- [ ] Test scripts derive their base URL from `APP_URL`, never a hardcoded port.

- [ ] Run `npm run polar:products` twice; confirm the same ID and intended price,
  currency, interval, and sandbox environment.
- [ ] For added tables, run `npm run db:generate`, review SQL, then
  `npm run db:migrate` (or the existing app's equivalent commands).
- [ ] Run `npm run test:checkout`; expect `https://sandbox.polar.sh/checkout/...`.
- [ ] With `npm run dev` running separately, run `npm run test:webhook` after
  writing the signed test from the polar-webhooks skill: unrelated product ignored,
  active access, duplicate receipt, signature 403, revocation, stale event ignored,
  and replay of the same event ID must pass.
- [ ] Run `npx tsc --noEmit`, `npm run lint`, and `npm run build`.
- [ ] Credential-free build: Next.js auto-loads `.env.local` from the project directory.
  Never move, rename, or delete `.env.local`; an interrupted agent can lose it.
  Build a clean copy made with `git clone` or `git worktree add` into a temp
  directory (untracked `.env.local` is not copied), then run `npm ci` and
  `npm run build` from the copied app directory with DB/billing credentials absent.
- [ ] Run `curl -I "$APP_URL/api/checkout"` without cookies; expect a
  redirect to `/sign-in`. In a signed-in browser, expect a sandbox checkout URL.
- [ ] Complete sandbox payment as a real app user; receive actual signed webhook
  delivery, confirm pending notice and eventual paid access, then test cancellation
  at period end and revocation separately (test card `4242 4242 4242 4242`, any
  future expiry and CVC). Browser payment and real delivery are manual steps:
  report them as NOT RUN unless you actually completed them.
- [ ] Visit `/dashboard?checkout_id=forged` without entitlement; paid UI and APIs
  must stay locked. Test duplicate webhooks cannot restore revoked access.

## Common mistakes

- Assuming a new webhook-id means a newer event: a new webhook-id does not mean a newer event: stale events can restore revoked access.
- Any product's subscription granting Pro; enforce the configured product allowlist.

- Guessing fields/imports from a major version. Inspect installed SDK `package.json`
  exports, `dist/2026-04/index.d.mts`, service types, and referenced model types.
  If unsure, read those types or https://polar.sh/docs before writing an API call.
- Treating multi-product checkout's 422 “product_id Field required” sibling error
  as proof `products` is wrong. Print the full validation detail with secrets/PII
  redacted; another sibling error may identify email deliverability. The smoke
  test prints sanitized message/status; add sanitized nested-detail diagnostics
  if these omit the real cause.
- Opening a portal for users who never checked out without handling 422
  “Customer does not exist.” Verified call:
  `customerSessions.create({ external_customer_id: user.id })`, returning
  `customer_portal_url`. In an authenticated portal route (e.g.
  `src/app/api/portal/route.ts`), a 422 whose detail or message contains
  "Customer does not exist" redirects to `/dashboard?billing=none` with a notice;
  other failures log a generic line (no SDK error payload) and redirect to
  `/dashboard?billing=error`. Inspect the installed SDK error types for where the
  status and detail live. Keep the final successful redirect outside the catch.
- Trusting success query parameters or event names as payment proof, recording
  receipts outside the entitlement transaction, mixing sandbox/production values,
  or reading required environment variables eagerly at module scope.

## Official Polar docs

- Sandbox: https://polar.sh/docs/integrate/sandbox
- TypeScript SDK: https://polar.sh/docs/integrate/sdk/typescript
- Checkout API: https://polar.sh/docs/features/checkout/session
- Create checkout session (2026-04): https://polar.sh/docs/api-reference/2026-04/checkouts/create-checkout-session
- Create customer session (portal): https://polar.sh/docs/api-reference/2026-04/customer-sessions/create-customer-session
- Webhooks: https://polar.sh/docs/integrate/webhooks/endpoints
