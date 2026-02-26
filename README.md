# ODEI Web Platform — api.odei.ai

[![Build](https://img.shields.io/badge/build-passing-brightgreen)](https://api.odei.ai/health)
[![Uptime](https://img.shields.io/badge/uptime-99.9%25-brightgreen)](https://api.odei.ai/health)
[![API Status](https://img.shields.io/badge/API-v2.0.0-blue)](https://api.odei.ai/api/v2/health)
[![License](https://img.shields.io/badge/license-proprietary-lightgrey)]()

Product site, API gateway, and developer hub for ODEI's constitutional world model.

---

## What is ODEI?

ODEI is a constitutional AI system built on a persistent knowledge graph. The world model captures entities, relationships, guardrails, and decisions across six architectural domains (Foundation, Vision, Strategy, Tactics, Execution, Track) with 10,665+ nodes in production.

The web platform at **api.odei.ai** serves three roles:

1. **Product site** -- Public-facing pages for token economics, holder analytics, security monitoring, integration guides, and the interactive world model explorer.
2. **API gateway** -- Authenticated REST endpoints for querying the knowledge graph, running constitutional guardrail checks, and purchasing AI services via the x402 payment protocol.
3. **Developer hub** -- OpenAPI spec, integration documentation, Solana Actions (Blinks), and marketplace connectors for Virtuals ACP, Clawlancer, and Fetch.ai.

---

## Architecture

```
                        Internet
                           |
                        [nginx]
                       /       \
              static files    reverse proxy
              /srv/.../current    :8800
                  |                 |
           HTML/CSS/JS        api-server.js
           Tailwind CSS       (Node.js, no framework)
           World Model           |
           Explorer         api-v2-routes.js
                            (v2 endpoints)
                                 |
                           Neo4j Graph DB
                           (bolt://...)
```

**Stack:**
- **Reverse proxy:** nginx with TLS (Let's Encrypt)
- **API server:** Node.js (vanilla `http` module, zero framework dependencies)
- **Static site:** Hand-crafted HTML + Tailwind CSS + premium dark theme (jade `#4FD1C5` / champagne `#F4C95D`)
- **Graph database:** Neo4j (10,665+ nodes, 92 relationship types)
- **Payments:** x402 protocol on Base chain
- **Hosting:** Google Cloud VM (`34.18.x.x`), managed by systemd (`odei-api.service`)

---

## API Endpoints

All endpoints are served from `https://api.odei.ai`. Paid endpoints accept x402 payment proofs or API key authentication.

### Paid Endpoints (x402 / API Key)

| Endpoint | Method | Price | Description |
|----------|--------|------:|-------------|
| `/api/v2/guardrail/check` | POST | $0.10 | Constitutional validation against ODEI's guardrail framework |
| `/api/v2/world-model/query` | POST | $0.25 | Knowledge graph query with domain/text/traversal modes |
| `/api/v2/world-model/signal` | POST | $0.05 | Trust score and signal classification |
| `/api/v2/world-model/live` | GET | $0.25 | Full graph snapshot with filtering (domains, types, edges) |
| `/api/v2/smart-contract/audit` | POST | $5.00 | EVM smart contract security audit |
| `/api/v2/knowledge-graph/build` | POST | $15.00 | Custom knowledge graph construction |
| `/api/v2/memory/design` | POST | $25.00 | Memory architecture design for AI agents |

### Free Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/token/price` | GET | Live $ODAI token price (DexScreener, 60s cache) |
| `/api/token/holders` | GET | Holder count from Basescan (5min cache) |
| `/api/context` | GET | Live project context snapshot (30s cache) |
| `/api/intake` | POST | Agent/human/enterprise onboarding registration |
| `/api/intake/stats` | GET | Registration counts by lane |
| `/health` | GET | Service health check |
| `/openapi.json` | GET | OpenAPI 3.0 specification |
| `/api/v2/health` | GET | v2 API health with upstream status |
| `/api/v2/schema` | GET | World model schema (node types, relationships, domains) |

### Solana Actions (Blinks)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/blink/guardrail` | GET/POST | Solana Actions standard -- guardrail check |
| `/blink/worldmodel` | GET/POST | Solana Actions standard -- world model query |

### Marketplace Webhooks

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/clawgig-webhook` | POST | ClawGig marketplace gig handler |
| `/fetch/` | * | Fetch.ai Agentverse protocol endpoint (proxied) |

---

## Payment

### x402 Protocol (Recommended)

ODEI supports the [x402 payment protocol](https://www.x402.org/) for per-request payments on Base chain. No account or API key needed.

1. **Discover pricing** at `https://api.odei.ai/.well-known/x402.json`
2. **Send payment** to `0x510a08954fD7096F67b58Ed210703b323a5f405E` on Base
3. **Include proof** in the `X-Payment-Proof` header:
   ```
   X-Payment-Proof: {"txHash":"0xabc123..."}
   ```
4. The API verifies the transaction via Basescan before serving the response.

### API Key

For high-frequency access, use an API key:

```
X-Api-Key: odei_pro_your_key_here
```

Tier quotas:
| Tier | Daily Limit | Price |
|------|------------|-------|
| Free | 100 queries | $0 |
| Basic | 1,000 queries | $9/mo |
| Pro | Unlimited | $49/mo |
| Enterprise | Unlimited + SLA | Contact us |

---

## Quick Start

### Query the World Model

```bash
curl -s https://api.odei.ai/api/v2/world-model/query \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: YOUR_KEY" \
  -d '{"domain": "FOUNDATION", "limit": 10}'
```

### Run a Guardrail Check

```bash
curl -s https://api.odei.ai/api/v2/guardrail/check \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: YOUR_KEY" \
  -d '{"action": "Transfer 500 USDC to vendor", "context": "Monthly payment"}'
```

### Get Live Token Price

```bash
curl -s https://api.odei.ai/api/token/price
```

For full request/response examples, see [docs/API.md](docs/API.md).

---

## Integration

ODEI is discoverable across multiple agent marketplaces:

| Platform | Agent ID | Protocol |
|----------|----------|----------|
| [Virtuals ACP](https://app.virtuals.io/acp/agent-details/3082) | #3082 | WebSocket + REST |
| [Fetch.ai Agentverse](https://agentverse.ai) | `agent1q2x5c6u...` | Almanac / ASI:One |
| [Clawlancer](https://clawlancer.ai) | `32e99873` | REST webhook |
| [ClawGig](https://clawgig.ai) | `dea086f2` | REST webhook |
| [ERC-8004](https://8004.app) | Agent #2065 (Base) | On-chain registry |
| Solana Actions | -- | Blinks standard |
| x402 | -- | `/.well-known/x402.json` |

For integration code examples, see the [examples repo](https://github.com/odei-ai/examples).

---

## Site Pages

| Path | Description |
|------|-------------|
| `/` | Landing page -- product overview, mission, metrics |
| `/token/` | $ODAI token dashboard with live price and charts |
| `/economics/` | Tokenomics, holder distribution map, revenue data |
| `/holders/` | Holder analytics and decentralization score |
| `/bidwall/` | BidWall trigger progress tracker |
| `/network/` | Network topology and agent connections |
| `/integrate/` | Developer integration guide with marketplace data |
| `/docs/` | API documentation and OpenAPI spec |
| `/security/` | Real-time threat monitoring dashboard |
| `/worldmodel.html` | Interactive 3D world model explorer |
| `/revenue/` | Live revenue dashboard |

---

## Development

### Prerequisites

- Node.js 20+
- Access to the deployment target (SSH alias: `google-cloud-api`)

### Local Development

The API server runs standalone without external dependencies:

```bash
# Start the API server locally
node api-server.js

# Start v2 routes in standalone mode (port 8801)
node api-v2-routes.js
```

Environment variables (all optional, with sensible defaults):

| Variable | Default | Description |
|----------|---------|-------------|
| `ODEI_API_HOST` | `127.0.0.1` | Bind address |
| `ODEI_API_PORT` | `8800` | Listen port |
| `ODEI_API_DATA_DIR` | `/srv/api.odei.ai/data` | Data directory for capsules/intakes |
| `ODEI_API_GRAPH_FALLBACK_PATH` | `./world-model-demo.json` | Fallback graph snapshot |
| `ODEI_API_V1_BASE` | `http://127.0.0.1:8800` | Upstream v1 API base URL |

### Build CSS

```bash
npm run build:css
```

This compiles `input.css` through Tailwind CSS. The output is `styles.css`.

### Deployment

Deployment uses an atomic symlink swap to achieve zero-downtime releases:

```bash
./scripts/deploy-api-site.sh
```

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for the full deployment workflow.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js 20 (vanilla `http`, no Express) |
| Reverse Proxy | nginx + Let's Encrypt TLS |
| CSS Framework | Tailwind CSS 3 |
| Design System | Premium dark theme -- jade `#4FD1C5`, champagne `#F4C95D` |
| Graph Database | Neo4j 5 |
| Payments | x402 protocol (Base L2) |
| Process Manager | systemd (`odei-api.service`) |
| Hosting | Google Cloud Compute Engine |
| Monitoring | PM2 health monitor + Telegram alerts |

---

## Repository Structure

```
.
├── api-server.js            # Main API server (v1 endpoints)
├── api-v2-routes.js         # v2 API routes (auth, quota, guardrails)
├── blinks-handler.js        # Solana Actions / Blinks support
├── config.js                # Site configuration
├── index.html               # Landing page
├── worldmodel.html          # Interactive world model explorer
├── world-model-demo.json    # Graph snapshot (exported pre-deploy)
├── sentinel-feed.json       # Sentinel feed (exported pre-deploy)
├── llms.txt                 # LLM-readable site manifest
├── robots.txt               # Search engine directives
├── sitemap.xml              # Sitemap for SEO
├── openapi-v2.yaml          # OpenAPI v2 specification
├── premium-theme.css        # Design system tokens
├── styles.css               # Compiled Tailwind CSS
├── nav-rail.js              # Navigation rail component
├── site-shell-router.js     # Client-side SPA router
├── token/                   # Token dashboard page
├── economics/               # Tokenomics + holder map
├── holders/                 # Holder analytics page
├── bidwall/                 # BidWall tracker page
├── network/                 # Network topology page
├── integrate/               # Developer integration page
├── security/                # Security monitoring page
├── revenue/                 # Revenue dashboard
├── docs/                    # Documentation pages
├── assets/                  # Images, icons, media
└── .github/workflows/       # CI/CD workflows
```

---

## Related Repositories

| Repo | Description |
|------|-------------|
| [odei-ai/memory](https://github.com/odei-ai/memory) | World model architecture documentation |
| [odei-ai/mcp-odei](https://github.com/odei-ai/mcp-odei) | MCP server for Neo4j graph access |
| [odei-ai/examples](https://github.com/odei-ai/examples) | Integration examples (Python, TypeScript) |
| [odei-ai/research](https://github.com/odei-ai/research) | Research papers and use cases |

---

## License

Proprietary. Copyright 2025-2026 ODEI Symbiosis. All rights reserved.

For API access, visit [api.odei.ai/integrate](https://api.odei.ai/integrate/) or contact via [Telegram](https://t.me/odei_ai).
