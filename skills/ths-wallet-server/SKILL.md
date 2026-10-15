---
name: ths-wallet-server
description: Use when changing Thus Spoke Zakura wallet sync, treasury rewards, send or faucet behavior, ths-server APIs, balances, activity, or persistent wallet state.
---

# THS wallet server

Read the current repository `AGENTS.md` and [wallet server](references/wallet-server.md). Trace node RPC and lightwalletd data through the pinned Zakura wallet crates, the server snapshot, and the API result. Use `zcash-wallet`, `zcash-payments`, or `zcash-light-client` for the Zcash semantics touched by the change.

Keep accounts 1–5 user-facing and account 6 internal. Preserve the wallet snapshot as the balance source. Send and faucet POSTs retain intent before wallet work; pending admission is distinct from payment confirmation. Trace the keyed status read and background worker separately. Account payments use `activities`; external-address faucets use `address_faucets`, sharing one idempotency key space. Preserve the original key across uncertain responses and restarts. Use the existing `ApiError`/`ApiResult` pattern and repository test seams. Run Rust checks after `crates/` changes and select a live Docker case when a real proof, node, or retained wallet store is necessary to establish the claim.

## Core workflow

1. Locate the owning step in RPC, lightwalletd download, wallet SDK, local store, snapshot, or API.
2. Follow a normal request and its hardest failure path, including retry after a persisted or broadcast effect.
3. Choose the nearest existing test seam and assert the consequential wallet, activity, and chain state.
4. Rebuild the app image before a live `ths` run and report which binary/image the evidence exercised.

## Review examples

- A pending Send replay returns retained intent; the background worker owns execution. Verify that POST admission and keyed GETs do not perform wallet, broadcast, or mining work, and that the worker progresses the same intent after the client disconnects.
- A wallet sync may publish new account balances before later activity reconciliation fails. Inspect status and balance publication at the actual failure step.
- An address faucet for a published user-account destination uses an account activity; an external destination uses `address_faucets`. Preserve an existing external claim's store on replay, and reject conflicting intent across the shared key space. Name the destination route before claiming its recovery behavior.
