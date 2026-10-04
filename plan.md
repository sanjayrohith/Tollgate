# Tollgate MVP — Execution Plan

Derived from `architecture.md` and `prd.md`. This document is the sequential build plan for Claude Code.
Execute one task at a time, top to bottom. Do not skip ahead. Do not start a phase until the previous
Phase Gate passes.

> ## Git policy — read before executing anything
>
> **The agent must never run git write commands.** No `git add`, `git commit`, `git push`,
> `git reset`, `git revert`, `git checkout`, `git switch`, `git stash`, `git merge`, `git rebase`,
> `git branch`, `git tag`, or `git clean`. Not even when a task below ends with a commit message.
>
> Every "**Commit.**" line in this document is a **suggested commit message for the human to use**.
> It is not an instruction to the agent.
>
> Read-only git (`status`, `diff`, `log`, `show`) is allowed and encouraged.
>
> When a task's verification passes, stop, print `git diff --stat` and the suggested message as
> text, and wait for the human to review and commit.
>
> The one exception is Task 0.1's `git init`, which creates the repository. Run that once, then
> touch git no further.

---

## Project goal

Build a working, demonstrable x402-compatible payment gateway that adds three things x402 lacks:
hard spend caps, sub-agent budget delegation via signed capability chains, and signed receipts.

The MVP is done when this runs end to end on a laptop against Base Sepolia:

> A supervisor agent signs three capability delegations with different caps. Three worker agents call
> paid endpoints through the gateway, receive 402s, pay with EIP-3009 authorisations, and get data.
> Worker 2 hits its cap and stops while workers 1 and 3 continue. The settler batches the
> authorisations into one on-chain transaction. The dashboard shows the delegation tree draining live.
> The expense report lists every charge with its transaction hash.

That is `prd.md` §9. Everything in this plan exists to make that paragraph true.

---

## MVP scope

**In scope (P0 from `prd.md` §7):**

| ID | Requirement | Phase |
|----|-------------|-------|
| F1 | Spec-compliant x402 402 response | 2 |
| F2 | x402 payment header accepted, EIP-3009 verified, request proxied | 2, 4 |
| F3 | Per-request cap, total cap, allowlist enforced atomically pre-upstream | 3, 5 |
| F4 | EIP-712 capability chain, attenuation-only, verified at the gateway | 5 |
| F5 | Batched settlement worker, one chain, off the request path | 6 |
| F6 | Signed receipt per charge + expense report endpoint | 7 |
| F7 | Provider registration, SSRF-safe | 1, 4 |
| F8 | Dashboard: provider view, agent view with delegation tree, public feed | 9 |
| F9 | Python client library: 402 handling, sign, retry, `delegate()` | 8 |

**Locked decisions (from `prd.md` §13 and the pre-build clarification):**

1. Speak x402 on the wire. Do not invent a protocol.
2. Direct-pull via EIP-3009. No escrow contract. Channel mode is out of scope.
3. Native USDC on Base Sepolia, chain 84532. One chain.
4. Delegation is the headline feature.
5. Gateway is Rust / Axum / Tower / Hyper, using `alloy` for all signature work.
6. Develop against a local `anvil` fork of Base Sepolia; deploy `BatchSettler` to real Base Sepolia
   in Phase 6 and demo against it.
7. Delegation funding uses **Resolution A** (`architecture.md` §5.3): the parent funds each child
   wallet with USDC up front. Capabilities are pure policy.
8. Dashboard ships all three views.

---

## Assumptions and ambiguities discovered

Resolved here so no task has to re-litigate them mid-build.

**A1.The two files present (`architecture.md`, `prd.md`) are the v2 documents and are canonical.
Ignore references to v1 behaviour except where they explain what *not* to build.

**A2. Empty repository.** The repo contains only the two markdown files and is not a git repository.
Phase 0 Task 0.1 initialises it. There is no existing code to integrate with.

**A3. Refund path conflict.** `prd.md` marks F12 (refunds) P1, but `architecture.md` §4.3 makes
"upstream 5xx → void the reservation, credit budget back, never settle" part of the core request
lifecycle. Resolution: **voiding is in the MVP** (Phase 4) because it is only a Redis decrement and a
status change and it prevents charging for failures. An on-chain refund of an already-settled charge
is out of scope.

**A4. Window budgets.** `architecture.md` §9.1 step 4 includes a window-bucket check in the Lua claim
script; `prd.md` F13 marks time-window budgets P1. Resolution: **omit window budgets.** The claim
script implements steps 1, 2, 3, 5, 6, 7 only. Do not add a `tg:window:*` key.

**A5. `upto` scheme.** P1 in `prd.md` (F10). Out of scope. The `providers` table keeps its `scheme`
and `max_price_wei` columns so the schema does not need migrating later, but the gateway only
implements `exact` and rejects any other scheme with a clear error.

**A6. `transferWithAuthorization` vs `receiveWithAuthorization`.** Open in `prd.md` §14, resolved in
`architecture.md` §3.1: batching requires `transferWithAuthorization` because
`receiveWithAuthorization` demands `msg.sender == to`. Use `transferWithAuthorization`. Document the
bounded front-running consequence in the README.

**A7. Contract on the critical path.** `prd.md` decision #5 says "no own contract on the critical
path"; `architecture.md` §6 ships `BatchSettler`. These are reconcilable: `BatchSettler` holds no
funds and cannot redirect one, so it carries no custody risk. It is required for batching. Build it.
Fallback if it slips: the settler submits one `transferWithAuthorization` per authorisation directly
to USDC, which is correct but costs more gas.

**A8. Latency targets.** `prd.md` §8 targets p50 under 15 ms of added gateway latency. Instrument for
it from Phase 2 (per-stage timers) and display it, but **do not gate any Phase Gate on hitting the
number.** Missing the target is a tuning problem, not a broken MVP.

**A9. Merkle allowlists.** `architecture.md` §5.1 mentions a merkle root as the privacy-preserving
version and explicitly says a plain sorted-list hash is enough for a first build. Use
`keccak256(sorted slugs joined by "\n")`.

**A10. Circuit breakers.** The `tg:cb:{provider_slug}` Redis key in `architecture.md` §8 is carried
over from a prior system and is not required by any P0 requirement. Out of scope.

**A11. Name availability check.** `prd.md` §12 asks for an npm/PyPI/crates.io/domain check before
writing code. That is a human task, not a code task. Noted, not planned.

**A12. Deny-by-default egress network policy** (`architecture.md` §10 SSRF item 4) is a deployment
control, not application code. The MVP implements resolve-validate-pin (items 1, 2, 3, 5) in the
connector and documents item 4 as an operational requirement.

**A13. Sample paid upstreams.** `architecture.md` §12 calls for two real public APIs plus one
deliberately slow one. The MVP uses a local mock upstream service for all tests (fast, deterministic,
no rate limits) and registers two real public APIs plus the slow mock for the demo.

---

## Target repository layout

Established incrementally; shown here so tasks can name files precisely.

```
tollgate/
├── Cargo.toml                      # workspace root
├── docker-compose.yml              # postgres + redis
├── .env.example
├── README.md
├── plan.md
├── architecture.md
├── prd.md
├── migrations/                     # sqlx migrations
│   └── 0001_init.sql
├── crates/
│   ├── tollgate-core/              # shared: config, types, db pool, errors
│   │   └── src/{lib,config,types,db,error}.rs
│   ├── tollgate-gateway/           # axum binary
│   │   └── src/
│   │       ├── main.rs
│   │       ├── state.rs
│   │       ├── routes/{mod,proxy,providers,capabilities,receipts,feed,metrics}.rs
│   │       ├── x402.rs             # 402 body, header decode, scheme dispatch
│   │       ├── verify.rs           # EIP-3009 recovery
│   │       ├── capability.rs       # chain decode + attenuation
│   │       ├── policy.rs           # Lua claim script driver
│   │       ├── proxy.rs            # SSRF-pinned connector + streaming
│   │       ├── ssrf.rs             # resolve + IP denylist
│   │       ├── ledger.rs           # stream drain, receipt signing
│   │       └── lua/claim.lua
│   └── tollgate-settler/           # settler binary
│       └── src/{main,batch,chain}.rs
├── contracts/                      # foundry
│   ├── foundry.toml
│   ├── src/BatchSettler.sol
│   ├── test/BatchSettler.t.sol
│   └── script/Deploy.s.sol
├── client/                         # python: tollgate-client
│   ├── pyproject.toml
│   └── tollgate/{__init__,session,capability,signing,errors}.py
├── dashboard/                      # next.js 14
│   └── app/{provider,agent,feed}/page.tsx
├── examples/
│   ├── mock_upstream.py
│   └── agents/{supervisor,worker}.py
└── scripts/
    ├── dev.sh
    └── seed_providers.sh
```

---

## Phase overview

| # | Phase | Ends with | Tasks | Commits |
|---|-------|-----------|-------|---------|
| 0 | Foundation | `cargo run` serves `/health`, Postgres + Redis up | 5 | 5 |
| 1 | Data layer and provider registry | Providers can be registered and listed via API | 4 | 4 |
| 2 | x402 protocol surface | Unpaid request gets a spec-shaped 402; a valid payment header verifies | 5 | 5 |
| 3 | Atomic claim | One Lua script does nonce + caps + decrement + stream, drained to Postgres | 4 | 4 |
| 4 | SSRF-safe proxy and lifecycle | Paid request reaches upstream and returns data; failures void | 4 | 4 |
| 5 | Capability delegation | Chains verified, every level decremented, revocation works | 6 | 6 |
| 6 | Contract and settler | Real batched settlement transaction on Base Sepolia | 6 | 6 |
| 7 | Receipts and expense report | Signed per-charge receipts, rolled-up expense report | 3 | 3 |
| 8 | Python client | `Session`, 402 retry loop, `delegate()` | 4 | 4 |
| 9 | Dashboard | Provider view, live delegation tree, public feed | 5 | 5 |
| 10 | Demo and hardening | Sample agents run the full `prd.md` §9 script | 4 | 4 |
|   | **Total** | | **46** | **46** |

Dependency chain is strictly linear: each phase depends on all previous ones. Phase 6 additionally
depends on nothing from Phase 5 except the `capability_hash` column existing, so if Phase 5 slips,
Phase 6 can proceed (see the fallback in `prd.md` §10: ship receipts alone).

---

# Phase 0 — Foundation

**Objective.** A running Rust workspace with Postgres and Redis reachable, a health endpoint, and
config loaded from the environment.

**After this phase exists:** `docker compose up -d && cargo run -p tollgate-gateway` starts a server
that answers `GET /health` with the liveness of both Postgres and Redis.

**Depends on:** nothing.

---

### Task 0.1 — Initialise repository and Cargo workspace

**Objective.** Create the git repository and an empty three-crate Rust workspace that compiles.

**Implementation.**
- `git init` at the repo root. This is the only git write command the agent ever runs; see the Git
  policy above. Do not `git add` or `git commit` afterwards. Add `.gitignore` covering `target/`, `.env`, `node_modules/`,
  `__pycache__/`, `.venv/`, `contracts/out/`, `contracts/cache/`, `dashboard/.next/`.
- Root `Cargo.toml`: `[workspace]` with `members = ["crates/tollgate-core", "crates/tollgate-gateway",
  "crates/tollgate-settler"]`, `resolver = "2"`, and a `[workspace.dependencies]` block pinning
  `tokio`, `axum`, `tower`, `hyper`, `serde`, `serde_json`, `sqlx`, `redis`, `alloy`, `tracing`,
  `tracing-subscriber`, `anyhow`, `thiserror`. Pin exact minor versions (`prd.md` risk table: pin
  versions, read changelogs).
