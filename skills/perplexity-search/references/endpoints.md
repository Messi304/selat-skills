# Endpoints — perplexity-search

Perplexity's API exposed as x402 endpoints by the **paysponge gateway**
(`pplx.x402.paysponge.com`), paid per call via selat-pay (USDC on Base), **routed**
through the SELAT Router. No API key.

Schemas below are pinned from the gateway's own OpenAPI (`GET
https://pplx.x402.paysponge.com/openapi.json`, title "Perplexity AI API", OpenAPI
3.1.0) and corroborated against Perplexity's public docs. Prices are the catalogue
list price; the live routed quote runs a few percent higher (router margin).

## The step this skill wires

| # | Step | Method | URL | Rail | ~Price |
|---|---|---|---|---|---|
| 1 | Web search | POST | `https://pplx.x402.paysponge.com/search` | routed x402 | $0.01 (live ≈ $0.0105) |

`maxAmount`: full-run cap **$0.03** (headroom over the ~$0.0105 live quote).

## All Perplexity endpoints on this gateway (enriched)

| Endpoint | Method | ~Price | Poll? |
|---|---|---|---|
| `/search` | POST | $0.01 | — |
| `/v1/sonar` | POST | $0.10 | — |
| `/v1/agent` | POST | $0.01 | — |
| `/v1/async/sonar` | POST | $0.01 | returns task id |
| `/v1/async/sonar/{api_request}` | GET | **free** (`routed-free`) | poll target |
| `/v1/models` | GET | — | list models |

### `POST /search` — Search the Web  ($0.01)

Body (`ApiSearchRequest`):

| Field | Req | Type | Values / default |
|---|---|---|---|
| `query` | ✅ | string \| string[] | the search query (or a batch of queries) |
| `max_results` | | integer | 1–20, default 10 |
| `search_context_size` | | string | `low` \| `medium` \| `high` (default `high`); conflicts with `max_tokens*` |
| `max_tokens` / `max_tokens_per_page` | | integer | 1–1,000,000 |
| `country` | | string | ISO 3166-1 alpha-2 |
| `search_recency_filter` | | string | `hour` \| `day` \| `week` \| `month` \| `year` |
| `search_domain_filter` | | string[] | ≤20 domains |
| `search_language_filter` | | string[] | ≤20 ISO 639-1 codes |
| `search_after_date_filter` / `search_before_date_filter` | | string | `MM/DD/YYYY` |
| `last_updated_after_filter` / `last_updated_before_filter` | | string | `MM/DD/YYYY` |

```bash
selat-pay POST "https://pplx.x402.paysponge.com/search" \
  --body '{"query":"latest x402 agentic payments adoption","max_results":8,"search_recency_filter":"month"}' \
  --chain base --max-amount 0.03
```

> Manifest note: the skill wires only the string-typed fields (`query`,
> `search_recency_filter`) because the skill runner substitutes `${param}` as a
> **string**. Numeric fields like `max_results` must be sent as real integers via a
> hand-built `selat-pay` call, not through `${…}` substitution.

### `POST /v1/sonar` — Create Chat Completion (synchronous answer)  ($0.10)

> ⚠ **Quote works, but a paid call currently fails with `431` — do not use yet.**
> Live-tested 2026-07-26: the free probe returns a quote at **$0.105** (upstream
> $0.10 + 5% routed markup), but the **paid** call returns `HTTP 431 Request Header
> Fields Too Large` and is **charged-but-not-delivered** ($0.105 lost, no chargeback).
>
> Root cause — the same 17 KB schema, now on the *request* leg. `/v1/sonar` serves an
> x402 **v2 `upto`/`permit2`** 402 whose `payment-required` header is **~17 KB** (it
> embeds the full 26-field body schema). The response-header fix (selat-pay 0.9.2 /
> router redeploy, raising `--max-http-header-size` to 64 KB) lets both legs *read*
> that challenge — which is why the quote now succeeds. But to pay the `upto`
> upstream, the router's outbound `X-PAYMENT` request header echoes the requirements,
> ballooning past **paysponge/Cloudflare's ~8–16 KB request-header limit** → `431`.
> The router can't fix this by raising its own cap (the limit is upstream's).
>
> Fix needs one of: **paysponge** moving the schema out of the HTTP header into the
> body (real fix), or the **router** trimming the echoed requirements from the
> outbound `upto` payment payload. Until then, use `/search` + agent synthesis,
> `/v1/agent`, or async `sonar-deep-research` — all confirmed working.
>
> (Note: selat-pay only ever signs `GatewayWalletBatched`; the Router adapts the
> outbound rail — for `/v1/sonar` it pays the `upto`/`permit2` upstream. The scheme
> was never the blocker; header size is, on both the response and now the request.)

Raw body (`ApiChatCompletionsRequest`) — **no wrapper**:

