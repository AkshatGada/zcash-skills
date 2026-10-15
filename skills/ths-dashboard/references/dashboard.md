# Dashboard data flow

## Current source map

| Concern | Owner |
| --- | --- |
| HTTP client, endpoints, errors, and runtime validation | `web/src/lib/api/` |
| Server-backed data and query keys | `web/src/hooks/queries.ts` |
| Send, faucet, mine, and query invalidation | `web/src/hooks/mutations.ts` |
| Send/Faucet fingerprints, retention, cross-tab locks, and restoration | `web/src/hooks/payment-identity.ts` |
| Keyed payment observation, terminal cleanup, and recovery controls | `web/src/features/wallet/PaymentStatus.tsx`, `PaymentUpdates.tsx` |
| SSE topic invalidation | `web/src/hooks/useServerEvents.ts` |
| Balance and activity UI | `web/src/features/wallet/` |
| Block, transaction, and address views | `web/src/features/explorer/` |

The server wallet snapshot is the authority for account balances. `WalletPage` reads `useAccounts()` and may sum its account values for display; it must not reconstruct balances from activity or chain data. Activity is a separate record of user operations and confirmation. Keep data ownership in TanStack Query and derive presentation in components.

The API client validates Rust responses with Zod at the boundary. Wallet account and activity zatoshi fields are checked and converted to `bigint` in `schemas.ts`; forms parse decimal strings in `web/src/lib/money.ts`. Explorer chain-value fields (`chainValueZat`, `valueDeltaZat`) currently remain JavaScript `number`, and `SupplyPanel` sums them as numbers. Valid ZEC supply is at most 2.1 quadrillion zatoshis, below JavaScript's safe-integer limit, so this representation is exact for valid chain values. Check the range and API representation if explorer money behavior changes. If Rust changes an endpoint shape, update the endpoint call, schema, affected hook/component, and boundary tests together.

## Mutation and event behavior

`useSend` and `useFaucet` both use `usePaymentMutation`. Trace identity through `payment-identity.ts`: `paymentName` fingerprints the intent, `withPaymentLock` coordinates tabs through `navigator.locks`, and `retainPayment` writes a persistent `localStorage` record, a shared fingerprint key, and this tab's `sessionStorage` key before POST. It verifies retention and attempts rollback on a write failure; missing locks or failed retention blocks submission. Restoration migrates legacy session keys and observes retained payments without automatically POSTing. `operationKey` still serves mining; it is not the Send/Faucet retention mechanism.

Capture the original key at the user action so a queued retry cannot become a new payment if terminal cleanup wins the lock first. An uncertain response retains that key across retries and reloads. Both mutations return `{ activity, key }`, with optional `cleanupError`. They cache the returned activity under `queryKeys.payment(key)` and invalidate wallet-related queries; HTTP success can mean pending admission. Only a terminal activity (`confirmed` or `failed`) triggers terminal identity cleanup. The rejection path releases identity only after a rejected request and a keyed lookup proving the claim absent; a failed lookup retains it. `clearPayment` compares stored identities before removing them, so delayed cleanup cannot erase another tab's newer key. A cleanup error must not change the known payment outcome.

`usePaymentStatus` polls the keyed activity until terminal; `PaymentStatus` handles terminal notification and cleanup, and `PaymentUpdates` keeps displayed outcomes readable after retained identity is released. Preserve server-backed status in Query and caller identity in the retention layer. Distinguish reading status, resuming the original request, and deliberately starting a new payment in tests and UI labels.

`useServerEvents` maps `wallet` to accounts/activity/send-quote, `chain` to status/blocks/mempool plus the `block`/`transaction`/`address` detail prefixes, `sync` to status, and `mining` to `['mining']`. An unknown topic invalidates all mapped keys. Check the actual key and event when a view is stale; adding local mirrored state will hide the broken invalidation.

The activity query key is `['activity', limit]` and block-page key is `['blocks', before ?? 'tip']`; invalidating the shorter prefixes reaches all mounted variants. Detail keys are `['block', id]`, `['transaction', txid]`, `['address', address]` — verify the current `TOPICS` map covers a key before asserting a refresh gap; this mapping has changed before.

