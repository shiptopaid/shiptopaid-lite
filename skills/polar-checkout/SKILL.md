---
name: polar-checkout
description: Add Polar subscription checkout to an existing authenticated app when asked to "add Polar checkout", "sell a subscription", or "create a Polar checkout session". Use for environment setup, reusable product creation, authenticated checkout, a success destination, and sandbox verification with the ShipToPaid starter.
---

## When to use

Add subscription checkout while preserving the app's auth, database, and UI.
Primary examples target Next.js 15 App Router, TypeScript, Better Auth, and
Drizzle/PostgreSQL. Paths below are relative to the ShipToPaid `starter/`.
Read its `src/`, `scripts/`, `drizzle/`, `.env.example`, `README.md`, and
`VERIFICATION.md` before adapting; snippets below come from that implementation.

## Prerequisites

- `@polar-sh/sdk` requires Node >=22 (SDK `engines`). Set:
  add `"engines": { "node": ">=22" }` to `package.json` and `.nvmrc` containing `22`.

- Existing users with stable IDs and email addresses; server-side session lookup.
- Polar sandbox organization and organization token with product/checkout access.
- PostgreSQL for entitlement state and webhook receipts; a reachable webhook URL
  for actual payment confirmation.
- Inspect installed package exports/types before choosing APIs. The reference has
  SDK **1.0.2**, using `@polar-sh/sdk/2026-04`, and endpoint API version **2026-04**.
  It does not export `new Polar` from the root or `@polar-sh/sdk/webhooks`.
- Use a supported Node version: installed SDK declares >=22; the optional Next.js
  adapter 1.1.0 declares >=24. Reuse existing dependencies/package manager and
  lockfile; the reference uses `tsx` for scripts and `standardwebhooks` for tests.

## Procedure

1. **Configure environment and a lazy Polar client.**
   Merge billing entries from `.env.example`; retain the existing app's database
   and auth configuration. Keep tokens and signing secrets server-only:

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
   Copy `src/lib/env.ts` and `src/lib/polar.ts`:

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
   without credentials; keep auth/DB initialization lazy too (see `src/lib/auth.ts`
   and `src/db/index.ts`). create-next-app ignores `.env*`, including `.env.example`;
   add `!.env.example` to the target app's ignore rules when including its template.

2. **Create the subscription product idempotently.**
   Copy `scripts/create-products.ts`:

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
   Merge these entries from `package.json` into the app's scripts:

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
   Copy this function and imports from `src/lib/billing.ts`:

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
   Reuse the app's session guard. Better Auth reference `src/lib/session.ts`:

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
   Copy `src/app/api/checkout/route.ts`:

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
   The reference uses `/dashboard` as its success destination. Its server page
   receives `searchParams: Promise<{ upgrade?: string; checkout_id?: string;
   billing?: string }>` in Next.js 15, authenticates, reads persisted entitlement,
   and awaits `searchParams`. Copy the full page's imports/props when adapting.
   Its success notice from `src/app/dashboard/page.tsx` is:

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
   Adapt `entitlements` and `webhookEvents` from `src/db/schema.ts` to the existing
   user table; `webhook_events.id` is a primary key containing `webhook-id`.
   Generate/review a migration; do not blindly apply example auth tables.
   Copy `src/lib/entitlements.ts`, `applySubscriptionEvent` from `src/lib/billing.ts`,
   and `src/app/api/webhooks/polar/route.ts`. Configure endpoint API version 2026-04
   and the six events the handler accepts: `subscription.created`, `.updated`,
   `.active`, `.canceled`, `.revoked`, `.uncanceled`.
   Critical sequence copied from the route (retain its imports and Node runtime):

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
   decodes it to the signing bytes. From `scripts/test-webhook.ts`:

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
   Enforce entitlement on paid APIs/server actions, too. The starter stores one
   subscription per user, allowlists `POLAR_PRO_PRODUCT_IDS` (fallback
   `POLAR_PRO_PRODUCT_ID`), and ignores other products with 202. Preserve its
   `modified_at` (fallback `created_at`) persistence and incoming >= stored guard.
   Multiple subscriptions still need explicit reconciliation.

7. **Add and run the checkout smoke test.**
   Copy the complete `scripts/test-checkout.ts`, including its sanitized error
   handler. Its core uses `randomUUID` from `node:crypto` and the billing helper:

```ts
  const checkout = await createCheckoutForUser({
    id: `checkout-test-${randomUUID()}`,
    email: process.env.TEST_CHECKOUT_EMAIL || 'shiptopaid-test@gmail.com',
  },);
  console.log(checkout.url);
```
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
  adapting the test script in skills/polar-webhooks: active access, duplicate receipt, signature 403,
  revocation, and replay of the same event ID must pass.
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
  at period end and revocation separately. `VERIFICATION.md` marks browser payment
  and real delivery as manual work, not checks already completed automatically.
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
  redacted; another sibling error may identify email deliverability. The starter
  test prints sanitized message/status; add sanitized nested-detail diagnostics
  if these omit the real cause.
- Opening a portal for users who never checked out without handling 422
  “Customer does not exist.” Verified call:
  `customerSessions.create({ external_customer_id: user.id })`, returning
  `customer_portal_url`. Adapt `src/app/api/portal/route.ts`: missing customers
  redirect to `/dashboard?billing=none` with a notice; other failures use
  `/dashboard?billing=error`. Keep the final successful redirect outside the catch.
- Trusting success query parameters or event names as payment proof, recording
  receipts outside the entitlement transaction, mixing sandbox/production values,
  or reading required environment variables eagerly at module scope.