| Field | Req | Type | Values |
|---|---|---|---|
| `model` | ✅ | string | enum: `sonar`, `sonar-pro`, `sonar-deep-research`, `sonar-reasoning-pro` |
| `messages` | ✅ | array | `[{ "role": "system\|user\|assistant\|tool", "content": "…" }]` |
| `max_tokens` | | integer | completion cap |
| `temperature` / `top_p` | | number | sampling |
| `search_mode` | | string | `web` \| `academic` \| `sec` |
| `search_recency_filter` | | string | `hour` \| `day` \| `week` \| `month` \| `year` |
| `search_domain_filter` | | string[] | ≤ domains |
| `search_after_date_filter` / `search_before_date_filter` | | string | `MM/DD/YYYY` |
| `reasoning_effort` | | string | reasoning budget |
| `return_related_questions` / `return_images` | | boolean | |
| `response_format` | | object | text or JSON-schema structured output |

```bash
selat-pay POST "https://pplx.x402.paysponge.com/v1/sonar" \
  --body '{"model":"sonar","messages":[{"role":"user","content":"What is the latest on x402 adoption? Cite sources."}]}' \
  --chain base --max-amount 0.15
```

### `POST /v1/agent` — Create Agent Response  ($0.01)  ✓ verified 200 (2026-07-25)

Payable via selat-pay (`exact` / routed-x402, ~$0.0105). Body:

| Field | Req | Type | Notes |
|---|---|---|---|
| `input` | ✅ | string \| InputItem[] | the task/prompt |
| **one of** `model` / `models` / `preset` | ✅ | — | **Required — the OpenAPI marks only `input`, but the live API rejects a body without one of these:** `400 "validation failed: model, models, or preset is required"`. |
| `model` | ➊ | string | `provider/model` form, e.g. `openai/gpt-5.4-mini`. |
| `models` | ➊ | array | fallback chain. |
| `preset` | ➊ | string | e.g. `fast-search` (simplest — used in the verified call below). |
| `tools` | | array | tools available to the model |
| `max_steps` | | integer | research-loop cap |
| `instructions` | | string | system instructions |
| `response_format` | | object | structured output |
| `stream` | | boolean | SSE if true |

_➊ = supply exactly one of `model`/`models`/`preset`._

```bash
selat-pay POST "https://pplx.x402.paysponge.com/v1/agent" \
  --body '{"input":"Summarize this week x402 news with sources.","preset":"fast-search"}' \
  --chain base --max-amount 0.03
```

### `POST /v1/async/sonar` — Create Async Chat Completion (deep research)  ($0.01)

Body — **wraps the chat request**:

| Field | Req | Type | Notes |
|---|---|---|---|
| `request` | ✅ | ApiChatCompletionsRequest | same shape as `/v1/sonar` body — but `model` **must** be `sonar-deep-research` |
| `idempotency_key` | | string | reuse to prevent a duplicate (costly) submission on retry |

Returns `{ "id": "<uuid>", "status": "CREATED", "response": null, … }`.

```bash
# 1. kick off (paid ~$0.01)
selat-pay POST "https://pplx.x402.paysponge.com/v1/async/sonar" \
  --body '{"request":{"model":"sonar-deep-research","messages":[{"role":"user","content":"Deep-research the latest x402 / agentic-payments adoption with sources."}]},"idempotency_key":"x402-adoption-2026-07"}' \
  --chain base --max-amount 0.03
```

### `GET /v1/async/sonar/{api_request}` — poll the async task  (**free**)

Path param `api_request` = the `id` from the POST. Free passthrough
(`mode=routed-free`, $0). Poll every ~15s until `status` is `COMPLETED` (or `FAILED`).

```bash
# 2. poll (free) until COMPLETED, then read response.choices[0].message.content
selat-pay GET "https://pplx.x402.paysponge.com/v1/async/sonar/<id>" \
  --chain base --max-amount 0.01
```

On completion, the answer is at `response.choices[0].message.content` and sources at
`response.search_results` (array of `{title,url}`) / `response.citations`.

## Rails & providers

- **routed x402** — every endpoint serves a native x402 challenge on
  `pplx.x402.paysponge.com`; the SELAT Router settles it (`mode=routed-x402`,
  `GatewayWalletBatched`, USDC on Base `eip155:8453`; Solana also advertised). The poll
  GET is served free (`mode=routed-free`).
- **Not MPP.** These endpoints are indexed only in the `agentic` catalogue and serve
  x402 exclusively — there is no MPP/Tempo route for Sponge/paysponge.

## Live probes (free; no wallet)

```bash
selat-pay POST "https://pplx.x402.paysponge.com/search" \
  --body '{"query":"agent payments","search_recency_filter":"month"}' --chain base --probe-only
selat-pay POST "https://pplx.x402.paysponge.com/v1/async/sonar" \
  --body '{"request":{"model":"sonar-deep-research","messages":[{"role":"user","content":"ping"}]}}' --chain base --probe-only
```

Both print `detected x402=yes … mode=routed-x402 price=$0.0105 on eip155:8453`. Probing
verifies **payability only** — it does not validate the body, so match the schemas above.
