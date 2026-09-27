# Soroban Keeper Bot v2

A high-performance, modular off-chain keeper bot for the [Soroban Keeper Network](https://github.com/soroban-tooling/soroban-keeper-network).

> [!NOTE]
> If you are a newcomer to Soroban, building a first integration, or exploring the smart contract ABI, start with the introductory single-file bot in [`examples/keeper-bot`](../keeper-bot).
>
> **Keeper Bot v2** is intended for operators running competitive keepers with requirements for concurrency, custom withdrawal schedules, lock-window-aware scheduling, and state persistence.

---

## 🚀 Key Architectural Capabilities

Keeper Bot v2 is intended for competitive operators. For an introductory,
single-file example, start with [`examples/keeper-bot`](../keeper-bot/).

### Concurrent rounds, prioritization, and spend safety (Issues #404, #407)
* Bounded concurrent workers process independent tasks without racing internally.
* Candidate tasks are ranked by estimated net profit so high-value work is evaluated first.
* `MAX_ROUND_SPEND_STROOPS` imposes an independent per-round fee ceiling (default: `5000000` stroops).
* Lost claim races are normal competitive skips, recorded separately from RPC and execution failures.

### Verifier support (Issue #412)
Verifier-aware proof generation is deferred until the on-chain contract exposes a
verifier field, entry points, and validation hooks. The bot does not implement
against a non-existent contract interface.

### Benchmarking (Issue #413)
Run the latency/throughput benchmark against v1 with `npm run benchmark`. The
benchmark report is committed at [`benchmark/REPORT.md`](benchmark/REPORT.md).

### Configuration

| Environment Variable | Description | Default |
|---|---|---|
| `SOROBAN_RPC_URL` | URL of the Soroban RPC node | Required |
| `KEEPER_SECRET_KEY` | Stellar secret key for signing transactions | Required |
| `KEEPER_CONTRACT_ID` | Contract address of KeeperRegistry | Required |
| `MAX_ROUND_SPEND_STROOPS` | Hard ceiling on round transaction fees | `5000000` |
| `MAX_CONCURRENCY` | Maximum concurrent task workers | `4` |
| `MIN_PROFIT_MARGIN_STROOPS` | Minimum net profit required before claiming | `0` |
| `POLL_INTERVAL_MS` | Delay between keeper rounds | `5000` |
| `SIMULATE_EXECUTION` | Enable fallback simulated executor for development | `false` |

### 1. Graceful Shutdown Guarantee Under Concurrency (`src/shutdown.js`)
* **Worker Draining**: When a `SIGINT` or `SIGTERM` signal is received, the bot stops accepting new candidate tasks and drains all active in-flight workers.
* **No Mid-Submission Kills**: Each concurrent worker finishes its current submission and persists its outcome before the process exits.
* **Bounded Maximum Wait**: Uses a configurable maximum drain ceiling (`maxDrainMs`) to prevent a deadlocked worker or hung network connection from blocking shutdown indefinitely.

### 2. Pluggable Withdrawal Strategies (`src/withdrawal.js`)
* **Default Fixed-Threshold Strategy**: Matches the v1 threshold behavior exactly (`WITHDRAW_THRESHOLD`, defaulting to 1 XLM), ensuring seamless zero-configuration migrations.
* **Fixed-Schedule Strategy**: Automatically triggers withdrawals on a time or ledger interval (e.g. hourly or daily) for accounting, liquidity, or tax management.
* **Fee-Aware Strategy**: Dynamically adjusts withdrawal floors to trigger payouts opportunistically when Stellar network base fees are low.
* **Custom Strategy Interface**: Fully pluggable interface allows operators to implement custom logic (e.g., epoch-based treasury sweeps).

### 3. Lock-Window-Aware Scheduling (`src/scheduling.js`)
* **Exact Contract Arithmetic**: Computes the exact ledger when a locked task becomes re-claimable (`claim_ledger + lock_ledgers`), matching `contracts/keeper-registry/src/internal.rs`.
* **Targeted Re-checks**: Rather than waiting for a full periodic event scan to rediscover expired locks, the bot schedules targeted re-checks right at the unlock ledger boundary.
* **Additive Discovery**: Re-check candidate tasks are merged additively with standard polling without disrupting the discovery of newly registered tasks.

---

## 🛠️ Testing & Verification

Run the test suite using Node's built-in test runner:

```bash
npm test
```

Run linting and the v1/v2 performance benchmark with `npm run lint` and
`npm run benchmark` respectively.

Run linting:

```bash
npm run lint
```

For design rationale, comparison with the Rust SDK example, and architecture specifications, see [`docs/KEEPER_BOT_V2_DESIGN.md`](../../docs/KEEPER_BOT_V2_DESIGN.md).