- Create the three crates with `cargo new --lib crates/tollgate-core`, `cargo new --bin
  crates/tollgate-gateway`, `cargo new --bin crates/tollgate-settler`.
- Gateway and settler both depend on `tollgate-core`.
- Add `rust-toolchain.toml` pinning a stable channel.

**Verification.**
```
cargo build --workspace        # succeeds
cargo run -p tollgate-gateway  # prints hello, exits 0
git status                     # target/ and .env do not appear as untracked
```

**Suggested commit message (for the human).** `chore: initialise cargo workspace and repository scaffolding`

---

### Task 0.2 — Docker Compose for Postgres and Redis

**Objective.** Local dependencies runnable with one command, with Redis configured per
`architecture.md` §9.3.

**Implementation.**
- `docker-compose.yml` with two services:
  - `postgres:16`, database `tollgate`, port 5432, named volume.
  - `redis:7`, port 6379, started with `--appendonly yes --appendfsync everysec` (required by
    `architecture.md` §9.3).
- Healthchecks on both (`pg_isready`, `redis-cli ping`).
- `scripts/dev.sh` that runs `docker compose up -d` and waits for both healthchecks.

**Verification.**
```
./scripts/dev.sh
docker compose ps                                  # both healthy
redis-cli CONFIG GET appendonly                    # -> yes
redis-cli CONFIG GET appendfsync                   # -> everysec
psql postgres://postgres:postgres@localhost/tollgate -c 'select 1'
```

**Commit.** `chore: add docker compose for postgres and redis`

---

### Task 0.3 — Configuration loading in `tollgate-core`

**Objective.** One typed `Config` struct loaded from environment variables, shared by both binaries.

**Implementation.**
- `crates/tollgate-core/src/config.rs`: a `Config` struct with `database_url`, `redis_url`,
  `bind_addr`, `chain_id` (default 84532), `rpc_url`, `usdc_address`, `batch_settler_address`,
  `settler_private_key`, `gateway_signing_key`, `quote_validity_seconds` (default 300, per
  `architecture.md` §9.2), `clock_skew_seconds` (default 60), `settler_batch_size` (default 50),
  `settler_flush_seconds` (default 10).
- Load with `envy` or manual `std::env::var` + `thiserror` errors. Fail loudly at startup on a
  missing required var; never silently default a secret.
- `.env.example` documenting every variable with a safe placeholder value. Note in a comment that
  `SETTLER_PRIVATE_KEY` is a hot key, per `architecture.md` §12.

**Verification.**
```
cargo test -p tollgate-core          # unit test: Config::from_env with a full env map parses;
                                     # missing DATABASE_URL returns a named error
```

**Commit.** `feat: add typed configuration loading to tollgate-core`

---

### Task 0.4 — Database and Redis connection pools

**Objective.** `tollgate-core` exposes constructors for a `sqlx::PgPool` and a
`redis::aio::ConnectionManager`.

**Implementation.**
- `crates/tollgate-core/src/db.rs`: `pub async fn pg_pool(cfg: &Config) -> Result<PgPool>` with
  `max_connections(10)` and a 5s acquire timeout.
- `pub async fn redis_conn(cfg: &Config) -> Result<ConnectionManager>`.
- Both return `tollgate_core::Error` on failure.

**Verification.**
```
docker compose up -d
cargo test -p tollgate-core -- --ignored connection   # integration test opens both, runs
                                                      # `SELECT 1` and `PING`, both succeed
```

**Commit.** `feat: add postgres and redis connection pools`

---

### Task 0.5 — Axum server with `/health` and structured logging

**Objective.** The gateway binary starts, logs structurally, and reports dependency health.

**Implementation.**
- `crates/tollgate-gateway/src/state.rs`: `AppState { cfg: Arc<Config>, pg: PgPool, redis:
  ConnectionManager }`.
- `main.rs`: init `tracing_subscriber` with an env filter and JSON-off (human) formatting; build
  `AppState`; build an `axum::Router` with `GET /health`; serve on `cfg.bind_addr`; handle
  SIGINT/SIGTERM for graceful shutdown.
- `routes/mod.rs` + a health handler returning
  `{"status":"ok","postgres":true,"redis":true}` with 200, or 503 with the failing component set
  false.

**Verification.**
```
cargo run -p tollgate-gateway
curl -s localhost:8080/health            # 200, both true
docker compose stop redis
curl -si localhost:8080/health           # 503, "redis": false
docker compose start redis
```

**Commit.** `feat: add axum server with health endpoint and tracing`

---

## Phase 0 Gate

- [ ] `cargo build --workspace` and `cargo clippy --workspace -- -D warnings` both clean
- [ ] `./scripts/dev.sh` brings up Postgres and Redis, both healthy
- [ ] `redis-cli CONFIG GET appendfsync` returns `everysec`
- [ ] `curl localhost:8080/health` returns 200 with both dependencies true
- [ ] Stopping Redis flips `/health` to 503; restarting flips it back
- [ ] `.env.example` lists every variable `Config` reads
- [ ] Ctrl-C shuts the server down without a panic

---

# Phase 1 — Data layer and provider registry

**Objective.** The Postgres schema from `architecture.md` §8 exists, and providers can be registered
and read back through the API.

**After this phase exists:** `POST /v1/providers` registers an upstream with a price and payout
address; `GET /v1/providers/{slug}` returns it. This is `prd.md` F7 minus the SSRF check, which lands
in Phase 4 alongside the connector that needs it.

**Depends on:** Phase 0.

---

### Task 1.1 — Initial migration: `providers`, `capabilities`, `authorisations`

**Objective.** The full schema, created once, exactly as specified.

**Implementation.**
- `migrations/0001_init.sql` containing the three `CREATE TABLE` statements and three
  `CREATE INDEX` statements verbatim from `architecture.md` §8.
- Add a `CHECK` constraint on `authorisations.status IN ('reserved','voided','queued','settled','failed')`.
- Add `sqlx-cli` usage to `scripts/dev.sh` (`sqlx migrate run`) and document `DATABASE_URL` in the
  README.
- Add `sqlx::migrate!()` at gateway startup so a fresh clone self-migrates.

**Verification.**
```
sqlx migrate run
psql $DATABASE_URL -c '\d providers'        # all columns present
psql $DATABASE_URL -c '\d authorisations'   # status CHECK present, batch_id/tx_hash nullable
psql $DATABASE_URL -c "\di"                 # the three indexes exist
psql $DATABASE_URL -c "insert into authorisations(nonce,payer,amount_wei,payload,signature,status) values('\x01','\x02',1,'{}','\x03','bogus')"
                                            # fails on the CHECK constraint
```

**Commit.** `feat: add initial postgres schema for providers, capabilities and authorisations`

---

### Task 1.2 — Provider domain type and repository

**Objective.** Typed reads and writes for `providers`, with no HTTP layer yet.

**Implementation.**
- `tollgate-core/src/types.rs`: `Provider { slug, upstream_url, price_wei: U256 (stored NUMERIC),
  max_price_wei: Option<U256>, scheme: Scheme, payout_address: Address, timeout_ms: i32 }`.
  `enum Scheme { Exact }` — deserialise `"exact"` only, reject anything else with a named error
  (assumption A5).
- Serde helpers converting `alloy::primitives::Address` ↔ `BYTEA` and `U256` ↔ `NUMERIC`.
- `tollgate-core/src/db.rs`: `insert_provider`, `get_provider(slug)`, `list_providers`.

**Verification.**
```
cargo test -p tollgate-core -- --ignored provider_repo
# integration test: insert a provider, read it back, assert address bytes and price round-trip
# exactly (U256 through NUMERIC is the failure-prone part); assert scheme "upto" fails to parse
```

**Commit.** `feat: add provider domain type and repository`

---

### Task 1.3 — Provider registration and lookup endpoints

**Objective.** HTTP surface for F7.

**Implementation.**
- `routes/providers.rs`:
  - `POST /v1/providers` — body `{slug, upstream_url, price, payout_address, timeout_ms?}` where
    `price` is a decimal USDC string (`"0.001"`) converted to base units (6 decimals) server-side.
    409 on duplicate slug. 400 on a malformed address or a non-positive price.
  - `GET /v1/providers/{slug}` — 200 or 404.
  - `GET /v1/providers` — list.
- A shared JSON error body: `{"error": {"code": "...", "message": "..."}}` used by every endpoint
  from here on. Define it once in `tollgate-core/src/error.rs`.

**Verification.**
```
curl -X POST localhost:8080/v1/providers -H 'content-type: application/json' \
  -d '{"slug":"geocode","upstream_url":"https://nominatim.openstreetmap.org","price":"0.001","payout_address":"0x0000000000000000000000000000000000000001"}'
# -> 201, body echoes price_wei 1000

curl localhost:8080/v1/providers/geocode        # 200
curl -s -o /dev/null -w '%{http_code}' localhost:8080/v1/providers/nope   # 404
# repeat the POST -> 409
# POST with "price":"0" -> 400 with code invalid_price
```

**Commit.** `feat: add provider registration and lookup endpoints`

---

### Task 1.4 — Mock upstream service and seed script

**Objective.** A deterministic upstream for every later test, including the slow and failing paths
(`architecture.md` §12, assumption A13).

**Implementation.**
- `examples/mock_upstream.py` — a stdlib-only HTTP server on port 9000 with four routes:
  - `GET /ok` → 200 `{"result":"ok"}`
  - `GET /slow` → sleeps 5s then 200
  - `GET /fail` → 500
  - `GET /echo` → 200 echoing the received headers and query
- `scripts/seed_providers.sh` registering `mock-ok`, `mock-slow` (`timeout_ms: 1000`), and
  `mock-fail`, all pointing at `http://127.0.0.1:9000`.
- README section: how to start the mock upstream.

**Verification.**
```
python3 examples/mock_upstream.py &
curl localhost:9000/ok                       # {"result":"ok"}
time curl localhost:9000/slow                # ~5s
curl -s -o /dev/null -w '%{http_code}' localhost:9000/fail   # 500
./scripts/seed_providers.sh
curl localhost:8080/v1/providers             # three providers listed
```

**Commit.** `test: add mock upstream service and provider seed script`

---

## Phase 1 Gate

- [ ] Fresh clone + `./scripts/dev.sh` + `cargo run` self-migrates and serves
- [ ] All three tables and three indexes exist with the specified columns
- [ ] A provider round-trips through the API with its price and address byte-exact
- [ ] Duplicate slug returns 409, bad address returns 400, unknown slug returns 404
- [ ] `scheme: "upto"` is rejected with a clear message, not silently accepted
- [ ] Mock upstream serves `/ok`, `/slow`, `/fail`; seed script registers all three
- [ ] `cargo clippy --workspace -- -D warnings` clean

---

# Phase 2 — x402 protocol surface

**Objective.** Speak x402 correctly on the wire (`prd.md` F1, F2 verification half). No proxying and
no budget enforcement yet — the request stops after verification.

**After this phase exists:** an unpaid request to a registered provider returns a spec-shaped 402
with payment requirements. A request carrying a valid `X-PAYMENT` header verifies its EIP-3009
signature, recovers the payer, and returns 200 with a debug body naming the recovered payer and
amount. An off-the-shelf x402 client can parse the 402.

**Depends on:** Phase 1 (provider lookup supplies price, payout address, and network).

---

### Task 2.1 — x402 payment-requirements types and the 402 responder

**Objective.** F1. The 402 body is spec-shaped so a client nobody wrote for Tollgate can pay it.

