---
name: polar-webhooks
description: Implement or debug Polar subscription webhooks when asked to “receive Polar webhooks”, “sync entitlements”, “test signed webhooks”, or “fix webhook 403s”. Use for a Next.js App Router webhook route in the user's own app, transactional idempotency, local signed tests, and real sandbox delivery through a tunnel.
---

## When to use

Use when subscription events must update access safely, retries duplicate events, or a Polar delivery fails
verification. Work in the user's own app: file paths below are suggestions relative to its root, so reuse
existing modules and follow its structure and import aliases. The snippets were tested together in a Next.js
App Router app; for anything not shown, use installed SDK types and the official Polar docs listed at the end.

## Prerequisites

- `@polar-sh/sdk` requires Node >=22 (SDK `engines`). Set:
  add `"engines": { "node": ">=22" }` to `package.json` and `.nvmrc` containing `22`.

- Next.js 15 App Router, TypeScript, Better Auth users, Drizzle/PostgreSQL.
- Before editing, read the app's auth setup and user table, DB client, schema and
  migration config (e.g. `drizzle.config.ts`), `package.json` scripts, env key
  names (never print secret values), any existing Polar code such as the checkout
  helper from the polar-checkout skill or an existing webhook route, and any
  middleware that could intercept `/api/webhooks/polar`.
- SDK **1.0.2** (the tested version) uses `@polar-sh/sdk/2026-04`, not the unavailable
  `@polar-sh/sdk/webhooks` export. Here the APIs are `webhooks.validateEvent`
  and `webhooks.PolarWebhookVerificationError`. Do not paste another version's
  `validateEvent` / `WebhookVerificationError` imports without checking exports.
- Use Node 22+ (installed SDK engine requirement). `.env.local` needs the app's
  PostgreSQL and auth settings plus the Polar entries from the polar-checkout
  skill (`APP_URL`, `POLAR_SERVER`, `POLAR_ACCESS_TOKEN`, `POLAR_PRO_PRODUCT_ID`,
  `POLAR_WEBHOOK_SECRET`). Keep secrets private.
- Use `POLAR_SERVER=sandbox`, sandbox tokens/products, and the endpoint's signing
  secret. Sandbox and production have separate orgs, tokens, products, and secrets.
  Synthetic tests need the local DB/app and matching secret; real checkout also
  needs `POLAR_ACCESS_TOKEN` and `POLAR_PRO_PRODUCT_ID`.

## Procedure

### 1. Preserve lazy configuration and checkout identity

Read environment values inside functions, using the env helper from polar-checkout (e.g. `src/lib/env.ts`):

```ts
export function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}
```

Keep the DB client and the `getPolar()` client (from polar-checkout) lazy. `getPolar()` uses `createPolar`
from the versioned package, with `environment: server` and `accessToken`. If the app's DB module creates its
pool at import time, change it to connect on first use (e.g. a cached `getDb()`, optionally behind a `Proxy`
exported as `db`). Eager env reads can break `next build` when runtime credentials are absent.

The checkout helper in the billing module (e.g. `src/lib/billing.ts`) establishes identity from the signed-in
user. This file's imports are shared with step 3:

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

Add an `entitlements` table to the app's schema: one row per local user, storing Polar
customer/subscription/product IDs, status, current period end, and the subscription's `modified_at`. Its user
foreign key requires an existing local user, including in signed test fixtures. Point it at the app's real
user table (Better Auth's is `user` with a text `id`; match the app's ID type):

```ts
export const entitlements = pgTable(
  'entitlements',
  {
    userId: text('user_id').primaryKey().references(() => user.id, { onDelete: 'cascade' }),
    polarCustomerId: text('polar_customer_id').notNull(),
    subscriptionId: text('subscription_id').notNull(),
    productId: text('product_id').notNull(),
    status: text('status').notNull(),
    subscriptionModifiedAt: timestamp('subscription_modified_at', { withTimezone: true }),
    currentPeriodEnd: timestamp('current_period_end', { withTimezone: true }),
    updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
  },
);
```

