# Monad AgentGuard

> This is the Monad deployment and benchmark of the AgentGuard foundation. It
> is not the ETHOnline submission repository. See
> [ethonline-agentguard](https://github.com/0xCaptain888/ethonline-agentguard)
> for the ETHOnline 2026 Continuity build and its sponsor evidence boundary.

[![CI](https://github.com/0xCaptain888/monad-agentguard/actions/workflows/ci.yml/badge.svg)](https://github.com/0xCaptain888/monad-agentguard/actions/workflows/ci.yml)
[![Demo](https://img.shields.io/badge/demo-live-36d399)](https://0xcaptain888.github.io/monad-agentguard/)
[![Network](https://img.shields.io/badge/Monad-Testnet-7dd3fc)](https://testnet.monadexplorer.com/address/0xee84007f8618c2c38Be8C45E8050144EbF00CE4a)
[![Source](https://img.shields.io/badge/source-Sourcify%20exact%20match-8b5cf6)](https://testnet.monadvision.com/contracts/full_match/10143/0xee84007f8618c2c38Be8C45E8050144EbF00CE4a/)
[![E2E](https://img.shields.io/badge/e2e-25%2F25%20VERIFIED-36d399)](docs/monad-performance.md)
[![Parallel V2](https://img.shields.io/badge/parallel%20V2-10%2F10%20VERIFIED-7dd3fc)](docs/benchmark-parallel-testnet.json)
[![Agent Wallet](https://img.shields.io/badge/MetaMask%20Agent%20Wallet-LIVE%20Task%2056-8b5cf6)](evidence/metamask-agent-wallet-live.json)

**Autonomous agents can act on Monad — but authority stays bounded and every result is independently verifiable.**

**Product thesis:** AgentGuard is the authorization and settlement firewall for
autonomous agent commerce. It lets Agents hire Agents without letting a planner
approve its own spending or grade its own work.

> Hackathon deployment: Monad Testnet chain ID `10143`. No Mainnet or production-security claim is made.

Monad AgentGuard is a Monad-native control and settlement layer for AI agents. A planner can propose a task, but it cannot authorize itself or certify its own output. Agent identity, policy authority, escrow, execution and verification are deliberately separated.

**Live proof:** two different seller Agents now use the same settlement layer.
[`YieldScout`](evidence/testnet-task-30.json) commits a real DeFiLlama liquidity
report, while [`ChainSentinel`](evidence/testnet-task-29.json) commits a fresh
Monad RPC network report. Both full reports are independently re-hashed by
`npm run evidence:verify` and match their on-chain result commitments.

**Live sponsor proof:** MetaMask Agent Wallet BYOK guard wallet
[`0xD71c…D9E`](https://testnet.monadexplorer.com/address/0xD71cf4282466b2197AC69ad027Fd64270a4C2D9E)
registered its identity, committed a `0.01 MON` policy, bound an independent
verifier and created YieldScout Task `56` with `0.001 MON` escrow. The seller
submitted a fresh DeFiLlama result and the independent verifier released it to
`VERIFIED`. Open the [complete receipt](evidence/metamask-agent-wallet-live.json)
or the [final settlement transaction](https://testnet.monadexplorer.com/tx/0x113d9c10506617dd2b408542bc7da242a0ab105faff345a911164dc82386a15a).

### The concrete Agent-to-Agent workflow

`TreasuryPlanner` (buyer) hires either `YieldScout` for a structured Monad
liquidity report or `ChainSentinel` for fresh Monad network telemetry. A
deterministic policy engine checks the seller
permission, per-task budget, daily budget, risk score and confirmation before
any escrow transaction is sent. The seller submits a result hash; an
independent verifier signs the decision; the contract settles only after that
signature is recovered on-chain. Run the inspectable local flow with
`npm run agent:flow`.

```text
Agent goal → identity → policy gate → Monad escrow → execution
           → task-specific verifier
           → VERIFIED / BLOCKED / FROZEN / REFUNDED → Receipt
```

## Why this is new for Metropolis

This repository is a new Monad-native implementation built during Metropolis. It carries forward the design research from [Binance AgentGuard](https://github.com/0xCaptain888/binance-agentguard), but the contracts, Monad deployment, task flow and evidence produced here are specific to this hackathon.

## Contract MVP

[`MonadAgentGuard.sol`](contracts/MonadAgentGuard.sol) is the verified reference
deployment used by the public judge console and the named Task 56 evidence.
[`MonadAgentGuardParallel.sol`](contracts/MonadAgentGuardParallel.sol) is the
deployed V2 execution architecture used to test independent buyer lanes.
Together they provide:

- Agent identity registration with metadata hash;
- per-buyer policy limits and confirmation flag;
- native MON escrow for agent-to-agent tasks;
- seller result submission;
- buyer-side independent verification;
- pre-execution `BLOCKED` refund;
- post-execution `FROZEN` isolation;
- `VERIFIED` release and event trail.

The policy engine is intentionally side-effect free and emits reason codes and
decision hashes before the transaction boundary. See the [architecture](docs/architecture.md)
and [judge quick review](docs/judge-guide.md) for the complete path. The
[scorecard](docs/scorecard.md) maps each judge question to a reproducible
artifact, while [Why Monad](docs/why-monad.md) explains the execution choice.

The AI planner is not trusted by the contract. The contract and verifier are the authority boundaries.

### Judge-operated Testnet task

The public Demo includes an opt-in **Live Judge Console**. A judge can connect
an injected wallet, switch to Monad Testnet and create a real YieldScout or
ChainSentinel escrow task. The browser first evaluates the visible budget,
permission, risk and confirmation checks. If any check fails, it returns
`BLOCKED` without requesting a wallet write. If the checks pass, it registers
the buyer only when needed, commits the displayed on-chain policy and creates
the task. Seller and verifier keys are never shipped to the browser.

### Measured complete settlement loop

A live benchmark covers 25 complete pipelines and 75 Testnet transactions:
`createTask → submitResult → independent signature → VERIFIED`. All 25
completed successfully, with 6,566 ms average end-to-end receipt latency,
6,479 ms P50, 7,543 ms P95 and 320,048 average total gas. See the
[measurement methodology](docs/monad-performance.md) and the
[machine-readable sample](docs/benchmark-e2e-testnet.json). This is a
sequential contract-workflow measurement, not a Monad protocol TPS claim.

### Measured parallel-safe V2

The first concurrent workload exposed a real V1 bottleneck: every buyer wrote
the same global `nextTaskId` slot during `createTask`. We preserved that result
instead of hiding it, then deployed Parallel V2 with deterministic task IDs
derived from `chainId + contract + buyer + per-buyer nonce`.

The V2 evidence contains 10/10 complete `VERIFIED` pipelines across five
independent Buyer/Seller lanes and 30 unique Testnet transactions. Public block
evidence shows up to four creates, five submissions and five verifications in
the same respective block. The interrupted run was safely resumed; only the
four recovered verification writes have a retained wall-clock measurement
(`2,606 ms`). See [the V1 → V2 analysis](docs/monad-performance.md), the
[machine-readable V2 evidence](docs/benchmark-parallel-testnet.json), and the
[deployment receipt](evidence/parallel-v2-deployment.json).

### MetaMask Agent Wallet sponsor integration

The [`integrations/metamask-agent-wallet`](integrations/metamask-agent-wallet)
package maps AgentGuard identity, policy, independent-verifier binding and task
escrow into explicit MetaMask Agent Wallet transaction requests. Task `56`
proves the authenticated BYOK guard path was actually broadcast on Monad
Testnet and settled after independent verification. Run
`npm run sponsor:metamask` to inspect the credential-free transaction commands
and human-readable intents; open
[`evidence/metamask-agent-wallet-live.json`](evidence/metamask-agent-wallet-live.json)
for the six-transaction receipt. The included Agent skill blocks writes when
policy fails and forbids silent Mainnet fallback.

### Verified deployment

- Contract: [`0xee84007f8618c2c38Be8C45E8050144EbF00CE4a`](https://testnet.monadexplorer.com/address/0xee84007f8618c2c38Be8C45E8050144EbF00CE4a)
- Source: [Sourcify exact match on MonadVision](https://testnet.monadvision.com/contracts/full_match/10143/0xee84007f8618c2c38Be8C45E8050144EbF00CE4a/)
- Compiler: Solidity `0.8.26`
- Optimizer: enabled, `200` runs
- Constructor arguments: none

Parallel V2 performance deployment:

- Contract: [`0x91A62595C8eF8c5E5cddcd782cAd7FDdd38D5169`](https://testnet.monadexplorer.com/address/0x91A62595C8eF8c5E5cddcd782cAd7FDdd38D5169)
- Deployment transaction: [`0x262c…3044e`](https://testnet.monadexplorer.com/tx/0x262c2b8f451b5d8ef5452f1181c7d2de5b2ab186729af113259da3a5a9f3044e)
- Source: [Sourcify exact match on MonadVision](https://testnet.monadvision.com/contracts/full_match/10143/0x91A62595C8eF8c5E5cddcd782cAd7FDdd38D5169/)
- Compiler: Solidity `0.8.26`, optimizer `200`
- Purpose: remove cross-buyer contention on the V1 global task counter
- Verification status: `EXACT_MATCH`

The committed `hardhat.config.ts` contains the same compiler and optimizer
settings used for deployment and the official Monad Sourcify endpoint used for
verification.

### A real seller task (DeFiLlama → receipt)

`YieldScout` is more than a label in the demo: it can perform a read-only
research task against DeFiLlama's public API, filter and rank Monad pools, and
return a bounded JSON report. The report includes its source URL and fetch
timestamp, then commits to the exact bytes with a `resultHash`. An independent
verifier checks the schema, source, chain, ranking and numeric bounds before the
hash is eligible for settlement. Run it with:

```bash
npm run yieldscout:report
```

This command is labelled `LIVE_EXTERNAL_DATA` and makes no blockchain write.
For a live task, use the printed `resultHash` as the seller commitment in
`submitResult`; the verifier must validate the report itself, not just trust the
seller's claim. The [task specification](docs/agent-task-spec.md) documents the
input, output and failure semantics.

## Network configuration

| Network | Chain ID | RPC | Explorer |
| --- | ---: | --- | --- |
| Monad Testnet | 10143 | `https://testnet-rpc.monad.xyz` | [Monad Testnet Explorer](https://testnet.monadexplorer.com) |
| Monad Mainnet | 143 | `https://rpc.monad.xyz` | [Monad Explorer](https://monadexplorer.com) |

No private key is committed. Use a dedicated testnet wallet through `DEPLOYER_PRIVATE_KEY` only in your local environment.

## Reproduce locally

```bash
npm install
npm run build
npm test
npm run demo
npm run benchmark
npm run benchmark:testnet # opt-in: 25 live Testnet task creations
npm run benchmark:e2e:testnet # opt-in: 25 complete, 75-transaction pipelines
npm run benchmark:e2e:verify
npm run benchmark:concurrent:verify
npm run benchmark:parallel:verify
npm run sponsor:metamask
npm run sponsor:metamask:rpc-bridge # local bridge for CLI 6.2.0 on chain 10143
npm run evidence:verify
npm run judge:demo
npm run judge:check
```

Deploy to Monad Testnet after funding a dedicated test wallet:

```bash
export DEPLOYER_PRIVATE_KEY=0x...
npm run deploy:testnet
```

The deploy command writes a local deployment record under `deployments/` (ignored by git). After deployment, configure a second seller wallet locally as `SELLER_PRIVATE_KEY` and run:

```bash
npm run task:testnet
```

The runner registers buyer and seller identities, creates a `0.01 MON` escrow task, submits a result, verifies it, reads the final task state back from Monad, and writes `evidence/testnet-task-<id>.json`. The evidence file includes the intent/policy/result hashes, receipt status, block number, Explorer links, and a deterministic evidence hash. It never prints or persists private keys.

To make that live task use the external seller output, opt in explicitly:

```bash
YIELDSCOUT_LIVE_DATA=1 npm run task:testnet
```

This fetches DeFiLlama, independently validates the structured report, uses
its `resultHash` in `submitResult`, and records the source, pool IDs and
verification checks in the receipt. Without the flag, no external request is
made and the deterministic demo result is used.

For a reproducible run when DeFiLlama is temporarily unreachable, first save a
fresh report with `npm run yieldscout:report`, then use
`YIELDSCOUT_LIVE_DATA=1 YIELDSCOUT_REPORT_FILE=evidence/yieldscout-latest.json npm run task:testnet`.
The same independent checks run before submission; the file is only a cache of
the real external response and is not accepted if its hash or schema is invalid.

The contract also supports a real independent verifier signature through `setVerifier` and `verifyTaskBySignature`. The seller (or another verifier wallet) signs the task decision off-chain; the contract recovers that signer before releasing or freezing escrow. `npm run benchmark` provides a reproducible local throughput baseline; it is deliberately labelled simulation until run against Monad Testnet. There is no claim of production security or Monad throughput from this local benchmark.

For a measured Testnet sample, run `BENCHMARK_TASKS=25 npm run benchmark:testnet`.
This records live `createTask` latency and gas in
[`docs/benchmark-testnet.json`](docs/benchmark-testnet.json); it is separate
from settlement evidence and does not imply end-to-end agent throughput.

For a first live run, use two dedicated Monad Testnet wallets:

| Role | Address | Minimum recommended balance |
| --- | --- | ---: |
| Buyer / deployer | `0xd64Fac11d711d7278a8Bb6D7be1E2De1fdBCC564` | `1 MON` |
| Seller agent | `0x637a61f2644E43aDa1eEeEb6Ff827B2aD60e669b` | `0.1 MON` |
| Independent verifier | `0xE01337d3F0E061017d8Ce547e11d86C0705e8526` | `0.1 MON` |

Get testnet MON from the [Monad faucet](https://faucet.monad.xyz). Keep `.env` local and never commit it.

## Trust boundaries

| Component | Propose | Execute | Authorize | Verify |
| --- | ---: | ---: | ---: | ---: |
| AI planner | ✓ | — | — | — |
| Policy engine | — | — | ✓ | — |
| Monad contract | — | ✓ | ✓ | — |
| Independent verifier | — | — | — | ✓ |

## Status

This is an active Metropolis build. A live Monad Testnet contract and end-to-end `VERIFIED`, `BLOCKED`, and `FROZEN` tasks are now available in [`docs/live-testnet-evidence.md`](docs/live-testnet-evidence.md). The dated build plan and submission checklist are in [`docs/metropolis-plan.md`](docs/metropolis-plan.md).

For adoption testing, the [external pilot](docs/external-pilot.md) lets a real
third-party wallet create a bounded Testnet task without receiving seller or
verifier credentials. Pilot counts remain zero until independently submitted
wallet evidence exists; no synthetic user claim is made.

## Judge demo

The demo is in [`site/index.html`](site/index.html) and is published by GitHub
Pages. It displays the live contract, independent verifier, three task states,
receipt links, browser-side RPC/evidence checks and an opt-in Live Judge Console.
The page is read-only until the judge explicitly clicks Connect Wallet; policy
failure produces `BLOCKED` before any write request.

## Evidence labels and limitations

- **LIVE_TESTNET**: the recovery-enabled deployment, real YieldScout and ChainSentinel receipts, the authenticated MetaMask Agent Wallet Task 56, failure paths and recovery receipts in [live evidence](docs/live-testnet-evidence.md).
- **LIVE_TESTNET_BENCHMARK**: V1 sequential/concurrent measurements and the
  Parallel V2 five-lane workload. These are application-level contract traces,
  never Monad protocol TPS claims.
- **SIMULATION**: `npm run agent:flow` and `npm run benchmark`; these move no funds.
- **DESIGN**: signed approvals, deadlines, neutral-arbiter quorum and production monitoring are follow-on work; the basic two-party frozen recovery is live on Testnet.

`npm run evidence:verify` is a read-only integrity check for the committed
receipts. It recomputes each evidence hash and validates the Monad Testnet
transaction trail.

The recommended three-minute recording sequence is documented in
[`docs/demo-script.md`](docs/demo-script.md); `npm run judge:demo` prints the
policy-first Agent-to-Agent story and the three LIVE_TESTNET links.

For a machine-readable entry point, open
[`evidence/judge-manifest.json`](evidence/judge-manifest.json). The
[Agent catalog](docs/agent-catalog.md) explains how the same protocol handles
two different real seller workloads.

`FROZEN` initially isolates escrow after a failed result. The deployed MVP now
supports a two-party buyer/seller recovery barrier; signed approvals, deadlines
and neutral arbitration remain production extensions.

## License

MIT