The SSE endpoint emits topics without event IDs or replay. If a connection drops or lags, a missed event does not prove the cache is fresh. Compare the current `/api/v1/accounts` result with `useAccounts()` and check whether a refetch occurred. If the API is stale, trace server sync and snapshot publication; if only the UI is stale, trace query invalidation, cache identity, and SSE delivery. Do not assume a fixed sync delay.

### Worked response-state diagnosis

**Input:** Send receives HTTP 202 with `status: preparing` and an empty txid. Show accepted/pending state and observe that key; the worker may prepare and broadcast after the dialog closes. A transient `last_error` while pending is not terminal failure. Show confirmation only when the keyed activity reports `confirmed`; a terminal `failed` result keeps its reason. Exercise acceptance, later confirmation/failure, lost response, and storage cleanup separately in `SendDialog.test.tsx`, `ActivityList.test.tsx`, and the shared Send/Faucet cases in `hooks/faucet.test.tsx`. A mocked status transition proves the UI behavior, not a real chain effect.

For an amount change, inspect `SendDialog`'s form and fee-quote validation. A displayed pool total is not proof of spendability; the server and wallet SDK still govern proposal validity. For identity changes, use the existing tests for cross-tab delayed cleanup, restored pending keys, unavailable locks/storage, and replay after terminal cleanup wins a queued lock. Assert the original request key and absence of unintended POSTs, not only a UUID-shaped value. Keep route, exact amount, and absent-versus-empty memo fingerprints distinct.

## UI and verification

For wallet forms, preserve server validation as the trust boundary while giving prompt field feedback. Check empty, invalid, pending, success, and error states. For explorer changes, distinguish chain data from wallet activity and check loading, not-found, and stale-query paths.

Use Node 24. Run the web lint, format, test, and build commands in `AGENTS.md` after `web/` changes. Nearby tests include `WalletPage.test.tsx`, `SendDialog.test.tsx`, `FaucetDialog.test.tsx`, `schemas.test.ts`, `useServerEvents.test.tsx`, and explorer tests (`ExplorerPage`, `BlockDetail`, `TransactionDetail`, `SearchBar`, `resolve-input`, `shielding`). For live frontend work, use `THS_DEV_API=<dashboard-url> npm run dev --prefix web` with the endpoint from `ths endpoints`; for a container run, rebuild with `cargo run -p thus-spoke-zakura -- build --dev` first.

## Worked task: explorer detail is stale after a chain change

**Input:** The user opens `/explorer/tx/:txid` or `/explorer/block/:id`, then mines or reorganizes the selected chain. `ExplorerPage` routes to `TransactionDetail` or `BlockDetail`; their hooks use `queryKeys.transaction(txid)` and `queryKeys.block(id)`. The `chain` SSE topic currently invalidates status, the `['blocks']` list prefix, mempool, and the `['block']`/`['transaction']`/`['address']` detail prefixes — confirm the mapping in `useServerEvents.ts` covers the affected key before assuming a refresh gap.

1. Reproduce with a mounted detail query and a controlled `chain` event. Record the API response before and after the event and whether that exact detail key refetched.
2. If the API changed but the view did not, fix invalidation at `useServerEvents.ts` or the owning query key; do not copy server data into component state. If the API itself is stale, trace the server RPC and snapshot path instead.
3. Assert a refetch for `['transaction', txid]`, `['block', id]`, and any affected `['address', address]` entry. Keep list pagination keyed by `before` intact. Test a missing detail and a same-height reorg, since a height-only comparison can miss the latter.

**Money oracle for a separate supply-format change:** `schemas.ts` leaves explorer `chainValueZat` and `valueDeltaZat` as numbers. `SupplyPanel` sums pool values as numbers before formatting. Test an ordinary block and a valid value near the 2.1 quadrillion zatoshi supply cap, including signed deltas. Those values are safely representable as integers. A value above `Number.MAX_SAFE_INTEGER` is malformed for ZEC supply; reject it at the API boundary if widening the accepted input range. Do not cite the current number arithmetic as a demonstrated loss of valid ZEC value.

```console
npm test --prefix web -- src/features/explorer/ExplorerPage.test.tsx
npm test --prefix web -- src/features/explorer/TransactionDetail.test.tsx
```

These are nearby tests, not a substitute for a new event-invalidation or supply-precision case. Run all web checks from `AGENTS.md` after editing `web/`.
