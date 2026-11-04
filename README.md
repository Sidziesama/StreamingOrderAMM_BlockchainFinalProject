# Streaming-Order AMM

**Sell a large amount of WBTC for USDC in equal slices over time instead of all at once, with every slice checked against Chainlink.**

Final project for **FRE-GY 6931 Blockchain & Cryptocurrency** (NYU Tandon, Fall 2026), Topic 1: Automated Market Maker.
Team: **Aditya Shah** (as22008), **Siddhant Yadav** (sy4583), **Dev Salvi** (ds8259). Everything runs on the **Ethereum Sepolia** test network
with mock tokens; no real money is involved.

## Deliverables

| Deliverable | Where |
|---|---|
| Final report (PDF, 8 sections, screenshots, references) | [`docs/report.pdf`](docs/report.pdf) |
| All Solidity source with line numbers (PDF) | [`docs/source-code.pdf`](docs/source-code.pdf) |
| Submission data (team, contracts, transactions, features) | [`submission.json`](submission.json) |
| Financial analysis (Python notebooks with charts) | [`docs/analysis_pool.ipynb`](docs/analysis_pool.ipynb), [`docs/analysis_slicing.ipynb`](docs/analysis_slicing.ipynb), [`docs/analysis_costs.ipynb`](docs/analysis_costs.ipynb) |
| Etherscan and application screenshots | [`docs/screenshots/`](docs/screenshots) |
| Demo video | submitted on Brightspace with the report |

The graded version of this repository is the tag **`final`**.

## The idea in one paragraph

Selling a large order into a constant-product pool moves the price against you with every coin sold. Splitting the order into slices
helps only if something pushes the price back between slices. We built a pool, an order contract that holds each order as an NFT and
sells it in equal slices, a Chainlink price guard, a keeper script that runs the slices, an arbitrage bot that refills the pool, and a web
app, and we measured the result on Sepolia: **a 5 WBTC order earned +66 bps over a lump sum, and our model matched all three on-chain
slices to the last unit.**

## Contracts on Sepolia (all source-verified on Etherscan)

