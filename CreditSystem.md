---
Summary: The credit economy. Users hold time-bounded credit cohorts; actions (asset/map generation) spend them FIFO oldest-first; unspent credits expire 3 months after grant. The money lives in a self-hosted **Formance OSS ledger** (double-entry, append-only) — Postgres holds no credit balances anymore. Domain code talks to `ICreditService`/`ICreditRepository`; the only implementation is `CreditRepositoryFormance`, which translates cohorts/FIFO/expiry/idempotency into Numscript transactions against the ledger. The ledger IS the audit trail, so user history reads from the same log that is the money. Neon Postgres keeps only *config* tables (costs, allotments, tiers). Temporal activities are the sole ledger I/O path for the orchestrator.
Tags: #service #credits #billing #formance #ledger #numscript #temporal #atlasforge
---

# CreditSystem

## Domain model & contracts

A **cohort** is one grant of credits with a `grantedAt` and an `expiresAt` (= grantedAt + 3 months). A user holds many cohorts. Spending drains cohorts **FIFO, oldest grant first**; a charge that exceeds the user's total active balance is rejected **atomically** (no partial drain). Unspent credits **expire** at `expiresAt` — credits are never deleted on downgrade/cancel, only burned on expiry.

The contracts live in `packages/schema/src/domain/credits.ts`:
- `ICreditService` — application surface: `deductCredits`, `grantCohort`, `getBalance`, `getActiveCohorts`, `expireStaleCohorts`, `getActionCosts(Map)`, `estimateMapForgeCredits`, `getCreditHistory`.
- `ICreditRepository` — persistence seam. **Only one impl: `CreditRepositoryFormance`** (`packages/credits/`). `CreditRepositoryPostgres` was deleted in the Formance cutover (Phase 2); the Postgres ledger tables `credit_cohorts`/`credit_deductions` are dropped.

`CreditService` is a thin delegate over the repository plus the `action_costs` pricing reads.

## How Formance / Numscript re-expresses the domain

The repository maps every domain concept to a ledger primitive — there is no impedance layer beyond `FormanceLedgerClient` (the deserialization seam; the `@formance/formance-sdk` never escapes it):

