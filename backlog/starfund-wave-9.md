# StarFund Contracts — Backlog Wave 9

Numbering starts at **#1**. Starting index was determined by checking GitHub for the highest
existing issue/PR number on `ushpraise/Starfund-contracts` via both `gh issue list --state all`
and `gh pr list --state all` (issues and PRs share one counter) and the equivalent REST endpoint
(`GET /repos/ushpraise/Starfund-contracts/issues?state=all&sort=created&direction=desc&per_page=1`).
All three returned an empty result — **no issues or PRs exist yet on this repo** — so numbering
starts fresh at #1.

**Read this first:** issues #1–#19 (Bug Fixes) describe why `cargo build -p starfund_escrow`
currently fails with **104 compiler errors**. Nothing else in this backlog can be verified by CI
until at least #1–#8 land, since the crate does not compile today. This was confirmed by
installing a local Rust toolchain and running `cargo clippy -p starfund_escrow --all-targets -- -D warnings`
from a clean `main` checkout.

---

## Bug Fixes

## #1: `DataKey::PauseState` and `EscrowError::PauseScopeMismatch` are missing — the scoped-pause feature does not compile

**Filed as:** [GitHub issue #4](https://github.com/ushpraise/Starfund-contracts/issues/4)

**Category:** Bug · **Size:** M

**Context:** `lib.rs` defines a full scoped-pause system — `PauseScope` enum (line ~1431),
`PauseState` struct (line ~1499), `StarfundEscrow::set_paused` (writes `PauseState` at line ~5285),
`StarfundEscrow::get_pause_state` (reads it at line ~2858) — that stores its state under
`DataKey::PauseState`. That variant does not exist on the `DataKey` enum (`escrow/src/lib.rs:1247`),
so every one of the ~16 references to it fails with `E0599: no variant ... named 'PauseState' found
for enum 'DataKey'`. The companion error `EscrowError::PauseScopeMismatch` (referenced 3 times,
e.g. at the scope-mismatch check in `set_paused`) is likewise never defined on `EscrowError`
(`escrow/src/lib.rs:578`).

**Files/functions:** `escrow/src/lib.rs` — `DataKey` enum, `EscrowError` enum, `PauseState`,
`PauseScope`, `set_paused`, `get_pause_state`, `paused_active`.

**Acceptance criteria:**
- Add `PauseState` as a new (appended, per ADR-007 Rule 6) `DataKey` variant.
- Add `PauseScopeMismatch` as a new (appended) `EscrowError` variant with a fresh, non-colliding
  discriminant (cross-check against #8 before picking a number).
- `cargo build -p starfund_escrow` succeeds through this portion of the enum.
- `docs/adr/ADR-007-storage-key-evolution.md` Rule 1 (additive-key policy) is satisfied: read with
  `.unwrap_or(default)`, no existing entrypoint semantics change.

---

## #2: `DataKey::ReleasedAmount` and `DataKey::AdminNonce` are missing — `release()` and admin-nonce replay protection don't compile

**Filed as:** [GitHub issue #3](https://github.com/ushpraise/Starfund-contracts/issues/3)

**Category:** Bug · **Size:** S

**Context:** `escrow/src/keys.rs::released_amount()` returns `DataKey::ReleasedAmount`, and it's the
only key `StarfundEscrow::release` (`escrow/src/lib.rs:6900`) uses to track cumulative SME
disbursement. `DataKey::AdminNonce` (5 references) backs `consume_admin_nonce` /
`get_admin_nonce` (`escrow/src/lib.rs:3777`, `:3829`), which every dual-auth and nonce-gated
entrypoint calls (`set_legal_hold`, `set_allowlist_active`, `set_investor_allowlisted(s)`,
`rotate_beneficiary`, `request_clear_legal_hold`). Neither variant exists on `DataKey`.

**Files/functions:** `escrow/src/lib.rs` `DataKey` enum; `escrow/src/keys.rs::released_amount`;
`consume_admin_nonce`, `get_admin_nonce`.

**Acceptance criteria:**
- Add `ReleasedAmount` and `AdminNonce` as new appended `DataKey` variants.
- `release()` and every nonce-consuming entrypoint compile and pass their existing tests once
  re-enabled.

---

## #3: `EscrowError` is missing 5 variants required by `release()`

**Filed as:** [GitHub issue #5](https://github.com/ushpraise/Starfund-contracts/issues/5)

**Category:** Bug · **Size:** S

**Context:** `StarfundEscrow::release` (`escrow/src/lib.rs:6900-6990`) references
`EscrowError::ReleaseAmountNotPositive`, `PausedBlocksRelease`, `LegalHoldBlocksRelease`,
`ReleaseNotFunded`, and `ReleaseExceedsRemaining` — none of which are defined on the enum
(`escrow/src/lib.rs:578`).

**Files/functions:** `escrow/src/lib.rs::release`, `EscrowError` enum.

**Acceptance criteria:**
- Add all 5 variants as new appended discriminants (pick values that don't collide — see #8).
- `release()` compiles; add/re-enable a happy-path and a per-error negative test for each variant.

---

## #4: `EscrowError` is missing 3 variants required by `partial_settle()`

**Filed as:** [GitHub issue #6](https://github.com/ushpraise/Starfund-contracts/issues/6)

**Category:** Bug · **Size:** S

**Context:** `StarfundEscrow::partial_settle` (`escrow/src/lib.rs:6715`) references
`EscrowError::PartialSettleNotOpen`, `PartialSettleUnauthorizedCaller`, and
`LegalHoldBlocksPartialSettle`. None exist on `EscrowError`. (`DisputeBlocksPartialSettle` already
exists at discriminant 240 but collides with `CallbackWrongOrigin` — see #8.)

**Files/functions:** `escrow/src/lib.rs::partial_settle`, `EscrowError` enum.

**Acceptance criteria:**
- Add the 3 missing variants with fresh discriminants.
- `partial_settle()` compiles; negative-auth and status-guard tests exist for each new variant.

---

## #5: `EscrowError` is missing 3 variants required by `rotate_payer()`

**Filed as:** [GitHub issue #7](https://github.com/ushpraise/Starfund-contracts/issues/7)

**Category:** Bug · **Size:** S

**Context:** `StarfundEscrow::rotate_payer` (`escrow/src/lib.rs:3689`) references
`EscrowError::NewPayerSameAsCurrent`, `PayerRotationNotOpen`, and `LegalHoldBlocksPayerRotation`
in its own rustdoc error table, none of which exist on `EscrowError`.

**Files/functions:** `escrow/src/lib.rs::rotate_payer`, `EscrowError` enum.

**Acceptance criteria:**
- Add the 3 missing variants with fresh discriminants.
- `rotate_payer()` compiles. See also #46/#50 (security) for the auth-model gaps in this same
  function — coordinate discriminant choice with whatever nonce parameter that work adds.

---

## #6: `EscrowError` is missing variants for the four monotonic raise/lower admin-config entrypoints

**Filed as:** [GitHub issue #8](https://github.com/ushpraise/Starfund-contracts/issues/8)

**Category:** Bug · **Size:** L

**Context:** Four entrypoints reference `EscrowError` variants that don't exist:
- `lower_min_contribution_floor` (`escrow/src/lib.rs:6090`) → `NewFloorNotLower`, `NewFloorNotPositive`
- `raise_max_per_investor` (`:6139`) → `MaxPerInvestorCapNotConfigured`, `MaxPerInvestorCapNotRaised`
- `extend_funding_deadline` (`:7565`) → `FundingDeadlineNotExtended`
- `raise_maturity_max_horizon` (`:7684`) → `HorizonNotRaised`

All four share the same shape of bug (a monotonic-adjustment guard whose error variant was never
added to the enum), which is why they're grouped, but each needs its own test coverage since the
guarded invariant differs per entrypoint.

**Files/functions:** `escrow/src/lib.rs` — the four functions above, `EscrowError` enum.

**Acceptance criteria:**
- Add all 6 variants with fresh discriminants.
- Each of the 4 entrypoints compiles and has a positive test (successful raise/lower) and a
  negative test (rejected no-op / wrong-direction adjustment).

---

## #7: `EscrowError` is missing `CollateralBatchEmpty`/`CollateralBatchTooLarge` for `batch_record_collateral`

**Filed as:** [GitHub issue #9](https://github.com/ushpraise/Starfund-contracts/issues/9)

**Category:** Bug · **Size:** S

**Context:** The batch collateral-recording entrypoint referenced by
`docs/escrow-security-checklist.md` §1 (`record_sme_collateral_commitment_batch` /
`MAX_COLLATERAL_BATCH`) needs `EscrowError::CollateralBatchEmpty` and `CollateralBatchTooLarge`,
neither of which is defined.

**Files/functions:** `escrow/src/lib.rs` — batch collateral entrypoint, `EscrowError` enum,
`MAX_COLLATERAL_BATCH` (line ~456).

**Acceptance criteria:**
- Add both variants with fresh discriminants.
- Add empty-batch and over-`MAX_COLLATERAL_BATCH` negative tests.

---

## #8: `EscrowError` enum integrity: 1 duplicate variant name + 8 duplicate discriminant values

**Filed as:** [GitHub issue #2](https://github.com/ushpraise/Starfund-contracts/issues/2)

**Category:** Bug · **Size:** L

**Context:** Independent of the *missing* variants in #1–#7, the `EscrowError` enum as currently
written has real duplication that breaks compilation on its own:

- **Duplicate variant name** (`E0428`): `AttestationNotRevoked` is defined twice, at
  `escrow/src/lib.rs:662` (`= 56`) and `:835` (`= 168`). Rust rejects two variants with the same
  identifier regardless of value — this alone is a hard compile error.
- **Duplicate discriminant values** (`E0081`, 8 collisions): `MaturityUnchanged`/`NoPendingAdmin`
  (both `81`), `AdminProposalExpired`/`AdminNonceMismatch` (both `85`),
  `NewCapNotHigher`/`InboundRecipientBalanceDeltaMismatch` (both `176`), and a 6-way collision
  across the `Dispute*` family and the `Callback*`/`Registry`/`Beneficiary`/`AdminRecovery` family
  spanning `240`–`247` (`DisputeBlocksPartialSettle`/`CallbackWrongOrigin` = 240,
  `DisputeBlocksRefund`/`CallbackWrongNonce` = 241, `DisputeBlocksUnfund`/`CallbackWrongPhase` = 242,
  `DisputeBlocksSweep`/`CallbackReplayed` = 243, `DisputeOpenUnauthorized`/`CallbackAfterCancellation`
  = 244, `DisputeCloseUnauthorized`/`CallbackNotFound` = 245, `DisputeAlreadyOpen`/
  `RegistryImmutableAfterFunding` = 246, `DisputeNotOpen`/`BeneficiaryImmutableAfterFunding` = 247).

Per `EscrowError`'s own doc comment ("Codes are append-only: never reuse or renumber a variant"),
the fix must **renumber one side of each collision to an unused value**, never repurpose a
colliding number for something else. Coordinate with #1–#7, which are adding new variants to the
same enum in the same PR wave.

**Files/functions:** `escrow/src/lib.rs::EscrowError` (the whole enum, lines ~578–977).

**Acceptance criteria:**
- Rename the second `AttestationNotRevoked` definition (line 835) to a distinct name (it guards a
  different call site — `revoke_attestation_digest` on a non-revoked index — than the original at
  line 662, which guards `unrevoke_attestation_digest`; do not merge them, they mean different
  things).
- Renumber one variant in each of the 8 colliding pairs to a value not used anywhere else in the
  enum (write a small script or test asserting all discriminants are unique — see #33).
- No existing, still-valid variant's numeric code changes (append-only policy).
- `cargo build -p starfund_escrow` compiles past the `EscrowError` definition.

---

## #9: Duplicate `MAX_INVESTOR_ALLOWLIST_BATCH` constant definition

**Filed as:** [GitHub issue #11](https://github.com/ushpraise/Starfund-contracts/issues/11)

**Category:** Bug · **Size:** S

**Context:** `pub const MAX_INVESTOR_ALLOWLIST_BATCH: u32 = 32;` is defined twice, at
`escrow/src/lib.rs:162` and again at `:407`, both with the same value. `E0428: the name
'MAX_INVESTOR_ALLOWLIST_BATCH' is defined multiple times`.

**Files/functions:** `escrow/src/lib.rs`.

**Acceptance criteria:** Delete the redundant definition at line 407 (or 162 — whichever is
determined to be in the more logical const-grouping location); keep exactly one.

---

## #10: Duplicate `AdminProposalCancelled` event struct (byte-identical copy-paste)

**Filed as:** [GitHub issue #12](https://github.com/ushpraise/Starfund-contracts/issues/12)

**Category:** Bug · **Size:** S

**Context:** `#[contractevent] pub struct AdminProposalCancelled { ... }` is defined twice, at
`escrow/src/lib.rs:2042` and `:2178`, with identical fields (`name`, `invoice_id` topics,
`cancelled_pending`). Unlike #11, the two copies are structurally identical — this is pure
copy-paste, not a divergent-implementation conflict. `docs/escrow-events.md` documents exactly one
`AdminProposalCancelled` shape, matching both copies.

**Files/functions:** `escrow/src/lib.rs`, `docs/escrow-events.md`.

**Acceptance criteria:** Delete one of the two identical struct definitions; confirm
`StarfundEscrow::cancel_pending_admin` still compiles and its existing test(s) pass unchanged.

---

## #11: Duplicate `CollateralClearedEvt` struct and two divergent `clear_sme_collateral_commitment` implementations

**Filed as:** [GitHub issue #13](https://github.com/ushpraise/Starfund-contracts/issues/13)

**Category:** Bug · **Size:** M

**Context:** Both `CollateralClearedEvt` (defined at `escrow/src/lib.rs:2358` and again at `:2598`)
and `clear_sme_collateral_commitment` (defined at `:3549` and again at `:4483`) exist twice — but
unlike #10, these two copies **genuinely diverge**:
- `:3549` returns `Result<(), EscrowError>`, uses `ok_or`/`?`, and publishes a 2-field
  `CollateralClearedEvt { invoice_id, amount }` with **no topic**.
- `:4483` returns `()` (panics via `fail()`), and publishes a 5-field
  `CollateralClearedEvt { name: symbol_short!("coll_clr"), invoice_id, asset, amount, recorded_at }`
  **with** a `coll_clr` topic.

`docs/escrow-events.md` §`CollateralClearedEvt` documents the `coll_clr` topic and the richer
field set — i.e. it documents the `:4483` (panic-style) implementation, not the `:3549`
(`Result`-style) one. That's a strong signal the `:4483` version is canonical and `:3549` is the
stale duplicate, but this needs a maintainer decision, not a guess, since deleting the wrong one
removes a `Result<>`-returning API surface that some caller might expect.

**Files/functions:** `escrow/src/lib.rs`, `docs/escrow-events.md`.

**Acceptance criteria:**
- Maintainer confirms which implementation is canonical (docs point to `:4483`).
- Delete the other `clear_sme_collateral_commitment` and the other `CollateralClearedEvt`.
- Existing/re-enabled tests exercise the surviving implementation's exact error and event shape.

---

## #12: `external_calls::transfer_into_escrow_with_balance_checks` doesn't exist — call-site name mismatch

**Filed as:** [GitHub issue #14](https://github.com/ushpraise/Starfund-contracts/issues/14)

**Category:** Bug · **Size:** S

**Context:** `escrow/src/lib.rs:6677` calls
`external_calls::transfer_into_escrow_with_balance_checks(...)`, but `external_calls.rs` only
defines `transfer_funding_token_inbound_with_balance_checks` (`escrow/src/external_calls.rs:169`).
`E0425: cannot find function 'transfer_into_escrow_with_balance_checks' in module 'external_calls'`.

**Files/functions:** `escrow/src/lib.rs:6677`, `escrow/src/external_calls.rs:169`.

**Acceptance criteria:** Fix the call site to use the real function name (or rename the function if
the caller's name is actually the intended public name — pick one, don't leave both). Confirm the
argument order/types match `transfer_funding_token_inbound_with_balance_checks`'s signature
exactly (`env, token_addr, investor, to, amount`).

---

## #13: Missing `CallbackRegisteredEvent`, `CallbackExecutedEvent`, and `FundingDeadlineUpdated` event struct definitions

**Filed as:** [GitHub issue #15](https://github.com/ushpraise/Starfund-contracts/issues/15)

**Category:** Bug · **Size:** M

**Context:** Three `#[contractevent]` structs are published but never defined:
- `register_callback` (`escrow/src/lib.rs:8483`) publishes `CallbackRegisteredEvent`.
- `execute_callback` (`:8563`) publishes `CallbackExecutedEvent`.
- `update_funding_deadline` (`:5966`) publishes `FundingDeadlineUpdated`.

The first two **are already documented** in `docs/escrow-events.md` (§`CallbackRegisteredEvent`
line 291, §`CallbackExecutedEvent` line 303) with their intended topic/field shape — the docs are
correct and ahead of the code; only the struct definitions are missing from `lib.rs`.
`FundingDeadlineUpdated` is not documented anywhere (grep of `docs/` and `.kiro/` returns zero
hits) and is a different event from the already-implemented, already-documented
`FundingDeadlineExtended` (`escrow/src/lib.rs:2229`, published by the separate
`extend_funding_deadline` entrypoint at `:7565`) — confirm with the maintainer whether
`update_funding_deadline` and `extend_funding_deadline` are supposed to be two distinct
entrypoints/events or the former is dead code that should be removed instead.

**Files/functions:** `escrow/src/lib.rs`, `docs/escrow-events.md`.

**Acceptance criteria:**
- Define `CallbackRegisteredEvent`/`CallbackExecutedEvent` matching `docs/escrow-events.md`'s
  documented field/topic shape exactly.
- Resolve `update_funding_deadline` vs `extend_funding_deadline` duplication (either document and
  implement `FundingDeadlineUpdated` properly, or remove `update_funding_deadline` in favor of the
  already-working `extend_funding_deadline`).

---

## #14: `set_investor_allowlisted` references undefined local `was_allowlisted`

**Filed as:** [GitHub issue #16](https://github.com/ushpraise/Starfund-contracts/issues/16)

**Category:** Bug · **Size:** S

**Context:** `escrow/src/lib.rs:5689` and `:5691` use `was_allowlisted` to decide whether to
push/remove the address from `DataKey::AllowlistIndex`, but no such binding exists anywhere in
`set_investor_allowlisted` (`escrow/src/lib.rs:5675`). `E0425: cannot find value 'was_allowlisted'
in this scope`.

**Files/functions:** `escrow/src/lib.rs::set_investor_allowlisted`.

**Acceptance criteria:** Add `let was_allowlisted = Self::is_investor_allowlisted(env.clone(),
investor.clone());` (read **before** the persistent-storage write that follows it) so the index
maintenance logic at lines 5689/5691 has the previous state to compare against. Add a test
covering: allowlisting a not-yet-indexed address (pushes), re-allowlisting an already-indexed
address (no duplicate push), and de-allowlisting (removes from index).

---

## #15: `fund_impl`'s tiered-commitment branch references undefined `res`/`resolution`

**Filed as:** [GitHub issue #17](https://github.com/ushpraise/Starfund-contracts/issues/17)

**Category:** Bug · **Size:** M

**Context:** Inside `fund_impl`'s `else` branch (the `fund_with_commitment` first-deposit path,
`escrow/src/lib.rs:6605-6623`), the code trails off with a bare `res` expression (line 6620) with
no prior binding, then immediately uses `resolution.effective_yield_bps` / `.matched_lock_secs`
(lines 6622-6623) — also never bound. This is the exact tier-selection logic documented in
`StarfundEscrow::preview_yield_tier` (`:4463`, which correctly calls
`Self::effective_yield_for_commitment(&env, escrow.yield_bps, lock)`), suggesting the equivalent
call inside `fund_impl` was accidentally deleted, leaving only its two downstream uses.

**Files/functions:** `escrow/src/lib.rs::fund_impl` (the `else` branch around lines 6605-6623),
compare against `preview_yield_tier` (`:4463`).

**Acceptance criteria:**
- Reconstruct the missing `let resolution = Self::effective_yield_for_commitment(&env,
  escrow.yield_bps, committed_lock_secs);` call (or equivalent) so `res`/`resolution` resolve.
- Add a test asserting `fund_with_commitment`'s first-deposit tier selection matches what
  `preview_yield_tier` would have previewed for the same `(amount, lock)` pair — this is exactly
  the kind of divergence this bug could otherwise reintroduce silently.

---

## #16: `propose_admin` and `transfer_admin` reference undefined locals (`validity_window_secs`, `invoice_id`)

**Filed as:** [GitHub issue #18](https://github.com/ushpraise/Starfund-contracts/issues/18)

**Category:** Bug · **Size:** S

**Context:** Two related, small breaks in the admin-handover code:
- `propose_admin(env: Env, new_admin: Address, expected_nonce: u32) -> Address`
  (`escrow/src/lib.rs:7859`) has no `validity_window_secs` parameter, but its body at line 7886
  reads `validity_window_secs.unwrap_or(DEFAULT_ADMIN_PROPOSAL_VALIDITY_SECS)` as if it were an
  `Option<u64>` parameter.
- `transfer_admin` (the `#[deprecated]` wrapper at `:7977`) already binds `let escrow =
  Self::get_escrow(env.clone());` at line 7985, but line 7989 publishes
  `DeprecatedTransferAdminUsed { invoice_id, ... }` using a bare `invoice_id` instead of
  `escrow.invoice_id.clone()`.

**Files/functions:** `escrow/src/lib.rs::propose_admin`, `::transfer_admin`.

**Acceptance criteria:**
- Add `validity_window_secs: Option<u64>` to `propose_admin`'s signature (matching the doc comment
  on `DEFAULT_ADMIN_PROPOSAL_VALIDITY_SECS` at line ~469, which already describes this parameter).
  Update the `#[deprecated]` `transfer_admin` call site accordingly.
- Fix `transfer_admin`'s event publish to use `escrow.invoice_id.clone()`.
- `cargo build` passes; re-enable/add a test for a non-default validity window.

---

## #17: `escrow/src/tests/coverage.rs` has an unclosed-delimiter brace mismatch

**Filed as:** [GitHub issue #1](https://github.com/ushpraise/Starfund-contracts/issues/1)

**Category:** Bug · **Size:** M

**Context:** `rustc` reports `error: this file contains an unclosed delimiter` rooted at
`typed_error_codes_cover_basic_escrow_guards` (starts `escrow/src/tests/coverage.rs:29`), with the
file's final `}` at line 4254 unable to close it — a missing (or extra) brace somewhere inside that
function swallows the rest of the 4254-line file into a single function body, which is why `cargo
test` currently reports only 11 errors for this file (it can't get far enough to report the real
count).

**Files/functions:** `escrow/src/tests/coverage.rs` (whole file, root cause inside
`typed_error_codes_cover_basic_escrow_guards`).

**Acceptance criteria:** Locate and fix the actual brace mismatch (likely bisectable by
temporarily truncating the file and reintroducing sections). `cargo test -p starfund_escrow`
parses `coverage.rs` as multiple distinct test functions again. This must land before any of the 6
`#[ignore]`d tests in this file (see #33) can be individually triaged.

---

## #18: `release()` has a redundant duplicate pause check and raw `.unwrap()`s instead of typed errors

**Filed as:** [GitHub issue #10](https://github.com/ushpraise/Starfund-contracts/issues/10)

**Category:** Bug · **Size:** S

**Context:** In `StarfundEscrow::release` (`escrow/src/lib.rs:6900-6923`):
- Lines 6903-6908 check the same condition twice: an inline
  `ensure(&env, !Self::paused_active(&env), EscrowError::PausedBlocksRelease)` immediately followed
  by `guard_not_paused(&env, EscrowError::PausedBlocksRelease)`. Harmless but dead weight, and a
  sign the function was assembled from two different snippets.
- Line 6912: `env.storage().instance().get(&DataKey::Escrow).unwrap()` panics with a generic
  message instead of the `EscrowError::EscrowNotInitialized` pattern every other entrypoint uses
  (`.unwrap_or_else(|| panic_with_error!(&env, EscrowError::EscrowNotInitialized))`).
- Line 6923: `escrow.funded_amount.checked_sub(released_amount).unwrap()` discards the checked
  result via `.unwrap()` instead of `.unwrap_or_else(|| fail(&env, EscrowError::...))`, so a
  theoretical underflow panics without a stable numeric error code — contradicting the crate's own
  documented policy on `EscrowError` ("client SDKs should branch on the numeric code rather than
  legacy panic strings").

**Files/functions:** `escrow/src/lib.rs::release`.

**Acceptance criteria:**
- Remove the redundant inline pause check; keep the `guard_not_paused` helper call (matches every
  other entrypoint's style).
- Replace both raw `.unwrap()`s with typed-error equivalents consistent with the rest of the file.
- Existing/new tests confirm the typed error codes, not panic message substrings.

---

## #19: `FeeSchedule` subsystem is fully implemented but never consulted by `withdraw()` or `release()`

**Filed as:** [GitHub issue #19](https://github.com/ushpraise/Starfund-contracts/issues/19)

**Category:** Bug · **Size:** M

**Context:** `FeeSchedule`/`FeeScheduleStorageKey`/`FeeScheduleError` (`escrow/src/lib.rs:339-368`)
back a complete admin-gated subsystem — `submit_fee_schedule` (`:2667`), `activate_fee_schedule`
(`:2709`), `get_active_fee_schedule`/`get_pending_fee_schedule`/`get_previous_fee_schedule`
(`:2736`, `:2750`, `:2762`) — that reads and writes `FeeScheduleStorageKey::{Active,Pending,Previous}`
exclusively within its own functions. Neither `withdraw()` (which applies the **separate**,
immutable, init-time `DataKey::ProtocolFeeBps` per the module rustdoc's "Immutable protocol fee"
section) nor `release()` (which applies **no** fee at all — see #46) ever reads
`FeeScheduleStorageKey::Active`. The subsystem compiles, is fully testable, and returns real data
from `get_active_fee_schedule()` — but activating a schedule has **zero effect** on any actual
token transfer.

**Files/functions:** `escrow/src/lib.rs:339-368, 2667-2770` (`FeeSchedule` module),
`escrow/src/lib.rs::withdraw`, `::release`.

**Acceptance criteria:** Maintainer decision required — either (a) wire `FeeScheduleStorageKey::Active`
into the actual disbursement calculation in `withdraw()` (and/or `release()`, pending #46's
resolution), with a test asserting an activated schedule changes the SME's net payout; or (b) if
this is deliberately a forward-looking/staged API not yet activated, add an explicit doc comment
on the module and a test asserting `withdraw()`'s payout is unaffected by any activated schedule,
so the disconnect is intentional and documented rather than silent.

---

## New Features

## #20: Add `raise_min_contribution_floor` and `lower_max_per_investor` (missing symmetric counterparts)

**Category:** Feature · **Size:** M

**Context:** `lower_min_contribution_floor` (`escrow/src/lib.rs:6090`) exists but there is no
`raise_min_contribution_floor`. `raise_max_per_investor` (`:6139`) exists but there is no
`lower_max_per_investor`. Both configured caps are otherwise only adjustable in one direction —
once an admin lowers the contribution floor or raises the per-investor cap, there's no on-chain way
to reverse course without redeploying.

**Files/functions:** `escrow/src/lib.rs::lower_min_contribution_floor`,
`::raise_max_per_investor` (as the pattern to mirror).

**Acceptance criteria:**
- Add `raise_min_contribution_floor(env, new_floor) -> i128` mirroring `lower_min_contribution_floor`'s
  guard shape (status-open guard, strictly-higher check, event emission).
- Add `lower_max_per_investor(env, new_cap) -> i128` mirroring `raise_max_per_investor`.
- New typed errors as needed (coordinate discriminants with #6, which touches the same functions'
  siblings).
- Update `docs/escrow-investor-caps.md` and `docs/escrow-lifecycle.md`'s valid-transitions table.

---

## #21: Add `lower_maturity_max_horizon` with a guard against invalidating in-flight maturities

**Category:** Feature · **Size:** M

**Context:** `raise_maturity_max_horizon` (`escrow/src/lib.rs:7684`) exists; there is no way to
lower `DataKey::MaturityMaxHorizon` back down once raised. Unlike #20's pair, this one needs an
extra safety constraint: lowering the horizon must not retroactively invalidate an already-set
`InvoiceEscrow::maturity` that was valid under the old (higher) horizon — `validate_maturity_bounds`
(`escrow/src/lib.rs:1206`) is only invoked at `init`/`update_maturity` time, so a horizon lowered
after maturity is set would silently leave the escrow in a state that couldn't be `init`'d fresh
under the new rule, which is a soft inconsistency worth guarding against explicitly rather than
leaving implicit.

**Files/functions:** `escrow/src/lib.rs::raise_maturity_max_horizon`,
`::validate_maturity_bounds`.

**Acceptance criteria:**
- Add `lower_maturity_max_horizon(env, new_horizon) -> u64`, admin-gated, strictly-lower check.
- Explicit guard: reject a new horizon that would put the escrow's **current** `maturity` outside
  `now + new_horizon` (typed error), or document why that's intentionally not checked.
- Test: set maturity near the current horizon ceiling, lower the horizon, confirm the guard fires
  (or confirm+document the alternative).

---

## #22: Add `get_released_amount()` public read view

**Category:** Feature · **Size:** S

**Context:** Once #2/#3 land, `release()` maintains a running total under `DataKey::ReleasedAmount`
via `keys::released_amount()`, but there is no public getter for it — contrast with
`get_distributed_principal()` (`escrow/src/lib.rs:8374`), which is the equivalent read view for the
refund-side ledger.

**Files/functions:** `escrow/src/lib.rs` (add near `get_distributed_principal`, `:8374`).

**Acceptance criteria:** Add `pub fn get_released_amount(env: Env) -> i128` reading
`keys::released_amount()` with `.unwrap_or(0)`, matching `get_distributed_principal`'s style. Add
a test asserting it reflects cumulative `release()` calls correctly across partial and final
releases.

---

## #23: Add `unfund_batch` to match the `_batch` pattern used by every other value-moving entrypoint

**Category:** Feature · **Size:** M

**Context:** `fund_batch` (`MAX_FUND_BATCH`), `refund_batch` (`MAX_REFUND_BATCH`), and
`settle_batch` (`MAX_SETTLE_BATCH`) all have batch siblings bounded by their own `MAX_*` constant.
`unfund` (`escrow/src/lib.rs:8277`, added per `.kiro/specs/unfund-entrypoint/`) does not — an
admin or integrator wanting to process multiple investors' unfund requests in one transaction has
no batch path, unlike every comparable value-moving entrypoint.

**Files/functions:** `escrow/src/lib.rs::unfund` (pattern to extend), `::fund_batch`,
`::refund_batch` (patterns to mirror for all-or-nothing batch semantics and duplicate-address
rejection).

**Acceptance criteria:**
- Add `unfund_batch(env, entries: Vec<(Address, i128)>) -> InvoiceEscrow`, bounded by a new
  `MAX_UNFUND_BATCH` constant, atomic (all-or-nothing), rejecting duplicate investor addresses in
  one call (mirroring `FundingBatchDuplicateInvestor`'s pattern).
- Each entry requires that specific investor's `require_auth()` (matching `unfund`'s single-call
  semantics — batch should not let one signer unfund on behalf of another).
- Tests: happy path, empty batch, over-limit batch, duplicate-address rejection, partial failure
  rolls back atomically.

---

## Documentation

## #24: `.kiro/specs/unfund-entrypoint/requirements.md` R11's error table doesn't match the shipped implementation

**Category:** Documentation · **Size:** S

**Context:** `requirements.md` §R11 specifies error variants `EscrowNotOpen` (code 165),
`OverWithdrawal` (166), `LegalHoldActive` (167). The actual shipped `unfund`
(`escrow/src/lib.rs:8277`) uses `UnfundEscrowNotOpen` (220), `OverWithdrawal` (221),
`UnfundLegalHoldActive` (222) — different names **and** different codes — plus an additional
`DisputeBlocksUnfund` (242) guard (`guard_not_disputed`, line 8288) that isn't in the spec at all.
`docs/escrow-lifecycle.md` (§"Investor unfund path", lines 214-233) **correctly** documents the
shipped names/codes — only the original `.kiro` spec is stale.

**Files/functions:** `.kiro/specs/unfund-entrypoint/requirements.md` §R11.

**Acceptance criteria:** Update R11's table to match `escrow/src/lib.rs`'s actual variant names and
codes, and add a row for the `DisputeBlocksUnfund` guard the implementation added beyond the
original spec.

---

## #25: Security checklist's Authentication Matrix and Trusted Addresses sections omit the `payer` role entirely

**Category:** Documentation · **Size:** M

**Context:** `InvoiceEscrow::payer` (`escrow/src/lib.rs:1526`) is a first-class field ("must
authorize funding"), defaults to `admin` at `init` (line 3233), has its own dual-auth rotation
entrypoint (`rotate_payer`, `:3689`), and is required-auth on **every** `fund`/`fund_with_commitment`/
`fund_batch` call via `fund_impl`'s `escrow.payer.require_auth()` (line 6583). None of this appears
in `docs/escrow-security-checklist.md`'s §1 Authentication Matrix (which lists `fund` as
investor-only) or §2 Trusted Addresses (which covers admin/SME/treasury/funding-token/registry but
not payer). See also #46 for the security implications this doc gap is hiding.

**Files/functions:** `docs/escrow-security-checklist.md` §1, §2;
`escrow/src/lib.rs::fund_impl`, `::rotate_payer`.

**Acceptance criteria:** Add a `payer` row to §1's Authentication Matrix for `fund`/
`fund_with_commitment`/`fund_batch`/`rotate_payer`, and a §2.x "Payer" subsection describing its
default-to-admin initialization, its rotation mechanics, and its risk profile (cross-reference
#46/#47).

---

## #26: Two contributors' absolute local filesystem paths are committed in docs, one of them broken across 5 links

**Category:** Documentation · **Size:** S

**Context:**
- `docs/escrow-security-checklist.md` line ~299 links to
  `file:///home/demigodjayydy/Desktop/Starfund-contracts/escrow/src/tests/admin.rs` — an absolute
  path on one contributor's machine, broken for everyone else, and it leaks that contributor's
  local username/directory layout.
- `docs/escrow-cancellation-refunds.md` lines 12, 20, 21, 30, 44 link to
  `file:///c:/Users/enwer/OneDrive/Documents/Code%20Projects/OS%20Contributions/Starfund-contracts/...`
  — a second contributor's absolute Windows path, broken the same way, in 5 separate links.

**Files/functions:** `docs/escrow-security-checklist.md`, `docs/escrow-cancellation-refunds.md`.

**Acceptance criteria:** Replace all 6 links with relative repo paths (e.g.
`../escrow/src/tests/admin.rs`) or GitHub permalinks. Since §6's own line-number references are
noted as needing "re-audit after refactors," take this pass as an opportunity to spot-check that
the line numbers those links point at are still accurate (see #27).

---

## #27: Security checklist's own "re-audit after refactors" trigger has been missed — line-number references and entrypoint tables are stale

**Category:** Documentation · **Size:** M

**Context:** `docs/escrow-security-checklist.md` §6 states "Line numbers refer to `escrow/src/lib.rs`
at schema version 6; re-audit after refactors." `SCHEMA_VERSION` is still 6, but `lib.rs` has grown
substantially since this table was last accurate — none of `release`, `partial_settle`,
`rotate_payer`, `unfund`, the `FeeSchedule` subsystem, or the `PauseState`/`PauseScope` system (see
#28) appear in §1's Authentication Matrix, §2's Trusted Addresses, or §6's Entrypoint checklist
table. The line numbers cited for existing rows (e.g. "`fund` / `fund_with_commitment` ... line
~1119") should also be spot-checked against current line numbers.

**Files/functions:** `docs/escrow-security-checklist.md` (whole document).

**Acceptance criteria:** Add rows to §1, §2, and §6 for `release`, `partial_settle`, `rotate_payer`,
`unfund`, and `FeeSchedule`; verify/update every cited line number; note this doc's line-number
citations drift quickly and consider switching to function-name-only references (no line numbers)
to reduce future maintenance burden — raise as an open question for the maintainer rather than
changing the doc's format unilaterally.

---

## #28: Pause-specific docs (`pauser-states.md`, `pause-auth.md`) don't mention the scoped `PauseState`/`PauseScope` system

**Category:** Documentation · **Size:** M

**Context:** `escrow/src/lib.rs:1431-1520` implements a full scoped-pause upgrade (`PauseScope`
enum, `PauseState` struct with a `reason` field, stored under `DataKey::PauseState`) beyond the
simple boolean `DataKey::Paused` flag that `docs/pauser-states.md` and `docs/pause-auth.md`
describe exclusively. Neither doc mentions `PauseState`, `PauseScope`, or the interaction between a
legacy boolean-only pause (`Paused` set, no `PauseState` recorded) and a scoped one — which
`get_effective_pause_scope` (referenced at `:2840`) explicitly has logic to distinguish.

**Files/functions:** `docs/pauser-states.md`, `docs/pause-auth.md`,
`escrow/src/lib.rs:1431-1520, 2799-2880, 5208-5320`.

**Acceptance criteria:** Document `PauseScope`'s variants, `PauseState`'s fields, the
legacy-vs-scoped pause interaction, and `PauseScopeMismatch`'s trigger condition (once #1 lands).
Depends on #1 landing first so the feature actually compiles and the doc describes real behavior.

---

## #29: `LOCAL_REPRODUCTION.md` understates ignored-test count by 4.6x and documents a coverage gate CI doesn't enforce

**Category:** Documentation · **Size:** S

**Context:** `escrow/LOCAL_REPRODUCTION.md` claims "10 tests are temporarily ignored" (8 funding cap
+ 1 external calls + 1 integration) and documents `cargo llvm-cov --fail-under-lines 95` as the
enforced CI gate. Actual counts: **46** `#[ignore]` attributes across 8 files (`funding.rs` ×27,
`coverage.rs` ×6, `external_calls.rs` ×7, `legal_hold.rs` ×2, `settlement.rs` ×5,
`external_calls_mocked.rs` ×1, `attestations.rs` ×2, `admin.rs` ×1). And
`.github/workflows/ci.yml`'s actual coverage step runs with `continue-on-error: true` and **no**
`--fail-under-lines` flag at all — it cannot fail the build on a coverage regression today,
contrary to what this doc implies.

**Files/functions:** `escrow/LOCAL_REPRODUCTION.md`, `.github/workflows/ci.yml`.

**Acceptance criteria:** Update the ignored-test count (or better, point to a `grep -c '#\[ignore'`
command instead of a hardcoded number that will go stale again — see #34 for the actual triage
work). Correct the coverage-gate description to match what CI actually runs today
(`continue-on-error`, report-only), or file the CI change separately (#41) and update this doc once
that lands.

---

## #30: `docs/escrow-events.md` documents `CallbackRegisteredEvent`/`CallbackExecutedEvent` correctly, but `update_funding_deadline`'s `FundingDeadlineUpdated` event is undocumented anywhere

**Category:** Documentation · **Size:** S

**Context:** See #13 for the code-side half of this. `docs/escrow-events.md` already has correct
entries for `CallbackRegisteredEvent` (line 291) and `CallbackExecutedEvent` (line 303) — no doc
work needed there once #13 lands. `FundingDeadlineUpdated` (published by `update_funding_deadline`,
`escrow/src/lib.rs:5966`) has zero documentation anywhere in `docs/` or `.kiro/`, unlike its
sibling `FundingDeadlineExtended` (documented at `docs/escrow-events.md:129`).

**Files/functions:** `docs/escrow-events.md`, `escrow/src/lib.rs::update_funding_deadline`.

**Acceptance criteria:** Once #13 resolves whether `update_funding_deadline` is a real, distinct
entrypoint from `extend_funding_deadline`, add (or don't, if it's removed) a
`docs/escrow-events.md` entry for `FundingDeadlineUpdated` matching `FundingDeadlineExtended`'s
level of detail.

---

## #31: `docs/openapi.yaml` is missing 5 real entrypoints entirely

**Category:** Documentation · **Size:** M

**Context:** `docs/openapi.yaml` has zero references to `unfund`, `release`, `partial_settle`,
`rotate_payer`, or `rotate_beneficiary` (confirmed via direct grep — none of the five appear
anywhere in the file). `docs/tests/openapi.test.js` presumably validates this spec against
something; it should be checked and extended once the spec catches up.

**Files/functions:** `docs/openapi.yaml`, `docs/tests/openapi.test.js`.

**Acceptance criteria:** Add OpenAPI operation definitions for all 5 missing entrypoints matching
their actual `lib.rs` signatures (including the typed errors from #4/#5/#24 once those land). Run
`docs/tests/openapi.test.js` and confirm it passes against the updated spec; if the test doesn't
already check entrypoint completeness against the contract, note that gap as a follow-up rather
than expanding this issue's scope.

---

## Testing

## #32: 46 `#[ignore]`d tests need individual triage, not a blanket "upstream latent: escrow API/test drift" reason

**Category:** Testing · **Size:** L

**Context:** `#[ignore = "upstream latent: escrow API/test drift"]` appears 35+ times across
`funding.rs`, `legal_hold.rs`, `admin.rs`, `external_calls_mocked.rs`, `attestations.rs`,
`external_calls.rs`, and `coverage.rs`. Given #1–#19, this generic reason is very likely masking
real, specific compile/behavior breaks — once those land, each ignored test needs to be
individually re-enabled, run, and either fixed or given a *specific* ignore reason (not a repeated
generic one). A handful have already-specific reasons worth checking first once the crate compiles:
`escrow/src/tests/settlement.rs:821,973,1041` ("HostError wraps contract panic; expected substring
not matched" — suggests brittle string-matching assertions that should assert on typed error codes
instead, consistent with #18's finding), and `:1075` ("body tests non-participant claim, not dust
sweep capping; panics without `#[should_panic]`" — a mislabeled/misplaced test).

**Files/functions:** All 8 files listed above; depends on #1–#19 landing first.

**Acceptance criteria:** For each of the 46 ignored tests: re-run once the crate compiles; if it
passes, remove the `#[ignore]`; if it fails, replace the generic reason with the specific failure
and either fix it or file a focused follow-up issue. Track completion as "0 tests carry the generic
'upstream latent' reason" rather than "0 ignored tests" (some may be legitimately still-pending
follow-on work).

---

## #33: Add a compile-time/test-time uniqueness check for `EscrowError` discriminants

**Category:** Testing · **Size:** S

**Context:** #8 fixes today's 8 duplicate-discriminant collisions, but nothing currently prevents a
future PR from reintroducing one — the enum is hand-numbered across ~160 variants with no
automated guard.

**Files/functions:** `escrow/src/lib.rs::EscrowError`; new test in `escrow/src/tests/coverage.rs`
or a new small test module.

**Acceptance criteria:** Add a test that enumerates all `EscrowError` discriminant values (via
`as u32` on each variant, or a `strum`-style iteration if a crate dependency is acceptable — prefer
a hand-written match returning all variants to avoid adding a new dependency) and asserts they're
pairwise distinct. This test must fail loudly (not just via `cargo clippy`) so it catches
regressions in normal `cargo test` runs.

---

## #34: `escrow/src/test_allowlist_tests.rs:1243` has a literal stub test body

**Category:** Testing · **Size:** S

**Context:** Line 1243 reads `// TODO: implement test body` inside what is otherwise a real `#[test]`
function in a 2017-line test file.

**Files/functions:** `escrow/src/test_allowlist_tests.rs:1243`.

**Acceptance criteria:** Read the surrounding test's name/doc comment to determine its intended
assertion, implement it, or remove the stub if it's superseded by another test in the same file
(check for a near-duplicate test name first).

---

## #35: `external_calls_mocked.rs`'s mock token client has 12 `unimplemented!()` method bodies

**Category:** Testing · **Size:** M

**Context:** `escrow/src/tests/external_calls_mocked.rs` (787 lines) implements what appears to be a
mock SEP-41 token client with `unimplemented!()` bodies at lines 47, 50, 53, 321, 324, 327, 412,
415, 418, 490, 493, 496 — any test path that actually exercises these methods will panic.
`escrow/src/external_calls.rs`'s module doc explicitly calls out "Mocked token scenarios (where
feasible) to detect divergence" as part of the test strategy for the balance-delta invariants —
these stubs are exactly where a fee-on-transfer or rebasing-token simulation would need real logic.

**Files/functions:** `escrow/src/tests/external_calls_mocked.rs` (12 call sites listed above).

**Acceptance criteria:** For each `unimplemented!()`, either implement the mock behavior it's
standing in for (likely a fee-on-transfer or balance-manipulating variant, per the surrounding
module's purpose) or, if the method is never actually called by any current test, remove it and
note why in a comment. Cross-reference `docs/escrow-token-safety.md` for the specific adversarial
token behaviors that should be simulated.

---

## #36: Zero test coverage for the `payer` dual-auth requirement in `fund_impl`

**Category:** Testing · **Size:** M

**Context:** `escrow/src/tests/` has zero references to `payer` anywhere (`grep -rn "payer"
escrow/src/tests/*.rs` returns nothing), despite `fund_impl` requiring
`escrow.payer.require_auth()` on every `fund`/`fund_with_commitment`/`fund_batch` call (line 6583).
`docs/escrow-security-checklist.md` §"Negative-auth test coverage" claims the `auth_audit_*` suite
in `escrow/src/tests/admin.rs` comprehensively tests "all state-mutating entrypoints" — it does not
test the payer requirement at all. See #46/#25 for the underlying design questions this gap is
hiding.

**Files/functions:** `escrow/src/tests/admin.rs` (`auth_audit_*` tests, pattern to extend);
`escrow/src/lib.rs::fund_impl`.

**Acceptance criteria:** Add `auth_audit_fund_requires_payer` (and `_fund_with_commitment`,
`_fund_batch` variants) proving a call signed by the investor alone (no payer signature in
`env.auths()`) is rejected, and a call signed by both succeeds. This will surface whatever the
maintainer decides in #46.

---

## #37: `rotate_payer` has no test coverage at all

**Category:** Testing · **Size:** S

**Context:** Same zero-hits grep as #36 — `rotate_payer` (`escrow/src/lib.rs:3689`) has no
dedicated test module, unlike its structural twin `rotate_beneficiary` which is covered by
`auth_audit_*` tests per the security checklist's table. Specifically needs a test proving/
disproving the missing-nonce gap identified in #47.

**Files/functions:** `escrow/src/lib.rs::rotate_payer`; new tests alongside
`rotate_beneficiary`'s existing coverage.

**Acceptance criteria:** Add: dual-auth requirement test (payer-only and admin-only signing both
rejected), no-op rejection (`NewPayerSameAsCurrent`), status-gate test (`PayerRotationNotOpen`),
legal-hold-block test, and — once #47 is resolved either way — a replay-protection test consistent
with whatever `rotate_beneficiary` does.

---

## #38: No property/invariant test exists for the `release()` conservation invariant

**Category:** Testing · **Size:** M

**Context:** `escrow/src/tests/properties.rs` (2816 lines, proptest-based) predates `release()` —
by construction, since `release()` doesn't even compile yet (#2/#3), `properties.rs` cannot
currently reference it. Once #1–#19 land, the `funded_amount`/`ReleasedAmount`/
`DistributedPrincipal` three-way relationship `release()` maintains
(`escrow/src/lib.rs:6953-6962, 6972-6980`) needs the same proptest-style conservation coverage the
existing invariants (I-1 through I-10 in `docs/escrow-security-checklist.md` §3) get for
`fund_impl`/`withdraw`/`settle`.

**Files/functions:** `escrow/src/tests/properties.rs`; `escrow/src/lib.rs::release`.

**Acceptance criteria:** Add a proptest asserting, across randomized sequences of partial
`release()` calls: `released_amount` never exceeds `funded_amount`; the final `release()` call
(bringing `released_amount == funded_amount`) transitions `status` to 3 and no further `release()`
call succeeds afterward. Add the corresponding entry to `docs/escrow-security-checklist.md` §3 as a
new invariant (I-11) once this is verified.

---

## #39: `callback_binding_tests.rs` cannot currently exercise the callback system it's meant to test

**Category:** Testing · **Size:** S

**Context:** `escrow/src/callback_binding_tests.rs` (315 lines) tests `execute_callback`'s 6 typed
errors (`CallbackWrongOrigin`, `CallbackWrongNonce`, `CallbackWrongPhase`, `CallbackReplayed`,
`CallbackAfterCancellation`, `CallbackNotFound`) — but since `CallbackRegisteredEvent`/
`CallbackExecutedEvent` don't compile (#13), this entire file is currently blocked.

**Files/functions:** `escrow/src/callback_binding_tests.rs`; depends on #13.

**Acceptance criteria:** Once #13 lands, confirm this file actually exercises all 6 error codes
with one negative test each (not just the happy path) — audit and fill any gaps found.

---

## #40: `escrow/proptest-regressions/` seed files should be reconciled with `properties.rs` once the crate compiles

**Category:** Testing · **Size:** S

**Context:** `escrow/proptest-regressions/test.txt` and `tests/properties.txt` are committed
regression seeds (intentionally tracked per the root `.gitignore`'s comment). Since the crate
doesn't currently compile, these seeds can't be replayed to confirm they still correspond to live
`proptest!` blocks in `properties.rs`. This is a follow-up dependent on #1–#19 and #38.

**Files/functions:** `escrow/proptest-regressions/test.txt`,
`escrow/proptest-regressions/tests/properties.txt`, `escrow/src/tests/properties.rs`.

**Acceptance criteria:** Once the crate compiles, run the proptest suite with these regression
files present and confirm every seed still maps to an existing property (no orphaned seeds for
deleted/renamed properties). Remove any that don't.

---

## Tooling / Infrastructure

## #41: No `[profile.release]` configured for a wasm32 Soroban contract

**Category:** Tooling · **Size:** S

**Context:** Neither `Cargo.toml` (workspace root) nor `escrow/Cargo.toml` defines a
`[profile.release]` section (confirmed via direct grep — zero `profile` matches in either file).
Soroban/wasm32 contracts typically want an explicit release profile
(`opt-level = "z"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = "symbols"`) to
minimize deployed wasm size, which directly affects deployment cost and reviewer audit surface.

**Files/functions:** `Cargo.toml` (root) or `escrow/Cargo.toml`.

**Acceptance criteria:** Add a `[profile.release]` section with the settings above (or the subset
appropriate after measuring their effect). Record the before/after wasm binary size (from the
existing `cargo build --target wasm32v1-none --release` CI step) in the PR description.

---

## #42: No `rust-toolchain.toml` pinning the exact Rust version

**Category:** Tooling · **Size:** S

**Context:** `.github/workflows/ci.yml` uses `dtolnay/rust-toolchain@stable`, a floating tag with no
version pin, and there's no `rust-toolchain.toml` anywhere in the repo (confirmed via `find`). A
future stable Rust release can silently change `rustc`/`clippy` lint behavior (the `-D warnings`
gate in CI makes this especially risky — a new clippy lint promoted to warn-by-default could break
CI with no repo change at all) and there's no way to reproduce a specific CI run's exact toolchain
locally.

**Files/functions:** New `rust-toolchain.toml` at repo root; `.github/workflows/ci.yml`.

**Acceptance criteria:** Add `rust-toolchain.toml` pinning a specific stable version (matching
whatever's currently green in CI). Update `escrow/LOCAL_REPRODUCTION.md` to reference it instead of
implying "any stable toolchain" works.

---

## #43: No dependency-vulnerability scanning in CI

**Category:** Tooling · **Size:** M

**Context:** `.github/workflows/ci.yml` (read in full) has no `cargo audit` or `cargo deny` step,
and there's no `deny.toml`/`audit.toml` anywhere in the repo. This is a contract that moves real
funds (per the README/security checklist's own framing) with zero automated dependency-CVE
scanning today.

**Files/functions:** `.github/workflows/ci.yml`; new `deny.toml` or equivalent.

**Acceptance criteria:** Add a `cargo audit` (or `cargo deny check advisories`) CI step. Decide
whether it should hard-fail the build or `continue-on-error` initially while the dependency tree is
triaged for existing advisories; document the choice in the workflow file's comments.

---

## #44: Coverage and workspace-clippy CI steps use `continue-on-error: true` with no enforced threshold

**Category:** Tooling · **Size:** S

**Context:** `.github/workflows/ci.yml`'s coverage step (`cargo llvm-cov --features testutils
--summary-only -p starfund_escrow`) and its second, `--all-targets` clippy pass both use
`continue-on-error: true`, and the coverage step has no `--fail-under-lines` flag at all — CI
cannot currently fail a PR for either a coverage regression or a test-scope lint regression, even
though `escrow/LOCAL_REPRODUCTION.md` documents a 95%-line-coverage gate as if it's enforced (see
#29 for the doc-side fix).

**Files/functions:** `.github/workflows/ci.yml`.

**Acceptance criteria:** Once #1–#19 land and coverage can actually be measured again, either
restore `--fail-under-lines 95` as a hard gate (removing `continue-on-error` from that step) or
explicitly document in the workflow file why it's report-only. Same decision for the
`--all-targets` clippy pass. Update #29's doc fix to match whatever is decided here.

---

## #45: Root `.gitignore` lists `Cargo.lock` as ignored, but it's git-tracked — dead/contradictory entry

**Category:** Tooling · **Size:** S

**Context:** Root `.gitignore`'s "Build" section lists `Cargo.lock` as ignored, but `git ls-files`
confirms the root `Cargo.lock` is tracked (correctly — this is a deployed contract, not a library
meant for downstream `cargo` consumers, so pinning the lockfile is the right call; the gitignore
entry is simply stale). This repo previously also had a second, independently-drifted
`escrow/Cargo.lock` committed alongside it (removed as part of this backlog's cleanup pass —
`diff`ing the two showed hundreds of lines of divergence, and `git status` after a `cargo clippy`
run from the workspace root never touched it, confirming cargo's workspace resolution ignores it
in favor of the root lockfile).

**Files/functions:** `.gitignore`.

**Acceptance criteria:** Remove the `Cargo.lock` line from `.gitignore`'s Build section (it's
misleading given the file is intentionally tracked), and add a one-line comment next to
`escrow/Cargo.toml`'s workspace membership (or in `README.md`'s Prerequisites section) stating only
the root `Cargo.lock` is authoritative, so a future contributor doesn't reintroduce a second one by
running `cargo build` from inside `escrow/` in a context where it forgets it's a workspace member.

---

## Security Hardening

## #46: `release()` bypasses `protocol_fee_bps` entirely — a fee-free principal-exit path parallel to `withdraw()`

**Category:** Security · **Size:** L

**What must not change:** the conservation invariant `sme_payout + fee == funded_amount` that
`withdraw()` enforces for every principal disbursement to the SME.

**Context:** `withdraw()` applies the immutable, init-time `DataKey::ProtocolFeeBps` split to the
full `funded_amount` (per the module rustdoc's "Immutable protocol fee" section,
`escrow/src/lib.rs:121-145`) and transitions `status` 1→3. `release()` (`:6900-6990`, added later)
**also** transitions `status` 1→3 (on its final tranche) and moves principal to the same
`escrow.sme_address`, but its transfer at lines 6941-6947 sends the **full requested `amount`**
directly to the SME with no fee computation, no treasury involvement, and no reference to
`ProtocolFeeBps` anywhere in the function. Both entrypoints are available whenever `status == 1`.
A fee-averse SME can simply always call `release(remaining)` instead of `withdraw()` and extract
100% of principal, permanently bypassing the protocol fee the contract's own top-level
documentation describes as a core economic guarantee.

**Files/functions:** `escrow/src/lib.rs::release` (:6900-6990), `::withdraw` (fee-split logic per
module doc lines 121-145).

**Acceptance criteria:** Maintainer decision required on intended design — either (a) `release()`
should apply the same `ProtocolFeeBps` split `withdraw()` does, with a test asserting fee parity
between the two paths for equivalent amounts; or (b) `release()` and `withdraw()` should be made
mutually exclusive per escrow instance (calling one disables the other), with a test proving the
exclusion; or (c) if `release()` is intentionally fee-exempt for a documented reason (e.g. it's
meant for a different, non-fee-liable disbursement category), that must be stated explicitly in the
module rustdoc's fee section so it isn't mistaken for an oversight by the next reader. Do not ship
a silent fix that just adds the fee without confirming intent — this changes real economic
behavior.

---

## #47: `fund_impl`'s undocumented `payer` dual-auth has no recovery path if the payer key is lost

**Category:** Security · **Size:** L

**What must not change:** the ability for a legitimately-configured escrow to keep accepting
investor funding indefinitely as long as at least the admin key remains available (mirrors the
existing admin-recovery guarantee).

**Context:** `fund_impl` requires `escrow.payer.require_auth()` on every funding call (line 6583),
and `payer` defaults to `admin` at `init` (line 3233). The **only** way to change `payer` is
`rotate_payer` (`:3689`), which itself requires **both** `escrow.payer.require_auth()` **and**
`escrow.admin.require_auth()` in a single atomic call — with no two-step proposal/expiry mechanism.
Contrast with admin-key loss, which has a documented recovery lever
(`propose_admin`/`accept_admin`, plus `execute_admin_recovery` after `PendingAdminExpiry` per
`EscrowError::AdminRecoveryNotExpired`). If the `payer` key is lost or its signer becomes
unavailable, `rotate_payer` can **never** be called (it needs the lost key's own signature to
authorize its own replacement), which means **all future funding halts permanently** for that
escrow instance — a strictly worse failure mode than losing the admin key, and currently
undocumented anywhere (see #25).

**Files/functions:** `escrow/src/lib.rs::fund_impl` (:6583), `::rotate_payer` (:3689), contrast with
`::propose_admin`/`::accept_admin`/`::execute_admin_recovery`.

**Acceptance criteria:** Maintainer decision required — either (a) add an admin-only emergency
payer-recovery path analogous to `execute_admin_recovery` (admin alone can reset `payer` after a
documented timelock, without needing the lost key's signature); or (b) if `payer` is intended to
always equal a governed multisig identical to `admin` in practice (making this risk moot by
deployment convention), document that constraint explicitly and add a test/warning for the
degenerate single-key case. Cross-reference #25 (doc gap) and #36 (test gap) — this issue is the
design decision those two are downstream of.

---

## #48: `investor.require_auth()` ordering in `fund_impl` doesn't match the documented guard-ordering table

**Category:** Security · **Size:** M

**What must not change:** the ADR-002 canonical sequence itself (read-only preconditions → auth →
writes) — this issue is about the checklist's table accuracy, not the underlying invariant, which
does hold (no storage write happens before `investor.require_auth()` succeeds).

**Context:** `docs/escrow-security-checklist.md` §6's Entrypoint checklist table lists, for
`fund`/`fund_with_commitment`: "Pre-auth reads (no writes): floor read | `require_auth`: line
~1119." That implies the floor read happens *before* `investor.require_auth()`. Reading the actual
code: `investor.require_auth()` is called at `escrow/src/lib.rs:6420`, and the floor read happens
afterward at lines 6435-6446 (along with the decimal-scale check, legal-hold check, status check,
and funding-deadline check — all also after the auth call). The order is reversed from what the
table documents. This doesn't violate the "no write before auth" invariant (all of these are
reads), but it does mean the specific ordering the checklist describes for audit purposes is wrong,
and an auditor trusting the table's literal claim would be misled about what's validated before a
signature is required.

**Files/functions:** `docs/escrow-security-checklist.md` §6 (the `fund`/`fund_with_commitment`
row); `escrow/src/lib.rs::fund_impl` (:6413-6473).

**Acceptance criteria:** Re-verify actual guard order in `fund_impl` line-by-line, update the
checklist table's "Pre-auth reads" column to match reality, and add a regression test (in the
spirit of `coverage.rs`'s `refactor_gate_helpers_*` tests) asserting the auth call happens at a
specific point relative to the other guards, so future refactors can't silently reorder them again
without a test failing.

---

## #49: `FeeSchedule` subsystem could mislead an auditor into believing schedule-based fees are enforced

**Category:** Security · **Size:** S

**What must not change:** `get_active_fee_schedule()`'s current read semantics (it should keep
returning whatever schedule was activated — the fix here is about disclosure/wiring, not about
hiding the stored data).

**Context:** This is the security-audit framing of #19's engineering gap — kept separate because
the fix/audience differs. An external auditor or integrator who calls `get_active_fee_schedule()`
and sees a non-null `FeeSchedule` with a specific `fee_bps` would reasonably conclude that value
governs SME disbursement fees. It does not — `withdraw()` only reads the separate, immutable
`DataKey::ProtocolFeeBps`. This is a trust-boundary/disclosure risk independent of whether #19 is
resolved by wiring the subsystem up or documenting it as inert: either way, anyone who audited the
contract *before* this issue is filed may have already drawn the wrong conclusion from the public
read API's apparent liveness.

**Files/functions:** `escrow/src/lib.rs:2667-2770` (`FeeSchedule` module).

**Acceptance criteria:** Once #19 lands (either direction), add a `# Security` rustdoc section on
`get_active_fee_schedule` (and `submit_fee_schedule`) explicitly stating whether an activated
schedule currently affects any token transfer, so the public API surface is self-documenting rather
than relying on a reader finding this backlog issue or the module-level fee docs.

---

## #50: `rotate_payer` lacks the admin-nonce replay protection its sibling `rotate_beneficiary` has

**Category:** Security · **Size:** M

**What must not change:** `rotate_beneficiary`'s existing nonce behavior — this issue only adds the
same protection to `rotate_payer`, it doesn't touch the beneficiary path.

**Context:** Every other dual-auth, admin-involved entrypoint consumes the replay-protection nonce
via `consume_admin_nonce`: `set_legal_hold`, `request_clear_legal_hold`, `set_allowlist_active`,
`set_investor_allowlisted(s)`, and — directly comparable in shape — `rotate_beneficiary`
(`escrow/src/lib.rs:3647`, which takes `expected_nonce: u32` and calls
`Self::consume_admin_nonce(&env, expected_nonce)` immediately after its own dual `require_auth`
calls). `rotate_payer` (`:3689`) has the **identical** dual-auth shape (current-role +
admin, both required) but its signature is `pub fn rotate_payer(env: Env, new_payer: Address) ->
InvoiceEscrow` — no `expected_nonce` parameter, and no call to `consume_admin_nonce` anywhere in
its body. The nonce exists specifically to bind a multi-signature authorization to a single
intended call so one collected signature can't be replayed in a different context; `rotate_payer`
requires the same multi-signature pattern without that binding.

**Files/functions:** `escrow/src/lib.rs::rotate_payer` (:3689), contrast with `::rotate_beneficiary`
(:3615-3647).

**Acceptance criteria:** Add `expected_nonce: u32` to `rotate_payer`'s signature and call
`Self::consume_admin_nonce(&env, expected_nonce)` in the same position `rotate_beneficiary` does
(after the dual `require_auth`, before the storage write). Coordinate with #5 (which is also
touching this function's error variants) and #37 (test coverage) in the same PR wave.
