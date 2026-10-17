---
name: launch-audit-lite
description: Quick pre-launch payments check for a Polar-powered web app. Use when asked "am I ready to launch?", "go live with Polar", or "switch from sandbox to production", to catch the mistakes that most often lose money or lock out paying users.
---

# Launch audit (lite)

A 15-minute go/no-go check before you take real payments with Polar. Report each
item as PASS, FAIL or NOT RUN, with the command or evidence. A successful build
or a working checkout URL alone does not mean you're ready.

## When to use
- Right before switching `POLAR_SERVER` from `sandbox` to `production`.
- After changing billing, auth or deployment config.

## Prerequisites
- The app runs locally and in its production deployment.
- You can see the production Polar dashboard (Settings → Webhooks, Products).

## Procedure and checklist

### 1. Environment
- [ ] `POLAR_SERVER=production`, and the access token, product IDs and webhook
      secret all come from the **production** org. Sandbox and production are
      separate orgs with separate IDs and secrets.
- [ ] `APP_URL` (and your auth URL) is the real HTTPS domain.
- [ ] Node matches the SDK: `@polar-sh/sdk` requires Node >= 22
      (`node -v`; set `"engines"` and `.nvmrc`).
- [ ] No secrets in git: `git log -p | grep -E 'polar_(oat|pat)_'` returns nothing,
      and `.env*` is gitignored except `.env.example`.

### 2. Webhooks
- [ ] A production webhook endpoint (HTTPS) is registered for `subscription.*`
      (and `order.*` if you sell one-time products).
- [ ] The handler reads the **raw** body (`await request.text()`) and verifies it
      before parsing; an invalid signature returns 403.
- [ ] Duplicate deliveries (same `webhook-id`) are ignored.
- [ ] Only your paid product IDs grant access. A subscription to any other product
      must not unlock the paid tier.
- [ ] An older event delivered late (new webhook-id, older `modified_at`) cannot
      restore access after revocation.

### 3. Access
- [ ] Logged out → sign-in; free user → upgrade; active/trialing subscriber → access.
- [ ] Paid API routes and server actions are checked server-side too, not just pages.
- [ ] A forged `?checkout_id=` on the success URL grants nothing (only webhooks grant).
- [ ] Customer portal works for subscribers and shows a friendly notice for users
      who never checked out (Polar returns 422 "Customer does not exist.").

### 4. Business basics
- [ ] Terms, privacy and refund pages are linked from the footer and checkout flow.
- [ ] A monitored support email is shown.
- [ ] Checkout errors show a retry screen instead of a 500.

### 5. Real money test
- [ ] Buy your own product with a real card in production, confirm the webhook
      arrived and access unlocked, then refund it and confirm access behaves as
      your policy says.
- [ ] `npm run build` passes in a clean clone (Next.js auto-loads `.env.local`, so
      never move or rename it to test this; use `git worktree add`).

## Common mistakes
- Shipping with a sandbox token or sandbox product ID in production env.
- Registering the webhook in sandbox but not in production (each org has its own).
- Parsing JSON before verifying the signature. Verification needs the exact raw bytes.
- Granting access from the checkout success redirect instead of the webhook.
- Testing with `@example.com` emails: Polar rejects undeliverable domains with 422.

## Official Polar docs
- Sandbox vs production: https://polar.sh/docs/integrate/sandbox
- Webhook endpoints and deliveries: https://polar.sh/docs/integrate/webhooks/endpoints,
  https://polar.sh/docs/integrate/webhooks/delivery