**Implementation.**
- `x402.rs`: `PaymentRequirements { scheme, network, max_amount_required, asset, pay_to, resource,
  description, mime_type, max_timeout_seconds, extra }` and the enclosing
  `{ x402_version: 1, accepts: [PaymentRequirements], error }` body. Serialise camelCase.
- `network` is the CAIP-2 identifier for Base Sepolia (`eip155:84532`), from config, not hardcoded in
  the handler.
- `extra` carries the EIP-712 domain the client needs for USDC: `{"name":"USDC","version":"2"}`.
- Route `ANY /v1/proxy/{provider_slug}/{*path}`: look up the provider, 404 with an x402-shaped error
  if unknown, otherwise 402 with the requirements built from provider price + payout address.
- `resource` is the full request URL.
- `max_timeout_seconds` is `cfg.quote_validity_seconds` (300, per `architecture.md` §9.2).

**Verification.**
```
curl -si localhost:8080/v1/proxy/mock-ok/ok
# 402; body has x402Version, accepts[0].scheme=="exact",
# accepts[0].network=="eip155:84532", maxAmountRequired=="1000",
# asset == the configured USDC address, payTo == the provider payout address,
# maxTimeoutSeconds == 300
cargo test -p tollgate-gateway x402::requirements_serialisation
# asserts exact JSON key casing against a fixture file
```

**Commit.** `feat: return spec-compliant x402 402 payment requirements`

---

### Task 2.2 — `X-PAYMENT` header decode

**Objective.** Parse the base64 x402 payment payload into typed data. Decode only; no crypto.

**Implementation.**
- `x402.rs`: `PaymentPayload { x402_version, scheme, network, payload: ExactEvmPayload }` where
  `ExactEvmPayload { signature, authorization: { from, to, value, valid_after, valid_before, nonce } }`.
- `decode_payment_header(&str) -> Result<PaymentPayload, X402Error>` — base64 decode, JSON parse,
  hex-parse addresses / `U256` / `bytes32`.
- Every failure mode is a distinct error variant with its own message: bad base64, bad JSON, unknown
  scheme, unknown network, malformed field.

**Verification.**
```
cargo test -p tollgate-gateway x402::decode
# table test over fixtures: one valid payload decodes with every field asserted;
# truncated base64, non-JSON, scheme "upto", network "eip155:1", and a 31-byte nonce
# each produce their own named error variant
```

**Commit.** `feat: decode x402 payment header into typed payload`

---

### Task 2.3 — Cheap field validation before any crypto

**Objective.** `architecture.md` §4.2 step 2. Ordering matters: reject cheap failures in ~10 µs
before spending 50 µs on signature recovery.

**Implementation.**
- `x402.rs::validate_fields(payload, provider, cfg, now)` checking, in this order:
  1. `valid_before > now + clock_skew` (not expired)
  2. `valid_after <= now + clock_skew` (already active)
  3. `valid_before - now <= quote_validity_seconds + clock_skew` (§9.2 rule 1: reject
     over-long validity windows, because the Redis nonce TTL must cover them)
  4. `network` matches `cfg.chain_id`
  5. `scheme == exact`
  6. `to == provider.payout_address`
  7. `value == provider.price_wei` for the `exact` scheme
- Returns a typed reason enum mapping to the response codes in `architecture.md` §4.3.
- Wire into the proxy route: on failure return 402 with the requirements body and an `error` string
  naming the reason (per §4.3, "signature invalid → 402 with requirements").

**Verification.**
```
cargo test -p tollgate-gateway x402::validate
# one case per rule, each asserting the specific reason variant, plus one all-valid case
curl -si localhost:8080/v1/proxy/mock-ok/ok -H "X-PAYMENT: $EXPIRED_PAYLOAD_B64"
# 402 with error mentioning expiry, and the accepts array still present
```

**Commit.** `feat: validate payment payload fields before signature recovery`

---

### Task 2.4 — EIP-3009 signature recovery with `alloy`

**Objective.** F2's verification half. Recover the payer address from the `TransferWithAuthorization`
typed data and confirm it equals `authorization.from`.

**Implementation.**
- `verify.rs`: define the EIP-712 domain (`name: "USDC"`, `version: "2"`, `chainId`,
  `verifyingContract: usdc_address` — take name/version from provider `extra` config so a different
  token works without a code change) and the `TransferWithAuthorization` struct via
  `alloy::sol!`/`SolStruct`.
- `recover_payer(payload, cfg) -> Result<Address>`: build the signing hash, `Signature::recover_address_from_prehash`.
- Reject high-`s` signatures (`architecture.md` §10, malleability) before recovery.
- Fail if `recovered != authorization.from`.
- Wire into the route after field validation. On success, for now, return
  `200 {"payer":"0x…","amount":"1000","provider":"mock-ok"}` as a debug body — the proxy replaces
  this in Phase 4.

**Verification.**
```
cargo test -p tollgate-gateway verify::recover
# fixture generated by the Python client's signing code (or a hardcoded known-good vector):
# recovers the expected address. Flipping one byte of `value` changes the recovered address
# so the from-check fails. A high-s signature is rejected before recovery.
curl -si localhost:8080/v1/proxy/mock-ok/ok -H "X-PAYMENT: $VALID_PAYLOAD_B64"   # 200 with the payer
```

**Commit.** `feat: recover and verify EIP-3009 payer signature with alloy`

---

### Task 2.5 — Per-stage latency instrumentation

**Objective.** `architecture.md` §11 and assumption A8. Instrument now, while there are only three
stages, rather than retrofitting.

**Implementation.**
- Add `metrics` + `metrics-exporter-prometheus`. Histograms named `tollgate_stage_seconds` with a
  `stage` label taking `decode`, `recover`, `capability`, `claim`, `upstream`, `total`.
- A small RAII timer helper in `tollgate-core` so each stage is two lines.
- Record `decode`, `recover`, `total` now; the other three are added by the phases that create them.
- `GET /metrics` exposing the Prometheus text format.

**Verification.**
```
curl -si localhost:8080/v1/proxy/mock-ok/ok -H "X-PAYMENT: $VALID_PAYLOAD_B64"
curl -s localhost:8080/metrics | grep tollgate_stage_seconds
# buckets present for stage="decode", stage="recover", stage="total", each with count >= 1
```

**Commit.** `feat: add per-stage prometheus latency histograms`

---

## Phase 2 Gate

The interop claim in `architecture.md` §4.1 is what this gate is really testing.

- [ ] Unpaid request returns 402 with a body that validates against the x402 payment-requirements
      shape (field names, casing, CAIP-2 network, base-unit amount as a string)
- [ ] **An off-the-shelf x402 client (the `x402` npm/pip package, not Tollgate code) parses the 402
      and produces a payment header the gateway accepts.** If no such client is available, verify
      against the published schema and record that substitution in the README
- [ ] A valid payment header returns 200 with the correct recovered payer
- [ ] Each of the seven field-validation rules rejects with its own reason
- [ ] Tampering with `value`, `to`, or `nonce` after signing causes rejection
- [ ] `/metrics` shows non-empty `decode`, `recover`, and `total` histograms
- [ ] `cargo test --workspace` green, `cargo clippy -- -D warnings` clean

---

# Phase 3 — Atomic claim

**Objective.** `architecture.md` §9. One Lua script performs nonce claim, cap checks, decrements, and
the reservation `XADD` atomically. A background task drains the stream into Postgres.

**After this phase exists:** a verified payment claims budget atomically. Replaying the same nonce
returns 409. Exceeding a cap returns 429 with a useful body. Reservations land in Postgres. Still no
proxying.

**Depends on:** Phase 2.

Note: caps at this stage are per-payer, keyed by the payer address, standing in for the single-level
case. Phase 5 generalises the same script to a capability chain. Write the script with the key list
already variadic so Phase 5 is a caller change, not a rewrite.

---

### Task 3.1 — The Lua claim script

**Objective.** The correctness centrepiece. Steps 1, 2, 3, 5, 6, 7 of `architecture.md` §9.1
(step 4, window budgets, is out of scope per assumption A4).

**Implementation.**
- `crates/tollgate-gateway/src/lua/claim.lua`, loaded with `SCRIPT LOAD` at startup, invoked by
  `EVALSHA` with `NOSCRIPT` fallback to `EVAL`.
- Keys: `KEYS[1]` = `tg:nonce:{nonce}`; `KEYS[2..n+1]` = `tg:cap:{hash}:spent` for each level;
  `KEYS[n+2]` = `tg:reservations`; plus a `tg:cap:{hash}:meta` key per level.
- Argv: amount, nonce TTL seconds, level count, and the reservation payload fields.
- Logic:
  1. `EXISTS KEYS[1]` → return `{"replay"}`
  2. for each level: read `meta`; if `amount > max_per_request` → return
     `{"cap_exceeded", level_index, cap_hash, max_per_request}`
  3. for each level: if `spent + amount > max_total` → return
     `{"insufficient", level_index, cap_hash, remaining}`
  4. `SET KEYS[1] 1 EX ttl`
  5. `INCRBY` every level's `spent`
  6. `XADD tg:reservations * ...` with nonce, payer, provider, amount, cap hashes
  7. return `{"ok", stream_id}`
- Missing `meta` for a level is an error return, never a silent pass.
- No `redis.call` after the first mutation may fail without leaving the script inconsistent: perform
  all reads and all checks before any write.

**Verification.**
```
cargo test -p tollgate-gateway policy::claim -- --ignored
# against a live Redis:
#  - first claim of a nonce returns ok; second returns replay and spent is unchanged
#  - amount > max_per_request returns cap_exceeded with the right level index and no spend recorded
#  - spend accumulating past max_total returns insufficient with the correct remaining
#  - after ok: nonce key exists with a TTL in range, spent incremented, XLEN tg:reservations grew by 1
#  - 100 concurrent claims of distinct nonces against max_total = 50*amount: exactly 50 return ok
#    (this is the v1 race in architecture.md §9.1 — the test must be concurrent, not sequential)
```

**Commit.** `feat: add atomic redis lua claim script for nonce and caps`

---

### Task 3.2 — Policy module: cap metadata seeding and script invocation

**Objective.** A typed Rust caller for the script, plus the code that populates `tg:cap:*:meta`.

**Implementation.**
- `policy.rs`: `ClaimRequest { nonce, amount, levels: Vec<CapLevel>, provider_slug, payer, ttl_secs }`
  and `enum ClaimOutcome { Ok { stream_id }, Replay, CapExceeded { level, cap_hash, limit },
  Insufficient { level, cap_hash, remaining } }`.
- Nonce TTL is computed as `valid_before - now + clock_skew`, never a constant
  (`architecture.md` §9.2 rule 2). Assert in code that it is `> 0`.
- A `POST /v1/budgets` dev endpoint seeding a payer-level budget:
  `{payer, max_total, max_per_request}` → writes `tg:cap:{keccak(payer)}:meta`. This is the
  single-level stand-in; Phase 5 replaces the source of these keys with real capabilities but keeps
  the key shape.

**Verification.**
```
curl -X POST localhost:8080/v1/budgets -d '{"payer":"0x…","max_total":"1.00","max_per_request":"0.05"}'
redis-cli HGETALL tg:cap:<hash>:meta          # max_total 1000000, max_per_request 50000
cargo test -p tollgate-gateway policy::ttl    # TTL = valid_before - now + skew, and a
                                              # payload whose validity already elapsed errors
```

**Commit.** `feat: add policy module wrapping the claim script`

---

### Task 3.3 — Wire the claim into the request path with correct status codes

**Objective.** `architecture.md` §4.3's failure table, for the outcomes that exist so far.

