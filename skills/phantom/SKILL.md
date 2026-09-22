---
name: phantom
description: Use the Phantom inference API (phantom.codes) — an OpenAI-compatible gateway whose keys are prepaid, hold no identity, can fund scoped child keys for subagents, and sign a receipt for every priced call. Use this skill whenever an agent needs to pay for its own inference, when work is being fanned out to subagents or workers that each need a spend limit, when a runaway loop needs a hard dollar or per-minute ceiling rather than a prompt asking it to be careful, when someone wants to check which model actually served a response, or when a Phantom key, `sk-phantom-` credential, child key, budget cap, `x-phantom-receipt` or a 402/429 from phantom.codes comes up. Reach for it even if the user only says "give the subagent a budget" or "make sure it can't burn through my credit" without naming Phantom.
---

# Phantom

An OpenAI-compatible inference API where the key is the account. Three things
follow from that, and they are the reason to use this skill rather than
treating Phantom as a drop-in base URL:

- A key holds credit, not an identity. Nothing links it to a person.
- A key can mint another key out of its own balance, with hard caps.
- Every priced response is signed, so what you were served is checkable.

Base URL `https://phantom.codes/v1`. Keys look like `sk-phantom-…`.

## Inference is just the OpenAI SDK

Nothing special here. Point any OpenAI client at the base URL:

```python
from openai import OpenAI
client = OpenAI(base_url="https://phantom.codes/v1", api_key=os.environ["PHANTOM_API_KEY"])
```

`/chat/completions`, `/embeddings`, `/images/generations`, `/audio/speech`,
`/audio/transcriptions`, `/rerank`, `/video/generations`, `/models`. Streaming
works. `/models` needs no key, so it is the cheapest way to check a model id
before spending anything.

Every priced response carries what it cost and what is left, so you rarely need
a second call to find out:

```
x-phantom-cost-usd: 0.000412
x-phantom-balance-usd: 2.499588
```

## Fanning work out: give each worker its own key

This is the part worth knowing. When you spawn subagents, do not hand them the
key you hold. Mint one per task, funded with what that task should cost:

```bash
curl -X POST https://phantom.codes/v1/key/child \
  -H "Authorization: Bearer $PHANTOM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"amount_usd": 0.50, "ttl_hours": 6, "rate_usd_per_min": 0.10}'
```

```json
{
  "api_key": "sk-phantom-…",
  "funded_usd": 0.5,
  "expires_at": "2026-09-22T17:09:39.000Z",
  "budget_usd": null,
  "rate_usd_per_min": 0.1,
  "parent_balance_usd": 2.5
}
```

The parent is debited in the same statement that writes the child, so a parent
that cannot cover the amount mints nothing — there is no half-funded state to
clean up. **The child key is returned once and stored only as a hash.** If you
lose it you cannot recover it; you can only burn it from the parent side.

Why bother instead of sharing the parent key: a worker with your key can spend
all of it, and a worker in a loop will. A child key makes the blast radius a
number you chose. It is also a full key — it can call every route and mint
children of its own — so a supervisor/worker tree works without any special
delegation protocol.

### Choosing the caps

Three independent limits, and they fail differently:

| Limit | Set with | Answers |
| --- | --- | --- |
| `amount_usd` | at mint | The total this task may ever cost. It is a transfer, not a promise — the credit physically moves. |
| `rate_usd_per_min` | at mint, or `PATCH /v1/key/budget` | A loop calling itself. This is the one that catches runaway recursion, because a monthly ceiling cannot notice anything inside the month. |
| `budget_usd` | `PATCH /v1/key/budget` | A rolling monthly ceiling, for a long-lived key rather than a task. |
| `ttl_hours` | at mint | Default 24. A task key should expire in hours, not months. |

Pick `amount_usd` from what the task should cost, then set `rate_usd_per_min`
to something a sane worker never touches but a loop hits within seconds. For a
task budgeted at $0.50, a $0.10/min cap means a runaway stalls after a few
calls instead of draining the whole allocation.

`PATCH /v1/key/budget` with `null` clears either cap.

### Getting the money back

When the task finishes, the worker burns its own key:

```bash
curl -X DELETE https://phantom.codes/v1/key -H "Authorization: Bearer $CHILD_KEY"
```

```json
{ "revoked": true, "swept_usd": 0.31, "swept_to_parent": true, "parent_balance_usd": 2.81 }
```

The unspent remainder returns to the parent. Burn is replay-safe — a second
call reports nothing left to burn rather than double-sweeping.

`swept_to_parent: false` means nothing was returned: either the key had no
parent, or the parent is no longer active. A key bought or granted directly has
no parent, so burning it strands whatever is left. That is deliberate — there
is nowhere to send it that would not mean recording who the key belongs to.

Two other credit moves:

- `POST /v1/key/topup` — `{"api_key": "sk-phantom-child", "amount_usd": 0.25}`.
  Only works on a key this one funded. A worker that runs long can be extended
  rather than re-minted.
