# axon-wallet skill

The AXON runtime skill — procedural guidance Claude Code (and compatible
agents) should follow when they're operating against an AXON wallet via
the [AXON MCP server](https://api.axon402.com/v1/mcp).

It teaches an agent:

- Always call `read_spending_policy` before a paid request.
- Use `governed_fetch` for every paid HTTP call.
- Handle `APPROVAL_REQUIRED` / `REJECTED_POLICY` / `INTENT_MISMATCH` /
  `RETRY_BUDGET_EXCEEDED` without giving up on the call.
- Prefer `check_approval_status` over `consume_approval_token`.
- Report outcomes honestly (`fulfilled` / `partial` / `unfulfilled`).
- Stay inside the runtime surface — operator actions (create wallets,
  fund treasury, rotate keys, modify mandates) are out of scope.

## Install

```bash
npx skills add alex-reysa/axon-wallet-skill
```

This uses [Vercel Labs' `skills` CLI](https://github.com/vercel-labs/skills)
to copy the `SKILL.md` into `.claude/skills/axon-wallet/SKILL.md` (or
`~/.claude/skills/axon-wallet/SKILL.md` for a global install). Claude
Code auto-discovers skills there.

The skill is wallet-agnostic — it pairs with whatever `axon` MCP server
you've configured in your client, so the same skill body works for
every wallet.

## Configure the paired MCP server

Install the skill and wire up MCP in one paste by using the Recommended
tab on any wallet's **Runtime Connection** panel at
[axon402.com](https://axon402.com). The dashboard hands you a short
setup prompt that:

1. Configures an `axon` MCP server with your wallet-scoped endpoint +
   runtime key.
2. Installs this skill via `npx skills add`.
3. Verifies the install by calling `read_spending_policy`.

## Canonical source

The `SKILL.md` in this repo is generated from `AXON_SKILL_BODY` in
[`axon-402-mcp/src/lib/connect-snippets.ts`](https://github.com/alex-reysa/AXON/blob/main/axon-402-mcp/src/lib/connect-snippets.ts).
The AXON monorepo has a vitest drift guard that pins that constant
against the on-disk file and a `publish:skill-repo` npm script that
mirrors the content here.

## License

Apache-2.0 — see [LICENSE](./LICENSE).