Receipt table in the same schema file (imports: `pgTable`, `text`, `timestamp` from `drizzle-orm/pg-core`):

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

Put the helpers in an entitlements module (e.g. `src/lib/entitlements.ts`). They accept a DB or transaction
executor. Module imports: `eq` and `sql` from `drizzle-orm`, the app's `db`, and both tables; declare
`type Executor = Pick<typeof db, 'insert'>`.

Use this `upsertEntitlement` exactly; its `setWhere` is the stale-event guard:

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

Read helper and receipt function in the same module:

```ts
export async function getEntitlement(userId: string) {
  return (
    (await db.select().from(entitlements).where(eq(entitlements.userId, userId)).limit(1))[0] ?? null
  );
}
```

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

Generate a migration with the app's tool (`npm run db:generate` with drizzle-kit), inspect the SQL, and confirm
it only adds `entitlements` and `webhook_events` (never recreate existing auth tables), then apply it
(`npm run db:migrate`). If those scripts are missing, add them; drizzle-kit needs `DATABASE_URL`, e.g.
`node --env-file=.env.local node_modules/drizzle-kit/bin.cjs generate`. Receipt insertion and entitlement
update must share a transaction: failure rolls back both for retry.

### 3. Apply subscription events and gate by status

Add to the same billing module (imports from step 1):

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

Access check in the entitlements module; call it server-side in paid pages, API routes, and server actions:

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

This handler allowlists `POLAR_PRO_PRODUCT_IDS` (comma-separated), falling back to
`POLAR_PRO_PRODUCT_ID`; unrelated products are ignored with 202. It stores subscription
`modified_at` (fallback `created_at`) and atomically updates only when incoming >= stored.
One entitlement per user still needs explicit aggregation for multiple subscriptions.

### 4. Receive and verify the raw request

