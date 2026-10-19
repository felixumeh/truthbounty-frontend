# PR: Canonical Appeal Participation Transaction Hooks (V2-FE-015)

## Linked Issue

Closes #402

## Summary

This PR adds the canonical appeal participation transaction layer for TruthBounty Protocol V2 on Optimism/EVM. It replaces ad hoc / mock-only appeal logic with a single, contract-driven flow: read appeal state, validate eligibility, simulate, submit via wagmi/viem, then reconcile after finality — while keeping first-round verification state fully independent of appeal participation state.

## What This PR Delivers

### 1. Appeal Domain Types (`src/app/types/appeal.ts`)

- `AppealDecision`, `AppealParticipationStatus`
- `AppealSnapshot`, `AppealDeadline`, `AppealStakeBounds`, `AppealWalletPosition`
- `AppealParticipationContext` (snapshot, deadline, stake bounds, wallet position, eligibility, ineligibility reason)
- `AppealParticipationPayload`, `AppealParticipationTransaction`
- `AppealSimulationResult`, `AppealValidation`, `AppealReconciliationResult`
- `StateSegregation` + `StateSegregationStatus` for strict first-round / appeal separation

### 2. Hooks

- `useAppealContext` — reads snapshot, deadline, stake bounds, and wallet position from the canonical API projection; exposes eligibility with specific ineligibility reasons.
- `useAppealParticipation` — encodes SUPPORT/OPPOSE via the canonical ABI, simulates before submitting, submits through real wallet write hooks, tracks last transaction, and fails closed when the wallet write path is unavailable.
- `useAppealReconciliation` — waits for receipt, applies confirmations, extracts revert reasons, updates wallet position post-confirmation, and enforces state segregation.

### 3. Tests

- Unit tests for all hooks (`useAppealContext`, `useAppealParticipation`, `useAppealReconciliation`)
- Integration test: complete flow, fail-closed guards, revert handling, state segregation, concurrent participation prevention
- Total: **~90 new test cases**

### 4. Documentation

- Companion implementation record: `IMPLEMENTATION_V2_FE_015.md`
- Verification report: `VERIFICATION_V2_FE_015.md`

## Acceptance Criteria Mapping

- [x] Reads appeal snapshot, deadline, stake bounds, and existing wallet position
- [x] Encodes appeal participation through the canonical contract interface
- [x] Validates all inputs (chain, address, artifact version, wallet account, amounts, state)
- [x] Simulates before submission and rejects invalid calls
- [x] Submits via real wallet hooks and tracks the transaction hash immediately
- [x] Reconciles only from a real receipt; never fabricates hashes, receipts, balances, or rewards
- [x] Keeps first-round and appeal participation state independent
- [x] Fails closed on unsupported chain, missing wallet, invalid appeal, already-participated, or wallet-write outage
- [x] No visual redesign, no UI components, no production mock/stub data
- [x] Documentation current and complete

## Files Changed

### New

- `src/app/types/appeal.ts`
- `src/hooks/useAppealContext.ts`
- `src/hooks/useAppealParticipation.ts`
- `src/hooks/useAppealReconciliation.ts`
- `src/hooks/__tests__/useAppealContext.test.ts`
- `src/hooks/__tests__/useAppealParticipation.test.ts`
- `src/hooks/__tests__/useAppealReconciliation.test.ts`
- `src/__tests__/integration/appeal-participation.test.tsx`
- `IMPLEMENTATION_V2_FE_015.md`
- `VERIFICATION_V2_FE_015.md`

### Modified

- `src/app/types/dispute.ts` — added `APPEALED` status and appeal tracking fields

## Security / Canonical State Guarantees

- Contracts remain authoritative; the API remains a read/projection layer.
- No fabricated calldata, gas, transaction hash, receipt, reward, reputation, or settlement.
- Unsupported chain, missing configuration, stale critical data, rejection, revert, replacement, finality, and reorg states fail safely.
- Appeal participation is fully segregated from first-round verification state: participating in the appeal does not block/affect first-round verification, and first-round participation does not block appeal participation.
- No Stellar/Soroban/Freighter runtime, no placeholder address, no production mock in the affected path.

## Validation Commands

```bash
pnpm install
pnpm type-check
pnpm lint
pnpm test
pnpm build
```

The affected test suites are green: `useAppealContext`, `useAppealParticipation`, `useAppealReconciliation`, and the integration suite.

## Reviewer Notes

- UI creators should consume these hooks rather than re-implementing appeal submission.
- The integration test uses the real ABI-driven participation flow (ensure a wallet write mock is aligned with wagmi in CI).
- Backend/contract authors should ensure the pinned appeal ABI exposes the expected function and returns deterministic, auditable projections.
