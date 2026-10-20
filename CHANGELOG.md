# Changelog

## 2026-10-20

- `launch-audit-lite`: replaced the "buy your own product with a real card in
  production, then refund it" step. Polar's docs ask you not to run test
  purchases with real card details (it looks like card testing and can block
  the card or the account). The production check now uses a 100% discount code
  limited to your product and to one redemption, with the Forever duration for
  subscriptions so no card is asked for, then cancels the test subscription.
  Added this to Common mistakes and linked the Polar docs for test transactions
  and discounts.
- README: the Limits section now says not to test production with a real card.

## 2026-10-17

- README: corrected the description of the paid pack (11 skills with Polar and
  Stripe support, 5 commands, a Next.js starter; $49 one-time for the first 100
  buyers, then $79). Removed the outdated "6 more skills / Stripe fallback" text.
- README: added Example, Limits and Contributing / issues sections.
- Removed the `stripe` repo topic. This repo is Polar only.
- Added this changelog.
- Made all three skills work standalone in your own app: removed references to
  the paid starter's files and `VERIFICATION.md`, described the checkout and
  signed webhook tests inline, and linked the official Polar docs.

## 2026-10-16

- First release with three skills: `polar-checkout`, `polar-webhooks` and
  `launch-audit-lite`.
- Made the skills self-contained so they no longer point to paths in the paid
  starter.
