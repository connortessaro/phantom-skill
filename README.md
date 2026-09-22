# Phantom skill

Teaches a coding agent to use [Phantom](https://phantom.codes) — an
OpenAI-compatible inference API whose keys are prepaid, hold no identity, can
fund scoped child keys for subagents, and sign a receipt for every priced call.

The skill covers the parts an API reference cannot: when to mint a child key
instead of sharing yours, how to pick the caps so a runaway loop stalls instead
of draining the balance, getting the remainder back on burn, and verifying a
receipt yourself against the published key.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills), for any of its
supported agents:

```bash
npx skills add connortessaro/phantom-skill
```

As a Claude Code plugin:

```
/plugin marketplace add connortessaro/phantom-skill
/plugin install phantom@phantom
```

Or copy `skills/phantom/` into your agent's skills directory by hand.

## What is in it

| File | Contents |
| --- | --- |
| `skills/phantom/SKILL.md` | The skill: key delegation, caps, burn and sweep, refusal codes, receipt verification, the CLI |
| `skills/phantom/references/endpoints.md` | Every route with request and response shapes, response headers, error codes |

## Also useful

- `npx phantom-key` — CLI for the same key operations from a shell
- [phantom.codes/docs/concepts](https://phantom.codes/docs/concepts) — the same
  ideas written for a person
- [phantom.codes/docs](https://phantom.codes/docs) — curl for every endpoint

MIT.
