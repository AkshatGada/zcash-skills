# Wallet server

## Current ownership map

| Boundary | Current owner |
| --- | --- |
| Initialization and service process | `crates/ths-server/src/main.rs` |
| Zakura JSON-RPC and node-selected chain checkpoint | `crates/ths-server/src/rpc.rs` |
| lightwalletd download, rewind, transparent discovery, scan ranges | `crates/ths-server/src/reconcile.rs` |
| Pinned Zakura wallet API, balances, payment creation | `crates/ths-server/src/wallet.rs` |
| Public accounts, activity, idempotency, treasury cursor type | `crates/ths-server/src/db.rs` |
| Atomic treasury reward import, cursor extension, rewind | `crates/ths-server/src/wallet/recovery.rs` |
| HTTP routing and published wallet snapshot | `crates/ths-server/src/api.rs`, `state.rs`, `api/status.rs` |
| Send/faucet admission and keyed address-faucet status | `crates/ths-server/src/api/payments.rs` |
| Accepted-intent traversal and background progression | `crates/ths-server/src/payments/progression.rs` |
| Preparation, submission, confirmation, and treasury replenishment | `crates/ths-server/src/payments/{execution,submission,confirmation,treasury}.rs` |

Inspect `Cargo.toml` and `crates/ths-server/Cargo.toml` before using upstream wallet documentation. This repository imports Zakura wallet crates (`zakura-keys`, `zakura-primitives`, `zakura-client-backend`, `zakura-client-sqlite`, `zakura-proofs`, `zakura-orchard`, `zakura-sapling-crypto`, `zakura-pczt`, `zakura-transparent`, `zakura-zip321`) under `zcash_*` aliases at exact-pinned versions. API semantics and compatibility claims must follow the resolved source.

The wallet database keeps legacy `ext_tsz_*` schema names and migration UUIDs on purpose — a comment in `wallet.rs` requires keeping them stable for existing wallets. Do not rename them when following the `tsz-*` → `ths-*` crate rename elsewhere.

## Sync and balance publication

The path is Zakura checkpoint → lightwalletd indexed data → wallet database → `AppState::refresh_wallet_snapshot` → `/api/v1/accounts` → dashboard query. `synchronize_wallet` holds the sync lock, checks a shared deadline, runs `reconcile::sync_wallet`, then publishes the snapshot. The ready snapshot is guarded by a scanned-versus-observed chain checkpoint check. Failures before that publication leave the earlier account snapshot in place. `refresh_wallet_snapshot` publishes accounts and ready status before `reconcile_unconfirmed`; if activity reconciliation then fails, sync changes status to error while those newer accounts remain published. Test and report this boundary precisely when changing sync behavior.

At startup, `main.rs::serve` checks dependencies, synchronizes, and provisions Account 1 with 5 Ironwood ZEC before starting the background sync loop. The loop checks about every two seconds. `/api/v1/health` can return 200 with a prior successful sync even if the current wallet-sync state is `error`; runtime readiness probes this health route. For a stale balance after startup, compare `/api/v1/status` wallet-sync state and scanned/observed height **and hash**, then `/api/v1/accounts`. A green health check establishes less than a current, ready wallet snapshot. `reconcile.rs::within_deadline` applies per-operation and shared sync deadlines, so locate the phase that timed out.

In `reconcile.rs`, separate reorg detection and rewind, subtree-root updates, transparent output discovery, and prioritized shielded scan ranges. Check which operation advances each cursor and whether the write is durable before a progress claim. The node tip, lightwalletd tip, wallet scanned checkpoint, and user-facing balance can differ transiently. Equal heights do not prove equal block hashes.

Public transparent discovery and treasury reward discovery deliberately use different paths. `wallet.rs::public_transparent_queries` covers accounts 1–5; `reconcile.rs::refresh_transparent_outputs` streams lightwalletd UTXOs for those queries with the selected start height. Account 6 is excluded. For treasury coinbase rewards, `payments/treasury.rs::replenish_treasury` walks the node's block/coinbase evidence through `discover_reward`, then `wallet/recovery.rs` imports the reward and advances `ext_tsz_treasury_cursor`. A claim that public UTXO streaming alone fixes treasury scale misses the separate reward path.

## Treasury and payments

Accounts 1–5 are public development accounts. Account 6 (`TREASURY_ACCOUNT_ID`) is the miner and faucet treasury; it must stay out of account lists, displayed balances, and routine user transparent queries. Reward discovery has a persisted treasury cursor; reorg handling and immature coinbase outputs affect when funds can be spent. A change to bulk or incremental reward processing must preserve the cursor's chain identity and make the next faucet request work after a large reward history.

