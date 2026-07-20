# X-Ray Report

> Midnight | 1755 nSLOC | 7538c438 (`HEAD`) | Foundry | 30/05/26

---

## 1. Protocol Overview

**What it does:** Midnight is a non-custodial fixed-rate lending protocol where immutable markets settle signed credit/debt offers against loan and collateral tokens.

- **Users**: Makers sign offers, takers consume them, borrowers supply/withdraw collateral, liquidators repay unhealthy or matured debt.
- **Core flow**: A market is touched/created, a maker-authorized ratifier validates an offer, `take` updates buyer/seller credit/debt, then callbacks and token transfers settle assets. ---
- **Key mechanism**: Fixed-maturity lending via zero-coupon-style units, lazy bad-debt socialization, piecewise settlement fees, and per-market optional gates.
- **Token model**: External ERC20 loan and collateral tokens; no protocol token or share token is minted.
- **Admin model**: Four mutable roles: `roleSetter`, `feeSetter`, `feeClaimer`, and `tickSpacingSetter`; no upgrade proxy found.

 see the [architecture diagram](architecture.svg).

### Contracts in Scope


| Subsystem | Key Contracts | nSLOC | Role |
|-----------|--------------|------:|------|
| Core | `Midnight` | 648 | Market creation, offers, positions, liquidation, flash loans, fees |
| Core interfaces | `IMidnight`, `ICallbacks`, `IOracle`, `IRatifier`, `IGate`, `IERC20` | 200 | External contracts and storage/API shape |
| Libraries | `ConstantsLib`, `EventsLib`, `IdLib`, `SafeTransferLib`, `TickLib`, `UtilsLib` | 225 | Math, market IDs, event schema, token calls |
| Periphery | `MidnightBundles`, `EcrecoverAuthorizer`, `TakeAmountsLib`, `ConsumableUnitsLib` | 385 | Batch flows and signed authorization helper |
| Periphery interfaces | `IMidnightBundles`, `IEcrecoverAuthorizer`, `IERC20Permit`, `IPermit2` | 97 | Permit and bundle API surface |
| Ratifiers | `EcrecoverRatifier`, `SetterRatifier`, `HashLib` | 167 | Offer Merkle/signature validation and root control |
| Ratifier interfaces | `IEcrecoverRatifier`, `ISetterRatifier` | 33 | Ratifier API and errors |

### How It Fits Together

The core trick: Midnight separates quoting from settlement, so signed offers do not escrow funds until `take` atomically updates positions and moves tokens.

### Market Creation

```text
Midnight.touchMarket()
├─ validates maturity, collateral list, LLTV, maxLif
├─ initializes fee/tick state from defaults
└─ IdLib.storeInCode()
   *market params become reconstructible from SSTORE2-style runtime code*
```

### Offer Settlement

```text
Midnight.take()
├─ touchMarket()
├─ IRatifier.isRatified()
├─ update buyer/seller credit, debt, pending fee, consumed
├─ buyer/seller callbacks
├─ ERC20 loan-token transfers
└─ isHealthy(seller)
   *seller liquidation is transiently locked across callbacks*
```

### Collateral & Repayment

```text
supplyCollateral() / withdrawCollateral()
├─ update collateral array and bitmap
└─ withdraw path checks isHealthy()

repay() / withdraw()
├─ debt down, withdrawable up
└─ credit/withdrawable/totalUnits down on withdraw
```

### Liquidation

```text
liquidate()
├─ oracle loop over activated collateral bitmap
├─ bad debt updates lossFactor and totalUnits
├─ repay/seize calculation
├─ optional callback
└─ collateral out, loan token in
```

---

## 2. Threat & Trust Model

### Protocol Threat Profile

> Protocol classified as: **Lending/Borrowing** with **orderbook-style off-chain offer** and **periphery batching** characteristics

Code signals are concentrated in borrow/repay/liquidate/health checks, oracle-priced collateral, and fixed-maturity debt units; offer signing/ratification adds signature and Merkle-root surfaces.

### Actors & Adversary Model

| Actor | Trust Level | Capabilities |
|-------|-------------|--------------|
| `roleSetter` | Trusted | Instant reassignment of all four roles; no timelock or pause layer in scope. |
| `feeSetter` | Trusted | Instant default and per-market settlement/continuous fee changes within caps. |
| `feeClaimer` | Trusted | Can claim settlement fees and continuous-fee credit to arbitrary receivers. |
| `tickSpacingSetter` | Bounded | Can only decrease/refine created-market tick spacing to a divisor. |
| Makers/takers | Untrusted | Sign/cancel/consume offers, delegate authorization, supply/withdraw/repay through self or authorized accounts. |
| Liquidators | Untrusted or gate-bounded | Can liquidate borrower debt if market gate allows and position is unhealthy or matured. |
| Ratifiers | Maker-authorized | Decide whether an offer is ratified; in-scope ratifiers are bound to a Midnight instance. |
| Oracles/gates/tokens/callbacks | External trust boundary | Price, access, transfer, and hook behavior are interface assumptions. |

