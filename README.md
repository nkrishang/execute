# Execute

**Never connect a wallet.** Execute ([execute.cash](https://execute.cash)) is infrastructure that lets apps
have their users earn onchain without wallet connections, token approvals or bridging.

The primitive is the **executable address**. An app describes what should happen with some tokens, for
example "deposit 1,000 USDC into this Morpho vault for this user". Execute returns a fresh address whose
CREATE2 derivation commits to those exact calls. The user sends the tokens from any wallet or exchange, and
the calls run. If they can't run before the address expires, the funds go back to a recovery address. Anyone
can recompute the address from the intent, so what you see is what executes.

*Submission to **Monad Metropolis 2026**.* Monad mainnet (chain 143) is a supported chain. The
`PaymentFactory` is deployed there, and execute.cash/morpho switches between 12 Monad Morpho vaults in a
single transfer.

## Links

| | |
|---|---|
| Website and dashboard | https://execute.cash |
| API | https://api.execute.cash |
| Docs | https://execute.cash/docs |
| Demo: switch Morpho vaults in one transfer | https://execute.cash/morpho |
| Demo: USDC to Bitso USDC Prime collateral in one transfer | https://execute.cash/bitso |

**This project's repos**

| Repo | What it is |
|---|---|
| [nkrishang/execute-cash](https://github.com/nkrishang/execute-cash) | The website: the landing page, docs, developer dashboard, address pages and demos |
| [nkrishang/execute-server](https://github.com/nkrishang/execute-server) | The Execute API: intents in, executable addresses out, and their lifecycle |

**The infrastructure they run on**

execute-server delegates every blockchain interaction to these. All three are needed to host the
whole stack.

| Repo | What it is |
|---|---|
| [nkrishang/gum-contracts](https://github.com/nkrishang/gum-contracts) | `PaymentFactory` and `Payment`, the onchain contracts behind every address |
| [nkrishang/gum-indexer](https://github.com/nkrishang/gum-indexer) | Watches addresses for incoming token transfers |
| [nkrishang/gum-engine](https://github.com/nkrishang/gum-engine) | Transaction relayer that sends `PaymentFactory.execute` |

## Architecture

```
browser ──▶ execute-cash (Vercel) ──/api/execute proxy──┐
app server ──POST /v1/address/create──────────────────┐ │
                                                      ▼ ▼
┌─ Railway project, private network ───────────────────────────────────────────────────────
│                                               execute-server (Postgres)
│                                                       │
│     gum-indexer  ◀──(outbox) watch the address ───────┤
│                  ──webhook──▶ payment.confirmed, threshold.reached
│                                                       │
│     gum-engine   ◀──(outbox) PaymentFactory.execute ──┤
│                  ──webhook──▶ transaction.included / confirmed / failed
│                                                       │
│     the app      ◀──(outbox) webhooks ────────────────┘
│                  address.funded / settled / refunded / …
│
└──────────────────────────────────────────────────────────────────────────────────────────
      gum-indexer reads and gum-engine sends through QuickNode RPCs, on:
      Monad · Base · Arbitrum (· Arc)   PaymentFactory 0x6D85B9706D8f076cB8A9fEA37d70Fcb8D22C2952
```

**gum-contracts** (Solidity, Foundry). `PaymentFactory` is an ownerless CREATE2 deployer. An address is
derived from its terms: token, amount, calls, expiry, recovery and salt. `execute(terms)` deploys a
`Payment` at that address, which runs the calls and returns any excess to `recovery`. After expiry it sends
everything to `recovery` instead. The factory has the same address on Monad, Base and Arbitrum.

**execute-server** (Rust, axum, sqlx, Postgres). This is the public API. `POST /v1/address/create` takes an
intent of human-readable function signatures and arguments, ABI-encodes it and derives the address. It
never opens an RPC connection. `GET /v1/address/{a}` returns the intent, and `/status` reports the state.
A transactional outbox registers a gum-indexer watch for each address. When the balance reaches the amount,
it asks gum-engine to execute. It retries failing calls until expiry, then refunds. Apps get signed
webhooks, and credentials are a secret API key or a publishable client ID locked to allowed origins.

**gum-indexer** (Rust, Postgres). It watches the addresses for ERC-20 transfers. WebSocket subscriptions
detect a payment within milliseconds. An `eth_getLogs` sweep from a durable cursor does the counting, so a
restart or RPC outage can delay a payment but never lose or double count it.

**gum-engine** (Rust, Postgres). This relayer signs with AWS KMS keys. It keeps a pool of signers that a
treasury tops up, tracks each transaction to a receipt, and reports the receipt by webhook. It runs one
leader instance at a time.

**execute-cash** (Next.js 16, React 19, Tailwind v4, three.js, viem, Privy). It has the landing page, the
docs with a live demo, and a dashboard behind Privy sign-in for API keys, client IDs, allowed origins,
webhooks and the address list. Every executable address gets a public page that recomputes the address in
the browser before showing it. The `/morpho` and `/bitso` demos build intents from live vault quotes. A
same-origin `/api/execute` proxy forwards an allowlist of routes to execute-server.

## Run and host it yourself

### Tools

- **Rust** (stable, 1.95+) and **Docker**, for the three Rust services and their local Postgres
- **PostgreSQL**: one database per Rust service
- **Foundry** (`forge`, `anvil`), to build and deploy the contracts and to run a local chain
- **Node.js** and **pnpm**, for execute-cash

### Accounts and API keys

| Service | Used by | What you need |
|---|---|---|
| [Railway](https://railway.com) | execute-server, gum-indexer, gum-engine | One project with three services and three Postgres databases, in one region. Private networking only works within a project. |
| [Vercel](https://vercel.com) | execute-cash | A Next.js project |
| [Privy](https://privy.io) | execute-cash, execute-server | An app ID and its verification key (PEM). Add your domains, and `localhost` for development, to its allowed domains. |
| [QuickNode](https://quicknode.com) | gum-indexer, gum-engine, execute-cash | An HTTP and a WSS endpoint per chain. Turn MEV Protection **off** on Base and Arbitrum. Arbitrum needs the Synchronous SendTransaction add-on. An API key is optional, for credit metrics. |
| [AWS KMS](https://aws.amazon.com/kms/) | gum-engine | `ECC_SECG_P256K1` sign/verify keys for the signers and the treasury, and an IAM user allowed only `kms:Sign` and `kms:GetPublicKey` |
| Gas | gum-engine | Native gas in each chain's treasury. Signers are topped up from it automatically. On Monad, keep the 10 MON reserve in mind. |
| A deployer key | gum-contracts | Only for deploying your own factory: a fresh key, at nonce 0 on every chain |

### Environment variables

Each service repo has a `.env.example` listing the variables it reads, and the READMEs explain each one.
Generate the secrets yourself, for example with `openssl rand -hex 32`.

**gum-indexer**: `DATABASE_URL`, `GUM_WEBHOOK__SECRET`,
`GUM_WEBHOOK__HOST_ALLOWLIST=["execute-server.railway.internal"]`, `GUM_CHAINS__<CHAIN>__HTTP_URL` and
`__WS_URL` per chain, `GUM_CHAINS__<CHAIN>__ENABLED`, `GUM_QUICKNODE__API_KEY` (optional).

**gum-engine**: `DATABASE_URL`, `GUM_WEBHOOK__SIGNING_SECRET`,
`GUM_WEBHOOK__HOST_ALLOWLIST=["execute-server.railway.internal"]`, `GUM_SIGNERS__KMS_KEY_IDS`,
`GUM_CHAINS__<CHAIN>__TREASURY_KEY_ID`, `RPC_URL_MONAD`, `RPC_URL_BASE`, `RPC_URL_ARBITRUM`, `RPC_URL_ARC`,
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`.

**execute-server**:

| Variable | Value |
|---|---|
| `DATABASE_URL`, `PORT` | Provided by Railway |
| `EXECUTE_SERVER__CALLBACK_BASE_URL` | `http://execute-server.railway.internal:8080` |
| `EXECUTE_ADDRESSES__FACTORY_ADDRESS` | `0x6D85B9706D8f076cB8A9fEA37d70Fcb8D22C2952`, or your own factory |
| `EXECUTE_PRIVY__APP_ID`, `EXECUTE_PRIVY__VERIFICATION_KEY` | From Privy |
| `EXECUTE_INDEXER__BASE_URL`, `EXECUTE_INDEXER__WEBHOOK_SECRET` | `http://gum-indexer.railway.internal:8080`, and gum-indexer's `GUM_WEBHOOK__SECRET` |
| `EXECUTE_ENGINE__BASE_URL`, `EXECUTE_ENGINE__WEBHOOK_SECRET` | `http://gum-engine.railway.internal:8080`, and gum-engine's `GUM_WEBHOOK__SIGNING_SECRET` |
| `EXECUTE_ADMIN__TOKEN` | A random token for `/v1/admin` |
| `EXECUTE_CHAINS__<CHAIN>__ENABLED` | Offers a chain for new addresses |

**execute-cash**:

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_PRIVY_APP_ID` | From Privy. `NEXT_PUBLIC_PRIVY_CLIENT_ID` is optional. |
| `EXECUTE_API_URL` | Your execute-server, server-side. The default is `https://api.execute.cash`. |
| `NEXT_PUBLIC_EXECUTE_API_URL` | The API URL shown in code snippets |
| `NEXT_PUBLIC_SITE_URL` | Your site's URL, for metadata. The default is `https://execute.cash`. |
| `EXECUTE_DEMO_API_KEY` | An API key from your own dashboard, for the live demo in the docs |
| `BASE_RPC_URL`, `MONAD_RPC_URL`, `ARC_RPC_URL` | Paid RPCs for the demo quotes. They fall back to public RPCs. |

### Deploy order

1. **Contracts.** Reuse the deployed `PaymentFactory` (`0x6D85…2952` on Monad, Base and Arbitrum), or run
   gum-contracts' `script/Bootstrap.s.sol` from a fresh key on each chain.
2. **gum-indexer**, then **gum-engine**, as services of one Railway project, each with its own Postgres.
   Give neither a public domain: their APIs have no authentication, and the private network is the
   boundary. Fund gum-engine's treasury on each chain.
3. **execute-server**, in the same Railway project, with its own Postgres. Migrations run at boot. Use
   `/readyz` as the health check, and give it your public API domain.
4. **execute-cash**, on Vercel, with `EXECUTE_API_URL` pointing at your execute-server. Add the domain to
   Privy's allowed domains.

To accept a new token, such as a vault's share token, add it to gum-indexer's `config/default.toml` and
deploy, then add it to execute-server's. Arc is off by default in both. Turn it on with
`GUM_CHAINS__ARC__ENABLED=true` and `EXECUTE_CHAINS__ARC__ENABLED=true`.

### Run locally

Each Rust service has a `docker-compose.yml` for Postgres and a local profile pointed at Anvil.
gum-contracts' `script/LocalBootstrap.s.sol` deploys a factory and mock stablecoins to a fresh Anvil, at the
addresses execute-server's `.env.example` expects.

```sh
# execute-server
docker compose up -d postgres && cp .env.example .env && set -a && . ./.env && set +a
cargo run                                    # set PORT to match execute-cash's EXECUTE_API_URL

# execute-cash
pnpm install && cp .env.example .env.local   # EXECUTE_API_URL=http://127.0.0.1:<that port>
pnpm dev
```

gum-indexer and gum-engine run the same way: `docker compose up -d`, then `cargo run`. Their READMEs cover
their local profiles, test suites and benchmarks.
