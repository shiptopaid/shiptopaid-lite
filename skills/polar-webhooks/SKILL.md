---
name: polar-webhooks
description: Implement or debug Polar subscription webhooks when asked to “receive Polar webhooks”, “sync entitlements”, “test signed webhooks”, or “fix webhook 403s”. Use for the ShipToPaid App Router handler, transactional idempotency, local signed tests, and real sandbox delivery through a tunnel.
---

## When to use

Use when subscription events must update access safely, retries duplicate events, or a Polar delivery fails
verification. Paths below are relative to your app root. Code snippets come from a tested Next.js App Router reference app.

## Prerequisites

- `@polar-sh/sdk` requires Node >=22 (SDK `engines`). Set:
  add `"engines": { "node": ">=22" }` to `package.json` and `.nvmrc` containing `22`.

- Next.js 15 App Router, TypeScript, Better Auth users, Drizzle/PostgreSQL.
- Read `src/app/api/webhooks/polar/route.ts`, `src/lib/{billing,entitlements}.ts`,
  `src/db/{index,schema}.ts`, `scripts/test-webhook.ts`, `.env.example`,
  `README.md`, and `VERIFICATION.md` before adapting.
- Installed SDK **1.0.2** uses `@polar-sh/sdk/2026-04`, not the unavailable
  `@polar-sh/sdk/webhooks` export. Here the APIs are `webhooks.validateEvent`
  and `webhooks.PolarWebhookVerificationError`. Do not paste another version's
  `validateEvent` / `WebhookVerificationError` imports without checking exports.
- Use Node 22+ (installed SDK engine requirement). Configure `.env.local` from
  `.env.example`, including PostgreSQL and app/auth settings. Keep secrets private.
- Use `POLAR_SERVER=sandbox`, sandbox tokens/products, and the endpoint's signing
  secret. Sandbox and production have separate orgs, tokens, products, and secrets.
  Synthetic tests need the local DB/app and matching secret; real checkout also
  needs `POLAR_ACCESS_TOKEN` and `POLAR_PRO_PRODUCT_ID`.

## Procedure

### 1. Preserve lazy configuration and checkout identity

Read environment values inside functions, as in `src/lib/env.ts`:

```ts
export function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}
```

Keep the lazy DB proxy in `src/db/index.ts` and lazy `getPolar()` client in `src/lib/polar.ts`. The latter
uses `createPolar` from the versioned package, with `environment: server` and `accessToken`. Eager env reads can break `next build` when runtime credentials are absent.

The helper in `src/lib/billing.ts` establishes identity from the signed-in user:

```ts
import { getPolar } from './polar';
import { requiredEnv } from './env';
import { upsertEntitlement } from './entitlements';
import type { models } from '@polar-sh/sdk/2026-04';
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

SDK requests/responses use snake_case: `external_customer_id`, `customer_email`, `success_url`, `product_id`,
`current_period_end`. `{CHECKOUT_ID}` is supported. Read user identity back as
`subscription.customer.external_id`, falling back to `metadata.userId`; no separate customer mapping table is
needed. A checkout redirect or query parameter never grants access.

### 2. Persist receipts and entitlement updates atomically

Keep `entitlements` from `src/db/schema.ts`: one row per local user, storing Polar
customer/subscription/product IDs, status, and current period end. Its user foreign key requires an existing local user, including in signed test fixtures.

Receipt schema from that file (imports: `pgTable`, `text`, `timestamp` from `drizzle-orm/pg-core`):

```ts
export const webhookEvents = pgTable(
  'webhook_events',
  {
    id: text('id').primaryKey(),
    type: text('type').notNull(),
    receivedAt: timestamp('received_at', { withTimezone: true }).defaultNow().notNull(),
  },
);
```

The helpers in `src/lib/entitlements.ts` accept a DB or transaction executor:

Reuse this exact `upsertEntitlement` excerpt; retain the module imports, including `sql` from `drizzle-orm`:

```ts
export async function upsertEntitlement(
  value: typeof entitlements.$inferInsert,
  executor: Executor = db,
) {
  await executor
    .insert(entitlements)
    .values(value)
    .onConflictDoUpdate({
      target: entitlements.userId,
      set: { ...value, updatedAt: new Date() },
      setWhere: sql`${entitlements.subscriptionModifiedAt} IS NULL
        OR excluded.subscription_modified_at >= ${entitlements.subscriptionModifiedAt}`,
    },);
}
```

Persist the schema field copied from `src/db/schema.ts`:

```ts
    subscriptionModifiedAt: timestamp('subscription_modified_at', { withTimezone: true }),
