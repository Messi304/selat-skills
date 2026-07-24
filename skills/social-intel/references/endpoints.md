# Endpoints — social-intel

Grounded web-context intelligence over two independent web-search engines, both
**routed** through the SELAT Router (native x402). Paid per call via selat-pay
(USDC), no API keys.

## Endpoints used

| # | Step | Method | URL | Rail | ~Price |
|---|---|---|---|---|---|
| 1 | Web context — Exa | POST | `https://api.exa.ai/search` (body `{"query":"${topic}",...}`) | routed x402 | $0.007 |
| 2 | Web corroboration — Tavily | POST | `https://x402.tavily.com/search` (body `{"query":"${topic}","search_depth":"advanced"}`) | routed x402 | $0.011 |

Full-run cap (`maxAmount`): **$0.10**; per-step cap **$0.05**. Live total ≈ $0.018
(prices probe-verified 2026-07-24).

## Rails & providers

- **routed x402 — Exa** (`api.exa.ai`) serves a native x402 challenge; the router
  settles it (`mode=routed-x402`). Neural/semantic search, returns page text.
- **routed x402 — Tavily** (`x402.tavily.com`) serves a native x402 challenge; the
  router settles it (`mode=routed-x402`, `GatewayWalletBatched`). Aggregation search
  with `search_depth: advanced`. Note: this is Tavily's own host — **not** the AIsa
  (`api.aisa.one/.../tavily/search`) or Locus
  (`tavily.mpp.paywithlocus.com`) re-hosts in the catalogue.

## Live probes (free; no wallet)

```bash
# web search (POST body)
selat-pay POST "https://api.exa.ai/search" \
  --body '{"query":"agent payments","numResults":5}' --chain base --probe-only
selat-pay POST "https://x402.tavily.com/search" \
  --body '{"query":"agent payments","search_depth":"advanced","max_results":5}' --chain base --probe-only
```

A served endpoint prints `detected ... price=$X on eip155:8453`. Both the Exa and
Tavily steps show `mode=routed-x402`.
