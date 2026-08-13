# Endpoints — financial-intel

Multi-signal financial intelligence across **three settlement modes** from the
SELAT federated catalogue: a Circle Gateway-batched nanopayment
(`x402 via Circle Gateway`, Alchemy), **MPP on Tempo** (CoinGecko, Alpha
Vantage, Exa), and **x402 on Base** (Nansen). Paid per call via selat-pay
(USDC via Circle Gateway), no API keys.

## Endpoints used

| # | Step | Method | URL | Rail | ~Price |
|---|---|---|---|---|---|
| 1 | Spot price — Alchemy | GET | `https://x402.alchemy.com/prices/v1/tokens/by-symbol?symbols=${symbol}` | x402 via Circle Gateway | $0.001 |
| 2 | Token market data — CoinGecko | POST | `https://coingecko.mpp.paywithlocus.com/coingecko/coins-markets` | MPP on Tempo | $0.063 |
| 3 | Equities / macro quote — Alpha Vantage | POST | `https://alphavantage.mpp.paywithlocus.com/alphavantage/global-quote` | MPP on Tempo | $0.0084 |
| 4 | On-chain smart-money holdings — Nansen | POST | `https://api.nansen.ai/api/v1/smart-money/holdings` | x402 on Base | $0.0525 |
| 5 | Market news / context — Exa | POST | `https://api.exa.ai/search` | MPP on Tempo | $0.00735 |

Full-run cap (`maxAmount`): **$1.00**; per-step caps range **$0.01–$0.10**. Live total ≈ $0.13.

## Rails & providers

This skill mixes a Circle Gateway nanopayment with MPP and x402 (`rail: mixed`).

- **Circle Gateway nanopayment** — Alchemy (`x402.alchemy.com`) serves an x402 challenge
  that settles as a Circle Gateway-batched nanopayment paid **straight to the
  upstream** (`x402 via Circle Gateway`), **no router hop**. This step does not require
  `SELAT_ROUTER_URL`.
- **MPP on Tempo** — CoinGecko (`coingecko.mpp.paywithlocus.com`), Alpha Vantage
  (`alphavantage.mpp.paywithlocus.com`), and Exa (`api.exa.ai`) settle MPP
  through the SELAT Router (`MPP on Tempo`). Sourced from the MPP catalog.
- **x402 on Base** — Nansen (`api.nansen.ai`) serves a native x402 challenge; the
  router settles it on Base (`x402 on Base`). Note: Nansen is listed in the MPP
  registry on Tempo, but the live host serves x402-only challenges (no MPP
  `WWW-Authenticate`) — the manifest follows the live 402, not the registry.

## Live probes (free; no wallet)

```bash
# Circle Gateway nanopayment (GET query)
selat-pay GET "https://x402.alchemy.com/prices/v1/tokens/by-symbol?symbols=ETH" \
  --chain base --probe-only

# MPP on Tempo (POST body)
selat-pay POST "https://coingecko.mpp.paywithlocus.com/coingecko/coins-markets" \
  --body '{"vs_currency":"usd","ids":"ethereum"}' --chain base --probe-only
selat-pay POST "https://alphavantage.mpp.paywithlocus.com/alphavantage/global-quote" \
  --body '{"symbol":"AAPL"}' --chain base --probe-only
selat-pay POST "https://api.exa.ai/search" \
  --body '{"query":"ethereum ETF flows","numResults":5}' --chain base --probe-only

# x402 on Base (POST body — note `chains` is an array)
selat-pay POST "https://api.nansen.ai/api/v1/smart-money/holdings" \
  --body '{"chains":["ethereum"]}' --chain base --probe-only
```

A served endpoint prints `detected ... price=$X on eip155:8453`. The Alchemy step
shows `x402 via Circle Gateway`; CoinGecko/Alpha Vantage/Exa show `MPP on Tempo`;
Nansen shows `x402 on Base`.