**Adversary Ranking**

1. **Oracle manipulator** — Oracle prices directly drive health, bad-debt, and liquidation math.
2. **Liquidation MEV searcher** — Liquidation profitability and post-maturity modes are central to bad-debt realization.
3. **Signature/root manipulator** — Offer validity depends on authorization mappings, Merkle proofs, domain separators, and ratifier roots.
4. **Malicious token/callback integrator** — Token and callback behavior is assumed by all fund-moving entry points.
5. **Compromised role holder** — Roles can instantly change fees, claim fees, or alter tick accessibility.

See [entry-points.md](entry-points.md) for the full permissionless entry point map.

### Trust Boundaries

- **Role boundary** — `roleSetter` can instantly rotate all other roles at `src/Midnight.sol:224-245`; no operational delay is present.

- **Fee boundary** &nbsp;[[I-4](invariants.md#i-4), [I-5](invariants.md#i-5)] — `feeSetter` changes default and market fee parameters instantly, bounded by cap guards at `src/Midnight.sol:258-301`.

- **Oracle boundary** &nbsp;[[X-2](invariants.md#x-2)] — `liquidate` and `isHealthy` trust `IOracle.price()` at `src/Midnight.sol:610` and `src/Midnight.sol:953`.

- **Token boundary** &nbsp;[[X-4](invariants.md#x-4)] — `SafeTransferLib` verifies code and optional boolean returns but not exact balance deltas at `src/libraries/SafeTransferLib.sol:12-33`.

- **Ratifier boundary** &nbsp;[[X-1](invariants.md#x-1)] — `take` delegates offer validity to maker-authorized ratifiers at `src/Midnight.sol:355-356`.

### Key Attack Surfaces

- **Core offer settlement accounting** &nbsp;[[G-1](invariants.md#g-1), [G-9](invariants.md#g-9), [G-10](invariants.md#g-10), [I-8](invariants.md#i-8)] — `take` mutates consumed, credit, debt, pending fees, total units, and claimable fees in one path at `src/Midnight.sol:346-418`; worth tracing every signed-offer edge and rounding direction.

- **Liquidation and bad-debt realization** &nbsp;[[G-20](invariants.md#g-20), [G-23](invariants.md#g-23), [G-24](invariants.md#g-24), [X-2](invariants.md#x-2)] — `liquidate` combines oracle loops, loss-factor math, RCF checks, collateral seizure, and callbacks at `src/Midnight.sol:595-717`.

- **Oracle-priced health checks** &nbsp;[[G-19](invariants.md#g-19), [X-2](invariants.md#x-2)] — `withdrawCollateral` and `take` depend on `isHealthy`, whose collateral loop reads arbitrary oracle implementations at `src/Midnight.sol:944-959`.

- **Token and callback ordering** &nbsp;[[X-4](invariants.md#x-4), [I-12](invariants.md#i-12)] — state is updated before callbacks and transfers in `take`, `repay`, `liquidate`, and `flashLoan`; worth checking token/callback assumptions against each flow.

- **Signature and Merkle-root authorization** &nbsp;[[X-1](invariants.md#x-1), [I-14](invariants.md#i-14)] — ratifiers and authorizer bind domains differently from market IDs across `src/ratifiers/EcrecoverRatifier.sol:33-45` and `src/periphery/EcrecoverAuthorizer.sol:24-47`.

- **Bundler skip-and-fill logic** &nbsp;[[I-13](invariants.md#i-13)] — `MidnightBundles` catches failed takes and continues through target loops at `src/periphery/MidnightBundles.sol:71-88`, `144-166`, `205-225`, and `282-304`.

- **Role operations without timelock** &nbsp;[[I-4](invariants.md#i-4), [I-5](invariants.md#i-5)] — fee, claimer, tick-spacing, and role setters are immediate at `src/Midnight.sol:224-324`.

### Protocol-Type Concerns

**As a Lending/Borrowing protocol:**
- `isHealthy` rounds max debt down at `src/Midnight.sol:953-955`; this is central to withdrawal, post-take, and liquidation eligibility.
- Loss factor updates round lenders collectively against themselves at `src/Midnight.sol:631-640`, matching the documented bad-debt socialization model.
- `MAX_COLLATERALS_PER_BORROWER` bounds active collateral but full market collateral arrays still define creation and hash surfaces at `src/Midnight.sol:759-770`.

### Temporal Risk Profile

**Market lifetime:**
- Market maturity is capped at 100 years on creation at `src/Midnight.sol:758`, while offer validity and post-maturity debt rules are checked at settlement time.

**Hard fork:**
- `INITIAL_CHAIN_ID` preserves market IDs at `src/Midnight.sol:871-872`, while signature helpers use current `block.chainid` at `src/periphery/EcrecoverAuthorizer.sol:29` and `src/ratifiers/EcrecoverRatifier.sol:40`.

### Composability & Dependency Risks

**Dependency Risk Map:**

> **ERC20 tokens** — via most value-moving `Midnight` and `MidnightBundles` functions
> - Assumes: exact transfer amounts, no transfer reentrancy, no no-op reverts
> - Validates: code exists and return data is empty or true
> - Mutability: external token-dependent
> - On failure: revert

> **Oracles** — via `isHealthy()` and `liquidate()`
> - Assumes: bounded, scaled, current prices
> - Validates: NONE beyond call success
> - Mutability: market-param dependent
> - On failure: revert in health/liquidation paths

> **Gates** — via `take()` and `liquidate()`
> - Assumes: gate policy implements intended market access
> - Validates: exact boolean true when required
> - Mutability: immutable per market params
> - On failure: revert

> **Callbacks** — via `take()`, `repay()`, `liquidate()`, `flashLoan()`
> - Assumes: callback returns `CALLBACK_SUCCESS` and respects token repayment expectations
> - Validates: exact return value
> - Mutability: caller/offer-provided
> - On failure: revert

**Token Assumptions**:
- Fee-on-transfer/rebasing/ERC777-like tokens are not validated by balance-delta checks; code comments require exact, non-reentrant ERC20 behavior.

---

## 3. Invariants

> ### Full invariant map: **[invariants.md](invariants.md)**
>
> A dedicated reference file contains the complete invariant analysis.
>
> - **32 Enforced Guards** (`G-1` … `G-32`) — per-call preconditions with `Check` / `Location` / `Purpose`
> - **14 Single-Contract Invariants** (`I-1` … `I-14`) — Conservation, Bound, Ratio, StateMachine, Temporal
> - **4 Cross-Contract Invariants** (`X-1` … `X-4`) — caller/callee pairs that cross scope boundaries
> - **2 Economic Invariants** (`E-1` … `E-2`) — higher-order properties deriving from `I-N` + `X-N`
>
> Every inferred block cites a concrete delta-pair, guard-lift, state edge, temporal predicate, or NatSpec claim.

---

## 4. Documentation Quality

| Aspect | Status | Notes |
|--------|--------|-------|
| README | Present | `README.md` gives concise protocol and developer overview. |
| NatSpec | 26 annotations | Core `Midnight.sol` has unusually dense design/liveness/rounding comments. |
| Spec/Whitepaper | Present | `protocol.md`, `README.md`, and `certora/README.md`; README links external whitepaper. |
| Inline Comments | Thorough | Many edge cases and accepted assumptions are documented directly in code. |

Spec-derived claims used above are tagged by source where they come from docs; most protocol mechanics are code-verified from `src/`.

---

## 5. Test Analysis

| Metric | Value | Source |
|--------|-------|--------|
| Test files | 38 | File scan |
| Test functions | 373 | File scan |
| Line coverage | Unavailable — `forge coverage` and `forge coverage --ir-minimum` both hit stack-too-deep compiler failures | Coverage tool |
| Branch coverage | Unavailable — same compiler failure | Coverage tool |

### Test Depth

| Category | Count | Contracts Covered |
|----------|------:|-------------------|
| Unit | 373 | Broad Foundry suite across Midnight, bundles, ratifiers, math, transfers |
| Stateless Fuzz | 1 | Detected by scan |
| Stateful Fuzz (Foundry) | 0 | none |
| Formal Verification (Certora) | 32 specs / 31 configs | Core accounting, health, liquidation, fees, authorization, transfers, bitmap, math |

### Gaps

- No Foundry invariant tests detected; Certora covers many invariants but runtime/stateful fuzz would exercise integration sequences differently.
- No Echidna, Medusa, Halmos, HEVM, or fork tests detected.
- Coverage metrics are unavailable due compiler stack-depth errors, not because tests are absent.

---

## 6. Developer & Git History

> Repo shape: normal_dev — 933 source-touching commits over 493 days on analyzed branch `HEAD` at `7538c438`.

### Contributors

| Author | Commits/Signal | Source Lines (+/-) | % of Source Changes |
|--------|---------------:|--------------------|--------------------:|
| MathisGD | top contributor | +7113 / n/a | 58.8% |
| Adrien Husson | contributor | +1531 / n/a | 12.7% |
| Quentin Garchery | contributor | +868 / n/a | 7.2% |
| prd-carapulse[bot] | contributor | +840 / n/a | 6.9% |
| peyha | contributor | +554 / n/a | 4.6% |
| Colin Gonzalez | contributor | +547 / n/a | 4.5% |
| PA | contributor | +452 / n/a | 3.7% |

### Review & Process Signals

| Signal | Value | Assessment |
|--------|-------|------------|
| Unique contributors | 15 | Larger team |
| Merge commits | 673 of 2492 | Strong PR/merge-process signal |
| Repo age | 2025-01-21 to 2026-05-29 | Active 493-day development history |
| Recent source activity (30d) | 102 late source commits | Late burst before snapshot |
| Test co-change rate | 45.7% | Measures source/test co-modification, not coverage |

### File Hotspots

| File | Modifications | Note |
|------|-------------:|------|
| `src/Midnight.sol` | 347 | Highest-churn core accounting and liquidation file |
| `src/libraries/EventsLib.sol` | 103 | Event/schema `churn` |
| `src/interfaces/IMidnight.sol` | 76 | API/storage shape churn |
| `src/libraries/UtilsLib.sol` | 72 | Low-level math/bitmap/transient storage helpers |
| `src/libraries/ConstantsLib.sol` | 60 | Fee, LLTV, LIF, callback constants |
| `src/periphery/MidnightBundles.sol` | 24 | Batch execution and permit surface |

### Security-Relevant Commits

| SHA | Date | Subject | Score | Key Signal |
|-----|------|---------|------:|------------|
| 0b8e2305 | 2026-03-04 | fix: consume attack | 17 | Explicit security language and runtime guard in `Midnight.sol` |
| 9e22a467 | 2026-05-05 | document price overflow | 15 | Oracle/pricing documentation in `Midnight.sol` |
| e00347a0 | 2026-04-30 | Certora liveness properties | 15 | Runtime guards and accounting specs |
| ccc7eda4 | 2026-04-15 | Custom error with commit fix | 15 | Broad guard rewrite across many source files |
| 6a28ed4a | 2026-04-04 | block liquidate during deferred callback health check | 15 | Liquidation/callback guard change |
| 122edd86 | 2026-05-29 | prevent debt increase after maturity more strictly | 14 | Late guard change touching maturity/debt |
| 5b3e7330 | 2026-05-19 | bind bundles to midnight at construction | 14 | Periphery signature/auth binding |
| fae73d2c | 2026-05-12 | permit/permit2 | 14 | Token transfer and signature handling |

### Dangerous Area Evolution

| Security Area | Commits | Key Files |
|--------------|--------:|-----------|
| liquidation | 462 | `src/Midnight.sol`, `src/interfaces/IMidnight.sol`, `src/libraries/ConstantsLib.sol` |
| fund_flows | 457 | `src/Midnight.sol`, `src/libraries/SafeTransferLib.sol`, `src/periphery/MidnightBundles.sol` |
| oracle_price | 410 | `src/Midnight.sol`, `src/libraries/TickLib.sol`, `src/periphery/TakeAmountsLib.sol` |
| access_control | 365 | `src/Midnight.sol`, `src/ratifiers/EcrecoverRatifier.sol`, `src/ratifiers/SetterRatifier.sol` |
| signatures | 44 | periphery and ratifier files |

### Forked Dependencies

No internalized forked dependency libraries were detected by the git analysis.

### Security Observations

- **Core concentration** — `src/Midnight.sol` is both the largest file and top hotspot at 347 modifications.
- **Late churn** — 102 source commits occurred in the late-change window, including liquidation, debt, and bundle changes.
- **Test co-change** — 45.7% of source-changing commits also touched tests; this is process signal, not coverage.
- **No TODO debt** — git analysis found no TODO/FIXME/HACK/XXX markers.
- **Formal depth** — 32 Certora specs and 31 configs provide unusually broad formal coverage.

### Cross-Reference Synthesis

- **`Midnight.sol` hotspot aligns with top surfaces** — offer settlement, liquidation, health checks, and fee roles all route through the same 648 nSLOC core file.
- **Late liquidation commits align with X-2/G-23/G-24** — recent changes cluster around oracle-priced liquidation and post-maturity behavior.
- **Bundle churn aligns with I-13** — periphery target/permit logic changed late and has several indexing/skip-and-fill preconditions.

---

## X-Ray Verdict

**HARDENED** — Strong docs, extensive unit tests, and broad Certora coverage are present, but operational roles are instant and coverage metrics could not be produced locally.

**Structural facts:**
1. 1755 in-scope nSLOC across core, libraries, periphery, and ratifiers.
2. 38 Foundry test files with 373 test functions and 32 Certora specs / 31 configs were detected.
3. No upgradeable contracts or proxy patterns were found in scope.
4. The current branch has 2492 commits, 673 merge commits, and 933 source-touching commits.