```
The receipt function below uses its existing `db`, `webhookEvents`, and `Executor` declarations (`Executor = Pick<typeof db, 'insert'>`).

```ts
export async function recordWebhookEvent(
  id: string,
  type: string,
  executor: Executor = db,
) {
  return (
    (await executor
      .insert(webhookEvents)
      .values({ id, type })
      .onConflictDoNothing()
      .returning({ id: webhookEvents.id })).length > 0
  );
}
```

Use `npm run db:migrate` for the checked-in migration. After schema changes run `npm run db:generate`, inspect
the SQL, then migrate. Receipt insertion and entitlement update must share a transaction: failure rolls back both for retry.

### 3. Apply subscription events and gate by status

The rest of `src/lib/billing.ts` uses the imports from step 1:

```ts
const subscriptionTypes = new Set([
  'subscription.created',
  'subscription.updated',
  'subscription.active',
  'subscription.canceled',
  'subscription.revoked',
  'subscription.uncanceled',
]);
export async function applySubscriptionEvent(
  event: {
    type: string;
    data: unknown;
  },
  executor?: Parameters<typeof upsertEntitlement>[1],
) {
  if (!subscriptionTypes.has(event.type)) return;
  const subscription = (event.data as models.Subscription);
  const productIds = (process.env.POLAR_PRO_PRODUCT_IDS || requiredEnv('POLAR_PRO_PRODUCT_ID'))
    .split(',').map((id) => id.trim()).filter(Boolean);
  if (!productIds.includes(subscription.product_id)) return;
  const userId = subscription.customer?.external_id || subscription.metadata?.userId;
  if (typeof userId !== 'string' || !userId)
    throw new Error('Subscription has no external user ID');
  const end = new Date(subscription.current_period_end);
  const subscriptionModifiedAt = new Date(subscription.modified_at ?? subscription.created_at);
  if (!subscription.id || !subscription.customer_id ||
    !subscription.product_id ||
    !subscription.status ||
    Number.isNaN(end.getTime()) ||
    Number.isNaN(subscriptionModifiedAt.getTime()))
    throw new Error('Invalid subscription payload');
  await upsertEntitlement(
    {
      userId,
      polarCustomerId: subscription.customer_id,
      subscriptionId: subscription.id,
      productId: subscription.product_id,
      status: subscription.status,
      currentPeriodEnd: end,
      subscriptionModifiedAt,
    },
    executor,
  );
}
```

The installed validator verifies signatures and recognized event types; it does not Zod-parse payloads. `as
models.Subscription` is not runtime validation. Preserve checks on consumed fields and let failures roll back the transaction.

Access check from `src/lib/entitlements.ts`:

```ts
export async function hasActiveSubscription(userId: string) {
  const item = await getEntitlement(userId);
  return !!item && ['active', 'trialing'].includes(item.status);
}
```

Do not revoke solely because the event is `subscription.canceled` or `canceled_at` is set. SDK `Subscription`
types document that cancellation can leave status active when `cancel_at_period_end` is true. Grant access
only for `active` or `trialing`; revocation removes access when status becomes canceled. Check
`SubscriptionStatus` and cancellation fields in installed types on upgrades.

The reference app allowlists `POLAR_PRO_PRODUCT_IDS` (comma-separated), falling back to
`POLAR_PRO_PRODUCT_ID`; unrelated products are ignored with 202. It stores subscription
`modified_at` (fallback `created_at`) and atomically updates only when incoming >= stored.
One entitlement per user still needs explicit aggregation for multiple subscriptions.

### 4. Receive and verify the raw request

Full `src/app/api/webhooks/polar/route.ts`:

```ts
import { webhooks } from '@polar-sh/sdk/2026-04';
import { db } from '@/db';
import { requiredEnv } from '@/lib/env';
import { recordWebhookEvent } from '@/lib/entitlements';
import { applySubscriptionEvent } from '@/lib/billing';
export const runtime = 'nodejs';
export async function POST(request: Request) {
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
}
```

Never call `request.json()` or reserialize before verification. Forward `webhook-id`, `webhook-timestamp`, and
`webhook-signature` unchanged. The SDK rejects missing signing headers before the non-null ID assertion is
reached. Return 403 for verification errors and 400 for other SDK webhook errors. Return 202 quickly after the
short DB transaction, including duplicates; let DB failures propagate for retry. Keep slow external work out
of the handler. Do not require a browser session for webhook delivery.

Other stacks: capture raw body before JSON middleware, forward signing headers, and preserve the same
verification and transactional receipt/update semantics. Use Node runtime for routes using this PostgreSQL adapter.

### 5. Run the signed local test

Reuse `scripts/test-webhook.ts`: its realistic snake_case fixture is typed as `models.Subscription`. It
creates a temporary local user and cleans up user and receipt records. Its `@example.com` addresses never reach Polar.

Signing excerpt from inside its `main()` (uses `Webhook` from `standardwebhooks` and `requiredEnv` from `src/lib/env.ts`):

```ts
  // Standard Webhooks decodes this base64 value to the raw UTF-8 secret.
  // Installed SDK accepts the UTF-8 secret directly (and decoded legacy secrets).
  const signer = new Webhook(
    Buffer.from(requiredEnv('POLAR_WEBHOOK_SECRET')).toString('base64'),
  );