**Implementation.**
- In the proxy route, after signature recovery, call the claim. Map outcomes:
  - `Ok` → continue (still the debug 200 body for now)
  - `Replay` → **409**, no budget change
  - `CapExceeded` → **429** with `{"reason":"cap_exceeded","cap_hash":…,"limit":…,"level":…}`
  - `Insufficient` → **402** with `{"reason":"insufficient_funds"}` plus the requirements body
- Record the `claim` stage histogram.
- Add a `#[deny]`-style comment and a code-structure guarantee that no upstream call can occur before
  this returns `Ok` (`architecture.md` §10, "free shopping"). The proxy call in Phase 4 must live
  inside the `Ok` match arm, not after the match.

**Verification.**
```
# fund a budget of 0.01 with max_per_request 0.005, price 0.001
curl -si .../mock-ok/ok -H "X-PAYMENT: $P1"   # 200
curl -si .../mock-ok/ok -H "X-PAYMENT: $P1"   # 409  (same nonce)
# a payload with value 0.01 against max_per_request 0.005 -> 429 with cap_exceeded
# spend the budget down, then one more -> 402 with insufficient_funds
curl -s localhost:8080/metrics | grep 'stage="claim"'   # count > 0
```

**Commit.** `feat: enforce atomic claim in the request path with x402 status codes`

---

### Task 3.4 — Reservation stream drain into Postgres

**Objective.** `architecture.md` §9.3. Redis is a projection, Postgres is the truth, the stream
bridges them.

**Implementation.**
- `ledger.rs`: a Tokio task started at gateway boot that:
  - creates the consumer group `tg-writer` on `tg:reservations` (`MKSTREAM`, ignore BUSYGROUP)
  - loops `XREADGROUP ... BLOCK 1000 COUNT 100`
  - inserts each entry into `authorisations` with `status = 'reserved'`, `ON CONFLICT (nonce) DO NOTHING`
    (the nonce is the idempotency key end to end, `architecture.md` §9.4)
  - `XACK`s only after the Postgres commit
  - on startup, first drains pending entries with `XREADGROUP ... 0` before reading new ones
- Log and continue on a single-row failure; never let one bad entry stall the drain.

**Verification.**
```
# make a successful paid request
psql $DATABASE_URL -c "select nonce, payer, amount_wei, status from authorisations"
# one row, status 'reserved', amount matching
redis-cli XPENDING tg:reservations tg-writer     # 0 pending after the drain

# crash-consistency check:
docker compose stop postgres          # drain now fails
# make a paid request -> still 200, XPENDING shows 1
docker compose start postgres
# within a few seconds the row appears and XPENDING returns to 0
```

**Commit.** `feat: drain redis reservation stream into postgres ledger`

---

## Phase 3 Gate

- [ ] 100 concurrent requests against a budget that permits exactly 50 result in exactly 50 successes
      and 50 `insufficient_funds` — the total spend never exceeds `max_total`
- [ ] Replaying a nonce returns 409 and does not change `spent`
- [ ] Cap exceeded returns 429 with the cap hash and the limit that tripped
- [ ] Nonce TTL in Redis is always ≥ the payload's remaining validity (`redis-cli TTL` vs the
      payload's `validBefore`)
- [ ] Every successful claim produces exactly one `authorisations` row with `status='reserved'`
- [ ] Killing Postgres mid-run loses no reservations once it returns
- [ ] Reading the route handler confirms no upstream call is reachable before a successful claim

---

# Phase 4 — SSRF-safe proxy and request lifecycle

**Objective.** Complete the core paid-request path: proxy to the upstream through a pinned-IP
connector, stream the response back, and promote or void the reservation based on the outcome.

**After this phase exists:** a paid request returns real upstream data. Upstream 5xx and timeouts
void the reservation and credit the budget back. Registering a provider pointing at internal address
space is refused.

**Depends on:** Phase 3.

---

### Task 4.1 — SSRF validation: resolve and check every IP

**Objective.** `architecture.md` §10 SSRF steps 1 and 2. This is the item `prd.md` §10 rates most
likely to be found live.

**Implementation.**
- `ssrf.rs`: `resolve_and_validate(host, port) -> Result<Vec<SocketAddr>, SsrfError>`.
- Resolve through the system resolver (documented as needing to be a public resolver, not an internal
  one, in deployment).
- Reject if **any** resolved address is in: RFC 1918 (10/8, 172.16/12, 192.168/16), loopback
  (127/8, ::1), link-local (169.254/16, fe80::/10), `169.254.169.254` specifically, IPv4-mapped IPv6
  (`::ffff:0:0/96`, checked after unmapping), unique-local IPv6 (fc00::/7), `0.0.0.0/8`,
  multicast, and CGNAT (100.64/10).
- Reject non-http/https schemes and any URL with credentials or a non-standard port outside 80/443
  unless explicitly allowed by config.
- **Test-mode exception:** an explicit `TOLLGATE_ALLOW_LOOPBACK_UPSTREAM=true` env flag permits
  127.0.0.1 so the mock upstream works locally. Default false. This flag is the one thing that must
  never be true in a deployed config; note it in the README.

**Verification.**
```
cargo test -p tollgate-gateway ssrf::denylist
# table test: 10.0.0.1, 127.0.0.1, 169.254.169.254, ::1, fc00::1, ::ffff:127.0.0.1,
# 100.64.0.1, 0.0.0.0 all rejected with a named variant; 1.1.1.1 and 8.8.8.8 accepted;
# a hostname resolving to a mix of public and private addresses is rejected
```

**Commit.** `feat: add ssrf ip resolution and denylist validation`

---

### Task 4.2 — SSRF validation at provider registration

**Objective.** `prd.md` F7's "SSRF-safe" clause, enforced at the earliest point.

**Implementation.**
- In `POST /v1/providers`, call `resolve_and_validate` on the submitted `upstream_url` before insert.
- Reject with 400 `{"code":"unsafe_upstream","message":"upstream resolves to a disallowed address: …"}`.
- Registration-time validation is necessary but **not sufficient** — DNS can change between
  registration and request. Add a code comment saying so and pointing at Task 4.3, which is the real
  defence.

**Verification.**
```
curl -X POST localhost:8080/v1/providers -d '{"slug":"evil","upstream_url":"http://169.254.169.254/latest/meta-data","price":"0.001","payout_address":"0x…01"}'
# 400 unsafe_upstream
curl -X POST ... -d '{"slug":"evil2","upstream_url":"http://10.0.0.5/","...":"..."}'   # 400
# with TOLLGATE_ALLOW_LOOPBACK_UPSTREAM=true, http://127.0.0.1:9000 is accepted
```

**Commit.** `feat: reject unsafe upstream urls at provider registration`

---

### Task 4.3 — Pinned-IP hyper connector and streaming proxy

**Objective.** `architecture.md` §10 SSRF steps 3 and 5: pin the connection to the validated
`SocketAddr` so DNS rebinding between check and connect is impossible.

**Implementation.**
- `proxy.rs`: a custom `tower::Service<Uri>` connector that, given a URI, calls
  `resolve_and_validate` once, picks the first validated `SocketAddr`, and **dials that address
  directly** — never handing the hostname back to the resolver. Set the TLS SNI and `Host` header to
  the original hostname.
- Build a `hyper_util::client::legacy::Client` over this connector, one per gateway (connection
  pooling matters for the latency budget).
- `proxy_request(provider, path, req)`: forward method, path, query, and a filtered header set
  (strip `X-PAYMENT`, `X-Tollgate-*`, `Host`, hop-by-hop headers; add `X-Forwarded-For`).
- Stream the response body back with `axum::body::Body::from_stream` — do not buffer.
- **Do not follow redirects.** Return the 3xx to the caller as-is. (`architecture.md` §10 step 5 says
  re-validate on every hop; not following at all is the simpler correct MVP choice. Document it.)
- Apply `provider.timeout_ms` with `tokio::time::timeout`.
- Record the `upstream` stage histogram separately from `total` (`prd.md` §8 insists these never be
  conflated).

**Verification.**
```
# with the mock upstream running and a funded budget:
curl -si .../mock-ok/ok -H "X-PAYMENT: $P"      # 200 with {"result":"ok"} — real upstream data
curl -si .../mock-slow/slow -H "X-PAYMENT: $P2" # 504 after ~1s (provider timeout_ms 1000)
curl -s .../mock-ok/echo?q=hi -H "X-PAYMENT: $P3" | jq .headers
# no X-PAYMENT present upstream; Host is the upstream host

# rebinding test (architecture.md §10: "test it with a rebinding payload before you rely on it"):
cargo test -p tollgate-gateway proxy::rebinding -- --ignored
# a stub resolver returning a public IP on the first lookup and 127.0.0.1 on the second:
# assert the connector dials the first (validated) address both times

curl -s localhost:8080/metrics | grep 'stage="upstream"'   # populated, and distinct from stage="total"
```

**Commit.** `feat: add ssrf-pinned streaming reverse proxy`

---

### Task 4.4 — Reservation promotion and voiding

**Objective.** `architecture.md` §4.2 step 7 and §4.3. A reservation only becomes settleable after a
2xx; a failure credits the budget back.

**Implementation.**
- After the proxy returns:
  - 2xx → update `authorisations` to `status='queued'`, record `upstream_status`
  - non-2xx or timeout → `status='voided'`, `DECRBY` every level's `spent` by the amount, record
    `upstream_status`
- **Do not delete the nonce key on void.** The signature is still replayable until `validBefore`;
  releasing the nonce would reopen the free-shopping window. Add a comment saying so.
- Because the drain is asynchronous, the row may not exist yet when the proxy returns. Handle it: the
  promotion path writes a small Redis marker `tg:outcome:{nonce}` that the drain applies, and also
  attempts a direct `UPDATE ... WHERE nonce = $1` for the common case where the row already landed.
  Whichever runs second wins idempotently.
- The status transition must be one-way: `reserved → queued` and `reserved → voided` only. Enforce
  with `WHERE status = 'reserved'` in the UPDATE.

**Verification.**
```
curl -s .../mock-ok/ok -H "X-PAYMENT: $P"
psql -c "select status, upstream_status from authorisations where nonce='\x…'"   # queued, 200

before=$(redis-cli GET tg:cap:<h>:spent)
curl -si .../mock-fail/fail -H "X-PAYMENT: $P2"    # 502
psql -c "select status from authorisations where nonce='\x…'"                    # voided
redis-cli GET tg:cap:<h>:spent                                                    # equals $before
redis-cli EXISTS tg:nonce:<n>                                                     # 1 — still claimed
curl -si .../mock-fail/fail -H "X-PAYMENT: $P2"                                   # 409, not a retry
```

**Commit.** `feat: promote reservations on 2xx and void on upstream failure`

---

## Phase 4 Gate

This gate is the "core path works" checkpoint. `prd.md` F1, F2, F3, F7 are complete after it.

- [ ] Unpaid → 402 → pay → 200 with real upstream data, in one flow, using only x402-shaped messages
- [ ] Upstream 500 → 502 to the caller, reservation `voided`, budget credited back exactly
- [ ] Upstream timeout → 504, reservation `voided`, budget credited back
- [ ] A voided nonce cannot be re-presented (still 409)
- [ ] Provider registration refuses metadata, loopback, and RFC 1918 upstreams
- [ ] The rebinding test proves the connection is pinned, not re-resolved
- [ ] `X-PAYMENT` never reaches the upstream
- [ ] `/metrics` reports `upstream` separately from `total`
- [ ] `cargo test --workspace` green including `--ignored` integration tests

---

# Phase 5 — Capability delegation

