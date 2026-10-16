# ShipToPaid Lite

Free AI coding-agent skills to add **Polar** payments to a vibe-coded Next.js app,
and to check it's actually ready to take money. Works with **Codex**, **Claude
Code** and **Cursor**.

| Skill | What it does |
|---|---|
| `polar-checkout` | Products script, server-side checkout tied to your user, success redirect |
| `polar-webhooks` | Signature-verified webhooks, idempotency, stale-event and product-allowlist guards, local signed tests |
| `launch-audit-lite` | A 15-minute go/no-go checklist before switching to production |

Every gotcha in these skills was hit for real while building a working app: snake_case
SDK fields, raw-body verification, `example.com` emails rejected with a 422, stale
events restoring revoked access, and more.

## Install

**Codex / Cursor:** copy `skills/` into your app and add to `AGENTS.md`:

```md
Skills live in skills/<slug>/SKILL.md. Before relevant work, read the matching
skill and follow its procedure and verification checklist:
- Polar checkout: polar-checkout. Webhooks: polar-webhooks.
- Before going live: launch-audit-lite.
```

**Claude Code:** copy each folder to `.claude/skills/<slug>/` in your app (or
`~/.claude/skills/`). Claude picks them up from their descriptions.

Then ask your agent: *"Add a $9/month Pro plan with Polar"* or *"Am I ready to launch?"*

Requirements: Node 22+ (required by `@polar-sh/sdk`), a Polar sandbox account.

## Want the whole path from shipped to paid?

**ShipToPaid** ($49, one-time, lifetime updates) adds:

- 6 more skills: entitlements and customer portal, usage and credit metering, license keys,
  Stripe fallback, a landing page and pricing copy, plus the **full** launch audit
- 5 slash commands (`/add-paywall`, `/debug-webhook`, `/launch-audit`, `/ship`, `/add-credits`)
- A working Next.js starter: Better Auth, Postgres/Drizzle, Polar checkout, webhooks,
  gated pages, portal, legal templates and end-to-end test scripts

👉 https://shiptopaid.com

## License

MIT
