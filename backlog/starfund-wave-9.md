# StarFund Contracts — Backlog Wave 9 (Published Archive)

## Backlog Status & Audit History

- **Repository:** `ushpraise/Starfund-contracts` (`https://github.com/ushpraise/Starfund-contracts`)
- **Status:** **Published to GitHub (All 110 Wave 9 Issues Live)**
- **Active / Unpublished Backlog Remaining:** **0** (all issues published)
- **Publication Waves:**
  - **Wave 9 Batch 1 (#1–#50):** Published as GitHub issues **#20 through #69**, archived in [`drips wave 9.md`](../drips%20wave%209.md).
  - **Wave 9 Batch 2 (#51–#110):** Published as GitHub issues **#70 through #129**, detailed below with full technical descriptions and GitHub issue links.
- **Traceability Guarantee:** Every issue below preserves its original Wave backlog number (#51–#110), its assigned GitHub issue number (#70–#129), complete technical context, reproduction evidence, and acceptance criteria.

---

# Published Wave 9 Issues (#51 through #110)

## Issue #51 — State Corruption: `sweep_terminal_dust` Unconditionally Sets `status = 2` (Settled) on Cancelled Escrows

**Filed as:** [GitHub issue #70](https://github.com/ushpraise/Starfund-contracts/issues/70)

**Category:** Bug / Security  
**Priority:** Critical  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::sweep_terminal_dust()` (lines 3485–3540, specifically line 3536)

**Problem**  
In `StarfundEscrow::sweep_terminal_dust()`, the function allows sweeping dust when `is_terminal_status(escrow.status)` is true (which includes `2: Settled`, `3: Withdrawn`, and `4: Cancelled`). At lines 3513–3527, it explicitly handles the cancelled state (`escrow.status == 4`) to compute the outstanding liability floor for unrefunded investors. However, immediately after executing the token transfer at line 3536, it unconditionally executes:
```rust
escrow.status = 2;
env.storage().instance().set(&DataKey::Escrow, &escrow);
```
If `sweep_terminal_dust` is called on a Cancelled escrow (`status == 4`), it mutates the contract status to Settled (`status == 2`).

**Why it matters**  
This state corruption has catastrophic consequences for escrow accounting and investor protection:
1. `StarfundEscrow::refund()` requires `escrow.status == 4` (`EscrowError::RefundNotCancelled`). Mutating a cancelled escrow to `status = 2` permanently locks remaining investors out from claiming their principal refunds.
2. Conversely, `StarfundEscrow::claim_investor_payout()` requires `escrow.status == 2` (`EscrowError::InvestorClaimNotSettled`). A cancelled escrow mutated to `status = 2` now allows investors to call `claim_investor_payout()`, which computes coupon and settlement pool payouts against nonexistent borrower repayments!
3. Furthermore, `sweep_terminal_dust` emits no event upon execution, leaving no on-chain trace of the sweep or status change.

**Proposed solution**  
1. Preserve the existing status rather than overwriting it with `2`. The status should remain unchanged (if status was 4, it remains 4; if 3, it remains 3; if 2, it remains 2).
2. Remove `escrow.status = 2;` or only update status if transitioning from a non-terminal state.
3. Emit a `TreasuryDustSwept` event with `invoice_id`, `treasury`, `amount`, and `remaining_balance`.

**Acceptance criteria**  
- Calling `sweep_terminal_dust` on an escrow with `status == 4` leaves `escrow.status` equal to `4`.
- Remaining investors can successfully call `refund()` after a partial dust sweep on a cancelled escrow.
- An event is emitted when dust is swept.

**Testing**  
- Add a unit test in `escrow/src/tests/` executing `sweep_terminal_dust` on a cancelled escrow and asserting `client.get_escrow().status == 4`.
- Assert that subsequent calls to `client.refund(&investor)` succeed.

---

## Issue #52 — `settle()` Never Writes `DataKey::SettledAt`, Breaking `get_settled_at()` and Settlement Audits

**Filed as:** [GitHub issue #71](https://github.com/ushpraise/Starfund-contracts/issues/71)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::settle()` (lines 6790–6845)
- `StarfundEscrow::get_settled_at()` (lines 4370–4372)

**Problem**  
The documentation for `settle()` at line 6790 states that `settle()` records the settled marker atomically:
`// writer of the SettledAt marker, so status == 2 uniquely identifies an escrow that has already been settled.`
Likewise, doc comments for `get_settled_at` at line 4361 explain:
`/// Returns the ledger timestamp (seconds since Unix epoch) at which StarfundEscrow::settle transitioned status from 1 -> 2, or None if the escrow has not yet been settled.`
However, in the actual implementation of `StarfundEscrow::settle()` (lines 6825–6830):
```rust
escrow.status = 2;
env.storage().instance().set(&DataKey::Escrow, &escrow);
extend_ttl_for_activity(&env, &escrow, None);
```
`DataKey::SettledAt` is **never written to storage**!

**Why it matters**  
Because `DataKey::SettledAt` is never set during settlement, `StarfundEscrow::get_settled_at()` (`env.storage().instance().get(&DataKey::SettledAt)`) always returns `None`, even after an escrow has been successfully settled. Off-chain indexers, UI dashboards, and smart contracts querying `get_settled_at()` cannot retrieve the settlement timestamp.

**Proposed solution**  
In `StarfundEscrow::settle()`, write the current ledger timestamp to `DataKey::SettledAt`:
```rust
let now = env.ledger().timestamp();
...
env.storage().instance().set(&DataKey::SettledAt, &now);
```

**Acceptance criteria**  
- `StarfundEscrow::settle()` writes `DataKey::SettledAt` atomically when transitioning status to 2.
- `StarfundEscrow::get_settled_at()` returns `Some(timestamp)` after `settle()` is called.

**Testing**  
- Add test asserting `client.get_settled_at() == Some(now)` immediately following `client.settle()`.

---

## Issue #53 — `settle()` Permits Transition to Settled Without Verifying Contract Token Balance Covers `settle_pool`

**Filed as:** [GitHub issue #72](https://github.com/ushpraise/Starfund-contracts/issues/72)

**Category:** Bug / Security  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::settle()` (lines 6810–6840)

**Problem**  
In `StarfundEscrow::settle()`, the contract checks that status is `1` (Funded) and that `maturity` has elapsed (`now >= escrow.maturity`). It then calculates `coupon` and `settle_pool = funded_amount + coupon`. It immediately sets `escrow.status = 2` (Settled) and publishes `EscrowSettled`.
However, `settle()` never checks whether the contract actually holds a token balance sufficient to cover `settle_pool` (`TokenClient::balance(&this) >= settle_pool`).

**Why it matters**  
If the SME or borrower has not yet deposited the repayment funds (principal + coupon) into the escrow contract, calling `settle()` will still succeed and permanently mark the escrow as Settled (`status = 2`).
Once settled:
1. The contract can never be cancelled via `cancel_funding` (which requires status 0).
2. The SME cannot withdraw.
3. When investors attempt to claim their payouts via `claim_investor_payout()`, `transfer_funding_token_with_balance_checks` will revert with `InsufficientTokenBalanceBeforeTransfer`.
4. The contract is permanently bricked in a pseudo-settled state with insolvent token balances.

**Proposed solution**  
In `StarfundEscrow::settle()`, query the contract's funding token balance before transitioning status:
```rust
let token_addr = Self::funding_token_or_fail(&env);
let balance = TokenClient::new(&env, &token_addr).balance(&env.current_contract_address());
ensure(&env, balance >= settle_pool, EscrowError::InsufficientContractBalance);
```

**Acceptance criteria**  
- `StarfundEscrow::settle()` reverts with `EscrowError::InsufficientContractBalance` if contract token balance is less than `settle_pool`.
- `settle()` succeeds when repayment has been deposited.

**Testing**  
- Add negative test asserting `client.try_settle()` reverts with `InsufficientContractBalance` when repayment funds are missing.
- Add test verifying settlement succeeds when tokens are properly deposited.

---

## Issue #54 — Duplicate Storage Writes and Redundant Checks in `StarfundEscrow::init`

**Filed as:** [GitHub issue #73](https://github.com/ushpraise/Starfund-contracts/issues/73)

**Category:** Refactor / Gas  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::init()` (lines 3131–3296)

**Problem**  
In `StarfundEscrow::init()`, initialization parameters are written to storage in lines 3131–3223, and then many of the exact same keys are written a second time in lines 3245–3296:
1. `DataKey::Version` is written at line 3151 and again at line 3248.
2. `DataKey::FundingToken` is written at line 3139 and again at line 3251.
3. `DataKey::Treasury` is written at line 3140 and again at line 3252.
4. `DataKey::MinContributionFloor` is written at line 3174 and again at line 3257.
5. `DataKey::UniqueFunderCount` is written at line 3181 and again at line 3261.
6. `DataKey::RegistryRef` is written at line 3154 and again at line 3266.
7. `DataKey::MaxPerInvestorCap` is validated and written at line 3187 and validated and written again at lines 3270–3274.
8. `DataKey::MaxUniqueInvestorsCap` is validated and written at line 3194 and validated and written again at lines 3277–3281.
9. `DataKey::LegalHoldClearDelay` is written at line 3201 and again at line 3287.
10. `DataKey::YieldTierTable` is written at line 3160 and again at line 3294.
11. `DataKey::FundingDeadline` is written at line 3134 *without validation*, and then validated and written again at line 3222.

**Why it matters**  
This duplicate logic wastes CPU instructions and storage write gas on every escrow deployment on Soroban. Even worse, the first write of `funding_deadline` at line 3134 occurs before any timestamp validation (`deadline > now` and `deadline < maturity`), which creates risk if an error aborts execution after partial writes.

**Proposed solution**  
Consolidate `init()` so each configuration key is validated once and written to instance storage exactly once.

**Acceptance criteria**  
- Each storage key in `init()` is written exactly once.
- Validation checks are performed before any storage writes occur.
- Total CPU instructions and ledger write operations in `init()` are reduced.

**Testing**  
- Run `init` unit tests in `escrow/src/tests/init.rs` to verify configuration values match expected keys.

---

## Issue #55 — `init()` References Non-Existent Error `EscrowError::FundingDeadlineBeyondMaturity`

**Filed as:** [GitHub issue #74](https://github.com/ushpraise/Starfund-contracts/issues/74)

**Category:** Bug / Compile-Blocker  
**Priority:** Critical  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::init()` (line 3217)
- `EscrowError` enum (lines 870)

**Problem**  
In `StarfundEscrow::init()` at line 3217, the funding deadline maturity check reads:
```rust
if maturity > 0 {
    ensure(
        &env,
        deadline < maturity,
        EscrowError::FundingDeadlineBeyondMaturity,
    );
}
```
However, in `EscrowError` (line 870), the error is defined as:
```rust
FundingDeadlineAtOrAfterMaturity = 218,
```
There is no variant named `FundingDeadlineBeyondMaturity` in `EscrowError`.

**Why it matters**  
This causes compiler error `E0599: no variant named 'FundingDeadlineBeyondMaturity' found for enum 'EscrowError'`. Any build of `starfund_escrow` with funding deadline validation active fails compilation.

**Proposed solution**  
Update line 3217 to use the defined enum variant:
```rust
ensure(
    &env,
    deadline < maturity,
    EscrowError::FundingDeadlineAtOrAfterMaturity,
);
```

**Acceptance criteria**  
- Line 3217 references `EscrowError::FundingDeadlineAtOrAfterMaturity`.
- Compilation succeeds past line 3217.

**Testing**  
- Verify with `cargo check -p starfund_escrow`.

---

## Issue #56 — `InvestorRefunded` Stored in Instance Storage Violates ADR-007 and Risks Instance Storage Exhaustion

**Filed as:** [GitHub issue #75](https://github.com/ushpraise/Starfund-contracts/issues/75)

**Category:** Bug / Storage  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `DataKey::InvestorRefunded(Address)` (line 1338)
- `StarfundEscrow::refund_impl()` (line 8168)
- `StarfundEscrow::refund_batch()` (line 8243)
- `StarfundEscrow::is_investor_refunded()` (line 8366)
- `docs/adr/ADR-007-storage-key-evolution.md`

**Problem**  
In `refund_impl` (line 8168):
```rust
env.storage()
    .instance()
    .set(&DataKey::InvestorRefunded(investor.clone()), &true);
```
`DataKey::InvestorRefunded(Address)` is stored in `instance()` storage. In Soroban, all instance storage keys reside in a single ledger entry loaded in its entirety whenever the contract is invoked, with a hard protocol limit of 64KB. ADR-007 Rule 5 explicitly mandates:
> "Per-investor data MUST be stored in persistent storage to prevent instance storage growth from exceeding ledger entry size limits."

All other per-investor keys (`InvestorContribution`, `InvestorEffectiveYield`, `InvestorClaimNotBefore`, `InvestorClaimed`, `InvestorAllowlisted`) correctly reside in persistent storage.

**Why it matters**  
In escrows with many participants, writing an unbounded number of `InvestorRefunded(Address)` keys into instance storage bloats the instance entry. Once the 64KB limit is exceeded, any transaction attempting to write or load instance storage will fail with a Soroban Host storage error, permanently bricking refund processing.

**Proposed solution**  
1. Store `DataKey::InvestorRefunded(Address)` in `persistent()` storage instead of `instance()` storage.
2. In `bump_ttl()`, include `DataKey::InvestorRefunded` when extending persistent TTL for investors.

**Acceptance criteria**  
- `InvestorRefunded` reads and writes use `env.storage().persistent()`.
- Instance storage entry size remains constant regardless of the number of refunded investors.
- ADR-007 Rule 5 compliance is restored.

**Testing**  
- Add a test verifying `env.storage().persistent().has(&DataKey::InvestorRefunded(investor))` is `true` after refund.

---

## Issue #57 — `unfund()` Emits Misleading `OverWithdrawal` on Non-Positive Amounts and Misplaces Validation

**Filed as:** [GitHub issue #76](https://github.com/ushpraise/Starfund-contracts/issues/76)

**Category:** Bug  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::unfund()` (lines 8298–8308)

**Problem**  
In `StarfundEscrow::unfund()`, line 8298 first executes:
```rust
let contribution: i128 = Self::get_persistent_investor_contribution(&env, investor.clone());
ensure(&env, amount <= contribution, EscrowError::OverWithdrawal);
let remaining_contribution = contribution
    .checked_sub(amount)
    .unwrap_or_else(|| fail(&env, EscrowError::OverWithdrawal));
```
Then, at line 8306, it checks:
```rust
if amount <= 0 {
    fail(&env, EscrowError::OverWithdrawal);
}
```
If a caller submits `amount = 0` or a negative amount, the contract fails with `EscrowError::OverWithdrawal` rather than an explicit error indicating that the unfund amount must be positive. Furthermore, the non-positive check is placed *after* checking contribution bounds.

**Why it matters**  
Failing with `OverWithdrawal` when `amount == 0` is misleading to API callers and off-chain clients, who expect `OverWithdrawal` to signify withdrawing more than their deposited principal.

**Proposed solution**  
1. Hoist the `amount > 0` check to the top of `unfund()` before reading storage.
2. Add and emit a dedicated typed error `EscrowError::UnfundAmountNotPositive = 239` (or reuse `TransferAmountNotPositive`).

**Acceptance criteria**  
- Calling `unfund` with `amount <= 0` immediately reverts with `EscrowError::UnfundAmountNotPositive`.
- Validations occur before storage lookups.

**Testing**  
- Add test verifying `client.try_unfund(&investor, &0)` returns `UnfundAmountNotPositive`.

---

## Issue #58 — `unfund()` to Zero Desynchronizes `InvestorIndex` and `UniqueFunderCount`, Corrupting Re-Funding

**Filed as:** [GitHub issue #77](https://github.com/ushpraise/Starfund-contracts/issues/77)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::unfund()` (lines 8319–8330)
- `StarfundEscrow::fund_impl()` (lines 6651–6665)

**Problem**  
When an investor calls `unfund()` to withdraw their entire principal balance, `remaining_contribution == 0`. At lines 8320–8329:
```rust
if remaining_contribution == 0 {
    let cur: u32 = env.storage().instance().get(&keys::unique_funder_count()).unwrap_or(0);
    env.storage().instance().set(&keys::unique_funder_count(), &cur.saturating_sub(1));
}
```
The contract decrements `UniqueFunderCount`. However:
1. The investor address is **not removed** from `InvestorIndex` (`keys::investor_index()`).
2. `InvestorEffectiveYield` and `InvestorClaimNotBefore` are not cleared.
3. If this investor deposits again later, `prev == 0` is true in `fund_impl`. `fund_impl` increments `UniqueFunderCount` and executes `index.push_back(investor.clone())`.

**Why it matters**  
`InvestorIndex` now contains duplicate entries for the same investor address. Any caller paginating through `get_investors()` or `get_funding_records()` will receive duplicate records for that investor. Furthermore, `InvestorIndex.len()` will exceed `UniqueFunderCount`, breaking the invariant `investor_index.len() == unique_funder_count`.

**Proposed solution**  
When `remaining_contribution == 0` in `unfund()`:
1. Remove `investor` from `InvestorIndex` (or retain a tombstone / set structure).
2. Clean up or reset `InvestorEffectiveYield` and `InvestorClaimNotBefore` if returning investors should re-qualify for tiers.

**Acceptance criteria**  
- Unfunding to 0 and re-funding does not create duplicate entries in `InvestorIndex`.
- `InvestorIndex.len()` accurately reflects `UniqueFunderCount`.

**Testing**  
- Add test: Investor A funds -> unfunds to 0 -> funds again. Assert `client.get_investors(0, 10).len() == 1`.

---

## Issue #59 — `bump_ttl()` Panics on Non-Existent Persistent Keys and Contains Duplicate Loops

**Filed as:** [GitHub issue #78](https://github.com/ushpraise/Starfund-contracts/issues/78)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::bump_ttl()` (lines 7779–7820)

**Problem**  
In `StarfundEscrow::bump_ttl()`, the implementation contains two sequential loops over `allowlisted.iter()`:
1. Lines 7780–7789: Iterates over `allowlisted` and extends `InvestorAllowlisted`, and calls `env.storage().instance().extend_ttl(ttl, ttl)` on every single iteration inside the loop.
2. Lines 7795–7819: Iterates over `allowlisted` a second time, extends `InvestorAllowlisted` a second time, and unconditionally calls:
```rust
env.storage().persistent().extend_ttl(&DataKey::InvestorContribution(addr.clone()), ttl, ttl);
env.storage().persistent().extend_ttl(&DataKey::InvestorEffectiveYield(addr.clone()), ttl, ttl);
env.storage().persistent().extend_ttl(&DataKey::InvestorClaimNotBefore(addr.clone()), ttl, ttl);
env.storage().persistent().extend_ttl(&DataKey::InvestorClaimed(addr.clone()), ttl, ttl);
```
In Soroban SDK 25, calling `extend_ttl` on a persistent storage key that **does not exist** causes a host panic.

**Why it matters**  
An allowlisted investor who has not yet funded has no `InvestorContribution`, `InvestorEffectiveYield`, or `InvestorClaimNotBefore` key. Similarly, an investor who has not yet claimed has no `InvestorClaimed` key. Calling `bump_ttl()` with any such address immediately panics and reverts the transaction. Furthermore, calling `instance().extend_ttl()` repeatedly inside a loop is redundant.

**Proposed solution**  
1. Call `instance().extend_ttl(ttl, ttl)` once outside the loop.
2. Guard every persistent `extend_ttl` call with `if env.storage().persistent().has(&key)`.
3. Consolidate the two redundant loops into a single loop.

**Acceptance criteria**  
- `bump_ttl` safely extends persistent keys for allowlisted addresses that have not yet deposited or claimed.
- `instance().extend_ttl()` is called exactly once.

**Testing**  
- Add unit test calling `bump_ttl` with allowlisted addresses that have zero deposits, verifying no host panic occurs.

---

## Issue #60 — Missing Implementation for `StarfundEscrow::batch_bump_ttl` (Orphaned Doc Comment)

**Filed as:** [GitHub issue #79](https://github.com/ushpraise/Starfund-contracts/issues/79)

**Category:** Bug / DevEx  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- Lines 7822–7859

**Problem**  
At lines 7822–7858, there is an extensive 37-line doc comment documenting `StarfundEscrow::batch_bump_ttl(env: Env, keys: Vec<DataKey>)`:
`/// Extend TTL for a bounded set of storage keys in one admin-authenticated call.`
`/// This is the admin-gated counterpart to [StarfundEscrow::bump_ttl]...`
`/// Bounds: MAX_BUMP_TTL_BATCH...`
Immediately following this doc comment at line 7859 is:
```rust
pub fn propose_admin(env: Env, new_admin: Address, expected_nonce: u32) -> Address {
```
The actual implementation of `batch_bump_ttl` is completely missing from the contract, leaving its documentation orphaned on top of `propose_admin`.

**Why it matters**  
The documented administrative batch TTL extension functionality cannot be called by operators. The misplaced doc comment also misleads developers and documentation generators into displaying `batch_bump_ttl` documentation for `propose_admin`.

**Proposed solution**  
1. Implement `pub fn batch_bump_ttl(env: Env, keys: Vec<DataKey>)` under its doc comment, respecting `MAX_BUMP_TTL_BATCH`, admin authorization, and `.has()` guards.
2. Ensure `propose_admin` has its own dedicated doc comment.

**Acceptance criteria**  
- `batch_bump_ttl` is implemented as an admin-authorized entrypoint accepting a bounded `Vec<DataKey>`.
- `propose_admin` has accurate doc comments.

**Testing**  
- Add unit tests verifying `batch_bump_ttl` successfully extends TTL for valid instance and persistent keys.

---

## Issue #61 — Unrealistic 1s/Ledger Assumptions in TTL Constants Exceed Soroban `max_entry_ttl`

**Filed as:** [GitHub issue #80](https://github.com/ushpraise/Starfund-contracts/issues/80)

**Category:** Bug / Storage  
**Priority:** High  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- Lines 483, 492, 494–496

**Problem**  
The TTL extension constants are calculated using an incorrect assumption of 1 second per ledger:
```rust
pub const INSTANCE_TTL_MIN_EXTENSION_LEDGERS: u32 = 60 * 60; // Approx. 1h at 1 ledger/sec.
pub const TTL_ACTIVE_ESCROW_LEDGERS: u32 = 90 * 24 * 60 * 60; // 7,776,000 ledgers
pub const TTL_DISPUTED_ESCROW_LEDGERS: u32 = 180 * 24 * 60 * 60; // 15,552,000 ledgers
pub const TTL_TERMINAL_ESCROW_LEDGERS: u32 = 30 * 24 * 60 * 60; // 2,592,000 ledgers
```
Stellar ledgers close approximately every 5 seconds, not 1 second.
More critically, on the Stellar Soroban network, the protocol parameter `max_entry_ttl` is 3,110,400 ledgers (~180 days at 5s/ledger). Passing 15,552,000 or 7,776,000 ledgers to `extend_ttl` exceeds `max_entry_ttl`.

**Why it matters**  
In Soroban, calling `extend_ttl(threshold, extend_to)` where `extend_to > max_entry_ttl` causes host execution errors or clamping failures depending on the Soroban host environment.

**Proposed solution**  
Recalculate all TTL constants assuming 5 seconds per ledger (12 ledgers per minute, 720 per hour, 17,280 per day):
- Active escrow (90 days): `90 * 17_280 = 1,555,200` ledgers.
- Disputed escrow (180 days): `180 * 17_280 = 3,110,400` ledgers (exactly `max_entry_ttl`).
- Terminal escrow (30 days): `30 * 17_280 = 518,400` ledgers.
- 1 hour minimum: `720` ledgers.

**Acceptance criteria**  
- All TTL constants are calibrated for 5-second ledger closing times.
- No TTL constant exceeds the Stellar protocol ceiling of 3,110,400 ledgers.

**Testing**  
- Assert in tests that `get_lifecycle_ttl(&escrow) <= 3_110_400`.

---

## Issue #62 — Triplicate Divergent Dispute Representations Allow Dispute Check Bypass in `close_escrow`

**Filed as:** [GitHub issue #81](https://github.com/ushpraise/Starfund-contracts/issues/81)

**Category:** Bug / Security  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `close_escrow()` (line 285)
- `is_dispute_active()` (line 3860)
- `open_dispute()` (line 3899)
- `get_lifecycle_ttl()` (line 499)

**Problem**  
The codebase maintains three separate, unsynchronized representations of dispute state:
1. `DataKey::DisputeActive` (bool) — written by `open_dispute()` and read by `is_dispute_active()`.
2. `DataKey::Dispute` (bool) — read by `close_escrow()` (`env.storage().instance().get(&DataKey::Dispute)`).
3. `InvoiceEscrow::dispute_active` (bool) — read by `get_lifecycle_ttl()`.
When `open_dispute()` is called, it sets `DataKey::DisputeActive = true`. It **never** sets `DataKey::Dispute` and **never** updates `escrow.dispute_active`.

**Why it matters**  
1. `close_escrow()` checks `DataKey::Dispute`. Because `open_dispute()` wrote to `DataKey::DisputeActive`, `close_escrow()` never detects that a dispute is active, allowing an admin to close an escrow undergoing active dispute resolution!
2. `get_lifecycle_ttl()` checks `escrow.dispute_active` to extend TTL to the disputed horizon. Because `escrow.dispute_active` is never updated by `open_dispute`, disputed escrows do not receive the extended dispute TTL.

**Proposed solution**  
Unify all dispute state under `DataKey::DisputeActive` and update `escrow.dispute_active` atomically in `open_dispute()` and `close_dispute()`. Eliminate the unused `DataKey::Dispute`.

**Acceptance criteria**  
- `close_escrow()` checks `is_dispute_active()`.
- `open_dispute()` updates `DataKey::DisputeActive` and `escrow.dispute_active` in lockstep.

**Testing**  
- Add test: open dispute -> call `close_escrow()`. Assert it reverts with `CloseError::ActiveDispute`.

---

## Issue #63 — Missing Events for `open_dispute` and `close_dispute`

**Filed as:** [GitHub issue #82](https://github.com/ushpraise/Starfund-contracts/issues/82)

**Category:** Bug / Events  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::open_dispute()` (lines 3878–3903)
- `StarfundEscrow::close_dispute()` (lines 3905–3940)

**Problem**  
`open_dispute` and `close_dispute` perform state transitions that freeze or unfreeze value movement across the contract. However, neither function emits any contract event.

**Why it matters**  
Off-chain indexers, liquidity providers, and SME dashboards have no event stream to observe when an escrow enters or exits a dispute. The only way to detect a dispute is by continually polling `is_dispute_active()`.

**Proposed solution**  
Define and emit:
```rust
#[contractevent]
pub struct DisputeOpenedEvt {
    #[topic]
    pub name: Symbol,
    #[topic]
    pub invoice_id: Symbol,
    pub opened_by: Address,
    pub opened_at: u64,
}

#[contractevent]
pub struct DisputeClosedEvt {
    #[topic]
    pub name: Symbol,
    #[topic]
    pub invoice_id: Symbol,
    pub closed_by: Address,
    pub closed_at: u64,
    pub resolved: bool,
}
```

**Acceptance criteria**  
- `open_dispute` emits `DisputeOpenedEvt`.
- `close_dispute` emits `DisputeClosedEvt`.

**Testing**  
- Verify event emission in dispute tests using `env.events().all()`.

---

## Issue #64 — `close_dispute` Silent Early Return When `resolved == false`

**Filed as:** [GitHub issue #83](https://github.com/ushpraise/Starfund-contracts/issues/83)

**Category:** Bug  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::close_dispute()` (lines 3936–3939)

**Problem**  
In `StarfundEscrow::close_dispute()`:
```rust
if resolved {
    record.state = DisputeState::Resolved;
    ...
    env.storage().instance().set(&DataKey::DisputeActive, &false);
} else {
    // leave dispute active if the admin chooses to keep it open; the freeze stays in force.
    return;
}
```
If an admin invokes `close_dispute(caller, false)`, the call verifies signatures, loads storage, and then silently returns `()` without modifying state or recording any action.

**Why it matters**  
Calling an entrypoint named `close_dispute` with `resolved = false` does not close the dispute or update the dispute record, giving the false impression that a resolution action was recorded.

**Proposed solution**  
Either:
1. Reject `resolved == false` with a typed error `EscrowError::DisputeResolutionRejected`.
2. Or record `DisputeState::Dismissed` / `DisputeState::Unresolved` and update the record accordingly.

**Acceptance criteria**  
- `close_dispute` does not silently no-op when passed `false`.

**Testing**  
- Test calling `close_dispute` with `resolved: false` asserts proper error or state recording.

---

## Issue #65 — `CloseError` Enum Discriminants Collide With `EscrowError` On-Chain

**Filed as:** [GitHub issue #84](https://github.com/ushpraise/Starfund-contracts/issues/84)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `CloseError` enum (lines 209–222)
- `EscrowError` enum (lines 578+)

**Problem**  
`CloseError` is annotated with `#[contracterror]` and defines variants:
```rust
pub enum CloseError {
    NotAuthorized = 0,
    NotInitialized = 1,
    AlreadyClosed = 2,
    ActiveBalance = 3,
    ActiveDispute = 4,
}
```
Meanwhile, `EscrowError` is also annotated with `#[contracterror]` and defines:
```rust
pub enum EscrowError {
    Unauthorized = 1,
    AlreadyInitialized = 2,
    ...
}
```
In Soroban, all contract error codes returned from a contract share the same contract error numerical space (u32 error code).

**Why it matters**  
When a client receives error code `2`, it cannot distinguish whether the error is `CloseError::AlreadyClosed` or `EscrowError::AlreadyInitialized`. Error code `1` collides between `NotInitialized` and `Unauthorized`.

**Proposed solution**  
Consolidate `CloseError` variants into `EscrowError` with unique, non-colliding discriminants, or assign `CloseError` an offset range (e.g. 500+).

**Acceptance criteria**  
- All contract error codes across all enums in the crate are globally unique.

**Testing**  
- Add a compile/test assertion checking discriminant uniqueness across all contract errors.

---

## Issue #66 — `request_clear_legal_hold` Allows Scheduling Clear on Non-Existent Legal Holds

**Filed as:** [GitHub issue #85](https://github.com/ushpraise/Starfund-contracts/issues/85)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::request_clear_legal_hold()` (lines 5574–5597)

**Problem**  
In `request_clear_legal_hold()`, the admin signature is checked and the admin nonce is consumed. The function then calculates `clearable_at = now + delay`, sets `DataKey::LegalHoldClearableAt`, and emits `LegalHoldClearRequested`.
However, it **never checks whether a legal hold is actually active** (`Self::legal_hold_active(&env)`).

**Why it matters**  
An admin can request clearing a legal hold when no hold exists, consuming nonces and emitting confusing `LegalHoldClearRequested` events for non-existent holds.

**Proposed solution**  
Add a check at the beginning of `request_clear_legal_hold`:
```rust
ensure(&env, Self::legal_hold_active(&env), EscrowError::LegalHoldNotActive);
```

**Acceptance criteria**  
- `request_clear_legal_hold` reverts with `EscrowError::LegalHoldNotActive` if no legal hold is active.

**Testing**  
- Add test verifying `try_request_clear_legal_hold` fails when `legal_hold == false`.

---

## Issue #67 — Redundant Consecutive Pause Checks in `withdraw()` and `settle()`

**Filed as:** [GitHub issue #86](https://github.com/ushpraise/Starfund-contracts/issues/86)

**Category:** Refactor / Gas  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::withdraw()` (lines 7040–7050)
- `StarfundEscrow::settle()` (lines 6771–6782)

**Problem**  
In `withdraw()`:
```rust
ensure(
    &env,
    !Self::paused_blocks(&env, PauseEntry::Withdrawal),
    EscrowError::PausedBlocksWithdrawal,
);
guard_not_paused(
    &env,
    EscrowError::PausedBlocksWithdrawal,
    PauseEntry::Withdrawal,
);
```
And identically in `settle()`:
```rust
ensure(
    &env,
    !Self::paused_blocks(&env, PauseEntry::Settlement),
    EscrowError::PausedBlocksSettlement,
);
guard_not_paused(
    &env,
    EscrowError::PausedBlocksSettlement,
    PauseEntry::Settlement,
);
```
Both functions execute `ensure(!Self::paused_blocks(...))` immediately followed by `guard_not_paused(...)`. `guard_not_paused` internally calls `ensure(!Self::paused_blocks(...))`.

**Why it matters**  
Calling the identical check twice consecutively performs redundant storage reads and logic execution, wasting gas and cluttering the code.

**Proposed solution**  
Remove the duplicate `ensure(!Self::paused_blocks(...))` call and keep only `guard_not_paused(...)`.

**Acceptance criteria**  
- Duplicate pause checks in `withdraw()` and `settle()` are eliminated.

**Testing**  
- Run pause tests in `escrow/src/tests/pause.rs`.

---

## Issue #68 — `withdraw()` Lacks `amount > 0` Guard, Permitting Zero-Payout Execution and Event

**Filed as:** [GitHub issue #87](https://github.com/ushpraise/Starfund-contracts/issues/87)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::withdraw()` (lines 7058–7110)

**Problem**  
In `StarfundEscrow::withdraw()`:
```rust
let released_amount: i128 = env.storage().instance().get(&keys::released_amount()).unwrap_or(0);
let amount = escrow.funded_amount.checked_sub(released_amount).unwrap();
```
If the entire funded amount was already released to the SME via `StarfundEscrow::release()`, `released_amount == escrow.funded_amount`, making `amount == 0`.
`withdraw()` does not check `amount > 0`. It proceeds to set `escrow.status = 3` (Withdrawn), skips fee and net transfers, and publishes `SmeWithdrew` with `payout: 0, fee: 0`.

**Why it matters**  
Invoking `withdraw()` when there is nothing left to withdraw should be rejected as an invalid operation rather than mutating state and emitting zero-value events.

**Proposed solution**  
Add validation:
```rust
ensure(&env, amount > 0, EscrowError::NothingToWithdraw);
```

**Acceptance criteria**  
- Calling `withdraw()` when `amount == 0` reverts with `EscrowError::NothingToWithdraw`.

**Testing**  
- Add test: Fully release funds via `release()`, then call `withdraw()`. Assert it reverts with `NothingToWithdraw`.

---

## Issue #69 — `settle_batch` Requires Target SME Cross-Contract Signatures, Failing Batch Settlements

**Filed as:** [GitHub issue #88](https://github.com/ushpraise/Starfund-contracts/issues/88)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::settle_batch()` (lines 6871–6885)
- `StarfundEscrow::settle()` (line 6787)

**Problem**  
`settle_batch` loops over a vector of escrow addresses and invokes `client.settle()` on each target contract.
In `settle()`, line 6787 executes:
`let mut escrow = Self::load_escrow_require_sme(&env);`
which requires `escrow.sme_address.require_auth()`.
In Soroban, when Contract A invokes Contract B, Contract B's `require_auth` on an address requires that address to have explicitly authorized the cross-contract invocation from Contract A to Contract B.

**Why it matters**  
An operator or admin attempting to settle multiple escrows in a batch cannot produce the SME signatures for every distinct SME in the batch, causing `settle_batch` to fail on authorization. Furthermore, once an escrow has reached maturity, settlement should be executable by either the SME, the admin, or permissionlessly.

**Proposed solution**  
Allow `settle()` to be authorized by either `escrow.sme_address` OR `escrow.admin` (or permissionlessly once `now >= escrow.maturity`), enabling batch settlement by operators.

**Acceptance criteria**  
- `settle_batch` can be executed by the admin without requiring individual SME signatures for each target escrow.

**Testing**  
- Add integration test settling 3 mature escrows via `settle_batch`.

---

## Issue #70 — `set_storage_limit` Lacks Admin Nonce Replay Protection and Event Emission

**Filed as:** [GitHub issue #89](https://github.com/ushpraise/Starfund-contracts/issues/89)

**Category:** Bug / Security  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::set_storage_limit()` (lines 7738–7750)

**Problem**  
In `StarfundEscrow::set_storage_limit(env: Env, limit: u32) -> u32`:
1. It calls `Self::load_escrow_require_admin(&env)`.
2. It validates `limit` in `MIN_STORAGE_LIMIT_LEDGERS..=MAX_STORAGE_LIMIT_LEDGERS`.
3. It sets `DataKey::StorageLimit`.
Unlike every other admin mutation entrypoint (`set_legal_hold`, `request_clear_legal_hold`, `rotate_beneficiary`, `propose_admin`, `cancel_funding`, `update_funding_target`), it has no `expected_nonce: u32` parameter, does not call `consume_admin_nonce`, and emits no event.

**Why it matters**  
1. Multi-sig admin wallets cannot bind a `set_storage_limit` authorization to a specific transaction sequence, exposing them to signature replay.
2. Indexers and monitoring systems cannot observe when contract storage TTL limits are modified.

**Proposed solution**  
Add `expected_nonce: u32` to `set_storage_limit`, consume the nonce via `Self::consume_admin_nonce(&env, expected_nonce)`, and define and emit `StorageLimitUpdatedEvent`.

**Acceptance criteria**  
- `set_storage_limit` consumes an admin nonce.
- `StorageLimitUpdatedEvent` is emitted with `invoice_id`, `old_limit`, and `new_limit`.

**Testing**  
- Test calling `set_storage_limit` with invalid nonce reverts with `AdminNonceMismatch`.

---

## Issue #71 — `cancel_funding` Lacks Dispute Gate, Permitting Cancellation During Active Dispute

**Filed as:** [GitHub issue #90](https://github.com/ushpraise/Starfund-contracts/issues/90)

**Category:** Bug / Security  
**Priority:** High  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::cancel_funding()` (lines 8108–8127)

**Problem**  
In `StarfundEscrow::cancel_funding()`:
```rust
Self::guard_not_legal_hold(&env, EscrowError::LegalHoldBlocksCancelFunding);
let mut escrow = Self::load_escrow_require_admin(&env);
Self::consume_admin_nonce(&env, expected_nonce);
guard_status_eq(&env, escrow.status, 0, EscrowError::CancelFundingNotOpen);
```
While `cancel_funding` checks legal hold and status, it **does not call `guard_not_disputed`**.

**Why it matters**  
If a dispute is active (`is_dispute_active() == true`), value-moving and lifecycle transitions must be frozen until resolution. Without a dispute guard, an admin can bypass the dispute freeze and unilaterally cancel the funding phase.

**Proposed solution**  
Add `guard_not_disputed(&env, EscrowError::DisputeBlocksCancelFunding);` to `cancel_funding()`.

**Acceptance criteria**  
- `cancel_funding` reverts with `DisputeBlocksCancelFunding` if an active dispute exists.

**Testing**  
- Add test: open dispute on open escrow -> attempt `cancel_funding` -> assert revert.

---

## Issue #72 — `cancel_funding` Ignores Operational Pause Gates

**Filed as:** [GitHub issue #91](https://github.com/ushpraise/Starfund-contracts/issues/91)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::cancel_funding()` (lines 8108–8127)

**Problem**  
`cancel_funding` lacks an operational pause check (`guard_not_paused`). While funding, refunds, claims, releases, and withdrawals all check pause state, an operator pause does not stop `cancel_funding`.

**Why it matters**  
An emergency operational pause should prevent state transitions across the escrow contract.

**Proposed solution**  
Add `guard_not_paused(&env, EscrowError::PausedBlocksCancelFunding, PauseEntry::Funding);` to `cancel_funding()`.

**Acceptance criteria**  
- `cancel_funding` reverts when funding operations are paused.

**Testing**  
- Add test pausing funding scope and asserting `cancel_funding` reverts.

---

## Issue #73 — `refund_batch` Passes `false` to `skip_zero_contribution`, Breaking Batch Execution on Zero Balance

**Filed as:** [GitHub issue #92](https://github.com/ushpraise/Starfund-contracts/issues/92)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::refund_batch()` (lines 8249–8252)
- `StarfundEscrow::refund_impl()` (lines 8146–8163)

**Problem**  
`refund_impl` has a parameter `skip_zero_contribution: bool`. At lines 8148–8150, its documentation states:
`/// When skip_zero_contribution is true, investors with no recorded contribution are skipped silently (batch mode). Otherwise a zero contribution fails with EscrowError::NoContributionToRefund.`
However, in `StarfundEscrow::refund_batch()` at line 8250:
```rust
Self::refund(env.clone(), investor);
```
`refund()` hardcodes `Self::refund_impl(&env, investor, false)`.

**Why it matters**  
Because `false` is passed, if any address in the batch has a zero contribution, `refund_impl` panics with `EscrowError::NoContributionToRefund`, terminating the entire batch transaction instead of skipping it as intended for batch mode.

**Proposed solution**  
In `refund_batch()`, call `Self::refund_impl(&env, investor, true)` directly.

**Acceptance criteria**  
- `refund_batch` silently skips addresses with zero contribution without aborting the batch.

**Testing**  
- Add test executing `refund_batch` containing a mix of funded investors and zero-contribution addresses.

---

## Issue #74 — `InvestorPayoutClaimed` Event Omits Payout Amount

**Filed as:** [GitHub issue #93](https://github.com/ushpraise/Starfund-contracts/issues/93)

**Category:** Bug / Events  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `InvestorPayoutClaimed` struct (lines 2385–2392)
- `StarfundEscrow::claim_investor_payout()` (lines 7235–7240)

**Problem**  
The event struct `InvestorPayoutClaimed` is defined as:
```rust
#[contractevent]
pub struct InvestorPayoutClaimed {
    #[topic]
    pub name: Symbol,
    #[topic]
    pub investor: Address,
    #[topic]
    pub invoice_id: Symbol,
}
```
It does not contain a field for the payout amount.

**Why it matters**  
All other financial events (`SmeWithdrew`, `PartialRelease`, `FinalRelease`, `EscrowSettled`, `EscrowFunded`, `EscrowUnfunded`) include the transferred token amount. Indexers monitoring `InvestorPayoutClaimed` cannot determine how many tokens were transferred to the investor without making an external query.

**Proposed solution**  
Add `pub payout: i128` to `InvestorPayoutClaimed` and populate it in `claim_investor_payout`.

**Acceptance criteria**  
- `InvestorPayoutClaimed` includes the `payout` amount field.

**Testing**  
- Assert event data payload contains `payout` in claim payout unit tests.

---

## Issue #75 — Economic Discrepancy: `settle()` and `get_settlement_pool()` Use Base Yield While `compute_investor_payout()` Uses Investor Tiered Yields

**Filed as:** [GitHub issue #94](https://github.com/ushpraise/Starfund-contracts/issues/94)

**Category:** Bug / Security  
**Priority:** High  
**Suggested Complexity:** High  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::settle()` (lines 6813–6823)
- `StarfundEscrow::get_settlement_pool()` (lines 7376–7450)
- `StarfundEscrow::compute_investor_payout()` (lines 7350–7374)
- `docs/escrow-pro-rata.md` (lines 106–120)

**Problem**  
In `settle()` and `get_settlement_pool()`, the settlement pool is calculated as:
```rust
let coupon = funded_amount * escrow.yield_bps / 10_000;
let settle_pool = funded_amount + coupon;
```
This calculation strictly uses the escrow's base `yield_bps`.
However, in `compute_investor_payout()`, each investor's payout is calculated using their individual tiered yield:
```rust
let effective_yield_bps = get_persistent_investor_effective_yield(...).unwrap_or(escrow.yield_bps);
let coupon = total_principal * effective_yield_bps / 10_000;
let settle_pool = total_principal + coupon;
payout = contribution * settle_pool / total_principal;
```
If investors locked funds under higher yield tiers (e.g. 12% vs base 8%), the sum of all investor payouts ($\sum \text{payout}_i$) will exceed the `settle_pool` calculated in `settle()` and `get_settlement_pool()`.

**Why it matters**  
If the SME repays the amount returned by `get_settlement_pool()`, the contract balance will be insufficient to pay all investors. The last investors to call `claim_investor_payout()` will have their claims revert with `InsufficientTokenBalanceBeforeTransfer`.

**Proposed solution**  
1. In `get_settlement_pool()`, compute the true total liability by summing the theoretical payouts across all recorded investors in `InvestorIndex`.
2. Document the difference between the base-yield pool and the tier-weighted pool.

**Acceptance criteria**  
- `get_settlement_pool()` returns a pool amount that guarantees solvency for all tiered investor claims.

**Testing**  
- Create an escrow with 2 investors in different yield tiers. Assert contract balance covers sum of both claims.

---

## Issue #76 — Tuple Destructuring Compile Error in `fund_impl` Tier Selection

**Filed as:** [GitHub issue #95](https://github.com/ushpraise/Starfund-contracts/issues/95)

**Category:** Bug / Compile-Blocker  
**Priority:** Critical  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::fund_impl()` (lines 6542–6543)
- `effective_yield_for_commitment()` (line 2993)

**Problem**  
At line 6542, `fund_impl` calls:
```rust
let (eff, lock) =
    Self::effective_yield_for_commitment(&env, escrow.yield_bps, committed_lock_secs);
investor_effective_yield_bps = eff;
tier_lock_secs = lock;
```
However, `effective_yield_for_commitment` returns `YieldResolution`:
```rust
fn effective_yield_for_commitment(...) -> YieldResolution
```
Rust does not permit destructuring a struct into a 2-tuple `(eff, lock)`.

**Why it matters**  
Causes compiler error `E0308: mismatched types: expected tuple (i64, u64), found struct YieldResolution`, blocking the build of `fund_impl`.

**Proposed solution**  
Change line 6542 to access the fields of `YieldResolution`:
```rust
let resolution =
    Self::effective_yield_for_commitment(&env, escrow.yield_bps, committed_lock_secs);
investor_effective_yield_bps = resolution.effective_yield_bps;
tier_lock_secs = resolution.matched_lock_secs;
```

**Acceptance criteria**  
- Struct `YieldResolution` is bound by name, not destructured as a tuple.
- Code compiles without `E0308`.

**Testing**  
- Verified during `cargo check -p starfund_escrow`.

---

## Issue #77 — Inbound Token Transfer Parameter Naming in `transfer_funding_token_with_balance_checks`

**Filed as:** [GitHub issue #96](https://github.com/ushpraise/Starfund-contracts/issues/96)

**Category:** Refactor / Code Quality  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/external_calls.rs` (lines 106–112)

**Problem**  
In `external_calls::transfer_funding_token_with_balance_checks`:
```rust
pub fn transfer_funding_token_with_balance_checks(
    env: &Env,
    token_addr: &Address,
    from: &Address,
    treasury: &Address,
    amount: i128,
)
```
The recipient parameter is named `treasury`. However, this function is the general-purpose outbound transfer helper called for transfers to the SME in `withdraw()`, to investors in `claim_investor_payout()` and `refund()`, and to the treasury in `sweep_terminal_dust()`.

**Why it matters**  
Naming the recipient `treasury` in a generic transfer function is confusing to contributors and static analysis tools.

**Proposed solution**  
Rename the parameter from `treasury` to `recipient` (or `to`).

**Acceptance criteria**  
- Parameter name reflects generic recipient semantics.

**Testing**  
- Ensure all calls in `lib.rs` and tests compile cleanly.

---

## Issue #78 — Lack of Self-Transfer Guard in `external_calls.rs` Causes Confusing Delta Errors

**Filed as:** [GitHub issue #97](https://github.com/ushpraise/Starfund-contracts/issues/97)

**Category:** Bug  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/external_calls.rs` (lines 113–145)

**Problem**  
In `transfer_funding_token_with_balance_checks`, if `from == treasury` (a self-transfer), `from_before` equals `treasury_before` and `from_after` equals `treasury_after`. As a result, `spent` and `received` evaluate to `0`, causing the transfer to fail with `EscrowError::SenderBalanceDeltaMismatch`.

**Why it matters**  
Failing with `SenderBalanceDeltaMismatch` masks the underlying bug (an address attempting to transfer to itself).

**Proposed solution**  
Add an explicit assertion at the start of the function:
```rust
ensure(env, from != treasury, EscrowError::SelfTransferNotAllowed);
```

**Acceptance criteria**  
- Self-transfers are explicitly rejected with a dedicated error.

**Testing**  
- Add unit test passing identical addresses to `transfer_funding_token_with_balance_checks`.

---

## Issue #79 — Clean Up Commented-Out Test Modules in `escrow/src/tests/mod.rs`

**Filed as:** [GitHub issue #98](https://github.com/ushpraise/Starfund-contracts/issues/98)

**Category:** Testing / DevEx  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/tests/mod.rs` (lines 64–66, 77, 88, 90)

**Problem**  
In `escrow/src/tests/mod.rs`, several test module declarations are commented out:
```rust
// mod collateral_boundary_tests; // file not present in this tree
// mod collateral_config_view;    // file not present in this tree
// mod collateral_limit_setter;   // file not present in this tree
// mod integration;
// mod settlement_limit; // file not present in this tree
// mod admin_recovery;  // file not present in this tree
```

**Why it matters**  
Commented-out code causes dead code noise and confusion about which test suites are active.

**Proposed solution**  
Remove the commented-out module lines or replace them with a single explanatory doc comment.

**Acceptance criteria**  
- Stale commented-out module lines are cleaned up.

**Testing**  
- `cargo test` runs without warnings.

---

## Issue #80 — Missing `collateral_pledge_key` Function Referenced in `escrow/src/keys.rs` Docs

**Filed as:** [GitHub issue #99](https://github.com/ushpraise/Starfund-contracts/issues/99)

**Category:** Documentation / DevEx  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/keys.rs` (lines 13–17)

**Problem**  
The documentation of `escrow/src/keys.rs` states:
`/// The collateral pledge key family is managed by collateral_pledge_key. All three collateral entrypoints ... call this function instead of constructing DataKey::SmeCollateralPledge inline.`
However, the function `collateral_pledge_key` is not defined anywhere in `keys.rs`.

**Why it matters**  
API documentation promises a key helper function that does not exist in the module.

**Proposed solution**  
Implement the missing helper in `escrow/src/keys.rs`:
```rust
pub(crate) fn collateral_pledge_key() -> DataKey {
    DataKey::SmeCollateralPledge
}
```
and use it at call sites in `lib.rs`.

**Acceptance criteria**  
- `collateral_pledge_key` exists in `keys.rs` and matches doc description.

**Testing**  
- Verify `keys::collateral_pledge_key()` returns `DataKey::SmeCollateralPledge`.

---

## Issue #81 — `test_allowlist_tests.rs` Fails Compilation Due to Outdated `client.init` Argument Count (15 vs 19)

**Filed as:** [GitHub issue #100](https://github.com/ushpraise/Starfund-contracts/issues/100)

**Category:** Testing / Compile-Blocker  
**Priority:** Critical  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/test_allowlist_tests.rs` (lines 19–35, 511–527)
- `escrow/src/lib.rs` (lines 3073–3094)

**Problem**  
In `escrow/src/test_allowlist_tests.rs`, helper functions `init` (line 19) and `init_gate` (line 511) call `client.init(...)` with 15 arguments:
`admin, invoice_id, sme, amount, yield_bps, maturity, token, registry, treasury, yield_tiers, min_contribution, max_unique_investors, max_per_investor, legal_hold_clear_delay, maturity_max_horizon`.
`StarfundEscrow::init` takes 19 parameters (excluding `env`). It is missing:
`funding_deadline`, `allowlist_active`, `protocol_fee_bps`, `token_decimals`.

**Why it matters**  
`cargo test` fails compilation with `E0061: this function takes 19 arguments but 15 arguments were supplied`. The entire `test_allowlist_tests` suite cannot run.

**Proposed solution**  
Update `client.init(...)` calls in `test_allowlist_tests.rs` to pass the 4 missing parameters (`&None, &None, &None, &None`).

**Acceptance criteria**  
- `test_allowlist_tests.rs` compiles without argument count mismatches.

**Testing**  
- Compile with `cargo test -p starfund_escrow --test test_allowlist_tests`.

---

## Issue #82 — `release_budget_tests.rs` Fails Compilation Due to Missing `token_decimals` Argument (18 vs 19)

**Filed as:** [GitHub issue #101](https://github.com/ushpraise/Starfund-contracts/issues/101)

**Category:** Testing / Compile-Blocker  
**Priority:** Critical  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/release_budget_tests.rs` (lines 54–73)
- `escrow/src/lib.rs` (lines 3073–3094)

**Problem**  
In `escrow/src/release_budget_tests.rs`, function `deploy` calls `client.init(...)` with 18 arguments, omitting the newly added 19th argument `token_decimals`.

**Why it matters**  
Causes compiler error `E0061: this function takes 19 arguments but 18 arguments were supplied`. The release budget regression test suite cannot run.

**Proposed solution**  
Pass `&None::<u32>` as the 19th argument in `release_budget_tests.rs`.

**Acceptance criteria**  
- `release_budget_tests.rs` compiles cleanly.

**Testing**  
- Compile with `cargo test -p starfund_escrow --test release_budget_tests`.

---

## Issue #83 — `tests/dispute_release.rs` Asserts Invalid State Transition (`withdraw()` at Status 0)

**Filed as:** [GitHub issue #102](https://github.com/ushpraise/Starfund-contracts/issues/102)

**Category:** Testing / Bug  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/tests/dispute_release.rs` (lines 35–43)

**Problem**  
In `tests/dispute_release.rs`:
```rust
#[test]
fn release_before_dispute_succeeds() {
    let (_, client, _, _) = funded_client();
    let before = client.get_escrow();
    assert_eq!(before.status, 0);

    let released = client.withdraw();
    assert_eq!(released.status, 3);
}
```
The test explicitly asserts `before.status == 0` and then invokes `client.withdraw()`.
In `withdraw()`, line 7056 enforces:
`guard_status_eq(&env, escrow.status, 1, EscrowError::WithdrawalNotFunded);`
`withdraw()` requires status `1` (Funded). Calling it when `status == 0` reverts with `WithdrawalNotFunded`.

**Why it matters**  
This test contains an invalid assumption about contract state transitions and will fail whenever executed against the real contract implementation.

**Proposed solution**  
Fund the escrow up to `funding_target` in `funded_client` so that `status == 1` before invoking `withdraw()`.

**Acceptance criteria**  
- Test sets escrow status to `1` before asserting `withdraw()` behavior.

**Testing**  
- Run `cargo test dispute_release`.

---

## Issue #84 — OpenAPI Test Suite in `docs/tests/openapi.test.js` is Not Executed in CI

**Filed as:** [GitHub issue #103](https://github.com/ushpraise/Starfund-contracts/issues/103)

**Category:** CI/CD  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `.github/workflows/ci.yml`
- `docs/package.json`
- `docs/tests/openapi.test.js`

**Problem**  
The repository includes a Node.js test suite in `docs/tests/openapi.test.js` that verifies OpenAPI 3.1 schema compliance. However, `.github/workflows/ci.yml` only runs Rust cargo steps and never installs Node dependencies or runs `npm test` from `docs/`.

**Why it matters**  
Schema regressions, syntax errors, or schema divergences in `docs/openapi.yaml` are not caught during pull requests or CI runs.

**Proposed solution**  
Add a job or step to `.github/workflows/ci.yml`:
```yaml
- name: Run OpenAPI validation
  working-directory: docs
  run: |
    npm ci
    npm test
```

**Acceptance criteria**  
- CI workflow validates `docs/openapi.yaml` using `openapi.test.js`.

**Testing**  
- Run `npm test` inside `docs/` and verify pass.

---

## Issue #85 — Severe Drift Between `docs/openapi.yaml` and Smart Contract Data Models

**Filed as:** [GitHub issue #104](https://github.com/ushpraise/Starfund-contracts/issues/104)

**Category:** Documentation  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `docs/openapi.yaml` (lines 67–95)
- `escrow/src/lib.rs` (`InvoiceEscrow` struct line 1410)

**Problem**  
In `docs/openapi.yaml`, the `InvoiceEscrow` component schema defines properties:
- `buyer_address` (required)
- `is_paid` (required)
- `amount` as `int64`
In the smart contract `InvoiceEscrow`:
- The role is `payer`, not `buyer_address`.
- There is an `admin: Address` field (missing from OpenAPI).
- There is a `dispute_active: bool` field (missing from OpenAPI).
- `amount`, `funding_target`, and `funded_amount` are 128-bit integers (`i128`), which exceed `int64`.

**Why it matters**  
API clients generating bindings from `openapi.yaml` will fail to deserialize escrow records returned from indexers or RPC endpoints.

**Proposed solution**  
Update `docs/openapi.yaml` to match `InvoiceEscrow` in `escrow/src/lib.rs`, replacing `buyer_address` with `payer`, adding `admin` and `dispute_active`, and updating integer schemas.

**Acceptance criteria**  
- `docs/openapi.yaml` field definitions match `InvoiceEscrow` in `escrow/src/lib.rs`.

**Testing**  
- Run `npm test` in `docs/` after updating test fixtures.

---

## Issue #86 — `docs/EVENT_SCHEMA.md` Falsely Documents Trailing `version` Topic for All Events

**Filed as:** [GitHub issue #105](https://github.com/ushpraise/Starfund-contracts/issues/105)

**Category:** Documentation  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `docs/EVENT_SCHEMA.md` (lines 34–38)
- `escrow/src/lib.rs` (contract event definitions)

**Problem**  
`docs/EVENT_SCHEMA.md` asserts:
`Every event in this contract emits a trailing schema version topic. It is a #[topic] Symbol named version with value v1. It is always the final topic in the topic list, after all other #[topic] fields. Indexers MUST ignore this extra topic...`
In `escrow/src/lib.rs`, **no event struct** defines a `version` topic. Events only emit the specific fields marked with `#[topic]` in their struct definitions.

**Why it matters**  
Off-chain indexers built according to `EVENT_SCHEMA.md` that expect a trailing `version` topic will fail to decode every single event emitted by `StarfundEscrow`.

**Proposed solution**  
Update `docs/EVENT_SCHEMA.md` to remove the incorrect claim regarding a universal trailing `version` topic, and accurately document each event's topic list.

**Acceptance criteria**  
- `docs/EVENT_SCHEMA.md` accurately describes actual Soroban contract event topics.

---

## Issue #87 — `AdminTransferredEvent` Defined in `lib.rs` and Documented in `EVENT_SCHEMA.md` is Never Emitted

**Filed as:** [GitHub issue #106](https://github.com/ushpraise/Starfund-contracts/issues/106)

**Category:** Bug / Events  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs` (`AdminTransferredEvent` line 2126, `accept_admin` line 7955)
- `docs/EVENT_SCHEMA.md` (line 52)

**Problem**  
`AdminTransferredEvent` is declared as a `#[contractevent]` struct at line 2126 and documented in `EVENT_SCHEMA.md` as emitted by `accept_admin`.
However, `accept_admin` actually emits `AdminAcceptedEvent`:
```rust
AdminAcceptedEvent {
    name: symbol_short!("adm_acc"),
    invoice_id: escrow.invoice_id.clone(),
    prior_admin,
    new_admin: pending,
}
.publish(&env);
```
`AdminTransferredEvent` is completely unreferenced dead code.

**Why it matters**  
Indexers following `EVENT_SCHEMA.md` listen for topic `admin` / `AdminTransferredEvent` and miss all admin acceptance events.

**Proposed solution**  
Reconcile `AdminAcceptedEvent` and `AdminTransferredEvent`. Either update `accept_admin` to emit `AdminTransferredEvent` or update documentation to reference `AdminAcceptedEvent`.

**Acceptance criteria**  
- Event definition, emission in `accept_admin`, and documentation in `EVENT_SCHEMA.md` are aligned.

**Testing**  
- Assert emitted event type in `tests/admin.rs`.

---

## Issue #88 — `update_maturity` Lacks Lower Bound Check Against `funding_deadline`

**Filed as:** [GitHub issue #107](https://github.com/ushpraise/Starfund-contracts/issues/107)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::update_maturity()` (lines 7453–7485)
- `validate_maturity_bounds()` (lines 1206–1220)

**Problem**  
In `update_maturity`, the new maturity timestamp is validated via `validate_maturity_bounds(&env, new_maturity, max_horizon)`. That helper only verifies:
`new_maturity >= now` and `new_maturity <= now + max_horizon`.
It **does not check `funding_deadline`**.
At `init` (line 3216) and `extend_funding_deadline` (line 7595), the contract strictly enforces:
`deadline < maturity` (`EscrowError::FundingDeadlineAtOrAfterMaturity`).

**Why it matters**  
An admin calling `update_maturity` can reduce `maturity` to a timestamp before `funding_deadline`, violating the invariant that funding closes before the invoice matures.

**Proposed solution**  
In `update_maturity`, if `funding_deadline` is set, ensure:
```rust
if let Some(deadline) = env.storage().instance().get(&keys::funding_deadline()) {
    ensure(&env, deadline < new_maturity, EscrowError::FundingDeadlineAtOrAfterMaturity);
}
```

**Acceptance criteria**  
- `update_maturity` rejects new maturities that are less than or equal to the configured funding deadline.

**Testing**  
- Add test setting funding deadline to 1000 and attempting `update_maturity` to 900.

---

## Issue #89 — `update_maturity` Lacks Admin Nonce Replay Protection

**Filed as:** [GitHub issue #108](https://github.com/ushpraise/Starfund-contracts/issues/108)

**Category:** Security  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::update_maturity()` (lines 7453–7485)

**Problem**  
`update_maturity(env: Env, new_maturity: u64)` requires admin authorization via `Self::load_escrow_require_admin(&env)`. However, it does not accept an `expected_nonce: u32` parameter and does not invoke `consume_admin_nonce`.

**Why it matters**  
Admin actions altering contract timelines should be protected against signature replay in multi-sig scenarios.

**Proposed solution**  
Add `expected_nonce: u32` to `update_maturity` and consume it with `Self::consume_admin_nonce(&env, expected_nonce)`.

**Acceptance criteria**  
- `update_maturity` consumes an admin nonce.

**Testing**  
- Verify nonce increment and error on nonce mismatch.

---

## Issue #90 — `update_yield_bps` Fails to Enforce Documented Zero-Funded Invariant

**Filed as:** [GitHub issue #109](https://github.com/ushpraise/Starfund-contracts/issues/109)

**Category:** Bug / Security  
**Priority:** High  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::update_yield_bps()` (lines 7487–7543)

**Problem**  
The rustdoc for `update_yield_bps` states:
`/// Only valid while the escrow is in open status (status == 0) — i.e. before any investor has funded... Once investors have committed principal the yield rate is effectively locked...`
However, the implementation only checks:
```rust
guard_status_eq(&env, escrow.status, 0, EscrowError::YieldBpsUpdateNotOpen);
```
It **never checks `escrow.funded_amount == 0`**.

**Why it matters**  
While an escrow is open (`status == 0`), investors may have already deposited funds (e.g. 5,000 out of a 10,000 target). An admin can call `update_yield_bps` and alter the base yield after investors have already committed funds, changing their expected payout.

**Proposed solution**  
Add validation:
```rust
ensure(&env, escrow.funded_amount == 0, EscrowError::YieldBpsUpdateFunded);
```

**Acceptance criteria**  
- `update_yield_bps` reverts if `escrow.funded_amount > 0`.

**Testing**  
- Add test funding 100 units and verifying `try_update_yield_bps` fails.

---

## Issue #91 — `update_funding_target` Lacks Upper Bound Check Against `MAX_INVOICE_AMOUNT`

**Filed as:** [GitHub issue #110](https://github.com/ushpraise/Starfund-contracts/issues/110)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Low  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::update_funding_target()` (lines 5862–5910)

**Problem**  
In `update_funding_target`, the function validates:
`new_target > 0` and `new_target >= escrow.funded_amount`.
However, it never verifies `new_target <= MAX_INVOICE_AMOUNT`.
At `init`, `amount <= MAX_INVOICE_AMOUNT` is enforced.

**Why it matters**  
An admin could set `new_target` to `i128::MAX`, creating arithmetic overflow vulnerabilities in coupon or pro-rata math.

**Proposed solution**  
Add `ensure(&env, new_target <= MAX_INVOICE_AMOUNT, EscrowError::AmountExceedsMax);`.

**Acceptance criteria**  
- `update_funding_target` rejects values exceeding `MAX_INVOICE_AMOUNT`.

**Testing**  
- Test calling `update_funding_target` with `MAX_INVOICE_AMOUNT + 1`.

---

## Issue #92 — `update_funding_target` Omits `FundingStateChanged` Event on Promotion to Funded

**Filed as:** [GitHub issue #111](https://github.com/ushpraise/Starfund-contracts/issues/111)

**Category:** Bug / Events  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::update_funding_target()` (lines 5880–5908)
- `docs/EVENT_SCHEMA.md` (line 49)

**Problem**  
When `update_funding_target` lowers the target such that `escrow.funded_amount >= new_target`, it mutates `escrow.status = 1` and creates the `FundingCloseSnapshot`.
However, it only emits `FundingTargetUpdated`. `docs/EVENT_SCHEMA.md` explicitly lists `update_funding_target` as an emitter of `FundingStateChanged`.

**Why it matters**  
Indexers listening for `FundingStateChanged` to track when escrows transition to status `1` will miss state changes triggered by target updates.

**Proposed solution**  
Emit `FundingStateChanged` when `update_funding_target` transitions `status` to `1`.

**Acceptance criteria**  
- `FundingStateChanged` is emitted upon status promotion in `update_funding_target`.

**Testing**  
- Verify `FundingStateChanged` event is present in `env.events().all()`.

---

## Issue #93 — `partial_settle` Permits Execution on Zero-Funded Escrows

**Filed as:** [GitHub issue #112](https://github.com/ushpraise/Starfund-contracts/issues/112)

**Category:** Bug  
**Priority:** High  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::partial_settle()` (lines 6715–6763)

**Problem**  
`partial_settle` transitions an open escrow (`status == 0`) to funded (`status == 1`) and captures `FundingCloseSnapshot`.
However, it does not check whether `escrow.funded_amount > 0`.

**Why it matters**  
An SME or admin can call `partial_settle` on an escrow with zero investor contributions. This writes a `FundingCloseSnapshot` with `total_principal = 0`. Subsequent calls to `withdraw()` or `compute_investor_payout()` encounter zero-principal division edge cases.

**Proposed solution**  
Add validation:
```rust
ensure(&env, escrow.funded_amount > 0, EscrowError::PartialSettleNoFunds);
```

**Acceptance criteria**  
- `partial_settle` reverts if `escrow.funded_amount == 0`.

**Testing**  
- Add test attempting `partial_settle` on a freshly initialized escrow.

---

## Issue #94 — `partial_settle` Lacks Operational Pause Check

**Filed as:** [GitHub issue #113](https://github.com/ushpraise/Starfund-contracts/issues/113)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::partial_settle()` (lines 6715–6763)

**Problem**  
`partial_settle` checks legal hold and dispute, but omits `guard_not_paused`.

**Why it matters**  
An operational pause should freeze all settlement and funding state transitions.

**Proposed solution**  
Add `guard_not_paused(&env, EscrowError::PausedBlocksSettlement, PauseEntry::Settlement);` to `partial_settle()`.

**Acceptance criteria**  
- `partial_settle` reverts when settlement operations are paused.

**Testing**  
- Add test pausing settlement and verifying `partial_settle` reverts.

---

## Issue #95 — `set_investors_allowlisted` (Batch) Never Saves Updated `AllowlistIndex` to Storage

**Filed as:** [GitHub issue #114](https://github.com/ushpraise/Starfund-contracts/issues/114)

**Category:** Bug  
**Priority:** Critical  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::set_investors_allowlisted()` (lines 5744–5782)

**Problem**  
In `StarfundEscrow::set_investors_allowlisted`:
Line 5744 loads the allowlist index:
```rust
let mut index: Vec<Address> = env.storage().instance().get(&DataKey::AllowlistIndex).unwrap_or_else(|| Vec::new(&env));
```
Inside the loop, `index.push_back(inv.clone())` or `index.remove(j)` is called.
However, at the end of the function (lines 5774–5782), **`env.storage().instance().set(&DataKey::AllowlistIndex, &index);` is never called!**
In contrast, the single-address variant `set_investor_allowlisted` correctly calls `.set(&DataKey::AllowlistIndex, &index)` at line 5701.

**Why it matters**  
Any address allowlisted via the batch function `set_investors_allowlisted` is never added to `AllowlistIndex` in storage. Consequently, `get_allowlisted_investors()` and `get_allowlisted_investors_count()` will return empty results or omit batch-allowlisted addresses entirely.

**Proposed solution**  
Add `env.storage().instance().set(&DataKey::AllowlistIndex, &index);` after the batch processing loop in `set_investors_allowlisted`.

**Acceptance criteria**  
- Batch-allowlisted addresses are persisted to `DataKey::AllowlistIndex`.
- `get_allowlisted_investors()` returns addresses added via `set_investors_allowlisted`.

**Testing**  
- Add test calling `set_investors_allowlisted` and asserting `client.get_allowlisted_investors_count() == batch_size`.

---

## Issue #96 — `get_allowlisted_investors_count` Performs Unbounded Persistent Storage Reads

**Filed as:** [GitHub issue #115](https://github.com/ushpraise/Starfund-contracts/issues/115)

**Category:** Performance / Gas  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::get_allowlisted_investors_count()` (lines 5835–5855)

**Problem**  
In `get_allowlisted_investors_count`:
```rust
let mut count: u32 = 0;
for i in 0..index.len() {
    let addr = index.get(i).unwrap();
    let is_al: bool = env.storage().persistent().get(&DataKey::InvestorAllowlisted(addr.clone())).unwrap_or(false);
    if is_al { count += 1; }
}
```
The function iterates over the entire `AllowlistIndex` and executes a persistent storage read for every address.

**Why it matters**  
If an escrow has hundreds of allowlisted investors, calling this view issues hundreds of storage read operations in a single invocation, easily exceeding Soroban transaction CPU and storage read limits.

**Proposed solution**  
Maintain an explicit `DataKey::AllowlistCount` counter in instance storage that increments when an address is allowlisted and decrements when revoked, allowing $O(1)$ reads.

**Acceptance criteria**  
- `get_allowlisted_investors_count` executes in $O(1)$ without looping over persistent storage.

**Testing**  
- Verify count remains accurate across multiple additions and revocations.

---

## Issue #97 — `AllowlistIndex` Stored in Instance Storage Risks Exceeding Soroban Ledger Size Limit

**Filed as:** [GitHub issue #116](https://github.com/ushpraise/Starfund-contracts/issues/116)

**Category:** Storage / Architecture  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `DataKey::AllowlistIndex` (lines 1354, 5686, 5747)

**Problem**  
`DataKey::AllowlistIndex` stores a `Vec<Address>` in `instance()` storage. In Soroban, an address is 32 bytes plus XDR overhead. A vector of hundreds of addresses stored in instance storage will exceed the 64KB ledger entry size limit.

**Why it matters**  
Once the 64KB instance storage limit is reached, any contract invocation that loads or updates instance storage will fail.

**Proposed solution**  
Migrate allowlist pagination to persistent storage pages or store allowlist indices under paginated keys `DataKey::AllowlistPage(u32)`.

**Acceptance criteria**  
- Instance storage does not contain unbounded address collections.

**Testing**  
- Add test benchmarking storage size with 500 allowlisted addresses.

---

## Issue #98 — `rotate_beneficiary` Lacks Operational Pause and Dispute Gates

**Filed as:** [GitHub issue #117](https://github.com/ushpraise/Starfund-contracts/issues/117)

**Category:** Bug / Security  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::rotate_beneficiary()` (lines 3615–3655)

**Problem**  
`rotate_beneficiary` verifies `guard_not_legal_hold`, but does not call `guard_not_paused` or `guard_not_disputed`.

**Why it matters**  
Beneficiary rotation redirects future SME disbursements. If an escrow is disputed or paused due to compromised credentials, rotating the beneficiary during the freeze undermines security controls.

**Proposed solution**  
Add:
```rust
guard_not_paused(&env, EscrowError::PausedBlocksBeneficiaryRotation, PauseEntry::Admin);
guard_not_disputed(&env, EscrowError::DisputeBlocksBeneficiaryRotation);
```

**Acceptance criteria**  
- `rotate_beneficiary` is blocked when paused or disputed.

**Testing**  
- Add tests verifying rotation reverts during active dispute or pause.

---

## Issue #99 — `rotate_payer` Lacks Operational Pause and Dispute Gates

**Filed as:** [GitHub issue #118](https://github.com/ushpraise/Starfund-contracts/issues/118)

**Category:** Bug / Security  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::rotate_payer()` (lines 3689–3722)

**Problem**  
`rotate_payer` verifies legal hold, but omits operational pause and dispute checks.

**Why it matters**  
Payer rotation changes the authorized settlement and repayment address. It must be frozen when a dispute or pause is active.

**Proposed solution**  
Add `guard_not_paused` and `guard_not_disputed` to `rotate_payer()`.

**Acceptance criteria**  
- `rotate_payer` is blocked when paused or disputed.

**Testing**  
- Test rotation reverts when dispute is open.

---

## Issue #100 — `rotate_payer` Allows Rotation After Funding Has Commenced

**Filed as:** [GitHub issue #119](https://github.com/ushpraise/Starfund-contracts/issues/119)

**Category:** Bug / Security  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::rotate_payer()` (lines 3694–3698)

**Problem**  
In `rotate_beneficiary`, line 3633 strictly requires:
`ensure(&env, escrow.funded_amount == 0, EscrowError::BeneficiaryImmutableAfterFunding);`
In `rotate_payer`, line 3694 only checks `escrow.status == 0 || escrow.status == 1`. It never checks `escrow.funded_amount == 0`.

**Why it matters**  
Allowing payer rotation after funding has begun allows changing the repayment counterparty mid-flight without investor consent.

**Proposed solution**  
Ensure `rotate_payer` requires `escrow.funded_amount == 0` or requires dual auth from both prior payer and SME.

**Acceptance criteria**  
- Payer cannot be arbitrarily rotated once funding is underway.

**Testing**  
- Add test attempting payer rotation on an escrow with non-zero funded amount.

---

## Issue #101 — Missing Non-Existent Paginated View Functions in `tests/paginated_views.rs`

**Filed as:** [GitHub issue #120](https://github.com/ushpraise/Starfund-contracts/issues/120)

**Category:** Documentation / Testing  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/tests/paginated_views.rs` (lines 1–4)

**Problem**  
The header of `tests/paginated_views.rs` claims:
`// Tests for the shared paginate_window helper and the public paginated read views:`
`//   get_investors, get_allowlisted_investors, get_revoked_attestation_digests,`
`//   get_collateral_records, get_pause_records, and get_settlement_records.`
Functions `get_collateral_records`, `get_pause_records`, and `get_settlement_records` do not exist in the contract.

**Why it matters**  
Misleads auditors into believing these paginated audit views exist.

**Proposed solution**  
Update the test file header to accurately list the existing views, or implement the missing paginated views.

**Acceptance criteria**  
- Test file header accurately reflects implemented contract functions.

---

## Issue #102 — Truncating Integer Division Causes Cumulative Rounding Residue in `compute_investor_payout`

**Filed as:** [GitHub issue #121](https://github.com/ushpraise/Starfund-contracts/issues/121)

**Category:** Bug / Math  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::compute_investor_payout()` (lines 7358–7374)

**Problem**  
In `compute_investor_payout`:
```rust
let coupon = total_principal
    .checked_mul(effective_yield_bps as i128).unwrap()
    .checked_div(10_000).unwrap();
let settle_pool = total_principal + coupon;
let payout = contribution.checked_mul(settle_pool).unwrap().checked_div(total_principal).unwrap();
```
Integer division truncates twice: first when computing `coupon = total_principal * bps / 10000`, and second when computing `payout = contribution * settle_pool / total_principal`.

**Why it matters**  
Truncating intermediate results prematurely exaggerates rounding loss against retail investors, leaving larger residual dust in the contract.

**Proposed solution**  
Compute payout with full 256-bit rational multiplication before dividing:
$$\text{payout} = \frac{\text{contribution} \times (10{,}000 + \text{effective\_yield\_bps})}{10{,}000}$$

**Acceptance criteria**  
- Single-division arithmetic reduces intermediate rounding loss.

**Testing**  
- Add property tests comparing intermediate vs single-division precision.

---

## Issue #103 — `get_funding_records` Returns 2-Tuple `Vec<(Address, i128)>` Instead of Typed Struct

**Filed as:** [GitHub issue #122](https://github.com/ushpraise/Starfund-contracts/issues/122)

**Category:** DevEx / API  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::get_funding_records()` (lines 4326–4348)

**Problem**  
`get_funding_records` returns `Vec<(Address, i128)>`. In Soroban contract interfaces, bare tuples in vectors have poorer client SDK code generation than named structs.

**Why it matters**  
Client SDKs (TypeScript / Python) generate awkward positional accessors (`entry[0]`, `entry[1]`) instead of self-documenting field names (`entry.investor`, `entry.contribution`).

**Proposed solution**  
Define a `#[contracttype]` struct:
```rust
#[contracttype]
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct FundingRecord {
    pub investor: Address,
    pub contribution: i128,
}
```
and return `Vec<FundingRecord>`.

**Acceptance criteria**  
- `get_funding_records` returns `Vec<FundingRecord>`.

**Testing**  
- Update pagination tests to assert on struct fields.

---

## Issue #104 — `get_funding_records` Returns Zero-Balance Entries for Unfunded Investors

**Filed as:** [GitHub issue #123](https://github.com/ushpraise/Starfund-contracts/issues/123)

**Category:** Bug  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::get_funding_records()` (lines 4341–4347)

**Problem**  
When an investor calls `unfund()` to withdraw all principal, their contribution becomes `0`. However, they remain in `InvestorIndex`. When `get_funding_records` is queried, it includes `(investor, 0)`.

**Why it matters**  
Callers expect `get_funding_records` to return active participants with positive contributions.

**Proposed solution**  
Filter out entries where `contribution == 0` or clean up `InvestorIndex` upon full unfund.

**Acceptance criteria**  
- `get_funding_records` only returns active investors with positive principal.

**Testing**  
- Test that an investor who unfunds to zero does not appear with contribution `0`.

---

## Issue #105 — `DistributedPrincipal` Accounting Mismatch Between `release()` and `withdraw()`

**Filed as:** [GitHub issue #124](https://github.com/ushpraise/Starfund-contracts/issues/124)

**Category:** Bug / Accounting  
**Priority:** High  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::release()` (lines 6920–6940)
- `StarfundEscrow::withdraw()` (lines 7101–7110)
- `StarfundEscrow::sweep_terminal_dust()` (lines 3514–3520)

**Problem**  
`DataKey::DistributedPrincipal` tracks how much principal has been disbursed.
In `withdraw()` (line 7108):
`DistributedPrincipal` is incremented by `amount = funded_amount - released_amount`.
In `release()`:
`DistributedPrincipal` is **never incremented**!
In `sweep_terminal_dust()` (line 3519):
`outstanding = escrow.funded_amount.saturating_sub(distributed);`

**Why it matters**  
If an escrow disburses principal via `release()`, `DistributedPrincipal` remains lower than the true disbursed amount. `sweep_terminal_dust` then falsely calculates that released funds are still "outstanding liability", causing valid dust sweeps to revert with `EscrowError::SweepExceedsLiabilityFloor`.

**Proposed solution**  
Increment `DataKey::DistributedPrincipal` atomically inside `release()` whenever principal is disbursed to the SME.

**Acceptance criteria**  
- `release()` updates `DataKey::DistributedPrincipal` by the released amount.
- `DistributedPrincipal` accurately equals total disbursed principal across both release paths.

**Testing**  
- Test calling `release()`, then verifying `client.get_distributed_principal() == released_amount`.

---

## Issue #106 — Missing Upper Bound Validation on `min_contribution_floor` in `raise_min_contribution_floor`

**Filed as:** [GitHub issue #125](https://github.com/ushpraise/Starfund-contracts/issues/125)

**Category:** Bug  
**Priority:** Medium  
**Suggested Complexity:** Trivial  

**Location:**
- `escrow/src/lib.rs`
- `raise_min_contribution_floor` (proposed admin setter)
- `init` (line 3166)

**Problem**  
At `init`, line 3166 enforces:
`ensure(&env, mc <= amount, EscrowError::MinContributionExceedsAmount);`
When the admin raises `min_contribution_floor`, there must be an upper bound check ensuring the new floor does not exceed `escrow.funding_target`.

**Why it matters**  
Setting `min_contribution_floor > funding_target` permanently prevents any investor from funding, because any contribution meeting the floor would exceed the funding target.

**Proposed solution**  
Ensure `new_floor <= escrow.funding_target` in the floor setter.

**Acceptance criteria**  
- Raising min contribution floor above `funding_target` reverts with `MinContributionExceedsAmount`.

**Testing**  
- Add test attempting to raise floor above funding target.

---

## Issue #107 — Missing Validation for `token_decimals` Against Live SEP-41 Token Metadata

**Filed as:** [GitHub issue #126](https://github.com/ushpraise/Starfund-contracts/issues/126)

**Category:** Enhancement / Safety  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/lib.rs`
- `StarfundEscrow::init()` (lines 3144–3148)
- `docs/ESCROW_TOKEN_INTEGRATION_CHECKLIST.md`

**Problem**  
`init()` accepts an optional `token_decimals: Option<u32>`. It persists this value directly without checking `TokenClient::new(&env, &funding_token).decimals()`.

**Why it matters**  
If an operator passes a mismatched decimal value (e.g. 6 instead of 7 for USDC), scale-dependent calculations or off-chain balance projections will miscalculate by orders of magnitude.

**Proposed solution**  
When `token_decimals` is provided, query `TokenClient::decimals()` and assert equality:
```rust
if let Some(dec) = token_decimals {
    let actual = TokenClient::new(&env, &funding_token).decimals();
    ensure(&env, dec == actual, EscrowError::TokenDecimalsMismatch);
}
```

**Acceptance criteria**  
- `init()` validates `token_decimals` against the token contract.

**Testing**  
- Test that initializing with wrong decimals reverts with `TokenDecimalsMismatch`.

---

## Issue #108 — `docs/escrow-ledger-time.md` References Non-Existent String Assert Messages

**Filed as:** [GitHub issue #127](https://github.com/ushpraise/Starfund-contracts/issues/127)

**Category:** Documentation  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `docs/escrow-ledger-time.md` (lines 29, 44)

**Problem**  
`docs/escrow-ledger-time.md` contains code snippets claiming the contract uses string assertion panics:
```rust
assert!(now >= escrow.maturity, "Escrow has not yet reached maturity");
assert!(now >= not_before, "Investor commitment lock not expired (ledger timestamp)");
```
The contract uses Soroban `ensure!(&env, ..., EscrowError::MaturityNotReached)` with typed error codes.

**Why it matters**  
Misleads integrators on error-handling mechanisms.

**Proposed solution**  
Update code snippets in `docs/escrow-ledger-time.md` to show actual `ensure!` and `EscrowError` usage.

**Acceptance criteria**  
- Documentation displays accurate typed error handling code snippets.

---

## Issue #109 — Cargo Workspace Configuration Excludes Tests From Coverage but CI Specifies `testutils`

**Filed as:** [GitHub issue #128](https://github.com/ushpraise/Starfund-contracts/issues/128)

**Category:** Tooling / CI  
**Priority:** Low  
**Suggested Complexity:** Trivial  

**Location:**
- `Cargo.toml` (lines 5–11)
- `.github/workflows/ci.yml` (lines 53–56)

**Problem**  
In root `Cargo.toml`:
```toml
[workspace.metadata.cargo-llvm-cov]
exclude = [
    "src/test/*",
    "**/*_test.rs",
    "tests/*"
]
```
In `.github/workflows/ci.yml`:
`cargo llvm-cov --features testutils --summary-only -p starfund_escrow`
The exclude pattern in `Cargo.toml` uses `tests/*` which does not match `escrow/src/tests/*`.

**Why it matters**  
Inconsistent exclusion paths lead to test scaffolding being counted in production code coverage metrics.

**Proposed solution**  
Update exclude patterns to `"escrow/src/tests/**"`.

**Acceptance criteria**  
- Coverage measurement only evaluates production contract code.

---

## Issue #110 — Missing Property Test for `unfund` Conservation and Unique Funder Count Monotonicity

**Filed as:** [GitHub issue #129](https://github.com/ushpraise/Starfund-contracts/issues/129)

**Category:** Testing  
**Priority:** Medium  
**Suggested Complexity:** Medium  

**Location:**
- `escrow/src/tests/properties.rs`

**Problem**  
The proptest suite in `escrow/src/tests/properties.rs` validates funding caps, fee splits, and release conservation, but has no property-based invariant test for arbitrary interleaved sequences of `fund` and `unfund` calls.

**Why it matters**  
Arbitrary user sequences of funding, partial unfunding, full unfunding, and re-funding are prime vectors for state desynchronization.

**Proposed solution**  
Add a stateful proptest in `properties.rs` asserting:
1. `escrow.funded_amount == sum(investor_contributions)`
2. `UniqueFunderCount == count(contributions > 0)`
3. Contract token balance equals `escrow.funded_amount` throughout all open-phase actions.

**Acceptance criteria**  
- Property test executes 100+ randomized iterations verifying conservation across fund/unfund operations.

**Testing**  
- Run `cargo test -p starfund_escrow --test properties`.

---

# Summary Table: Published Issues (#51 through #110)

| Wave # | GitHub Issue | Title | Category | Priority | Complexity | Primary Location |
|---|---|---|---|---|---|---|
| #51 | [#70](https://github.com/ushpraise/Starfund-contracts/issues/70) | State Corruption: `sweep_terminal_dust` unconditionally sets `status = 2` on Cancelled escrows | Bug / Security | Critical | Medium | `escrow/src/lib.rs:3536` |
| #52 | [#71](https://github.com/ushpraise/Starfund-contracts/issues/71) | `settle()` never writes `DataKey::SettledAt`, breaking `get_settled_at()` and settlement audits | Bug | High | Trivial | `escrow/src/lib.rs:6825–6830` |
| #53 | [#72](https://github.com/ushpraise/Starfund-contracts/issues/72) | `settle()` permits transition to Settled without verifying contract token balance covers `settle_pool` | Bug / Security | High | Medium | `escrow/src/lib.rs:6810–6830` |
| #54 | [#73](https://github.com/ushpraise/Starfund-contracts/issues/73) | Duplicate storage writes and redundant checks in `StarfundEscrow::init` | Refactor / Gas | Medium | Medium | `escrow/src/lib.rs:3131–3296` |
| #55 | [#74](https://github.com/ushpraise/Starfund-contracts/issues/74) | `init()` references non-existent error `EscrowError::FundingDeadlineBeyondMaturity` | Bug / Compile-Blocker | Critical | Trivial | `escrow/src/lib.rs:3217` |
| #56 | [#75](https://github.com/ushpraise/Starfund-contracts/issues/75) | `InvestorRefunded` stored in instance storage violates ADR-007 and risks instance storage exhaustion | Bug / Storage | High | Medium | `escrow/src/lib.rs:8168` |
| #57 | [#76](https://github.com/ushpraise/Starfund-contracts/issues/76) | `unfund()` emits misleading `OverWithdrawal` on non-positive amounts and misplaces validation | Bug | Low | Trivial | `escrow/src/lib.rs:8306–8308` |
| #58 | [#77](https://github.com/ushpraise/Starfund-contracts/issues/77) | `unfund()` to zero desynchronizes `InvestorIndex` and `UniqueFunderCount`, corrupting re-funding | Bug | High | Medium | `escrow/src/lib.rs:8319–8330` |
| #59 | [#78](https://github.com/ushpraise/Starfund-contracts/issues/78) | `bump_ttl()` panics on non-existent persistent keys and contains duplicate loops | Bug | High | Medium | `escrow/src/lib.rs:7779–7820` |
| #60 | [#79](https://github.com/ushpraise/Starfund-contracts/issues/79) | Missing implementation for `StarfundEscrow::batch_bump_ttl` (orphaned doc comment) | Bug / DevEx | Medium | Medium | `escrow/src/lib.rs:7822–7859` |
| #61 | [#80](https://github.com/ushpraise/Starfund-contracts/issues/80) | Unrealistic 1s/ledger assumptions in TTL constants exceed Soroban `max_entry_ttl` | Bug / Storage | High | Low | `escrow/src/lib.rs:494–496` |
| #62 | [#81](https://github.com/ushpraise/Starfund-contracts/issues/81) | Triplicate divergent dispute representations allow dispute check bypass in `close_escrow` | Bug / Security | High | Medium | `escrow/src/lib.rs:285, 3863, 3899` |
| #63 | [#82](https://github.com/ushpraise/Starfund-contracts/issues/82) | Missing events for `open_dispute` and `close_dispute` | Bug / Events | Medium | Low | `escrow/src/lib.rs:3878–3940` |
| #64 | [#83](https://github.com/ushpraise/Starfund-contracts/issues/83) | `close_dispute` silent early return when `resolved == false` | Bug | Low | Trivial | `escrow/src/lib.rs:3936–3939` |
| #65 | [#84](https://github.com/ushpraise/Starfund-contracts/issues/84) | `CloseError` enum discriminants collide with `EscrowError` on-chain | Bug | High | Low | `escrow/src/lib.rs:209–222` |
| #66 | [#85](https://github.com/ushpraise/Starfund-contracts/issues/85) | `request_clear_legal_hold` allows scheduling clear on non-existent legal holds | Bug | Medium | Trivial | `escrow/src/lib.rs:5574–5597` |
| #67 | [#86](https://github.com/ushpraise/Starfund-contracts/issues/86) | Redundant consecutive pause checks in `withdraw()` and `settle()` | Refactor / Gas | Low | Trivial | `escrow/src/lib.rs:7040–7050` |
| #68 | [#87](https://github.com/ushpraise/Starfund-contracts/issues/87) | `withdraw()` lacks `amount > 0` guard, permitting zero-payout execution and event | Bug | Medium | Trivial | `escrow/src/lib.rs:7058–7110` |
| #69 | [#88](https://github.com/ushpraise/Starfund-contracts/issues/88) | `settle_batch` requires target SME cross-contract signatures, failing batch settlements | Bug | High | Medium | `escrow/src/lib.rs:6871–6885` |
| #70 | [#89](https://github.com/ushpraise/Starfund-contracts/issues/89) | `set_storage_limit` lacks admin nonce replay protection and event emission | Bug / Security | Medium | Low | `escrow/src/lib.rs:7738–7750` |
| #71 | [#90](https://github.com/ushpraise/Starfund-contracts/issues/90) | `cancel_funding` lacks dispute gate, permitting cancellation during active dispute | Bug / Security | High | Trivial | `escrow/src/lib.rs:8108–8127` |
| #72 | [#91](https://github.com/ushpraise/Starfund-contracts/issues/91) | `cancel_funding` ignores operational pause gates | Bug | Medium | Trivial | `escrow/src/lib.rs:8108–8127` |
| #73 | [#92](https://github.com/ushpraise/Starfund-contracts/issues/92) | `refund_batch` passes `false` to `skip_zero_contribution`, breaking batch execution on zero balance | Bug | Medium | Trivial | `escrow/src/lib.rs:8250` |
| #74 | [#93](https://github.com/ushpraise/Starfund-contracts/issues/93) | `InvestorPayoutClaimed` event omits payout amount | Bug / Events | Medium | Trivial | `escrow/src/lib.rs:2385–2392` |
| #75 | [#94](https://github.com/ushpraise/Starfund-contracts/issues/94) | Economic discrepancy: `settle()` and `get_settlement_pool()` use base yield while `compute_investor_payout()` uses investor tiered yields | Bug / Security | High | High | `escrow/src/lib.rs:6810–6823, 7350–7374` |
| #76 | [#95](https://github.com/ushpraise/Starfund-contracts/issues/95) | Tuple destructuring compile error in `fund_impl` tier selection | Bug / Compile-Blocker | Critical | Trivial | `escrow/src/lib.rs:6542–6543` |
| #77 | [#96](https://github.com/ushpraise/Starfund-contracts/issues/96) | Inbound token transfer parameter naming in `transfer_funding_token_with_balance_checks` | Refactor / Code Quality | Low | Trivial | `escrow/src/external_calls.rs:106–112` |
| #78 | [#97](https://github.com/ushpraise/Starfund-contracts/issues/97) | Lack of self-transfer guard in `external_calls.rs` causes confusing delta errors | Bug | Low | Trivial | `escrow/src/external_calls.rs:113–145` |
| #79 | [#98](https://github.com/ushpraise/Starfund-contracts/issues/98) | Clean up commented-out test modules in `escrow/src/tests/mod.rs` | Testing / DevEx | Low | Trivial | `escrow/src/tests/mod.rs:64–66` |
| #80 | [#99](https://github.com/ushpraise/Starfund-contracts/issues/99) | Missing `collateral_pledge_key` function referenced in `escrow/src/keys.rs` docs | Documentation / DevEx | Low | Trivial | `escrow/src/keys.rs:13–17` |
| #81 | [#100](https://github.com/ushpraise/Starfund-contracts/issues/100) | `test_allowlist_tests.rs` fails compilation due to outdated `client.init` argument count (15 vs 19) | Testing / Compile-Blocker | Critical | Low | `escrow/src/test_allowlist_tests.rs:19–35` |
| #82 | [#101](https://github.com/ushpraise/Starfund-contracts/issues/101) | `release_budget_tests.rs` fails compilation due to missing `token_decimals` argument (18 vs 19) | Testing / Compile-Blocker | Critical | Low | `escrow/src/release_budget_tests.rs:54–73` |
| #83 | [#102](https://github.com/ushpraise/Starfund-contracts/issues/102) | `tests/dispute_release.rs` asserts invalid state transition (`withdraw()` at status 0) | Testing / Bug | Medium | Low | `escrow/src/tests/dispute_release.rs:35–43` |
| #84 | [#103](https://github.com/ushpraise/Starfund-contracts/issues/103) | OpenAPI test suite in `docs/tests/openapi.test.js` is not executed in CI | CI/CD | Medium | Low | `.github/workflows/ci.yml` |
| #85 | [#104](https://github.com/ushpraise/Starfund-contracts/issues/104) | Severe drift between `docs/openapi.yaml` and smart contract data models | Documentation | Medium | Medium | `docs/openapi.yaml:67–95` |
| #86 | [#105](https://github.com/ushpraise/Starfund-contracts/issues/105) | `docs/EVENT_SCHEMA.md` falsely documents trailing `version` topic for all events | Documentation | Medium | Low | `docs/EVENT_SCHEMA.md:34–38` |
| #87 | [#106](https://github.com/ushpraise/Starfund-contracts/issues/106) | `AdminTransferredEvent` defined in `lib.rs` and documented in `EVENT_SCHEMA.md` is never emitted | Bug / Events | Medium | Low | `escrow/src/lib.rs:2126, 7955` |
| #88 | [#107](https://github.com/ushpraise/Starfund-contracts/issues/107) | `update_maturity` lacks lower bound check against `funding_deadline` | Bug | High | Low | `escrow/src/lib.rs:7453–7485` |
| #89 | [#108](https://github.com/ushpraise/Starfund-contracts/issues/108) | `update_maturity` lacks admin nonce replay protection | Security | Medium | Low | `escrow/src/lib.rs:7453–7485` |
| #90 | [#109](https://github.com/ushpraise/Starfund-contracts/issues/109) | `update_yield_bps` fails to enforce documented zero-funded invariant | Bug / Security | High | Low | `escrow/src/lib.rs:7487–7543` |
| #91 | [#110](https://github.com/ushpraise/Starfund-contracts/issues/110) | `update_funding_target` lacks upper bound check against `MAX_INVOICE_AMOUNT` | Bug | Medium | Low | `escrow/src/lib.rs:5862–5910` |
| #92 | [#111](https://github.com/ushpraise/Starfund-contracts/issues/111) | `update_funding_target` omits `FundingStateChanged` event on promotion to funded | Bug / Events | Low | Trivial | `escrow/src/lib.rs:5880–5908` |
| #93 | [#112](https://github.com/ushpraise/Starfund-contracts/issues/112) | `partial_settle` permits execution on zero-funded escrows | Bug | High | Trivial | `escrow/src/lib.rs:6715–6763` |
| #94 | [#113](https://github.com/ushpraise/Starfund-contracts/issues/113) | `partial_settle` lacks operational pause check | Bug | Medium | Trivial | `escrow/src/lib.rs:6715–6763` |
| #95 | [#114](https://github.com/ushpraise/Starfund-contracts/issues/114) | `set_investors_allowlisted` (batch) never saves updated `AllowlistIndex` to storage | Bug | Critical | Trivial | `escrow/src/lib.rs:5744–5782` |
| #96 | [#115](https://github.com/ushpraise/Starfund-contracts/issues/115) | `get_allowlisted_investors_count` performs unbounded persistent storage reads | Performance / Gas | High | Medium | `escrow/src/lib.rs:5835–5855` |
| #97 | [#116](https://github.com/ushpraise/Starfund-contracts/issues/116) | `AllowlistIndex` stored in instance storage risks exceeding Soroban ledger size limit | Storage / Architecture | High | Medium | `escrow/src/lib.rs:5686, 5747` |
| #98 | [#117](https://github.com/ushpraise/Starfund-contracts/issues/117) | `rotate_beneficiary` lacks operational pause and dispute gates | Bug / Security | Medium | Trivial | `escrow/src/lib.rs:3615–3655` |
| #99 | [#118](https://github.com/ushpraise/Starfund-contracts/issues/118) | `rotate_payer` lacks operational pause and dispute gates | Bug / Security | Medium | Trivial | `escrow/src/lib.rs:3689–3722` |
| #100 | [#119](https://github.com/ushpraise/Starfund-contracts/issues/119) | `rotate_payer` allows rotation after funding has commenced | Bug / Security | Medium | Trivial | `escrow/src/lib.rs:3694–3698` |
| #101 | [#120](https://github.com/ushpraise/Starfund-contracts/issues/120) | Missing non-existent paginated view functions in `tests/paginated_views.rs` | Documentation / Testing | Low | Trivial | `escrow/src/tests/paginated_views.rs:1–4` |
| #102 | [#121](https://github.com/ushpraise/Starfund-contracts/issues/121) | Truncating integer division causes cumulative rounding residue in `compute_investor_payout` | Bug / Math | Medium | Medium | `escrow/src/lib.rs:7358–7374` |
| #103 | [#122](https://github.com/ushpraise/Starfund-contracts/issues/122) | `get_funding_records` returns 2-tuple `Vec<(Address, i128)>` instead of typed struct | DevEx / API | Low | Trivial | `escrow/src/lib.rs:4326–4348` |
| #104 | [#123](https://github.com/ushpraise/Starfund-contracts/issues/123) | `get_funding_records` returns zero-balance entries for unfunded investors | Bug | Low | Trivial | `escrow/src/lib.rs:4341–4347` |
| #105 | [#124](https://github.com/ushpraise/Starfund-contracts/issues/124) | `DistributedPrincipal` accounting mismatch between `release()` and `withdraw()` | Bug / Accounting | High | Medium | `escrow/src/lib.rs:6920–6940, 7108` |
| #106 | [#125](https://github.com/ushpraise/Starfund-contracts/issues/125) | Missing upper bound validation on `min_contribution_floor` in `raise_min_contribution_floor` | Bug | Medium | Trivial | `escrow/src/lib.rs:3166` |
| #107 | [#126](https://github.com/ushpraise/Starfund-contracts/issues/126) | Missing validation for `token_decimals` against live SEP-41 token metadata | Enhancement / Safety | Medium | Medium | `escrow/src/lib.rs:3144–3148` |
| #108 | [#127](https://github.com/ushpraise/Starfund-contracts/issues/127) | `docs/escrow-ledger-time.md` references non-existent string assert messages | Documentation | Low | Trivial | `docs/escrow-ledger-time.md:29, 44` |
| #109 | [#128](https://github.com/ushpraise/Starfund-contracts/issues/128) | Cargo workspace configuration excludes tests from coverage but CI specifies `testutils` | Tooling / CI | Low | Trivial | `Cargo.toml:5–11`, `.github/workflows/ci.yml:53–56` |
| #110 | [#129](https://github.com/ushpraise/Starfund-contracts/issues/129) | Missing property test for `unfund` conservation and unique funder count monotonicity | Testing | Medium | Medium | `escrow/src/tests/properties.rs` |