```

Keep this base64 conversion: Standard Webhooks decodes its constructor secret. Older SDK helpers
base64-encoded the secret before passing it to Standard Webhooks. This installed SDK instead verifies HMAC
directly, accepting raw UTF-8 and legacy decoded secrets. Inspect `dist/webhooks-*.mjs` on encoding mismatches; do not assume the older implementation is present.

The script serializes `{ type, timestamp, api_version: '2026-04', data }` once, signs that exact body using
`signer.sign(id, date, body)`, and sends epoch seconds in `webhook-timestamp`. The signing ID must equal
`webhook-id`. It first checks SDK acceptance, then POSTs to `${APP_URL}/api/webhooks/polar`.

In your app, configure a local test DB and `POLAR_WEBHOOK_SECRET=test_secret_local` in `.env.local`:

```sh
npm run db:migrate
npm run dev
```

With the app running, in another terminal:

```sh
npm run test:webhook
```

Expect PASS for active entitlement, duplicate receipt, invalid signature 403, revocation, and replay
protection. Replaying the original active event ID after revocation must not restore access. This proves local
signature/DB behavior, not real dashboard delivery or completed payment.

### 6. Verify real sandbox delivery through a tunnel

Use one installed tunnel tool:

```sh
ngrok http 3000
```

Or:

```sh
cloudflared tunnel --url http://localhost:3000
```

In sandbox Settings → Webhooks, configure `https://<tunnel-host>/api/webhooks/polar` with API version
**2026-04**. Subscribe to `subscription.created`, `subscription.updated`, `subscription.active`,
`subscription.canceled`, `subscription.revoked`, and `subscription.uncanceled`. Put that endpoint's signing
secret in `POLAR_WEBHOOK_SECRET` and restart the app. Do not use `test_secret_local` for real delivery. Update
the endpoint URL when the tunnel hostname changes. Keep app/auth origins consistent with the browser origin; update them if browsing through the tunnel.

Sign in with a real deliverable email, follow `/api/checkout`, and complete a sandbox payment (reference app
verification notes use `4242 4242 4242 4242`). Inspect the dashboard delivery result, matching receipt,
entitlement, and `/dashboard/pro`. Test scheduled cancellation separately from revocation: active status
preserves access until period end; canceled status removes it. `VERIFICATION.md` marks browser/payment and real delivery as not yet done; record actual results.

### 7. Diagnose failures without weakening verification

- **403:** check endpoint/environment secret and restart after env changes.
  Preserve the raw body and all three signing headers through middleware/proxies.
  Do not confuse the organization access token with the endpoint signing secret.