| Domain | Formance / Numscript |
|---|---|
| Cohort | A holding **account** `users:<id>:cohorts:<cohortKey>` (one per grant) |
| Grant | `mint` from `world` → cohort account (Numscript `grantScript`) |
| FIFO spend | `spend` with an **ordered source set** of cohort keys (oldest first); Numscript drains in order |
| No overdraft | Numscript posts atomically; insufficient funds → ledger rejects → `InsufficientCreditsError` |
| Expiry | `burn` cohort balance → `platform:expired` |
| Idempotency | The ledger **`Idempotency-Key`** (`grant:<id>:<cohortKey>`, `expire:<id>:<cohortKey>`, spend's `dedupeKey`) — not a DB pre-check |
| Tagging / audit | account + transaction **metadata** (`type`, `userId`, `tier`, `actionType`, `dedupeKey`) |
| Balance | server-side `getBalancesAggregated` (no row materialization) |

`cohortKey` is the **billing-cycle-start date** (`YYYY-MM-DD`, via `cohortKey(grantedAt)` in `formance/addressing.ts`). This is load-bearing: keying the cohort on the deterministic billing-cycle start (not `now()`) is what makes a **re-published grant idempotent** — the same `(user, cycle)` maps to the same account + mint IK, so the first grant in a cycle wins and re-deliveries no-op. This *replaced* the old Postgres `(user_id, granted_at)` uniqueness pre-check (`CreditSyncService`).

**Idempotency-conflict subtlety:** Formance binds the IK to a body hash. An identical-body replay returns the stored txn (silent no-op); a *same-IK, different-amount* re-grant (e.g. a mid-cycle upgrade diff vs the initial grant) returns **`400 VALIDATION`**. `FormanceLedgerClient.mint` swallows exactly that conflict (`isIdempotencyConflict`: `errorCode === 'VALIDATION'` + message has `'idempotency key'` AND `'is stored'`) as first-write-wins — faithfully preserving the old "one cohort per cycle, mid-cycle upgrade diff dropped" behavior. `V2ErrorsEnum` has no finer code, so the message tokens are the necessary discriminator; any other VALIDATION error fails loud.

## The Temporal seam (orchestrator)

Asset-forge runs on Temporal. Two distinct credit activities exist by design (see [[Temporal-Workflow-Activity-Boundary]]):
- **`checkCredits`** — advisory pre-flight (before doing expensive work): does the user *probably* have enough? Cheap, may be stale.
- **`deductCredits`** — **authoritative**, post-persist: charge only after an asset was actually produced. Idempotent via the ledger IK = `dedupeKey` (`generationId`, or `${generationId}:${assetId}` per-asset), so a Temporal retry re-running the activity can't double-charge. `InsufficientCreditsError` → non-retryable `ApplicationFailure(type: 'InsufficientCreditsError')`.

**All ledger I/O is activities-only** — workflows stay deterministic (no ledger calls, no `Date`/random). The `creditService` reaches activities through the `createActivities(deps)` closure-DI factory (deps in the closure, only serializable args cross the proxy).

## Grants & expiry (lifecycle)

- **Grants** — `CreditSyncService` (worker) consumes Patreon tier-change Pub/Sub events and calls `grantCohort(userId, credits, tier, expiresAt, grantedAt=billingCycleStart)`. The `user_tiers` upsert (Neon) runs on **every** event (fresh + redelivered); the mint is idempotent by IK. No DB pre-check.
- **Expiry** — a daily **Temporal Schedule** (`registerExpirySchedule` → `expireCohortsWorkflow` → `burnExpiredCohorts` activity → `expireStaleCohorts()`). The cross-user sweep lists all cohort accounts (cursor-paginated) and burns each stale one (idempotent per-cohort by `expire:` IK). This replaced the worker `setInterval` + `pg_advisory_lock` cron. ⚠️ **Prod gap (2026-06): the orchestrator isn't deployed to prod yet, so the Schedule doesn't run there** — see the project memory; deploy the orchestrator (or restore the interim worker cron) before real expirable grants accrue.

## Account / asset conventions

The **single source of address truth** is `formance/addressing.ts` — nothing else concatenates ledger addresses.

| Address | Role |
|---|---|
| `world` | Built-in infinite mint source (grants flow out of here) |
| `users:<userId>:cohorts:<cohortKey>` | One holding account per grant; holds that cohort's remaining balance |
| `platform:redeemed` | Destination for spent credits (redemption, not cash revenue) |
| `platform:expired` | Burn sink for expired credits (kept distinct from redeemed so churn ≠ consumption) |

Asset: `CREDIT`, scale 0 (whole numbers, `[CREDIT 5]`). Cross-user list filters use Formance's trailing-colon (`users::cohorts:`) wildcard.

**Numscript injection guard (§4.1).** `addressing.ts` is the **single splice site** where externally-sourced values (the grant path's `userId`/`tier` come from a Patreon Pub/Sub event, NOT a JWT) enter addresses + Numscript — so the charset guard lives there and is authoritative. Address **segments** (userId, cohortKey) must match `^[A-Za-z0-9_-]+$` (`assertAddressSegment`, applied in `cohortAccount`/`userCohortsPrefix`, so spend/burn inherit it); metadata **values** (tier) forbid only `"`, `\`, and control chars (`assertMetadataValue` on `tier` in `grantScript`) so a multi-word tier still works. Both throw `ValidationError` — fail fast, never emit injected Numscript. `CreditSyncService.validateTierChangeEvent` reuses the same predicates as boundary defense-in-depth (ack-and-skip a poison message). This charset guard is the §4.1 *stopgap*; the structural fix (Numscript `vars` + the typed `metadata` field instead of inline `set_account_meta`) + a DB-backed tier allowlist (§4.2) remain the pre-launch Phase 4 target.

**Pagination:** Formance v2 list endpoints are **cursor-only** (no offset/skip; pageSize ≤ 15). `listAccountsMatching` / `listTransactions` walk the cursor to exhaustion (continuation passes `{ledger, cursor}` alone); a defensive page cap fails loud on a runaway. Prefer aggregate endpoints (`getBalancesAggregated`) over materializing rows when you only need a sum.

## Auditability — `GET /api/credits/history`

The append-only ledger *is* the audit trail, so user-facing history reads the same log that is the money — the user view and system truth cannot disagree. `getCreditHistory(userId, {cursor, pageSize})` projects ledger transactions (filtered to `metadata[userId]`) into `CreditHistoryEntry { type: 'grant'|'spend'|'expiry', amount, timestamp, actionType?, tier? }`, cursor-paginated (consumer-driven, not accumulated server-side). `userId` is JWT-only (never from query) — same rule as `/balance`.

## What stays in Neon Postgres

Only **config** (not money): `action_costs` (per-action credit price), `tier_credit_allotments` (credits per tier), `user_tiers` (current tier). `CreditRepositoryFormance` still holds a `pg.Pool` purely to read `action_costs` for `getActionCosts`/`estimateMapForgeCredits`. No balances, no cohorts, no deductions in Postgres.

## Cross-references
- [[Temporal-Workflow-Activity-Boundary]] — why `checkCredits` (advisory) and `deductCredits` (authoritative) both exist; activities-only ledger I/O.
- [[Orchestrator-Service]] — the Temporal worker that hosts the credit activities + expiry schedule.
- [[Worker-Service]] — hosts `CreditSyncService` (Patreon grants).
- [[API-Service]] — `/api/credits/balance` + `/api/credits/history` routes.
- [[Architecture-Decoupling]] — repository seam, typed outbound client (`FormanceLedgerClient`), SDK confinement.
- [[Type-Discipline]] — the deserialization boundary where SDK shapes (bigint, metadata maps) become domain types.
- [[Deployment]] — the self-hosted OSS ledger on Cloud Run (IAM-gated, idtoken auth) + its migrate/init steps.