- `POST /v1/key/merge` — `{"api_key": "sk-phantom-just-bought"}` folds another
  key's credit into the calling one and retires it. This is how you top up a
  key that already exists, since a purchase always issues a new key. The later
  of the two expiries wins.

## When a call is refused

Read the status before retrying. They mean different things:

| Status | `code` | What to do |
| --- | --- | --- |
| 401 | `invalid_key` | The key is wrong, burned, or never existed. Do not retry — get a key. |
| 402 | `insufficient_balance` | Out of credit, over the monthly budget, or expired. Top up, merge, or ask the parent. Retrying changes nothing. |
| 429 | `spend_rate_exceeded` | The per-minute cap. The key is fine, the minute is spent. Honour `Retry-After` — it says exactly how many seconds. |

A 429 here is the cap doing its job, not an outage. If you see it repeatedly on
a worker, the worker is looping — stop it rather than raising the cap.

## Checking what you were served

A gateway decides which model answered and how many tokens it used, and you
cannot see either. Every priced response carries a signed statement of both:

```
x-phantom-receipt: <base64url payload>.<base64url signature>
```

Streams cannot use a header, so the same payload arrives as the final SSE
frame, `event: phantom.receipt`, immediately before `data: [DONE]`.

The payload names `model_requested` and `model_served`, prompt/completion/
reasoning token counts, the cost in integer micro-USD, and SHA-256 hashes of
the request and response. No prompt or completion text.

Verify it yourself — Phantom publishes the key rather than an `is_valid` route,
so a verdict never requires trusting the thing you are auditing:

```js
const [payload, signature] = receipt.split('.');
const { public_key_jwk } = await fetch('https://phantom.codes/v1/receipts/key').then(r => r.json());
const key = await crypto.subtle.importKey('jwk', public_key_jwk, { name: 'Ed25519' }, false, ['verify']);
const ok = await crypto.subtle.verify({ name: 'Ed25519' }, key, b64url(signature), b64url(payload));

const claim = JSON.parse(new TextDecoder().decode(b64url(payload)));
claim.model_served === claim.model_requested;  // the check that usually matters
```

Worth checking when the model identity matters — an eval, a benchmark, a
customer-facing claim about which model ran, a bill you are reconciling. Not
worth checking on every chat turn.

Receipts are never stored server-side. Keep the ones you may need, or anchor
one to Solana for $0.01 with `POST /v1/receipts/anchor`, which fixes the time
it existed. `https://phantom.codes/verify` does the whole check in a browser.

**What a valid signature does not prove:** `model_served` is read from the
upstream's own response body, so it proves what the upstream reported, not that
the upstream was honest. Say it that way if you are reporting on it.

## Getting a key in the first place

- A human can buy one at `https://phantom.codes/checkout`.
- Programmatically: `POST /v1/purchase/native` returns an address to pay, a
  `payment_id` and a `recovery_code`. Poll `GET /v1/purchase/{payment_id}/status`
  with the code in the **`x-phantom-recovery-code` header** — not a query
  parameter — and the key comes back once the payment confirms. A poll without
  the header returns status and no key, which is the usual reason a caller
  thinks the purchase failed.
- No key at all: `POST /api/v1/x402/chat/completions` answers `402` with a
  price and an address, and serves the completion once payment is proved in
  `X-PAYMENT`. Note this rail is **testnet only** right now, so treat it as
  something to integrate against rather than a way to buy real inference today.

## The CLI

`phantom-key` wraps the key routes for shell use. JSON on stdout by default,
`--table` for reading, and exit code 2 specifically means the key was rejected
— so a script can branch on 2 as "get a new key" rather than "retry".

```bash
export PHANTOM_API_KEY=sk-phantom-...
npx phantom-key balance
npx phantom-key child --amount 0.50 --ttl 6 --rate 0.10
npx phantom-key burn
```

Also `budget get|set|clear`, `topup`, `merge`, `rotate`. `PHANTOM_BASE_URL`
overrides the host.

## Things that will bite you

- **A child key is shown once.** Capture it from the mint response. There is no
  "show me that key again".
- **Rotating does not reset a cap.** The balance, both caps, the spent figures
  and the children all travel to the new key. `POST /v1/key/rotate` is for
  replacing a credential you think leaked, not for clearing a ceiling.
- **`topup` only reaches your own children.** Credit cannot be moved between
  two keys that never funded each other. Use `merge` for that.
- **Don't build a key registry keyed by user.** The whole point is that
  `api_keys` holds no identity. Two identity-to-key links do exist and are
  disclosed rather than hidden — claiming the free grant, and asking bought
  credit to land on a key you already hold. `GET /v1/privacy` reports whether
  either names your key, by counting rows rather than asserting. Send that
  endpoint rather than repeating a privacy claim you cannot check.
- **No prompt or completion text is written by the API.** If a task needs an
  audit trail of content, you have to keep it yourself.

Full route list, request and response shapes: `references/endpoints.md`, and
`https://phantom.codes/docs` for curl for every endpoint.