**Objective.** The headline feature (`prd.md` decision #4, F4). EIP-712 capability chains, verified
attenuation, and every level decremented on every charge.

**After this phase exists:** a supervisor can sign a capability for a worker offline, the worker
presents the chain on each request, and a charge draws down every ancestor's budget. A worker hitting
its own cap stops while its siblings continue; a supervisor running dry stops all of them.

**Depends on:** Phases 3 and 4.

---

### Task 5.1 — Capability EIP-712 type and hashing

**Objective.** The struct from `architecture.md` §5.1, its EIP-712 hash, and signature recovery.

**Implementation.**
- `capability.rs`: `alloy::sol!` definition of `Capability { address issuer; address subject;
  bytes32 parent; uint256 maxTotal; uint256 maxPerRequest; bytes32 allowlistHash; uint64 notBefore;
  uint64 expiry; uint32 maxDepth; bytes32 salt; }` — field order matters, it is part of the type hash.
- EIP-712 domain: `name: "Tollgate"`, `version: "1"`, `chainId` from config (this is the cross-chain
  replay guard, `architecture.md` §10), no `verifyingContract`.
- `allowlist_hash(slugs: &[String]) -> B256` = `keccak256` of the slugs sorted lexicographically and
  joined with `\n`. `0x0` means inherit from the parent (assumption A9).
- `struct_hash()` and `recover_issuer(signature)`.
- The same domain and hashing must be reimplemented in Python in Phase 8. Emit a
  `tests/fixtures/capability_vectors.json` containing three signed capabilities with their expected
  hashes so both implementations test against the same vectors.

**Verification.**
```
cargo test -p tollgate-gateway capability::hash
# - a known capability hashes to a fixed expected value (golden test, so a field-order change breaks it)
# - recover_issuer returns the signer for a valid signature
# - changing maxTotal by 1 changes the hash
# - allowlist_hash(["b","a"]) == allowlist_hash(["a","b"])
# - the fixture file is written and readable
```

**Commit.** `feat: add eip-712 capability struct, hashing and recovery`

---

### Task 5.2 — Capability chain decode from the request header

**Objective.** Parse `X-Tollgate-Capability` into an ordered chain, root first.

**Implementation.**
- Header value: base64 of a JSON array `[{capability: {...}, signature: "0x…", allowlist: ["…"]}, …]`,
  root first, the plaintext allowlist required whenever `allowlistHash != 0`.
- **Reject depth > 4 before any signature work** (`architecture.md` §5.2 rule 1) — this is a
  denial-of-service guard, so it must come first.
- Reject an empty chain, a chain whose declared allowlist does not hash to `allowlistHash`, and any
  malformed link, each with its own error.

**Verification.**
```
cargo test -p tollgate-gateway capability::decode
# valid 3-link chain decodes in root-first order; 5-link chain rejected with depth_exceeded and
# zero signature recoveries performed (assert with a counter or by timing);
# allowlist not matching its hash rejected; empty array rejected
```

**Commit.** `feat: decode capability chain from request header`

---

### Task 5.3 — Chain verification and attenuation rules

**Objective.** All six rules in `architecture.md` §5.2. This is the security core of the headline
feature; it gets its own task and its own exhaustive test.

**Implementation.**
- `verify_chain(chain, payment_signer, provider_slug, now, revoked_set) -> Result<Vec<CapLevel>, CapError>`,
  walking root-first and returning on the first failure:
  1. depth ≤ 4 (already done in 5.2, re-asserted)
  2. every link's signature recovers to its declared `issuer`
  3. for `i > 0`: `issuer(i) == subject(i-1)` and `parent(i) == hash(link i-1)`; for the root,
     `parent == 0x0`
  4. attenuation: `maxTotal(i) <= maxTotal(i-1)`, `maxPerRequest(i) <= maxPerRequest(i-1)`,
     `expiry(i) <= expiry(i-1)`, `notBefore(i) >= notBefore(i-1)`, `maxDepth(i) < maxDepth(i-1)`,
     and `allowlist(i) ⊆ allowlist(i-1)`
  5. `subject(last) == payment_signer` (this is what makes a stolen capability useless)
  6. no link's hash is in `tg:revoked`
  7. `now` within every link's `[notBefore, expiry]`
  8. `provider_slug` is in the effective allowlist (the leaf's, or the nearest ancestor's if the leaf
     inherits)
- Error → HTTP mapping from `architecture.md` §4.3:
  - attenuation violation → **400** (malformed chain, not a policy failure)
  - expired / revoked → **403** with `capability_expired` / `capability_revoked`
  - provider not in allowlist → **403** `provider_not_allowed`
  - subject mismatch → **403** `subject_mismatch`
  - bad signature or broken parent link → **400**
- Record the `capability` stage histogram.

**Verification.**
```
cargo test -p tollgate-gateway capability::verify
# one test per rule, each with a chain that violates exactly that rule and asserts the exact
# error variant, plus a fully valid 3-link chain that passes. Specifically:
#   - child maxTotal > parent maxTotal            -> 400 attenuation
#   - child expiry > parent expiry                -> 400 attenuation
#   - child allowlist adds a slug parent lacks    -> 400 attenuation
#   - child maxDepth >= parent maxDepth           -> 400 attenuation
#   - link 1's issuer != link 0's subject         -> 400 broken_chain
#   - link 1's parent != hash(link 0)             -> 400 broken_chain
#   - leaf subject != payment signer              -> 403 subject_mismatch
#   - expiry in the past                          -> 403 capability_expired
#   - hash present in tg:revoked                  -> 403 capability_revoked
#   - request to a slug outside the allowlist     -> 403 provider_not_allowed
```

**Commit.** `feat: verify capability chain attenuation and binding rules`

---

### Task 5.4 — Multi-level decrement in the claim script

**Objective.** `architecture.md` §5.2: "Every charge decrements every capability in the chain, not
just the leaf." This is what makes the delegation tree visibly drain.

**Implementation.**
- Replace the single payer-level key list with one key pair per chain link (up to 4 levels + the
  funding root). The script from Task 3.1 was written variadic, so this is a caller change plus
  `meta` seeding.
- On a verified chain, upsert each link's `tg:cap:{hash}:meta` from the capability itself
  (`maxTotal`, `maxPerRequest`, `expiry` TTL) — the capability is self-describing, so no server-side
  registration step is needed, which is what makes `delegate()` a pure local signature
  (`architecture.md` §7.3).
- Persist each newly seen capability into the `capabilities` table on first use (issuer, subject,
  parent hash, depth, limits, allowlist, signature).
- `cap_exceeded` and `insufficient` responses now name **which link** tripped. Build the 429 body
  properly (`architecture.md` §4.3: "the 429 body is worth getting right"):
  `{"reason":"cap_exceeded","tripped_level":1,"capability_hash":"0x…","chain":[{"hash":"0x…",
  "subject":"0x…","max_total":"…","spent":"…","remaining":"…"}, …]}`.
- Voiding (Task 4.4) must now credit back **every** level.

**Verification.**
```
cargo test -p tollgate-gateway policy::multilevel -- --ignored
#  - one charge against a 3-link chain increments all three spent counters by the amount
#  - a void decrements all three back
#  - a leaf with a small maxTotal trips first and names level 2
#  - a root with a small maxTotal trips even when the leaf has room, and names level 0

# end to end:
# supervisor cap 1.00, workerA cap 0.50, workerB cap 0.50, price 0.10
# spend 5 requests as workerA -> workerA blocked at its own cap, supervisor spent == 0.50
# spend 5 requests as workerB -> succeeds
# a 6th from either -> supervisor 429 naming level 0
redis-cli GET tg:cap:<supervisor>:spent    # 1000000
```

**Commit.** `feat: decrement every capability level atomically on each charge`

---

### Task 5.5 — Revocation endpoint

**Objective.** `architecture.md` §5.2: revocation is off-chain, stored in a Redis set, and that is
sufficient because the gateway is the enforcement point.

**Implementation.**
- `POST /v1/capabilities/{hash}/revoke` with a body carrying an EIP-712 `Revocation { bytes32
  capabilityHash; uint64 issuedAt; }` signed by the capability's **issuer** (or any ancestor issuer
  in the chain — an ancestor can revoke a descendant).
- Verify the signature, confirm the signer is the issuer or an ancestor issuer (look the chain up in
  the `capabilities` table), then `SADD tg:revoked {hash}` and set `revoked_at` in Postgres.
- 403 if the signer is unrelated to the capability.
- `GET /v1/capabilities/{hash}` returning the stored capability with `revoked_at` and current spend.

**Verification.**
```
# with a working delegation chain:
curl -X POST localhost:8080/v1/capabilities/0x…/revoke -d "$SIGNED_REVOCATION"   # 200
curl -si .../mock-ok/ok -H "X-PAYMENT: $P" -H "X-Tollgate-Capability: $CHAIN"    # 403 capability_revoked
redis-cli SISMEMBER tg:revoked 0x…                                               # 1
psql -c "select revoked_at from capabilities where hash='\x…'"                   # not null
# a revocation signed by an unrelated key -> 403
# revoking a parent also blocks the child (the child's chain includes the revoked link)
```

**Commit.** `feat: add signed capability revocation endpoint`

---

### Task 5.6 — Rebuild capability counters from Postgres at boot

**Objective.** `architecture.md` §9.3: "Rebuild at boot, not lazily. A cold-start miss that lets an
agent overspend is exactly the bug that surfaces later."

