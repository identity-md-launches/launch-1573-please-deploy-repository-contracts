# Adaptation for launch B (contracts only, Ethereum mainnet)

This file records what was changed to make the repository launchable through IdentityMD's
`evm_contracts` factory, and why. The short answer: **no source file, test, document or
configuration was changed.** The project arrived already in the form the launch requires, and the
checks below confirm it. The only file this adaptation adds is this one.

## 1. What the launch requires, and where the code already meets it

| Launch rule | Where the project meets it | Checked by |
|---|---|---|
| Each contract is fully configured in a nonpayable constructor taking only static words (address, uintN, bool, bytes32), at most 16 of them | `WorkProof(address)`, `WorkOrders(address,address,address)`, `CrowdPots(address,address)`, `Doorbell(address,address,bytes32,uint96,uint96,uint96,uint32,bool)` | `test_launch_stateChangingFunctionsAreExactlyTheDocumentedOnes` (walks each constructor's ABI inputs) |
| A constructor runs on an empty chain: no external call, no code-size check | Every constructor stores its arguments and refuses only the zero address or an out-of-range value; `Doorbell` checks its bond and delivery time against constants, not against the desk | `test_launch_constructorsMakeNoCallAndNoCodeCheck` (walks constructor bytecode for CALL, STATICCALL, DELEGATECALL, CALLCODE, EXTCODE*, BALANCE, CREATE, CREATE2, SELFDESTRUCT); `test_launch_deploysInManifestOrderOnAnEmptyChain` (asserts the three mainnet dependencies have no code, then deploys in manifest order) |
| No owner, admin or setter; `$owner` is never an argument; `msg.sender` (the factory) and `tx.origin` are never stored | No contract has an owner. No constructor reads `msg.sender` or `tx.origin` | `test_manifest_namesTheFourContractsInOrder` (the manifest holds no `$owner` and no `$token`); `test_launch_noFunctionNamesAnOwnerOrAPowerToChange` |
| No minting, pausing, freezing, blacklisting, seizing, proxy, initializer or upgrade path | None exists. The 28 state-changing functions are the ones `README.md` lists and no others; no contract can receive ETH | `test_launch_stateChangingFunctionsAreExactlyTheDocumentedOnes` |
| No DELEGATECALL, CALLCODE or SELFDESTRUCT in runtime bytecode | None | `test_launch_runtimeBytecodeHasNoForbiddenOpcode`, `test_bytecodeWalk_findsWhatItLooksForAndSkipsPushData`; reproduced independently with the protected harness (section 3) |
| Nothing is called after deployment | Nothing needs to be: no initializer, no configuration. Orders, pots and rings start from zero | `test_launch_deploysInManifestOrderOnAnEmptyChain` |
| Every library function the contracts call is internal (no unlinked bytecode) | `SpecText`, `OrderHeader` and `JsonScan` have only internal functions; the four build artifacts have empty `linkReferences` | `forge build` output; the protected harness deploys the bytecode as built |
| Contracts-only launch: no `LaunchToken`, no `MerkleDistributor`, no pool; no `launch.json` in the tree | None of these exists. The manifest is in `docs/LAUNCH.md` for the manifest step to write | `test_manifest_*` |
| Runtime at most 24,576 bytes, initcode at most 49,152 | 4,579 / 19,289 / 14,177 / 5,972 runtime; 4,744 / 19,726 / 14,646 / 7,032 initcode with arguments | `test_launch_fitsOurSizeAndGasGates`; `forge build --sizes` |
| Build configuration | `foundry.toml` already has solc 0.8.26, cancun, optimizer 200 runs, `via_ir`, `bytecode_hash = "none"`, `cbor_metadata = false`. It was not touched | `forge build` |

The manifest the brief gives matches `docs/LAUNCH.md` word for word and is what the tests check
against the code: `test_manifest_everyArgumentIsWhatTheCodeExpects`,
`test_manifest_theDoorbellsValuesAreTheOnesTheDocumentExplains`,
`test_manifest_theDoorbellsKindIsTheTextTheDocumentNames`,
`test_manifest_payrollIsTheFirstLaunchsLiveAddress`.

## 2. The imported audit: what reproduced, what was done

The audit reported four entries, all at level **info**, and stated of each that it is a documented
trust assumption or a coverage note, not a code defect against the stated design. Each was
reproduced against the unchanged code before deciding. **None required a change**: fixing any of
the three behavioural entries would replace a rule the requester chose and documented, which the
launch rules do not ask for and the brief forbids ("deploy this repository as it is").

### c309d5e6 (coverage, no defect)

Confirmed. `forge build` and the full `forge test` suite pass (654 tests, 30 suites, invariant
suites included, with an empty environment and no network). The bytecode scan, the constructor
ABI shapes, the nonpayable constructors and the absence of launch-forbidden features were all
re-checked by the tests named in section 1 and by the rehearsal in section 3. No change.

### 46190b9b: with `allowClaimed = 1`, a Doorbell order can be paid on an unchallenged false claim

**Reproduced** step by step with the manifest's exact Doorbell values on the real desk: one IMD
transferred to the Doorbell, one address with 0.25 IMD rings 25 times, claims, registers a
fabricated job id, claims delivery with an arbitrary hash, and after 24 hours anyone finalizes.
The address ends with 1.15 IMD (0.9 runner pay plus its 0.25 bond), Payroll receives 0.1 IMD, the
Doorbell receives nothing back, and the order is delivered but unproven.

**Not changed.** This is the behaviour the requester asks for: the value `1` is a launch-day
decision explained in `docs/LAUNCH.md` section 5 (receipts stalled since about 8 October 2026),
stated at the head of `src/pots/Doorbell.sol`, and mitigated operationally by the watcher that
section 6, point 4 of `docs/LAUNCH.md` requires before the Doorbell is funded. The same scenario
is pinned by the existing tests `test_claimsOn_theOrderAcceptsAnUnchallengedDeliveryClaim`
(`test/pots/Doorbell.t.sol`) and `test_runbook_theFirstPot_anUntrueClaimIsStoppedByAChallenge`
(`test/launch/LaunchRunbook.t.sol`). Setting the argument to `0` is the requester's call on the
day, not an adaptation; the manifest step is told the same value.

### be85d51c: `CrowdPots` never calls `finalize`, so an unchallenged claim leaves a pot Hiring

**Reproduced**: a pot with the claimed path on, launched, taken, registered and claimed delivered;
after 25 hours and again 30 days later `claimRefund` reverts `WrongStage(Hiring)`, `stageOf`
reads `Hiring` and `reconcile` makes no desk call; after any address calls
`WorkOrders.finalize(orderId)`, `reconcile` moves the pot to `Delivered` and `claimRefund` pays.

**Not changed.** The contract header (`src/pots/CrowdPots.sol`, "The one call they do not make is
`finalize`") and the `reconcile` notes document this as a design choice, and the existing test
`test_reconcile_leavesAnUnchallengedClaimToTheRunner` (`test/pots/CrowdPots.t.sol`) pins it.
Having `reconcile` call `finalize` would add the two Payroll calls and the callback to every
reconcile of such a pot (about 2.15 million gas of required allowance) and would make a
contributor's refund call pay the runner, which is a change of the product's rules rather than a
launch requirement. Contributors lose nothing to it: the runner is paid either way, and
`finalize` is open to anyone. Left for the requester to decide after launch review.

### 157f2546: a minimum bond holds a receipt-only pot's escrow for its whole delivery time

**Reproduced**: a pot of 1,000 IMD with the desk's minimum bond (0.25 IMD), 30 days of delivery
and 30 days of hiring, receipt-only; an address with 0.25 IMD claims and registers a job id that
has no receipt. For the 30 days `reconcile` returns `Hiring`, `claimRefund` reverts
`WrongStage(Hiring)` and `reportFailed` reverts `NoReceipt`. At the deadline `reconcile` has the
desk time the claim out; the pot is refundable for 1,000.25 IMD (target plus the forfeited bond).

**Not changed.** The bond and the delivery time are the opener's choices, published in `Terms`
before anyone chips in and bounded by the contract (bond at least the desk's minimum, delivery at
most 30 days). The `Terms` documentation states the trade-off, `docs/THREATS.md` L6 and C3 state
it, and the existing tests `test_provenPath_theSameStallCostsTheWholeBond_andTheOrderReopens` and
`test_claimedPath_anUndecidedDisputeEndsThePotEarly_forATenthOfTheBond` pin the cost. Raising the
minimum bond or shortening the maximum delivery would change what every pot and every order may
ask for, which is not a launch requirement.

## 3. Checks run for this adaptation

- `forge build --sizes` with the project's own `foundry.toml`: builds; sizes as in section 1.
- `forge test` (default profile, no network, empty environment): 654 passed, 0 failed.
- The launch kind's protected harness (`Contracts.protected.t.sol`) was run in `test/scratch/`
  with chain id 1, the real ProjectFactory address etched with the probe, and the four initcodes
  built from the manifest's arguments with `$contract:` references resolved to the CREATE2
  addresses predicted for each earlier contract. All four deployed at their predicted addresses
  and the runtime scan found no DELEGATECALL, CALLCODE or SELFDESTRUCT.
- The three audit reproductions above were run in `test/scratch/` against the launch's real
  `WorkOrders`, `CrowdPots` and `Doorbell`. `test/scratch/` is scaffolding and is not part of the
  submission; the permanent tests that pin the same behaviour are named beside each finding.
- The fork suite (`FOUNDRY_PROFILE=fork`) was not run: it needs a mainnet RPC, and the launch's
  verification runs without network.

## 4. Open items for the launch, none of which needs a code change

- The Doorbell's last argument (`allowClaimed`) is `1` in the manifest. It is permanent once
  deployed. `docs/LAUNCH.md` section 5 gives the command that decides it on launch day.
- The three mainnet addresses (IMD, WorkRegistry, Payroll of launch 1531) are the requester's
  values from `docs/LAUNCH.md` and `src/interfaces/external/Mainnet.sol`. They were not re-read
  from the chain here; section 6 of `docs/LAUNCH.md` says how the requester checked Payroll.
- The rule the requester asked to keep (anyone may call `claimRefund` and `claimRefunds` for any
  contributor, contracts included) is kept as it is. `docs/DEVIATIONS.md` D7 gives the reasoning.
