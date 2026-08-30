# Context Trimmer — Product Overview

## Problem

AI agents waste money on tokens they don't need.

When an agent calls an external API — a search engine, a database, a weather service, a CRM — it gets back a raw payload. That payload gets stuffed directly into the LLM context window. Most of it is structural noise: repeated key names across array rows, pretty-print whitespace, URL fields, API metadata objects. But critically: null fields and empty arrays are **not** noise — they carry meaning.

A typical API response is 2,000–10,000 tokens. A typical agent makes dozens of calls per session. The token cost of raw payloads often exceeds the cost of the actual reasoning.

**The agent doesn't need raw JSON. It needs the same information in fewer tokens — with zero context lost.**

---

## What We Do

A single HTTP endpoint agents call between fetching data and passing it to the LLM. Send a raw API payload, get back a token-optimized version that preserves 100% of the information. No setup, no SDK, no config. Pay per call in USDC via x402.

---

## Design Principle: Lossless by Default

We do **not** strip null values or empty arrays. An agent needs to know:
- `customer_phone: null` → the field exists; this customer has no phone on file
- `discounts: []` → discounts are supported; none were applied (different from null)

These are semantically different from a field being absent entirely. Stripping them loses context an agent may need to make correct decisions.

**Only tracking/analytics keys are removed** (`utm_*`, `fbclid`, `_ga`, `__utm`, `msclkid`, etc.) — these are never useful to agents.

---

## Output Format Design

Based on research across 11 format benchmarks and live token tests, the format that gives agents the most information per token is:

### Scalar fields — pipe-grouped key:value lines
```
id: in_1234 | status: paid | amount_due: 4999 | currency: usd
customer_email: jane@acme.com | livemode: false
null: customer_phone, customer_address, application, footer, discount
empty: discounts, total_discount_amounts, tax_amounts
```
- Present values: up to 4 short pairs per line, long values on their own line
- Null fields: one `null: field1, field2, ...` line — ~1.5 tokens/field vs ~4 tokens for individual lines
- Empty arrays/objects: separate `empty: field1, field2, ...` line — preserves the type distinction (null ≠ empty array)

### Arrays of objects — markdown tables
```
| id | description | amount | quantity | type |
| --- | --- | --- | --- | --- |
| il_1 | Pro Plan | 2999 | 1 | subscription |
| il_2 | Usage fee | 2000 | 200 | invoiceitem |
```
- Eliminates repeated key names across rows (biggest single token saving for list data)
- Null cells shown explicitly as `null` — agent knows the field exists per row
- Columns absent from all rows are dropped entirely

### API envelope unwrapping
Stripe-style `{ data: [...] }` wrappers and similar (`results`, `items`, `records`, `list`) are transparently unwrapped — the array is treated as a direct field value.

### Research backing
| Format | Token cost | LLM accuracy |
|--------|------------|--------------|
| Pretty JSON (baseline) | 100% | baseline |
| Compact JSON | −37% | good |
| Markdown table (flat data) | −54% | 51.9% |
| CSV | −60% | 44.3% (worst) |
| Key:value grouped | best accuracy/token balance | 60.7% |

Key finding: CSV is cheapest but has the worst comprehension. Markdown tables are the best balance for list data. Key:value lines outperform JSON on accuracy per token.

---

## Two Tiers

### Tier 1 — `POST /v1/clean` ($0.001)
**Lossless. No LLM. ~200ms.**

Deterministic structural reformatting. Removes only tracking keys. Everything else preserved.

**Input:**
```json
{ "data": <string | object | array>, "format"?: "auto" | "json" | "markdown" | "text" }
```

**Output:**
```json
{
  "output": "<formatted string>",
  "format": "mixed | markdown_table | compact_json | text",
  "inputTokens": 926,
  "outputTokens": 654,
  "reductionPct": 29
}
```