| Contract | What it does | Address |
|---|---|---|
| `Pool` | Constant-product pool, 30 bps fee, TWAP accumulator; **also the ERC-20 LP token `SALP`** | [`0x9461fd23335242663ddf7a9fae994995df7fc8e9`](https://sepolia.etherscan.io/address/0x9461fd23335242663ddf7a9fae994995df7fc8e9#code) |
| `OrderBook` | **ERC-721 `SORDER`**: one NFT per order; equal slices; claim and cancel by the holder | [`0x9c20dbb0bb17a0532348e69502af3b8b44fefa62`](https://sepolia.etherscan.io/address/0x9c20dbb0bb17a0532348e69502af3b8b44fefa62#code) |
| `PriceGuard` | Chainlink BTC/USD checks (staleness, sign, round) and a pool deviation band | [`0xe52d276bec7b96bcdd0632050cad5717956289df`](https://sepolia.etherscan.io/address/0xe52d276bec7b96bcdd0632050cad5717956289df#code) |
| `MockWBTC` | Test WBTC, 8 decimals, capped faucet | [`0x2c95a78b3a3fc73c16bdd6394bfe30cc51d37d12`](https://sepolia.etherscan.io/address/0x2c95a78b3a3fc73c16bdd6394bfe30cc51d37d12#code) |
| `MockUSDC` | Test USDC, 6 decimals, capped faucet | [`0x9cd5587b22d15790a5880d37837cdd091c9ea0b5`](https://sepolia.etherscan.io/address/0x9cd5587b22d15790a5880d37837cdd091c9ea0b5#code) |
| Chainlink BTC/USD (external) | Price feed | [`0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43`](https://sepolia.etherscan.io/address/0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43) |

Key transactions (order creation, keeper slices, arbitrage refills, claims) are listed in [`submission.json`](submission.json) and in
Appendix A of the report.

## Results at a glance

- **Slicing pays only with a refill.** Model: 50 WBTC into a 500 WBTC pool costs 934 bps as a lump sum and 108 bps in 20 slices with a refill; without a refill slicing is slightly *worse* than a lump sum.
- **On Sepolia:** order of 5 WBTC in 3 slices returned 387,501.20 USDC against 384,946.34 for one swap (+66 bps); the model equals the chain for 3 of 3 slices.
- **Testing:** 92 tests in one command, including revert, reentrancy, invariant and end-to-end tests; mutation checks confirm the tests catch bugs.
- **Security:** Slither with no high-severity finding; every finding fixed or explained.
- **Gas:** two measured optimizations (pool storage packing, about -8% on swaps; two-slot orders, -22,054 gas on `createOrder`).

## Repository structure

```
packages/
  hardhat/
    contracts/            Pool, OrderBook, PriceGuard, libraries/PriceMath, interfaces/, mocks/
    deploy/               deployment and seeding scripts (01 to 05, 98 writes deployments.json)
    test/                 Pool, PriceGuard, OrderBook, EndToEnd and gas tests
    scripts/              keeper.ts, arb.ts, offchain/ (shared bot code), demoCheck, evidence and funding tools
    deployments.json      Sepolia addresses, deployment transactions and blocks
    deployments/sepolia/  ABI and compiler metadata per contract
  nextjs/                 web application (Scaffold-ETH 2: Next.js, wagmi, viem, RainbowKit)
docs/
  report.pdf, report.md   final report and its source
  source-code.pdf         Solidity source listing
  analysis_*.ipynb        financial analysis (Parts A, B, C); data/ and figures/ hold their inputs and charts
  screenshots/            Etherscan and application screenshots used in the report
  slither-*.md, gas-*.md  security scan and gas measurements
  deployment.md, sepolia-runbook.md, offchain.md, frontend.md   how to deploy and run everything
  ai-use.md               log of how AI assistance was used
  internal/               the team's working notes and earlier report drafts
submission.json           course submission data
```

## Quick start

Requires **Node 22.10 or later**. Yarn 4 is bundled in the repository: if `yarn` is not installed, run `corepack enable`, or replace `yarn` below
with `node .yarn/releases/yarn-4.13.0.cjs`.

```bash
git clone https://github.com/Sidziesama/StreamingOrderAMM_BlockchainFinalProject.git
cd StreamingOrderAMM_BlockchainFinalProject
yarn install
yarn test                                   # 92 tests, all on the local Hardhat network
```

Run everything locally, each in its own terminal:

```bash
yarn chain                                  # local blockchain
yarn deploy --tags StreamingAMM             # deploy and seed the pool
yarn start                                  # web app at http://localhost:3000
```

Run the app against the live Sepolia contracts: `NEXT_PUBLIC_TARGET_NETWORK=sepolia yarn start`.

Keeper and arbitrage bot (configure `packages/hardhat/.env` from `.env.example` with **testnet-only** keys; never commit `.env`):

```bash
yarn keeper                                 # executes due slices; logs each result with its transaction hash
yarn arb                                    # trades the pool back toward Chainlink when profitable after the fee
yarn demo-check                             # read-only health check of the Sepolia deployment
```

The full Sepolia procedure (credentials, deploy, verify, bots, evidence) is in [`docs/sepolia-runbook.md`](docs/sepolia-runbook.md).

## Known limitations

- A slice can be sandwiched within the guard band and the seller's tolerance (no private mempool).
- The proposed execution fee and protocol fee share are not implemented (future work), so as built the protocol earns no revenue.
- No external audit. Testnet only.

## AI use

AI was used in the making of this project; see Section 8 of the report and [`docs/ai-use.md`](docs/ai-use.md).

## License

MIT. Built on [Scaffold-ETH 2](https://scaffoldeth.io) by BuidlGuidl (see [`LICENCE`](LICENCE)).