- **Timestamp:** installed verifier tolerance is 300 seconds in either direction.
  Correct system time and sign fresh payloads. Do not disable timestamp checks.
- **Encoding:** reproduce with the local signer and inspect installed verifier
  code; do not randomly strip prefixes or double-base64 the secret.
- **400:** inspect event type/API-version compatibility and JSON. This differs
  from signature failure; do not convert all handler errors to 403.
- **500/retry:** inspect DB connectivity, migrations, user foreign key, and
  consumed subscription fields. Failed transactions must leave no receipt.
- **Checkout 422:** Polar validates `customer_email` deliverability; `@example.com`
  can yield “domain does not accept email”. Use a real domain for API tests.
  Multi-product errors can include a misleading sibling “product_id Field required”.
  Print full error detail with secrets redacted rather than changing the verified
  `products` shape based on that sibling alone.
- **Portal 422:** `customerSessions.create({ external_customer_id: user.id })`
  reports “Customer does not exist.” for users who never checked out. Follow
  `src/app/api/portal/route.ts`: redirect to `/dashboard?billing=none` with a notice.

For APIs not shown here, inspect installed package exports and `dist/**/*.d.mts` types first. If
uncertain, verify against https://polar.sh/docs; never invent request fields, error properties, or subscription status semantics.

## Verification checklist

- [ ] An unrelated product subscription returns 202 and grants no Pro access;
  test `POLAR_PRO_PRODUCT_IDS` (comma-separated) and fallback `POLAR_PRO_PRODUCT_ID`.
- [ ] Revoke, then send a stale active event with a distinct webhook ID: still 202,
  access stays revoked. Persist `modified_at` (fallback `created_at`); only update
  when incoming >= stored. Test equal timestamps and the created-at fallback.

- [ ] Test scripts derive their base URL from `APP_URL`, never a hardcoded port.

Run commands from your app root; DB/Polar npm scripts load `.env.local`.

- [ ] `npm run db:migrate` creates `webhook_events` and `entitlements`.
- [ ] With `npm run dev` running, `npm run test:webhook` passes all assertions.
- [ ] Invalid signature returns 403 without changing access; duplicate ID returns
  202 with one receipt; failed updates roll back their receipt.
- [ ] Run `npx tsc --noEmit`, `npm run lint`, and `npm run build`.
- [ ] Credential-free build: Next.js auto-loads `.env.local` from the project directory.
  Never move, rename, or delete `.env.local`; an interrupted agent can lose it.
  Build a clean copy made with `git clone` or `git worktree add` into a temp
  directory (untracked `.env.local` is not copied), then run `npm ci` and
  `npm run build` from the copied app directory with DB/billing credentials absent.
- [ ] `npm run db:studio` shows the matching receipt and entitlement after real
  tunnel delivery; sandbox dashboard reports 202.
- [ ] Paid signed-in user can open `/dashboard/pro`; scheduled cancellation keeps
  access while active, and revocation/non-entitled status removes access.
- [ ] Optional `npm run test:checkout` prints a sandbox checkout URL. Its fake user
  ID is not a signed-in DB user; use an actual account for payment/webhook tests.
- [ ] Secrets remain private. In new create-next-app templates, `.env*` also
  ignores `.env.example`; add `!.env.example` when packaging the public template.
  Never commit `.env.local`.

## Common mistakes

- Assuming a new webhook-id means a newer event: a new webhook-id does not mean a newer event: stale events can restore revoked access.
- Any product's subscription granting Pro; enforce the configured product allowlist.

- Copying unavailable unversioned webhook imports or camelCase API fields.
- Reading env eagerly, parsing JSON before verification, or using Edge runtime
  with the Node PostgreSQL adapter.
- Deduplicating by subscription ID instead of `webhook-id`, or storing a receipt
  outside the update transaction and permanently blocking failed deliveries.
- Granting access from checkout redirects, revoking on scheduled cancellation,
  or assuming duplicate-ID checks order distinct events.
- Mixing sandbox/production configuration or treating a synthetic test as proof
  that real payment completion and remote delivery work.