**Auto-routing logic:**
- Top-level array of objects → markdown table
- Object with nested arrays (incl. Stripe envelopes) → mixed (key:value scalars + tables)
- Flat object → compact key:value lines
- String → whitespace normalization + HTML entity decoding

**Measured token savings:**
- Stripe invoice (926 tokens raw): → 654 tokens (−29%)
- Transaction array (213 tokens raw): → 126 tokens (−41%)
- Flat API response with many nulls: → up to −64% (tracking keys + whitespace removed)

---

### Tier 2 — `POST /v1/compress` ($0.008)
**LLM-powered. Preserves all facts. ~800ms.**

Runs Tier 1 first (to reduce LLM input cost), then passes the pre-cleaned output through Claude Haiku 4.5 with an engineered system prompt.

**Input:**
```json
{ "data": <string | object | array>, "context"?: "<optional hint about what this data is>" }
```

**Output:** same shape as Tier 1.

**System prompt design (the moat):** The model is explicitly instructed to:
- Preserve every fact, number, ID, name, date, amount, currency, status, URL, and relationship
- Remove verbosity, redundant restatements, and filler phrases
- Never infer, synthesise, or omit a unique piece of data
- When in doubt: keep it

**Target reduction:** 50–80% on top of Tier 1 output. A 926-token Stripe invoice → ~185–330 tokens after both tiers.

> The system prompt is the moat. Model choice is not.

---

## How It Works

### End-to-End Request Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        AGENT PIPELINE                           │
│                                                                 │
│  1. Agent needs data                                            │
│     e.g. "fetch latest Stripe invoice for customer X"          │
│                    │                                            │
│                    ▼                                            │
│  2. Agent calls external API                                    │
│     GET https://api.stripe.com/v1/invoices?customer=cus_X      │
│     ← returns ~900-token pretty-printed JSON blob              │
│                    │                                            │
│                    ▼                                            │
│  3. Agent calls Context Trimmer                                 │
│     POST https://api.contexttrim.dev/v1/clean                  │
│     body: { "data": <raw API response> }                       │
│                    │                                            │
│           ┌────────┴────────┐                                   │
│           │  x402 PAYMENT   │                                   │
│           │                 │                                   │
│           │ First call:     │                                   │
│           │  → Server returns HTTP 402 + payment challenge      │
│           │  → Agent signs USDC authorization (no gas needed)  │
│           │  → Agent retries with X-Payment header             │
│           │  → Facilitator verifies + settles on Base          │
│           │  → Server returns 200                              │
│           │                 │                                   │
│           │ Subsequent calls:                                   │
│           │  → Single round trip, no 402 step                  │
│           └────────┬────────┘                                   │
│                    │                                            │
│                    ▼                                            │
│  4. Context Trimmer processes payload                           │
│                                                                 │
│     Tier 1 /v1/clean (no LLM):                                 │
│     ┌─────────────────────────────────────────┐                │
│     │ Remove tracking keys only               │                │
│     │ Route to optimal format (table/kv/json) │                │
│     │ Group nulls + empty fields compactly    │                │
│     │ ~200ms · fully lossless · 29–64% less  │                │
│     └─────────────────────────────────────────┘                │
│                                                                 │
│     Tier 2 /v1/compress (Haiku 4.5):                           │
│     ┌─────────────────────────────────────────┐                │
│     │ Run Tier 1 first (reduces LLM cost)     │                │
│     │ Haiku 4.5 with fact-preserving prompt   │                │
│     │ · Keep every fact, ID, number, status   │                │
│     │ · Cut verbosity & redundancy            │                │
│     │ ~800ms · 50–80% smaller                │                │
│     └─────────────────────────────────────────┘                │
│                    │                                            │
│                    ▼                                            │
│  5. Agent receives optimized payload                            │
│     { output, format, inputTokens, outputTokens, reductionPct }│
│                    │                                            │
│                    ▼                                            │
│  6. Agent passes output to LLM                                  │
│     Saves ~$0.015 in Claude Sonnet tokens per Stripe call      │
│     vs $0.001 spent on /v1/clean = 15x positive ROI            │
└─────────────────────────────────────────────────────────────────┘
```

### x402 Payment Flow (Detail)

```
Agent                    Context Trimmer             Facilitator (Coinbase CDP)
  │                            │                              │
  │── POST /v1/clean ─────────▶│                              │
  │                            │                              │
  │◀─ HTTP 402 ────────────────│                              │
  │   { accepts: [             │                              │
  │       { network: "eip155:8453", price: "$0.001",         │
  │         payTo: "0x..." },  │                              │
  │       { network: "eip155:1", ... },                      │
  │       { network: "eip155:137", ... },                    │
  │       { network: "eip155:42161", ... }                   │
  │     ] }                    │                              │
  │                            │                              │
  │  [Agent picks whichever chain it has USDC on]            │
  │  [Signs EIP-3009 authorization — no gas required]        │
  │                            │                              │
  │── POST /v1/clean ─────────▶│                              │
  │   X-Payment: <signed auth> │                              │
  │                            │── verify + settle USDC ────▶│
  │◀─ HTTP 200 ────────────────│                              │
  │   { output, format,        │                              │
  │     inputTokens: 926,      │                              │
  │     outputTokens: 654,     │                              │
  │     reductionPct: 29 }     │                              │