Trace admission separately from execution. `api/payments.rs` validates Send/Faucet requests and persists intent through `claim_transfer`, `claim_faucet`, or `claim_address_faucet` before wallet work. A new or unresolved payment returns HTTP **202**; a replay of a stored `confirmed` or `failed` payment returns HTTP **200**. Both carry a relative **Location** with the exact encoded idempotency key. Neither admission nor a successful POST proves inclusion; a pending row can have an empty txid.

Follow the returned Location with GET. Account Send/Faucet status is `GET /api/v1/activity?idempotency_key=<key>`, returning a one-item array or `[]`. The address endpoint uses `GET /api/v1/faucet/address?idempotency_key=<key>`, returning an object or HTTP 404. These reads observe local stored state and never start execution. If an admission response is lost, replay the unchanged POST with the **same key** to recover the retained intent; a timeout or missing observation is not permission to create a different key.

`payments/progression.rs::payment_confirmation_loop` traverses accepted intents and drives preparation, submission, and reconciliation independently of HTTP clients. Inspect `payments/execution.rs`, `submission.rs`, `confirmation.rs`, and the wallet journal when locating a failure window. Restart resumes retained work. `preparing`, `prepared`, and `broadcast` are pending; a transient `last_error` does not make them terminal. `confirmed` and `failed` are terminal results. The worker reconciles a retained transaction's identity, chain presence, and expiry before any replacement; replacement must keep the operation key. A deliberate new payment after terminal failure requires a new key.

For the address faucet, `faucet_address` first preserves an existing external claim's store. Otherwise a destination matching a published user-account address is claimed as an account faucet in `activities`; an external destination is claimed in `address_faucets`. The two stores share a key space: changed intent or a conflicting cross-store key is rejected with `IdempotencyConflict`. Keep validation, account boundaries, and exact zatoshi arithmetic intact. `execute_address_faucet` and its `AddressFaucetRuntime` seam are test-only compatibility helpers at the reviewed revision; do not mistake them for the production admission handler. Locate the production worker and its nearest test seam before adding abstractions. Use existing `ApiError`/`ApiResult` for HTTP failures.

### Reward-discovery trace

`prepare_with_replenishment` distinguishes the wallet's `ScanRequired` from `InsufficientFunds`. The first path retries after sync and may mine forward from the birthday; the second uses the SDK's deficit to seek enough spendable rewards. `replenish_treasury` checks the persisted cursor's receiver and selected-chain hash, walks reward candidates, and mines until a candidate can mature. `discover_reward` verifies coinbase, unspent/spender evidence, and the selected chain hash around the wallet commit. `wallet/recovery.rs::commit_treasury_discovery` stores the reward and advances the cursor in one database transaction with a compare-and-swap check.

**Worked large-history diagnosis:** A faucet stalls after 10,000 mined blocks. First determine whether the wallet returned `ScanRequired` or `InsufficientFunds`; those trigger different work. Then compare the persisted cursor's height, hash, and receiver with the selected chain, inspect coinbase maturity and SDK spendability, and measure bulk catch-up separately from the next incremental block. The narrow test `insufficient_funds_discovers_only_the_needed_mature_reward_and_retries` checks the deficit path; the ignored large-history Docker case checks the real stack. A treasury total balance is not a substitute for the wallet's proposal result.

**Cursor interruption example:** Let the persisted treasury cursor be `(H, hash H, receiver R)`. Discovery checks block `H+1` on the selected chain and, if it contains a relevant reward, commits both the imported transaction and cursor `(H+1, hash H+1, R)` in one wallet database transaction. A failure before that commit must leave both at `H`; a restart rechecks `H+1`. A reorg that replaces `hash H+1` must rewind the imported reward and cursor together. The tests `failed_cursor_commit_rolls_back_reward_import_and_reopen_keeps_progress` and `rewind_updates_sdk_and_cursor_atomically` directly exercise these boundaries; `known_spender_must_be_stored_and_spend_the_exact_reward_output` checks spent-reward evidence. These are stronger oracles than a successful faucet HTTP response.

### Send failure windows

