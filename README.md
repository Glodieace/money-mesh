# MoneyMesh

MoneyMesh is a synthetic fraud-operations prototype. PostgreSQL is the transactional source of truth. FastAPI applies deterministic fraud rules and synchronizes graph relationships to Neo4j when connected. The admin explorer renders the graph API response with Cytoscape.js. Investigation evidence is stored as canonical JSON in PostgreSQL, fingerprinted with SHA-256, and anchored to a minimal Solidity registry on a local EVM chain.

The EVM stores only a hashed case key, evidence digest, version, and timestamp. It does not store customer data, evidence JSON, account balances, or banking transfers. Local mode is a real but disposable local EVM blockchain; it is not a public testnet or production chain.

## Run the full stack with Docker Compose

Requirements: Docker Desktop with Compose.

```powershell
Copy-Item .env.example .env
docker compose up --build
```

Open [http://localhost:3000](http://localhost:3000), with API docs at [http://localhost:8000/docs](http://localhost:8000/docs). Compose starts PostgreSQL, Neo4j, the local Hardhat EVM node, FastAPI, and Next.js. The API compiles the contract artifact in its build and deploys the registry automatically when the local node is available. No public RPC key, wallet key, or manual contract address is needed.

Compose’s local Neo4j defaults are for development only (`neo4j` / `moneymesh-demo`). To use Neo4j Aura instead, fill in `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, and optionally `NEO4J_DATABASE` in `.env`. The local container may remain running, but the API uses the configured URI. If Neo4j is unreachable, PostgreSQL graph fallback remains available and the admin UI labels it **FALLBACK MODE**.

The EVM node is ephemeral: stopping or recreating it clears its chain state. Evidence JSON and anchor metadata remain in PostgreSQL. A subsequent API health check redeploys the registry if the old contract address has no code; anchors from the previous local chain then need to be retried.

## Run locally without Docker

Requirements: Node.js 22+, Python 3.11+. PostgreSQL is optional for local development; SQLite can run the API. Compile the registry once:

```powershell
npm install
npm run blockchain:compile
```

Terminal 1 — local EVM:

```powershell
npm run blockchain:start
```

Hardhat's funded development keys are local-only fixtures; the launcher redacts the private-key fields from its standard output. The API uses an unlocked account on this private node and does not read a key from source control.

Terminal 2 — API:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:PYTHONPATH = "apps/api"
$env:DATABASE_URL = "sqlite:///./moneymesh.db"
$env:BLOCKCHAIN_MODE = "local"
$env:BLOCKCHAIN_RPC_URL = "http://127.0.0.1:8545"
$env:BLOCKCHAIN_CHAIN_ID = "31337"
uvicorn app.main:app --app-dir apps/api --reload --port 8000
```

At startup the API deploys the registry if it is not already deployed for that local chain. For an explicit manual deployment, run `npm run blockchain:deploy` after the node starts; the generated address is saved under `apps/chain/deployments/local.json` and read by the API.

Terminal 3 — frontend:

```powershell
npm run dev
```

The frontend uses `http://localhost:8000` by default. `DATABASE_URL`, `JWT_SECRET`, `DEMO_PASSWORD`, and risk thresholds are configurable in the environment. `.env.example` contains development-only synthetic credentials; replace them before any hosted deployment. Production requires a unique JWT secret and refuses the default demo password. Synthetic demo data and a private local EVM can be explicitly opted into for a hosted hackathon demonstration; both opt-ins are off by default.

### Investigator risk explanations

The deterministic fraud engine remains authoritative for every risk score and signal. An administrator can request a GPT-4o mini explanation from a transaction investigation. Configure `OPENAI_API_KEY` only in the FastAPI environment; `OPENAI_MODEL` defaults to `gpt-4o-mini`. If the key is missing or the API is unavailable, the backend returns a deterministic explanation from the recorded signals. The request contains the amount, currency, risk level, rule signals, and an anonymized graph path; it omits customer names, account/device/IP identifiers, login location, and transaction description. Explanations are not persisted in the evidence package or anchored on-chain.

## Live demo controls

The admin dashboard includes **Normal activity**, **Simulate money-mule attack**, and **Reset demo**. The resettable fixture has four synthetic customers, five external accounts, one wallet, three known devices, and nine curated transactions. The attack adds two fixed-ID transfer attempts, so the complete demo remains small and the graph does not grow random nodes on each run. All people, accounts, IPs, and locations are simulated.

The suspicious scenario creates a new-device, simulated UAE context for Rahul, runs his ₹85,000 transfer through the existing transfer endpoint, then scores Meera’s ₹82,000 forwarding attempt on her usual simulated location and shared demo device. The deterministic fraud engine returns 78 and 94 from its existing rules. Both attempts remain held, so the second is not treated as settled proceeds from the first. The main path is Rahul → Meera → `MM90003` → `WALLET-X91`; the final wallet relationship is explicitly synthetic graph context, not a posted customer payment. Location is one behavioral signal and never decides fraud by itself. The focused explorer includes only the selected transfer path and its relevant demo device, IP, or simulated-location context. **Presentation mode** enlarges the activity feed, scores, and graph for a live walkthrough.

The demo event API is `GET /admin/demo/state` and `GET /admin/demo/events`; start with `POST /admin/demo/simulate` using `NORMAL_ACTIVITY` or `SUSPICIOUS_NETWORK`. `POST /admin/demo/reset` restores the curated PostgreSQL scenario and graph mirror with fixed entity and transaction identifiers. Demo-only tampering is available under `/admin/demo/evidence/{id}/tamper` and `/restore`; it changes only local evidence JSON, leaves the on-chain fingerprint/history untouched, and is reversible from its saved snapshot.

Open the frontend's `/technology` page for plain-language technology cards, the architecture flow, and a short guide to why each component is used.

## Neo4j graph

The official Neo4j Python driver creates idempotent namespaced constraints and synchronizes customers, accounts, transactions, devices, IP addresses, wallets, and investigations from PostgreSQL. Transactions commit in PostgreSQL first; graph synchronization failure does not roll back or invalidate a bank transfer. `/health/graph` reports the actual connection state.

To rebuild the Neo4j mirror from the configured PostgreSQL database:

```powershell
$env:PYTHONPATH = "apps/api"
python -m app.graph_sync
```

The admin-only graph endpoints return parameterized Cypher paths and analytics for multi-hop movement, rapid forwarding, shared devices/IPs, circular flow, and high connectivity. These results are risk signals, not proof of fraud. When Neo4j is connected, Cytoscape uses its selected transaction trace; otherwise it uses the PostgreSQL fallback path. Both return a focused subgraph rather than the whole database. The seeded `MM90003 → WALLET-X91` edge is explicitly marked **SIMULATED CONTEXT** and is not presented as settled money.

## Local blockchain and evidence verification

`apps/chain/contracts/MoneyMeshEvidenceRegistry.sol` is a minimal owner-controlled Solidity registry. A case can anchor sequential evidence versions; retrying an identical version is idempotent, while a conflicting digest or skipped version is rejected. Backend ABI and bytecode are generated by `npm run blockchain:compile`.

The API submits evidence using an unlocked local Hardhat development account on chain ID `31337`; it waits for a real receipt and stores the actual transaction hash, block number, contract address, chain ID, and chain timestamp in PostgreSQL. If the RPC is down, the evidence package is retained with **PENDING** status and **Retry anchor** becomes available. Verification recalculates the local SHA-256 digest and reads the stored `bytes32` value from the contract; only a matching local/on-chain digest is reported as verified. The local chain is private development infrastructure; the fingerprint can show that the current evidence matches an anchored version, not that the evidence itself is true.

With the local node running, run the contract tests and Python adapter acceptance check:

```powershell
npm run blockchain:test
$env:PYTHONPATH = "apps/api"
python -m app.blockchain_verify
```

`python -m app.blockchain_verify` deploys a contract if needed, submits a test evidence transaction, waits for its receipt, reads the digest back, and asserts the on-chain value equals the local SHA-256. `npm run blockchain:verify` is an additional Hardhat/Ethers acceptance script for an already-running local node.

The optional non-local EVM adapter can use `BLOCKCHAIN_MODE=testnet`, `BLOCKCHAIN_RPC_URL`, `BLOCKCHAIN_PRIVATE_KEY`, `EVIDENCE_CONTRACT_ADDRESS`, and `BLOCKCHAIN_CHAIN_ID` (`CHAIN_ID` remains a legacy alias). Those values are blank in `.env.example`; local mode does not need a wallet key or contract address. Never place a real funded key in source control.

## Railway hackathon deployment

See [the Railway deployment guide](docs/railway-deployment.md) for service setup, reference variables, private Hardhat networking, service restart behavior, and live health checks. It describes a synthetic hackathon deployment, not production readiness. Railway docs currently recommend Infrastructure as Code for new projects; follow the guide's root-directory and Dockerfile-path setup in the Railway dashboard.

## Demo accounts and flow

All seeded accounts use the development password `MoneyMeshDemo!2026`:

| Role | Email |
| --- | --- |
| Admin | `admin@moneymesh.demo` |
| Customer | `arun@moneymesh.demo` |
| Customer | `priya@moneymesh.demo` |
| Customer | `rahul@moneymesh.demo` |
| Customer | `meera@moneymesh.demo` |

1. Sign in as the admin and start **Simulate money-mule attack**. The backend scores Rahul’s ₹85,000 transfer to Meera at 78 and Meera’s ₹82,000 forwarding attempt to `MM90003` at 94 using the existing deterministic rule engine.
2. The first transfer has a simulated new-device/location context; the second uses the shared demo device from Meera’s usual simulated location. Both transfers are held and neither moves a balance.
3. Sign in as admin, investigate the alert, and request **Trace the money**. The graph path shows Rahul → Meera → `MM90003` → `WALLET-X91`. The two live held edges are unsettled attempts; the wallet edge is seeded synthetic context.
4. Record an audited case action, anchor the evidence package, then verify integrity. A confirmed result includes the mined EVM transaction hash and block metadata.

Use **Settings & demo → Reset demo data** to restore seeded balances, transactions, alerts, cases, and restrictions. Reset does not erase the EVM node or deploy another contract.

## Tests and limits

```powershell
$env:PYTHONPATH = "apps/api"
python -m pytest -q
npm run typecheck
npm run build
npm run blockchain:test
```

The backend suite uses isolated SQLite databases unless a test explicitly connects to the local EVM. `test_blockchain_integration.py` skips when the EVM node is not running. PostgreSQL and Neo4j container behavior requires Docker. The API currently uses SQLAlchemy `create_all`; schema migration tooling, production fraud calibration, bank integrations, independent public-chain persistence, and production key management are outside this prototype. The local EVM is suitable for a hackathon demonstration, not production evidence custody.
