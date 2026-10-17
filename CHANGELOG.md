# Changelog

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
