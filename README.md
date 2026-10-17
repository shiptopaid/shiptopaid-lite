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

## Example

With the skills installed, type a plain request into Claude Code, Cursor or Codex:

> Add a $9/month Pro plan with Polar. Only Pro users should be able to open /dashboard/pro.

The agent reads `polar-checkout` and `polar-webhooks`, then roughly:

1. Creates the Pro plan in your Polar sandbox with a small script you can re-run.
2. Adds an upgrade button that sends the signed-in user to Polar checkout and
   back to a success page.
3. Sets up the payment notifications Polar sends, checks each one really came
   from Polar, and turns Pro on (and off again when a subscription ends).
4. Locks `/dashboard/pro` so only Pro users can open it.
5. Runs the included sandbox tests and tells you what passed and what it couldn't run.

Later, ask *"Am I ready to launch?"* and `launch-audit-lite` walks through the
checklist, marking each item PASS, FAIL or NOT RUN.

Results depend on your app and your agent. Read the changes before you commit them.

## Limits

- Written for Next.js App Router with TypeScript. Examples use Better Auth and
  Postgres + Drizzle; on another stack the agent has to adapt them.
- Polar only. There is no Stripe support in this repo.
- These are instructions for your coding agent, not a library. There is nothing
  to install from npm; the agent writes the code into your app.
- Test in the Polar sandbox before switching to production.

## Contributing / issues

Found a mistake, an outdated Polar detail, or a step that didn't work with your
agent? [Open an issue](https://github.com/shiptopaid/shiptopaid-lite/issues) with
what you asked, which agent you used and what happened. Small pull requests that
fix or clarify a skill are welcome. Please never paste API keys or other secrets
into an issue.

## The paid pack

The paid [ShipToPaid](https://www.shiptopaid.com) pack goes further, with full
Polar **and** Stripe support:

- 11 skills: checkout, verified payment notifications, access control, customer
  portal, usage credits, license keys, the full launch audit, a landing page and
  pricing copy
- 5 commands: `/add-paywall`, `/debug-webhook`, `/launch-audit`, `/ship`, `/add-credits`
- A Next.js starter: Better Auth, Postgres + Drizzle, Polar

$49 one-time for the first 100 buyers, then $79. The skills in this repo are free
under MIT and work on their own.

## License

MIT