```

No persistent storage. No data retention. Each request is stateless.

---

## What We Are Not

**Not Headroom Labs.** Headroom (25k GitHub stars, open source) is a local-first tool that compresses the *entire context window* on a developer's machine using ML (ModernBERT). It targets developers running local coding agents. We are a cloud API for agents calling external services — zero setup, pay per use, x402-native. Different use case, different customer, not competing.

---

## Infrastructure

| Layer | Choice | Why |
|-------|--------|-----|
| Runtime | Node.js + TypeScript + Hono | Lightweight, edge-compatible |
| Hosting | Vercel free tier → Railway $5/mo | Start free; migrate only if 10s execution limit is hit on large payloads |
| Payment | x402 (USDC) via Coinbase CDP facilitator | Native agent payments, no subscription friction |
| Chains | Base, Ethereum, Polygon, Arbitrum (testnet: Base Sepolia only) | Same 0x address works on all EVM chains — no extra wallet |
| Chain switching | Automatic — detected from `FACILITATOR_URL` env var | x402.org URL = testnet (Base Sepolia); CDP URL = mainnet (all 4 chains) |
| LLM (Tier 2) | Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) | $0.80/M input + $4/M output; ~$0.004 per average call |
| DNS/CDN | Cloudflare | Free DDoS, SSL termination |

---

## Unit Economics

### Tier 1 — `/v1/clean`
| Item | Cost |
|------|------|
| Hosting + gas settlement | ~$0.0002/call |
| LLM | None |
| Price charged | $0.001 |
| **Net margin** | **~80%** |

### Tier 2 — `/v1/compress`
| Item | Cost |
|------|------|
| Hosting + gas settlement | ~$0.0002/call |
| Haiku 4.5 (avg 2K pre-cleaned input / 600 output tokens) | ~$0.004/call |
| Price charged | $0.008 |
| **Net margin** | **~45–50%** |

At 10,000 calls/day (70% clean / 30% compress): ~**$350/month revenue**, ~**$180/month margin**.

Agent ROI on Tier 1 with Sonnet 4.5: pay $0.001 to clean, save $0.015 in Sonnet input tokens = **15x positive ROI**.

---

## Monetisation

- **x402 micropayments** — no subscriptions, no API keys, no accounts. Agents pay per call in USDC on whichever EVM chain they hold funds.
- **Volume pricing (future)** — agents pre-fund a channel via x402 batch-settlement and pay a lower per-call rate.
- **White-label / embedded (future)** — other agent framework builders embed the endpoint; take a revenue share.

---

## Launch Checklist

### 1. Get a Wallet Address
- [ ] Create a wallet in [Coinbase Wallet](https://wallet.coinbase.com) or MetaMask
- [ ] Copy your `0x...` EVM address — same address works on Base, Ethereum, Polygon, and Arbitrum
- [ ] Save it somewhere safe — this is where all USDC payments land

### 2. Get an Anthropic API Key (for Tier 2)
- [ ] Go to [console.anthropic.com](https://console.anthropic.com) → API Keys → Create key
- [ ] Save the key — needed for `/v1/compress` (Haiku 4.5 LLM compression)
- [ ] Without this, only `/v1/clean` (Tier 1) works

### 3. Local Testnet Setup
- [ ] `cp .env.example .env`
- [ ] Set `RESOURCE_WALLET_ADDRESS=0xYourAddress` in `.env`
- [ ] Set `ANTHROPIC_API_KEY=sk-ant-...` in `.env`
- [ ] Keep `FACILITATOR_URL=https://x402.org/facilitator` — server auto-uses Base Sepolia testnet when this URL is set
- [ ] Get free testnet USDC at [faucet.coinbase.com](https://faucet.coinbase.com) (select Base Sepolia)
- [ ] `npm run dev` — confirm `GET /health` returns `{"status":"ok"}`
- [ ] Confirm `GET /llms.txt` returns the endpoint documentation
- [ ] Send a request to `/v1/clean` without payment — confirm HTTP 402 response
- [ ] Send a paid test request and confirm end-to-end payment flow works

### 4. Switch to Mainnet
- [ ] In `.env`: set `FACILITATOR_URL=https://api.cdp.coinbase.com/platform/v2/x402`
- [ ] The server automatically enables all 4 chains (Base, Ethereum, Polygon, Arbitrum) — no code changes needed
- [ ] Fund your wallet with real USDC on Base (via [Coinbase](https://coinbase.com) or any exchange with Base withdrawals)
- [ ] Test one real paid request to confirm mainnet settlement works

### 5. Deploy to Vercel (free tier — start here)
- [ ] `git init && git add . && git commit -m "Initial release"`
- [ ] Push to GitHub: `gh repo create token-api --private --push --source=.`
- [ ] Go to [vercel.com](https://vercel.com) → New Project → Import from GitHub → select `token-api`
- [ ] Add env vars in Vercel dashboard (Settings → Environment Variables): `RESOURCE_WALLET_ADDRESS`, `FACILITATOR_URL`, `ANTHROPIC_API_KEY`
- [ ] Deploy and confirm `GET /health` responds on the Vercel URL
- [ ] If large payloads hit the 10s execution limit → migrate to Railway ($5/mo) instead (see `railway.toml`)

### 6. Domain & CDN
- [ ] Buy a domain at [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/) (e.g. `contexttrim.dev`)
- [ ] In Cloudflare DNS: add CNAME `api` → your Vercel deployment URL
- [ ] Enable Cloudflare proxy (orange cloud) — free DDoS protection + SSL
- [ ] Confirm `https://api.contexttrim.dev/health` responds

### 7. Distribute
- [ ] Decide on final product name (see Open Questions below)
- [ ] Confirm `GET /llms.txt` is publicly accessible on your live domain
- [ ] Publish benchmarks in README: exact token savings on Stripe, GitHub, weather API, etc.
- [ ] Submit to [Circle Agent Marketplace](https://www.circle.com/products/agent-marketplace)
- [ ] Post in [x402 community Slack](http://slack.x402.org)

---

## Open Questions

- [ ] Naming: "Context Trimmer" is a working name — decide before listing on Circle Marketplace
- [ ] Should `/v1/compress` accept a `max_tokens` target parameter so agents can specify a ceiling?
- [ ] Add constant-column hoisting to tables: columns where all rows have the same value can be moved to a header note to save more tokens
- [ ] Tier 2 system prompt needs testing on diverse real-world payloads before launch — this is the core product risk