| Point of failure | Persisted or external effect to check |
| --- | --- |
| Before durable admission | Validation can reject the request before a new claim; query the original key if the response was lost |
| After claim, before preparation | Intent is retained even with an empty txid; the worker owns further progress |
| After preparation or broadcast | Inspect the server row, wallet journal, original txid, chain evidence, and expiry before replacement |
| Admission response lost or client disconnected | The worker continues; recover the intent with the same-key POST and observe its keyed GET |
| Stored `failed` result | Same-key replay returns the retained failure; only an intentional new payment gets a new key |

This table describes possible boundaries, not proof that each state is currently recovered automatically. For a money-flow change, force the relevant boundary and assert the activity row, txid, chain inclusion, and final wallet snapshot independently.

## Evidence by claim

| Claim | Nearest useful evidence |
| --- | --- |
| Snapshot reads and pre-publication failure | `account_reads_return_the_snapshot_without_contacting_lightwalletd`, `synchronization_failure_preserves_the_last_good_snapshot`; neither proves preservation after activity reconciliation fails |
| Reorg or scan range | `same_height_and_lower_tip_reorgs_find_the_last_common_block`, `scan_ranges_are_split_without_gaps` |
| Hidden treasury or transparent scope | `creates_user_accounts_and_hidden_treasury`, `routine_transparent_queries_exclude_the_treasury` |
| Admission without execution, replay status and Location | `all_payment_posts_admit_without_execution_while_payments_are_busy`, `payment_replays_use_stored_status_and_exact_encoded_location` in `tests/unit/api/payments/tests.rs` |
| Input validation and replay identity | `invalid_sends_are_rejected_before_replay_or_synchronization`, `a_valid_replay_returns_the_original_without_the_wallet_or_network`, `payment_admission_replays_enforce_fingerprints_and_cross_store_identity` |
| Pending uncertainty and terminal failure | `insufficient_send_keeps_the_uncertain_original_lookup_error_pending`, `insufficient_send_is_terminal_for_same_key_but_a_new_key_can_retry` in `tests/unit/payments/progression/tests.rs` |
| Address-faucet replay and key conflicts | `confirmed_address_faucet_replays_without_sending_a_second_payment`, `address_faucet_replay_survives_restart_and_rejects_conflicting_intents`, `concurrent_address_faucet_claims_share_one_operation` |
| Real broadcast recovery | The ignored live `broadcast_recovers_after_auto_mine_failure` case through `ths-regtest-verifier` |

Run `AGENTS.md` Rust checks after `crates/` edits. Rebuild the app image before a live `ths` run reflects server source changes. Keep Docker-free seam tests for logic that does not require a real node or proof.

## Worked task: external-address faucet times out

**Input:** A client posts an address faucet request with a valid Regtest destination and loses the admission response. The original key and unchanged address/amount remain the retry identity. Replay that POST to retrieve its durable result and Location, then GET that status URL. Check whether the destination used an internal account activity or an external `address_faucets` record. Restart the server after admission to test worker recovery; the client does not have to stay connected or repeatedly POST to drive execution.

| Observation after timeout | Safe conclusion |
| --- | --- |
| Original txid and confirmed block found | Payment occurred; a replayed key does not pay again |
| Row in `broadcast` but no confirmed block | Payment remains pending; inspect worker reconciliation and the saved transaction |
| Prepared row past `expiry_height` and tx presence uncertain | Preserve the operation key; expiry alone is insufficient evidence for replacement |
| `preparing` row with a transient error | Observe retained intent and worker retry; do not discard the claim or invent a new key |
| Status GET returns 404 or `[]` | No record was observed at that endpoint; check the endpoint/store and recover admission with the same-key POST |

Start with the admission/replay tests above and `address_faucet_status_reads_internal_and_external_records`. Store-reopen tests such as `address_faucet_replay_survives_restart_and_rejects_conflicting_intents` prove retained identity, not that a real background worker paid. Use the ignored live `address_faucet_retry_after_server_restart_pays_each_destination_once` through `ths-regtest-verifier` when claiming real restart recovery and chain effects. For unit-level worker changes, inject failure at the actual production owner rather than exercising only the test-only dispatch helper.

```console
rg -n 'fn (send|faucet|faucet_address|payment_response)|payment_confirmation_loop|claim_address_faucet' crates/ths-server/src/api/payments.rs crates/ths-server/src/payments crates/ths-server/src/db
cargo test -p ths-server all_payment_posts_admit_without_execution_while_payments_are_busy
cargo test -p ths-server payment_replays_use_stored_status_and_exact_encoded_location
```

Test both an account faucet replay and the address endpoint — they share the idempotency key space, and the same key is rejected across `activities` and `address_faucets`.
