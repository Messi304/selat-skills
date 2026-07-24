# Endpoints — social-intel

Cross-platform social intelligence over the SELAT federated catalogues. Web search
and Reddit are **routed** through the SELAT Router (x402 + MPP). Paid per call via
selat-pay (USDC), no API keys.

## Endpoints used

| # | Step | Method | URL | Rail | ~Price |
|---|---|---|---|---|---|
| 1 | Web context — Exa | POST | `https://api.exa.ai/search` | routed x402 | $0.007 |
| 2 | Web corroboration — Parallel | POST | `https://parallelmpp.dev/api/search` | routed MPP | $0.011 |
| 3 | Reddit keyword search — StableSocial | POST | `https://stablesocial.dev/api/reddit/search` (body `{"query":"${topic}"}`) | routed MPP | $0.063 |
| 4 | Reddit subreddit top posts — StableSocial | POST | `https://stablesocial.dev/api/reddit/subreddit` (body `{"subreddit":"${subreddit}"}`) | routed MPP | $0.063 |

Full-run cap (`maxAmount`): **$0.50**; per-step caps **$0.05** (web) and
**$0.20** (StableSocial Reddit steps). Live total ≈ $0.14 (prices probe-verified
2026-07-10).

## Rails & providers

- **routed x402** — Exa (`api.exa.ai`) serves a native x402 challenge; the router
  settles it (`mode=routed-x402`). Sourced from the Agentic Market / MPP catalogs.
- **routed MPP** — Parallel (`parallelmpp.dev`) and StableSocial (Reddit,
  a direct MPP merchant at `stablesocial.dev/api/reddit/...`) settle MPP through
  the SELAT Router (`mode=routed-mpp`). Sourced from the MPP catalog.

## Live probes (free; no wallet)

```bash
# web search (POST body)
selat-pay POST "https://api.exa.ai/search" \
  --body '{"query":"agent payments","numResults":5}' --chain base --probe-only
selat-pay POST "https://parallelmpp.dev/api/search" \
  --body '{"objective":"agent payments","search_queries":["agent payments"]}' --chain base --probe-only

# Reddit — StableSocial, routed MPP (POST body)
selat-pay POST "https://stablesocial.dev/api/reddit/search" \
  --body '{"query":"usdc"}' --chain base --probe-only
selat-pay POST "https://stablesocial.dev/api/reddit/subreddit" \
  --body '{"subreddit":"ethereum"}' --chain base --probe-only
```

A served endpoint prints `detected ... price=$X on eip155:8453`. The Exa step shows
`mode=routed-x402`; Parallel and the StableSocial (Reddit) steps show
`mode=routed-mpp`.
