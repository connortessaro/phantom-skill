# Phantom endpoints

Base URL `https://phantom.codes/v1`. Every route also answers on `/api/v1/…`;
the `/v1` prefix is the documented one.

Auth is `Authorization: Bearer sk-phantom-…` unless a row says otherwise.
`https://phantom.codes/docs` has a copyable curl for every one of these.

## Contents

- [Inference](#inference)
- [Keys](#keys)
- [Verification](#verification)
- [Buying credit](#buying-credit)
- [Account](#account)
- [Response headers](#response-headers)
- [Error shapes](#error-shapes)

## Inference

| Method | Path | Notes |
| --- | --- | --- |
| POST | `/chat/completions` | Streaming and non-streaming. Usage is captured mid-stream, so a stream that fails part way is billed for what it delivered and refunded for what it did not. |
| POST | `/embeddings` | |
| POST | `/rerank` | |
| POST | `/images/generations` | |
| POST | `/audio/speech` | |
| POST | `/audio/transcriptions` | multipart |
| POST | `/video/generations` | Enqueues a job, returns an id. The receipt lands on the poll, not here — this response holds a reservation, not a result. |
| GET | `/video/generations/{id}` | Poll. Carries the receipt once complete. |
| GET | `/models` | **No key.** Live catalog and per-token rates. |
| GET | `/` | **No key.** Machine-readable service index: every route, including the key lifecycle. Send `Accept: application/json`. |
| POST | `/api/v1/x402/chat/completions` | **No key.** Pay per call over x402: unpaid POST answers 402 with a price and address, repeat with the proof in `X-PAYMENT`. Flat price, pinned model, no streaming. Testnet only at present. Use the `/api/v1` prefix here. |

## Keys

| Method | Path | Body | Returns |
| --- | --- | --- | --- |
| GET | `/key/balance` | — | `active`, `kind`, `credit_balance_usd`, `credit_spent_usd`, `expires_at` |
| GET | `/key/budget` | — | current caps |
| PATCH | `/key/budget` | `{"budget_usd": 25, "rate_usd_per_min": 0.10}` | `null` clears either |
| POST | `/key/child` | `{"amount_usd": 0.5, "ttl_hours": 6, "budget_usd": null, "rate_usd_per_min": 0.1}` | `api_key` (once), `funded_usd`, `expires_at`, `budget_usd`, `rate_usd_per_min`, `parent_balance_usd` |
| POST | `/key/topup` | `{"api_key": "sk-phantom-child", "amount_usd": 0.25}` | Target must be a key this one funded |
| POST | `/key/merge` | `{"api_key": "sk-phantom-other"}` | Folds that key in and retires it. Later expiry wins |
| POST | `/key/rotate` | — | New key; balance, expiry, both caps, spent figures and children all travel |
| DELETE | `/key` | — | `revoked`, `swept_usd`, `swept_to_parent`, `parent_balance_usd` |
| GET | `/privacy` | — | Accepts a key, a session cookie, or both. Counts live rows |

`ttl_hours` defaults to 24 and cannot exceed a year. `amount_usd`,
`budget_usd` and `rate_usd_per_min` must be positive numbers; the cap fields
also accept `null`.

### /privacy

Answers for the key you present rather than asserting a policy:

```json
{
  "tier": "anonymous",
  "key": {
    "identity": null,
    "identity_linkable": false,
    "prompts_stored": 0,
    "usage_rows": 41,
    "payment_linkable": false
  },
  "invariants": {
    "prompts_written_by_api": false,
    "key_has_identity_column": false,
    "payment_to_key_join_exists": true,
    "identity_to_key_join_exists": true
  }
}
```

The two `*_join_exists` flags are true because those joins exist in the schema
— claiming the free grant writes one, and asking bought credit to land on an
existing key writes the other. Whether either names *your* key is
`identity_linkable` / `payment_linkable`.

## Verification

| Method | Path | Notes |
| --- | --- | --- |
| GET | `/receipts/key` | **No key.** Ed25519 public key as JWK and PEM, plus `signs: true/false` |
| POST | `/receipts/anchor` | Writes the receipt's leaf hash to Solana. $0.01. Anchoring the same leaf twice is free |
| GET | `/receipts/anchor?leaf=<sha256>` | **No key.** Look up an anchor |

Receipt payload fields:

```
v, request_id, ts, model_requested, model_served, upstream,
upstream_request_id, prompt_tokens, completion_tokens, reasoning_tokens,
cost_micro_usd, params_hash, input_hash, output_hash
```

The leaf hash is SHA-256 over the compact `payload.signature` string exactly as
received. `v` is pinned — a verifier refuses a receipt from another version, so
an old receipt stops verifying if the payload format changes.

## Buying credit

| Method | Path | Notes |
| --- | --- | --- |
| GET | `/bundles` | **No key.** Purchase bounds and presets |
| GET | `/coins` | **No key.** Accepted crypto |
| POST | `/purchase/native` | Returns an address, `payment_id` and `recovery_code`. No hosted invoice |
| POST | `/purchase` | Hosted invoice instead |
| POST | `/purchase/card` | Stripe PaymentIntent |
| GET | `/purchase/{id}/status` | Poll. Send `x-phantom-recovery-code: <code>` **as a header** or the key is withheld |
| POST | `/purchase/recover` | Re-issue with a recovery code |

`target_api_key` on a purchase lands the credit on a key you already hold
instead of issuing a new one. That is the write that sets
`payment_linkable: true` for that key.

## Account

Session cookie, not a key. An autonomous agent will not use these.

| Method | Path | Notes |
| --- | --- | --- |
| GET/POST | `/account/grant` | Free developer grant, once per identity |
| DELETE | `/account` | Erases grants and referrals, revokes sessions. API keys survive by design |

## Response headers

| Header | On | Meaning |
| --- | --- | --- |
| `x-phantom-cost-usd` | every priced call | What this call cost |
| `x-phantom-balance-usd` | every priced call | Credit left after billing. Absent means the debit found no key to charge — not zero |
| `x-phantom-receipt` | priced calls | `<base64url payload>.<base64url signature>` |
| `x-phantom-upstream` | priced calls | Which upstream served it |
| `x-phantom-content-logged` | priced calls | Always `false` |
| `Retry-After` | 429 | Seconds until the rate window reopens |

Two priced calls carry no receipt: `POST /video/generations`, where the receipt
lands on the poll instead, and `POST /receipts/anchor`.

## Error shapes

Refusals follow the OpenAI error envelope:

```json
{ "error": { "message": "…", "type": "rate_limit_error", "code": "spend_rate_exceeded" } }
```

| Status | `code` | Cause |
| --- | --- | --- |
| 401 | `invalid_key` | No such key, or it was burned |
| 402 | `insufficient_balance` | Out of credit, over the monthly budget, inactive, or expired |
| 429 | `spend_rate_exceeded` | Per-minute cap. `Retry-After` says how long |
| 404 | — | Unknown model id. Check `GET /models` |
| 502 | — | Upstream failed. Anything already billed is refunded |