Create the route at `src/app/api/webhooks/polar/route.ts` (adjust `@/` imports to the app's aliases):

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
of the handler. Do not require a browser session for webhook delivery: exclude this route from auth middleware,
and make sure it does not redirect (Polar treats 3xx responses as failed deliveries).

Other stacks: capture raw body before JSON middleware, forward signing headers, and preserve the same
verification and transactional receipt/update semantics. Use Node runtime for routes using this PostgreSQL adapter.

### 5. Run the signed local test

Write `scripts/test-webhook.ts` in the app (run by the `test:webhook` npm script from polar-checkout; needs
`standardwebhooks` and `tsx` as dev dependencies). It imports the app's `db`, schema tables, entitlement helpers,
`requiredEnv`, and `webhooks` / `models` from `@polar-sh/sdk/2026-04`. It should:

- Insert a temporary local user (fill the app's required user columns) so the entitlement foreign key works.
  In `finally`, delete that user and every receipt ID the script sent, then close the DB pool.
- Build a realistic snake_case fixture typed as `models.Subscription`, letting the type checker list the
  required fields: `status: 'active'`, `product_id` = the first allowed Pro product ID, `customer.external_id`
  and `metadata.userId` = the temporary user ID, `current_period_end` about 30 days ahead, `modified_at: null`.
  Fixture `@example.com` addresses never reach Polar.
- Assert, in order:
  1. Active event for an unrelated product ID: 202, no entitlement row, no access.
  2. Active event (keep its webhook ID): 202, status `active`, `hasActiveSubscription` true.
  3. Same webhook ID again: 202, exactly one `webhook_events` row for it.
  4. Signature `v1,invalid`: 403, access unchanged.
  5. `subscription.revoked` with `status: 'canceled'`, `ended_at` set, `modified_at` one second later: 202,
     no access.
  6. Older active payload (`modified_at` = original time) with a new webhook ID: 202, status stays `canceled`.
  7. Replay of step 2's webhook ID: 202, still no access.
- Print one PASS line naming the cases. On failure print only the error message and set `process.exitCode = 1`.

Signing setup inside `main()` (`Webhook` from `standardwebhooks`):

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

Each send serializes `{ type, timestamp, api_version: '2026-04', data }` once, signs that exact body, and sends
epoch seconds in `webhook-timestamp`. The signing ID must equal `webhook-id`. Valid events are first checked
with the SDK, then POSTed to `${APP_URL}/api/webhooks/polar`:

```ts
  const body = JSON.stringify({ type, timestamp: new Date().toISOString(), api_version: '2026-04', data });
  const date = new Date();
  const headers = {
    'content-type': 'application/json',
    'webhook-id': id,
    'webhook-timestamp': String(Math.floor(date.getTime() / 1000)),
    'webhook-signature': bad ? 'v1,invalid' : signer.sign(id, date, body),
  };
  if (!bad) await webhooks.validateEvent(body, headers, requiredEnv('POLAR_WEBHOOK_SECRET'));
  const response = await fetch(`${requiredEnv('APP_URL')}/api/webhooks/polar`, { method: 'POST', headers, body });
```

In your app, configure a local test DB and `POLAR_WEBHOOK_SECRET=test_secret_local` in `.env.local`:

```sh
npm run db:migrate
npm run dev
```

With the app running, in another terminal:

```sh
npm run test:webhook
```

Expect PASS for unrelated product ignored, active entitlement, duplicate receipt, invalid signature 403,
revocation, stale event ignored, and replay protection. Replaying the original active event ID after revocation must not restore access. This proves local
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

Sign in with a real deliverable email, follow `/api/checkout`, and complete a sandbox payment (Polar's sandbox
accepts the test card `4242 4242 4242 4242` with any future expiry and CVC). Inspect the delivery result in
the Polar dashboard, the matching receipt and entitlement rows, and the app's paid page (e.g. `/dashboard/pro`).
Test scheduled cancellation separately from revocation: active status preserves access until period end;
canceled status removes it. Record what you actually observed; if browser payment or real delivery was not
done, report it as not run rather than passed.

### 7. Diagnose failures without weakening verification

- **403:** check endpoint/environment secret and restart after env changes. If the handler never ran, auth
  middleware, a firewall, or bot protection may be blocking Polar; exclude the route instead.
  Preserve the raw body and all three signing headers through middleware/proxies.
  Do not confuse the organization access token with the endpoint signing secret.
- **Timestamp:** installed verifier tolerance is 300 seconds in either direction.
  Correct system time and sign fresh payloads. Do not disable timestamp checks.
- **Encoding:** reproduce with the local signer and inspect installed verifier
  code; do not randomly strip prefixes or double-base64 the secret.
- **400:** inspect event type/API-version compatibility and JSON. This differs
  from signature failure; do not convert all handler errors to 403. An event type
  the installed SDK doesn't know raises `PolarWebhookUnknownTypeError` (a
  `PolarWebhookError`, so 400 here); subscribe only to the six events above.
- **500/retry:** inspect DB connectivity, migrations, user foreign key, and
  consumed subscription fields. Failed transactions must leave no receipt.
- **Checkout 422:** Polar validates `customer_email` deliverability; `@example.com`
  can yield “domain does not accept email”. Use a real domain for API tests.
  Multi-product errors can include a misleading sibling “product_id Field required”.
  Print full error detail with secrets redacted rather than changing the verified
  `products` shape based on that sibling alone.
- **Portal 422:** `customerSessions.create({ external_customer_id: user.id })`
  reports “Customer does not exist.” for users who never checked out. In the portal
  route, redirect those users to `/dashboard?billing=none` with a notice.

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
- [ ] The database (`npm run db:studio` or `psql`) shows the matching receipt and
  entitlement after real tunnel delivery; sandbox dashboard reports 202.
- [ ] Paid signed-in user can open the paid page; scheduled cancellation keeps
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

## Official Polar docs

- Set up webhook endpoints: https://polar.sh/docs/integrate/webhooks/endpoints
- Validate and monitor deliveries: https://polar.sh/docs/integrate/webhooks/delivery
- Webhook events: https://polar.sh/docs/integrate/webhooks/events
- Sandbox: https://polar.sh/docs/integrate/sandbox
- TypeScript SDK: https://polar.sh/docs/integrate/sdk/typescript