**Implementation.**
- At gateway startup, before binding the listener:
  1. `SELECT capability_hash, SUM(amount_wei) FROM authorisations WHERE status IN
     ('reserved','queued','settled') GROUP BY capability_hash` — voided and failed excluded.
  2. Because every level is charged, the per-level attribution needs the chain: store the full chain
     of hashes on the authorisation row (add a `chain_hashes BYTEA[]` column in a second migration
     `0002_chain_hashes.sql`) and sum per hash across the array.
  3. `SET tg:cap:{hash}:spent` to the computed total for every hash found.
  4. Rehydrate `tg:cap:{hash}:meta` from the `capabilities` table.
  5. Reload `tg:revoked` from `capabilities WHERE revoked_at IS NOT NULL`.
  6. Then drain any unacknowledged stream entries (already implemented in Task 3.4's startup path).
- Log a one-line summary: capabilities rebuilt, total reconstructed spend.

**Verification.**
```
# spend 0.30 across a 3-link chain
redis-cli GET tg:cap:<leaf>:spent            # 300000
redis-cli FLUSHALL                            # simulate total redis loss
# restart the gateway
redis-cli GET tg:cap:<leaf>:spent            # 300000 again, rebuilt from postgres
redis-cli SMEMBERS tg:revoked                 # revoked hashes restored
# a request that would exceed the rebuilt cap is still rejected -> no cold-start overspend
```

**Commit.** `feat: rebuild capability spend counters from postgres at boot`

---

## Phase 5 Gate

This gate proves the headline feature. `prd.md` F3 and F4 complete here.

- [ ] A supervisor signs a capability for a worker with **no server round-trip**, and the worker's
      first request is accepted on the strength of the signature alone
- [ ] A 3-link chain verifies and charges all three levels per request
- [ ] Every one of the ten attenuation/binding violations in Task 5.3 is rejected with its documented
      status code and reason
- [ ] **The `prd.md` §9 step 3 scenario:** three workers with different caps run concurrently; worker
      2 hits its cap and is refused — **402 `insufficient_funds`**, naming its own capability as the
      tripped level — while workers 1 and 3 continue succeeding

      > Corrected 2026-08-06: this line originally said 429. Exhausting a **total** cap is
      > `insufficient_funds` → **402** (`architecture.md` §4.3, implemented in Task 3.3, and asserted
      > by the Phase 3 Gate's "50 `insufficient_funds`"). **429 `cap_exceeded`** is the **per-request**
      > cap, which is the item two lines below. `prd.md` §9 step 3 — the requirement this encodes —
      > names no status code, only that worker 2 "hits its cap mid-task and stops". Changing the code
      > to 429 here would have regressed the Phase 3 Gate. See `NOTES.md` decision D11.

- [ ] When the supervisor's pool empties, all three workers stop at once even though none has hit its
      own cap
- [ ] The 429 body contains the tripped level, the capability hash, and remaining budget per level
      (the 402 body carries the same three, so either refusal is equally diagnosable)
- [ ] A revoked capability is refused; revoking a parent refuses the child
- [ ] `FLUSHALL` + restart restores exact spend from Postgres with no overspend window
- [ ] A stolen capability presented by a different signer is refused (`subject_mismatch`)

---

# Phase 6 — Contract and settler

**Objective.** `prd.md` F5. Queued authorisations become one on-chain transaction on Base Sepolia.

**After this phase exists:** the settler binary drains `status='queued'` rows in batches of 50 (or
every 10s), submits them via `BatchSettler`, and writes back `tx_hash` and `status='settled'`. A
block explorer shows one transaction covering many charges.

**Depends on:** Phase 4 (queued rows exist). Independent of Phase 5.

---

### Task 6.1 — Foundry project and `BatchSettler` contract

**Objective.** The ~50-line pass-through from `architecture.md` §6.

**Implementation.**
- `forge init contracts --no-git`. `foundry.toml` with `optimizer = true`, `optimizer_runs = 200`,
  a `base_sepolia` RPC endpoint entry, and `solc` pinned.
- `contracts/src/BatchSettler.sol` exactly as specified in `architecture.md` §6: the `Auth` struct,
  `Settled` / `SettleFailed` events, and `settle(Auth[] calldata)` using **try/catch per item so one
  failure never reverts the batch**.
- `IERC3009` interface with `transferWithAuthorization`.
- No owner, no pause, no upgradeability, no withdraw. Add a header comment stating those absences and
  why (`architecture.md` §6 properties list).

**Verification.**
```
cd contracts && forge build            # compiles
grep -c 'function' src/BatchSettler.sol  # exactly one external function
grep -iE 'owner|pause|upgrade|withdraw|selfdestruct' src/BatchSettler.sol   # no matches
```

**Commit.** `feat: add BatchSettler pass-through settlement contract`

---

### Task 6.2 — Contract tests including the griefing case

**Objective.** Prove skip-and-continue, and measure gas so the `prd.md` §8 cost claim is measured
rather than asserted.

**Implementation.**
- `contracts/test/BatchSettler.t.sol` against a mock EIP-3009 token (or a fork of Base Sepolia USDC
  via `forge test --fork-url`):
  - a batch of 3 valid auths: all transfer, 3 `Settled` events
  - a batch where item 2 has a used nonce: items 1 and 3 still settle, one `SettleFailed` emitted,
    **the transaction does not revert** (this is the v1 griefing vector the design exists to fix)
  - a batch of 50 valid auths: succeeds
  - an empty batch: no-op, no revert
- `forge snapshot` committed as `contracts/.gas-snapshot`.

**Verification.**
```
cd contracts
forge test -vv                    # all green; the mixed batch shows 2 Settled + 1 SettleFailed
forge snapshot                    # writes .gas-snapshot
# read gas for the 50-item batch; divide by 50; record the per-charge gas in the README
# alongside a Base Sepolia gas price, and confirm the "under $0.001" claim from prd.md §8
# with the measured number, not an estimate
```

**Commit.** `test: add BatchSettler tests and gas snapshot`

---

### Task 6.3 — Deploy script and local anvil fork setup

**Objective.** A repeatable deploy, first to a fork, later to real Base Sepolia.

**Implementation.**
- `contracts/script/Deploy.s.sol` deploying `BatchSettler` and logging the address.
- `scripts/anvil.sh` starting `anvil --fork-url $BASE_SEPOLIA_RPC` and, using `anvil_setBalance` /
  `anvil_impersonateAccount`, funding a test EOA with forked USDC so integration tests have real
  USDC semantics without a faucet.
- README section: how to run the fork, deploy, and point `.env` at the deployed address.

**Verification.**
```
./scripts/anvil.sh &
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast --private-key $ANVIL_KEY
cast code $BATCH_SETTLER_ADDRESS --rpc-url http://localhost:8545      # non-empty
cast call $USDC 'balanceOf(address)(uint256)' $TEST_EOA --rpc-url http://localhost:8545  # funded
```

**Commit.** `chore: add BatchSettler deploy script and anvil fork setup`

---

### Task 6.4 — Settler worker: batch selection and submission

**Objective.** `architecture.md` §7.2. A separate binary, separate process.

**Implementation.**
- `crates/tollgate-settler/src/main.rs`: loop that flushes at **50 authorisations or 10 seconds,
  whichever comes first**.
- `batch.rs`: `SELECT ... FROM authorisations WHERE status = 'queued' ORDER BY created_at LIMIT 50
  FOR UPDATE SKIP LOCKED`. **Reads only `queued`** — a `reserved` row is never visible to it, which
  closes the v1 race where a flush could settle a charge the request handler was about to void.
- `batch_id = keccak256(sorted nonce set)` so a duplicate submission is a deterministic no-op.
  Mark the rows with `batch_id` inside the same transaction that selects them.
- `chain.rs`: build the `Auth[]` from each row's stored payload and signature, call
  `BatchSettler.settle` via an `alloy` provider, EIP-1559 with a fixed multiplier (**no gas oracle**).
- One in-flight transaction at a time, sequential EOA nonces. Document the resulting ceiling
  (~25 settlements/sec at 50/batch and 2s blocks) in the README rather than hiding it.

**Verification.**
```
# against anvil with 3 queued authorisations:
cargo run -p tollgate-settler
# logs one batch of 3, one tx hash
cast receipt $TX --rpc-url http://localhost:8545    # status 1, 3 Settled events
psql -c "select status, tx_hash, batch_id from authorisations"   # all 3 settled with the same batch_id
# run the settler again -> no batch submitted (nothing queued)
```

**Commit.** `feat: add settler worker with batched on-chain submission`

---

### Task 6.5 — Settlement result handling and retries

**Objective.** `architecture.md` §4.3 last row and §7.2: a failing item requeues alone; the other 49
settle. Never blind-retry after an ambiguous timeout.

**Implementation.**
- Parse the receipt logs: for each `Settled(nonce)` → `status='settled'`, `settled_at=now()`,
  `tx_hash`. For each `SettleFailed(nonce, reason)` → increment a `retry_count` column (migration
  `0003_retry_count.sql`), return the row to `status='queued'`, and log the reason.
- After 3 failures → `status='failed'` and an error-level log line (the alert, per §4.3).
- Before resubmitting after a transaction timeout or an ambiguous result, call
  `USDC.authorizationState(from, nonce)` on-chain. If true, the charge already settled — mark it
  settled and do not resubmit.
- Handle receipt-not-found by polling with a bounded deadline, then falling into the
  `authorizationState` check.

**Verification.**
```
# construct a batch of 3 where one nonce was already used on-chain:
cargo run -p tollgate-settler
psql -c "select nonce, status, retry_count from authorisations"
# two settled, one queued with retry_count = 1
# run twice more -> that row reaches status 'failed', an error log is emitted,
# and the two good rows are untouched

# ambiguous-timeout path: unit test with a stubbed provider that times out but where
# authorizationState returns true -> the row is marked settled, no second submission
```

**Commit.** `fix: requeue failed settlement items individually with bounded retries`

---

### Task 6.6 — Deploy to Base Sepolia and settle for real

**Objective.** The block-explorer moment in `prd.md` §9 step 4.

**Implementation.**
- Deploy `BatchSettler` to Base Sepolia (84532) with `Deploy.s.sol`; verify on Basescan.
- Record the address in `.env.example` and the README, with the Basescan link.
- Point a `.env.base-sepolia` profile at the real RPC and native USDC (`0x036CbD53842c5426634e7929541eC2318f3dCF7e`
  — **confirm this address against Circle's documentation before use; do not trust it from this
  document**). Confirm it is native USDC, not bridged (`prd.md` decision #3).
- Fund a test payer wallet with testnet USDC, run 10 paid requests through the gateway, run the
  settler once.

**Verification.**
```
# after the run:
psql -c "select count(*) from authorisations where status='settled'"     # 10
# open the tx on sepolia.basescan.org: one transaction, 10 Settled events,
# 10 USDC Transfer events to the provider payout address
cast call $USDC 'balanceOf(address)(uint256)' $PAYOUT --rpc-url $BASE_SEPOLIA_RPC
# increased by exactly 10 * price
# record the actual gas used and the per-charge cost in the README
```

**Commit.** `chore: deploy BatchSettler to base sepolia and verify live settlement`

---

## Phase 6 Gate

`prd.md` F5 complete. This is the end-to-end money path.

- [ ] `forge test` green, including the mixed batch where one item fails and the rest settle
- [ ] `.gas-snapshot` committed; measured per-charge cost recorded in the README with the arithmetic
- [ ] The settler never selects a `reserved` row
- [ ] 10 real paid requests settle in one Base Sepolia transaction, visible on Basescan
- [ ] The provider's payout address balance increases by exactly the sum of the charges
- [ ] A failed item requeues alone and reaches `failed` after 3 attempts without affecting others
- [ ] An ambiguous timeout consults `authorizationState` and never double-submits
- [ ] The settler EOA throughput ceiling is stated in the README

---

# Phase 7 — Receipts and expense report

**Objective.** `prd.md` F6 and user story 8: a signed artifact proving what an agent spent, on what,
and whether the upstream returned 200 — the thing the chain alone cannot show.

**After this phase exists:** every charge produces a gateway-signed receipt, and an expense report
endpoint rolls spend up across a whole delegation tree.

**Depends on:** Phases 5 and 6 (needs the chain for rollup and `tx_hash` for proof).

---

### Task 7.1 — Sign a receipt per charge

**Objective.** A verifiable per-charge artifact.

**Implementation.**
- `ledger.rs`: an EIP-712 `Receipt { bytes32 nonce; address payer; bytes32 capabilityHash;
  string providerSlug; uint256 amount; uint16 upstreamStatus; uint64 timestamp; }` signed with
  `cfg.gateway_signing_key`.
- Sign at promotion time (on 2xx), store in `authorisations.receipt_sig`.
- Return the receipt inline on the successful proxy response as an `X-Tollgate-Receipt` header
  (base64 JSON with the signature) so the client gets it in the same round trip.
- `GET /v1/receipts/{nonce}` returning the receipt plus `tx_hash` once settled.
- `GET /v1/gateway-key` returning the gateway's public signing address so anyone can verify.

**Verification.**
```
curl -si .../mock-ok/ok -H "X-PAYMENT: $P" | grep X-Tollgate-Receipt   # present
curl -s localhost:8080/v1/receipts/0x<nonce> | jq
# amount, provider, upstream_status 200, signature present; tx_hash null before settlement
cargo test -p tollgate-gateway ledger::receipt_signature
# recovering the signature yields the address returned by /v1/gateway-key;
# altering the amount in the receipt body breaks verification
# after running the settler: tx_hash is populated
```

**Commit.** `feat: sign and expose per-charge spend receipts`

---

### Task 7.2 — Expense report endpoint

**Objective.** `prd.md` user story 8 and the `sup.expense_report()` call in `architecture.md` §7.3:
rolled up across the whole delegation tree.

**Implementation.**
- `GET /v1/expense-report?capability=0x…` (or `?payer=0x…` for the no-delegation case).
- Walk the `capabilities` table downward from the given hash to build the subtree, then aggregate
  `authorisations` whose `chain_hashes` contains any node.
- Response shape:
  ```json
  {"root": "0x…",
   "total_spent": "0.42",
   "nodes": [{"hash":"0x…","subject":"0x…","depth":0,"spent":"0.42","max_total":"1.00",
              "remaining":"0.58","children":[…]}],
   "charges": [{"nonce":"0x…","provider":"geocode","amount":"0.001","upstream_status":200,
                "tx_hash":"0x…","settled_at":"…","receipt_sig":"0x…"}]}
  ```
- Exclude `voided` charges from totals; include `failed` ones in a separate count.
- Sign the whole report with the gateway key (`prd.md` story 8 says "signed expense report").

**Verification.**
```
# after the three-worker scenario from the Phase 5 gate:
curl -s "localhost:8080/v1/expense-report?capability=0x<supervisor>" | jq
# nodes: supervisor + 3 workers, correct depths, each remaining = max_total - spent
# total_spent equals the sum of non-voided charges
# every settled charge has a tx_hash; the voided one is absent from totals
# the report signature verifies against /v1/gateway-key
```

**Commit.** `feat: add signed expense report endpoint with delegation rollup`

---

### Task 7.3 — Public feed endpoint

**Objective.** Data source for the dashboard's public feed (`architecture.md` §7.4 view 3).

**Implementation.**
- `GET /v1/feed?limit=50&since=<cursor>` returning recent charges: payer, provider, amount,
  upstream status, tx hash, timestamp. Payer address shown in full (this is testnet, and the demo
  depends on being able to check it on Basescan).
- `GET /v1/stats` returning the latency percentiles from the metrics registry, split into
  `gateway_p50`, `gateway_p99`, `upstream_p50`, plus settlement stats: batch size, gas per batch, gas
  per charge, queue depth, and mean queued→settled time (`architecture.md` §11).
- Simple polling, not SSE. Post-MVP if a live stream is wanted.

**Verification.**
```
curl -s 'localhost:8080/v1/feed?limit=10' | jq 'length'          # <= 10, newest first
curl -s localhost:8080/v1/stats | jq
# gateway_p50 and upstream_p50 are separate fields with plausible values;
# gas_per_charge is populated after a settlement run;
# queue_depth matches `select count(*) from authorisations where status='queued'`
```

**Commit.** `feat: add public charge feed and latency stats endpoints`

---

## Phase 7 Gate

- [ ] Every successful charge carries a receipt header verifiable against `/v1/gateway-key`
- [ ] Tampering with any receipt field breaks verification
- [ ] The expense report reproduces the three-worker run: per-worker spend, per-endpoint breakdown,
      a tx hash on every settled charge
- [ ] Voided charges are excluded from expense totals
- [ ] `/v1/stats` reports gateway and upstream latency as separate numbers, never summed
- [ ] `gas_per_charge` in `/v1/stats` matches the `forge snapshot` figure within a reasonable margin

---

# Phase 8 — Python client library

**Objective.** `prd.md` F9 and user story 7. `pip install`-able, handles the 402/sign/retry loop, and
exposes `delegate()` as one local signature.

**After this phase exists:** the code sample in `architecture.md` §7.3 runs verbatim.

**Depends on:** Phases 5 and 7 (capability format and receipt format must be frozen).

---

### Task 8.1 — Package skeleton and EIP-3009 payload signing

**Objective.** Sign a payment payload in Python that the Rust gateway accepts.

**Implementation.**
- `client/pyproject.toml`: package `tollgate-client`, deps `httpx`, `eth-account`, `eth-utils`.
- `client/tollgate/signing.py`: build the USDC EIP-712 `TransferWithAuthorization` typed data and
  sign it with `eth_account.Account.sign_typed_data`. Nonce is 32 random bytes.
- `validBefore = now + 300` by default, matching `quote_validity_seconds` (`architecture.md` §9.2).
- `encode_payment_header(payload) -> str` producing the base64 x402 header.

**Verification.**
```
cd client && pip install -e . && pytest tests/test_signing.py
# the Python-produced header decodes and verifies in Rust:
cargo test -p tollgate-gateway verify::python_vector
# (the Python test writes tests/fixtures/payment_vector.json; the Rust test reads it —
#  this cross-language fixture is the point of the task)
```

**Commit.** `feat: add python client package with eip-3009 payment signing`

---

### Task 8.2 — `Session` with the 402 retry loop

**Objective.** `prd.md` user story 7: 402 → sign → retry → data, in one call.

**Implementation.**
- `client/tollgate/session.py`: `Session(signer, network="base-sepolia", capability=None)` with
  `get`, `post`, and a generic `request`.
- Flow: send unpaid → on 402 parse `accepts[0]` → sign for the quoted amount and `payTo` → resend
  with `X-PAYMENT` (and `X-Tollgate-Capability` if a capability is set) → return the response.
- **Never retry more than once** on a 402; a second 402 is an error, not a loop.
- Map gateway errors to typed exceptions: `InsufficientBudget`, `CapExceeded` (carrying the tripped
  level and remaining budget from the 429 body), `CapabilityRevoked`, `CapabilityExpired`,
  `NonceReplay`, `UpstreamFailed`.
- Collect the `X-Tollgate-Receipt` header into `session.receipts()`.
- Mirror **only** per-request cap and expiry locally for fast failure — do not mirror the whole
  policy engine (`architecture.md` §7.3: the server is the security boundary).

**Verification.**
```
pytest client/tests/test_session.py -k integration
# against a running gateway + mock upstream:
#  - s.get(".../mock-ok/ok") returns 200 and the upstream body, having transparently paid
#  - exactly two HTTP requests were made (assert with a transport recorder)
#  - s.receipts() has one entry with a verifiable signature
#  - exceeding the cap raises CapExceeded with .tripped_level and .remaining set
#  - a request above the locally mirrored per-request cap raises before any HTTP call is made
```

**Commit.** `feat: add session with automatic 402 payment retry`

---

### Task 8.3 — `delegate()` and capability signing

**Objective.** The line that sells the feature: one local signature, no server round-trip.

**Implementation.**
- `client/tollgate/capability.py`: the Tollgate EIP-712 domain and `Capability` type, matching
  Task 5.1 byte for byte, verified against `tests/fixtures/capability_vectors.json`.
- `Session.delegate(subject, max_total, max_per_request, allowlist, expiry_seconds, max_depth=1)`
  returning a serialisable capability chain (the parent's chain plus the new link).
- Client-side attenuation check before signing: refuse to sign a link that exceeds the parent's
  limits, with a clear error. The server enforces it anyway, but failing locally is better UX.
- `Capability.serialize()` / `deserialize()` producing exactly the base64 header format from
  Task 5.2.
- `Session.expense_report()` calling `/v1/expense-report` for the session's root capability.

**Verification.**
```
pytest client/tests/test_capability.py
#  - the hash of a fixture capability matches the Rust golden value exactly
#  - delegate() makes zero HTTP requests (assert with a transport recorder — this is the claim)
#  - delegating more than the parent holds raises AttenuationError locally
#  - a chain produced by delegate() is accepted by the live gateway
pytest client/tests/test_e2e.py
#  - the architecture.md §7.3 code sample runs verbatim and prints receipts and an expense report
```

**Commit.** `feat: add local capability delegation to the python client`

---

### Task 8.4 — Client README and quickstart

**Objective.** `prd.md` metric: "provider integration measured live, under 3 minutes from zero."

**Implementation.**
- `client/README.md`: install, the provider path (register an endpoint with one curl), the agent path
  (the §7.3 sample), the exception table, and what is mocked.
- A `client/examples/quickstart.py` that runs against a local stack in under 20 lines.

**Verification.**
```
# from a clean shell, following only the README:
# start stack -> register a provider -> run quickstart.py -> paid response printed
# time the whole thing; record the measured duration in the root README
```

**Commit.** `docs: add python client readme and quickstart example`

---

## Phase 8 Gate

- [ ] `pip install -e client/` then the `architecture.md` §7.3 sample runs verbatim, unmodified
- [ ] `delegate()` issues zero HTTP requests
- [ ] Python capability hashes match the Rust golden vectors byte for byte
- [ ] The 402 loop makes exactly two HTTP requests and never loops
- [ ] Every gateway error maps to a typed exception carrying the useful fields
- [ ] Following the README from zero produces a paid response in under 3 minutes, timed

---

# Phase 9 — Dashboard

**Objective.** `prd.md` F8 and risk "reads as invisible infrastructure". `architecture.md` §7.4:
"The delegation tree is the product surface."

**After this phase exists:** three working views — provider, agent (delegation tree), public feed.

**Depends on:** Phase 7 (all three views read Phase 7 endpoints).

---

### Task 9.1 — Next.js app scaffold and API client

**Objective.** A running dashboard shell with typed access to the gateway.

**Implementation.**
- `npx create-next-app@latest dashboard --ts --tailwind --app --no-src-dir`, Next.js 14, App Router.
- `dashboard/lib/api.ts`: typed fetchers for `/v1/providers`, `/v1/feed`, `/v1/stats`,
  `/v1/expense-report`, `/v1/receipts/{nonce}`.
- A shared polling hook `usePolling(fetcher, intervalMs)` — polling, not websockets (MVP simplicity).
- Layout with nav across the three views. Dark, dense, monospace for addresses and hashes.
- Enable CORS on the gateway for the dashboard origin (small gateway change, include it here).

**Verification.**
```
cd dashboard && npm run build && npm run dev
# http://localhost:3000 loads, nav links to /provider, /agent, /feed
# browser console has no CORS errors when the gateway is running
```

**Commit.** `feat: scaffold next.js dashboard with gateway api client`

---

### Task 9.2 — Provider view

**Objective.** `architecture.md` §7.4 view 1 and `prd.md` stories 1 and 2.

**Implementation.**
- `/provider`: a registration form (slug, upstream URL, price in USDC, payout address, timeout)
  posting to `/v1/providers`, showing the `unsafe_upstream` error inline when SSRF validation refuses.
- Below it, a live request log for the selected provider: payer, path, upstream status, amount,
  tx hash linked to Basescan, timestamp. Polled every 2s.
- Empty state explaining what to do first.

**Verification.**
```
# in the browser: register "geocode" through the form -> appears in the provider list
# submit http://169.254.169.254/ -> the inline error shows "unsafe_upstream"
# run a paid request from the Python client -> the row appears in the log within ~2s
# after running the settler, the tx hash appears and links to a real Basescan page
```

**Commit.** `feat: add provider dashboard view with registration and live log`

---

### Task 9.3 — Delegation tree view

**Objective.** The screen that matters most (`architecture.md` §7.4, `prd.md` risk mitigation).

**Implementation.**
- `/agent?capability=0x…`: render the `expense-report` node tree as an actual tree — nested nodes,
  connector lines, one card per capability showing subject address, allowlist, expiry, and a budget
  bar (`spent` / `max_total`).
- Poll every 1s; animate the bar so budgets are visibly *draining*, not stepping.
- A node at its cap turns red and shows "capped". Its siblings stay green. **This visual difference
  is the deliverable** — `prd.md` §9 step 3.
- A revoked node is struck through.
- Below the tree, that subtree's charge list.

**Verification.**
```
# run the three-worker scenario while the page is open:
# - supervisor's bar visibly drains as all three workers spend
# - worker 2's card turns red and reads "capped" while 1 and 3 keep filling
# - when the supervisor's bar empties, all three stop
# - revoke a worker's capability -> its node is struck through within ~1s
# take a screenshot of the capped state and save it to docs/ for the README
```

**Commit.** `feat: add live delegation tree view with draining budget bars`

---

### Task 9.4 — Public feed and latency panel

**Objective.** `architecture.md` §7.4 view 3.

**Implementation.**
- `/feed`: a scrolling table of every settled charge — payer, provider, amount, upstream status,
  tx hash (Basescan link), age. New rows enter at the top with a brief highlight.
- A latency panel above it showing gateway p50 / p99 and upstream p50 as **separate** numbers with
  explicit labels ("Tollgate added" vs "upstream"), because `prd.md` §8 warns they will be conflated.
- A settlement panel: last batch size, gas used, gas per charge, cost per charge in USD, queue depth.

**Verification.**
```
# drive 100 requests in under 60 seconds from the sample agents (prd.md §8 metric):
# - the feed scrolls continuously without the page stalling
# - the latency panel shows two clearly distinct numbers
# - gas per charge matches the .gas-snapshot figure
# - cost per charge displays and can be checked against the "under $0.001" claim
```

**Commit.** `feat: add public charge feed with latency and settlement panels`

---

### Task 9.5 — Dashboard build and lint gate

**Objective.** Keep the dashboard from silently rotting.

**Implementation.**
- `npm run lint` clean, `tsc --noEmit` clean, `npm run build` succeeds.
- Add all three to `scripts/check.sh` alongside `cargo test`, `cargo clippy`, `forge test`, `pytest`.

**Verification.**
```
./scripts/check.sh     # runs every check across all four languages, exits 0
```

**Commit.** `chore: add unified check script across rust, solidity, python and typescript`

---

## Phase 9 Gate

`prd.md` F8 complete.

- [ ] All three views load and poll live data
- [ ] Provider registration works from the form, including the SSRF rejection path
- [ ] The delegation tree renders as a tree with visibly draining budget bars
- [ ] A capped worker is visually distinct from its still-running siblings — screenshotted
- [ ] The feed handles 100 charges in under 60 seconds without stalling
- [ ] Gateway and upstream latency are shown as separate, labelled numbers
- [ ] `./scripts/check.sh` exits 0

---

# Phase 10 — Demo, documentation, hardening

**Objective.** Make `prd.md` §9 runnable as a single script, and state the boundaries honestly.

**Depends on:** all previous phases.

---

### Task 10.1 — Sample agents

**Objective.** `architecture.md` §13: one supervisor, three workers, under 150 lines total.

**Implementation.**
- `examples/agents/supervisor.py`: funds three worker wallets (Resolution A), signs three
  capabilities with deliberately different caps and allowlists — worker 2's cap is set so it runs out
  partway through — and prints the expense report at the end.
- `examples/agents/worker.py`: loops calling its assigned endpoints through a `Session` until it is
  capped or done, printing each result and each receipt.
- `examples/agents/run_demo.sh` orchestrating all four with the dashboard open.
- Keep the total under 150 lines. If it grows past that, the client library is doing too little.

**Verification.**
```
./examples/agents/run_demo.sh
# worker 2 stops with a CapExceeded naming its level; workers 1 and 3 finish
# the supervisor prints a signed expense report with three sub-agents and per-endpoint totals
wc -l examples/agents/*.py    # under 150 total
```

**Commit.** `feat: add supervisor and worker sample agents`

---

### Task 10.2 — Load test hitting the `prd.md` §8 targets

**Objective.** Measure the claims rather than assert them.

**Implementation.**
- `scripts/loadtest.py`: 100 paid requests within 60 seconds across parallel workers against the mock
  upstream, printing gateway p50/p99, upstream p50, end-to-end p50, error counts by reason, and
  total settled.
- Assert no overspend: the sum of settled + queued + reserved amounts never exceeds the root cap.
- Record the measured numbers in the README next to the `prd.md` §8 targets, marking any target that
  was missed rather than quietly omitting it (assumption A8: missing a target does not fail the MVP).

**Verification.**
```
python3 scripts/loadtest.py
# 100 requests complete inside 60 seconds
# no overspend assertion fires
# the printed numbers are written into README.md under a "Measured" table beside the targets
```

**Commit.** `test: add load test measuring latency and cap-enforcement targets`

---

### Task 10.3 — README with the stated boundaries

**Objective.** `architecture.md` §12: "A stated boundary reads far better than a discovered one."

**Implementation.**
- Root `README.md`: what Tollgate is (the `prd.md` §1 one-liner and the "why not just x402" answer up
  front, per the §10 risk table), quickstart, architecture diagram, and:
- **What is mocked / limits**, stated plainly:
  - Base Sepolia only, testnet USDC, no real value
  - One chain; multichain is designed, not deployed
  - Provider onboarding is a form; no verification, no reputation
  - The settler key is a hot key in an env var; production wants a KMS or threshold signer. Note that
    this matters less than it looks, because the settler cannot steal (`architecture.md` §6)
  - Delegation ships with Resolution A funding; channel mode is designed, not built
  - Redis loss loses only undrained in-flight reservations, a bounded window of seconds. **Do not
    claim exactly-once across a Redis wipe** (`architecture.md` §9.3)
  - Settlement is **effectively-once**, not exactly-once — explain the composition
    (`architecture.md` §9.4)
  - Settler throughput ceiling ~25 settlements/sec, single EOA
  - `transferWithAuthorization` front-running is bounded: an observer executes exactly the signed
    transfer to the signed recipient and pays the gas. Ordering changes, custody does not
  - The deny-by-default egress policy (assumption A12) is required in deployment and not enforced by
    the application
- A "measured, not estimated" table: gas per charge, cost per charge, latency percentiles.

**Verification.**
```
# a reader who has never seen the repo follows the README from a clean clone and reaches
# a paid response. Every mocked boundary above appears verbatim in the Limits section.
# every command in the README is executed once and exits 0
```

**Commit.** `docs: add readme with quickstart, measured numbers and stated limits`

---

### Task 10.4 — Full demo rehearsal and fix pass

**Objective.** `architecture.md` §14 Phase 4: "Walk through it end to end at least a few times,
including the failure where a worker hits its cap, before calling it done."

**Implementation.**
- Run the complete `prd.md` §9 script three times from a clean database and a clean Redis:
  1. supervisor signs three delegations
  2. workers run, feed scrolls
  3. worker 2 caps out, 1 and 3 continue
  4. settlement transaction appears on Basescan
  5. signed expense report with tx hashes
  6. wrap a brand-new public endpoint live and have a worker pay it
- Fix every defect found. Keep each fix a separate commit if it is substantive; roll trivial ones
  into this task's commit.
- Add `scripts/reset.sh` (drop + migrate + `FLUSHALL`) so a clean rehearsal is one command.

**Verification.**
```
./scripts/reset.sh && ./examples/agents/run_demo.sh    # three consecutive clean runs
# step 6 specifically: register a public API not previously seen, from the dashboard form,
# and have a worker pay it successfully within the same session
# all six steps observable without touching a terminal except to start the agents
```

**Commit.** `chore: add reset script and fix defects found in demo rehearsal`

---

## Phase 10 Gate — MVP acceptance

- [ ] `./scripts/reset.sh && ./examples/agents/run_demo.sh` completes cleanly three times running
- [ ] All six steps of `prd.md` §9 are observable, step 3 (a worker capping while siblings continue)
      most visibly
- [ ] `./scripts/check.sh` exits 0 across Rust, Solidity, Python, and TypeScript
- [ ] README limits section matches `architecture.md` §12 with nothing quietly omitted
- [ ] Measured numbers are in the README, including any target that was missed

---

# Final MVP acceptance checklist

Every P0 requirement from `prd.md` §7, verifiable independently:

- [ ] **F1** — An x402 client nobody wrote for Tollgate parses the 402 and pays it
- [ ] **F2** — A payment header verifies via `alloy` and the request reaches the upstream
- [ ] **F3** — Per-request cap, total cap, and allowlist enforced atomically before the upstream call;
      100 concurrent requests never overspend
- [ ] **F4** — A capability chain is verified at the gateway with no round-trip to the parent;
      attenuation violations are refused
- [ ] **F5** — Queued charges settle in batches on Base Sepolia, off the request path
- [ ] **F6** — Every charge carries a verifiable signed receipt; the expense report rolls up the tree
- [ ] **F7** — Providers register with an SSRF-validated upstream and a pinned-IP proxy connection
- [ ] **F8** — Three dashboard views, with the delegation tree draining live
- [ ] **F9** — The Python client handles 402/sign/retry and `delegate()` locally

Correctness properties that must hold:

- [ ] No response is ever released before the atomic claim returns ok (free-shopping class)
- [ ] The Redis nonce TTL is never shorter than the signature's remaining validity
- [ ] A DNS rebinding payload cannot redirect the proxy after validation
- [ ] Redis loss followed by a restart restores exact spend with no cold-start overspend window
- [ ] One failing settlement item never blocks the other 49
- [ ] A capability presented by anyone other than its subject is refused

---

# Post-MVP / Out of scope

Explicitly excluded from the execution path above. Do not build these while working the plan.

**From `prd.md` P1 (F10–F14):**

- `upto` scheme — authorise a maximum, settle the actual lower amount (assumption A5)
- Payment channel mode / x402 `batch-settlement` — Resolution B in `architecture.md` §5.3
- On-chain refund of an already-settled charge (voiding pre-settlement *is* in scope, per A3)
- Time-window budgets and `tg:window:*` keys (assumption A4)
- `tollgate wrap ./openapi.yaml --price 0.001` CLI

**From `prd.md` P2 (F15–F19):** second chain, MCP server wrapper, provider SLA with automatic
refunds, streaming settlement, dispute window and arbitration.

**From `architecture.md`, deliberately simplified:**

- Merkle-root allowlists (plain sorted-list hash instead, A9)
- Circuit breakers on upstreams, `tg:cb:*` (A10)
- Priority queueing in the Tower stack
- Multiple settler EOAs behind a nonce allocator — the single-EOA ceiling is stated, not fixed
- A gas oracle — fixed EIP-1559 multiplier instead
- Redirect following with per-hop re-validation — redirects are simply not followed
- KMS or threshold signing for the settler key
- SSE or websocket streaming on the dashboard — polling instead
- Deny-by-default egress network policy (A12) — documented as a deployment requirement
- OTel traces; Prometheus histograms only

**Cut order if time runs short** (`architecture.md` §14, in this order): the CLI, the `upto` scheme,
window budgets, the refund path, channel mode, the second chain.

**Never cut:** the atomic Lua claim, SSRF connection pinning, and the delegation tree view.

---

## Commit summary

These are the commits **the human** will make, one per completed and verified task. The agent
proposes messages and never commits.

| Phase | Tasks | Commits |
|-------|-------|---------|
| 0 Foundation | 0.1–0.5 | 5 |
| 1 Data and providers | 1.1–1.4 | 4 |
| 2 x402 surface | 2.1–2.5 | 5 |
| 3 Atomic claim | 3.1–3.4 | 4 |
| 4 Proxy and lifecycle | 4.1–4.4 | 4 |
| 5 Delegation | 5.1–5.6 | 6 |
| 6 Contract and settler | 6.1–6.6 | 6 |
| 7 Receipts | 7.1–7.3 | 3 |
| 8 Python client | 8.1–8.4 | 4 |
| 9 Dashboard | 9.1–9.5 | 5 |
| 10 Demo and docs | 10.1–10.4 | 4 |
| **Total** | **46** | **50** |

Plus any defect-fix commits from Task 10.4, landing the history in the 46–52 range.
