# DChain Customizations

This document describes the changes layered on top of upstream `cosmos/evm` (currently rebased onto `v0.7.3`) by the d-foundation fork.

> In one sentence: dchain removes the value-receiver branch of the bank-keeper type switch so the erc20 precompile compiles cleanly against the d-foundation/cosmos-sdk fork's pointer-only `bankkeeper.NewMsgServerImpl`.

## Contents

- [Patches](#patches)
  - [1. BankKeeper pointer-only type switch](#1-bankkeeper-pointer-only-type-switch)
- [Deliberately NOT Patched](#deliberately-not-patched)
  - [ante IBCKeeper nil-check](#ante-ibckeeper-nil-check)

## Patches

### 1. BankKeeper pointer-only type switch

**File:** `precompiles/erc20/msgsrv.go`

Upstream keeps two arms in the `Send` type switch to accommodate both value and pointer receivers for `bankkeeper.BaseKeeper`:

```go
case bankkeeper.BaseKeeper:
    ...
case *bankkeeper.BaseKeeper:
    ...
```

The d-foundation/cosmos-sdk fork narrowed `bankkeeper.NewMsgServerImpl` to accept `*BaseKeeper` (pointer) only as part of the bank-hooks feature (`SetHooks` mutates the keeper; a value copy would lose the mutation). This makes the `case bankkeeper.BaseKeeper:` arm fail to compile against the fork.

**Change:** the value-receiver case is removed; only `case *bankkeeper.BaseKeeper:` is retained. This is behavior-neutral on dchain — dchain stores `BankKeeper` as `*BaseKeeper` exclusively, so the value arm was unreachable at runtime anyway.

This patch must be re-applied on every future upstream rebase.

## Deliberately NOT Patched

### ante IBCKeeper nil-check

`ante/ante.go` contains `HandlerOptions.Validate()`, which checks that `IBCKeeper` is non-nil. An earlier draft of the dchain v0.7.3 migration patched this nil-check out, because the Protocol branch that consumes this fork (the sdk-0.54.3/evm-0.7.3 upgrade, which also removes IBC from dchain) wires no `IBCKeeper` into the evm ante handler.

**Decision:** no patch here. As of that same Protocol branch, Protocol does not call upstream `Validate()` — it validates its own `HandlerOptions` fields at app initialization, omitting the `IBCKeeper` check entirely. (Protocol versions pinned to `v0.5.0-dchain-v1` predate this and still call `Validate()` with a wired IBCKeeper — the check only becomes moot from `v0.7.3-dchain-v1` consumers onward.) The fork's `ante/ante.go` therefore tracks upstream byte-for-byte, keeping the diff minimal.

## File-Level Diff Summary

Total against `v0.7.3`: **2 files**; +5 / −7 excluding this changelog.

| File | +added / −removed | Category |
|------|-------------------|----------|
| `CHANGELOG_DCHAIN.md` | — | Docs (this file) |
| `precompiles/erc20/msgsrv.go` | +5 / −7 | Bank pointer patch (incl. dropping upstream's stale "both pointer and non-pointer" comment) |
