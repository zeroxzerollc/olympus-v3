# PolicyEnabler — Restart Pattern

Design doc for adding a `restart()` entry point to `PolicyEnabler` / `PeripheryEnabler`, with the LZ bridge (`LZBridgeGateway` + `LZCrossChainBridge`) as the first consumer.

## Problem

The disable/enable lifecycle is asymmetric:

- `disable(bytes)` — `onlyEmergencyOrAdminRole` (Emergency MS **or** OCG Timelock)
- `enable(bytes)` — `onlyAdminRole` (OCG Timelock only)

When Emergency MS pauses a policy (e.g. the LZ bridge during the rsETH precaution), bringing it back online requires a full OCG cycle even when no parameters need to change — the stored config is still good, we just need to flip `isEnabled` back to `true`.

**Goal:** let a faster signer (DAO MS) re-enable a paused policy from stored state — without an OCG vote, and without granting authority to mutate config or push fresh parameters.

## Landscape

Consumers split into two groups:

**No params on enable** — stored state is the config:

- `LZBridgeGateway` (`_enable()` only flips `isReceiveEnabled`; peers/options/rate limits/`bridgedSupply` live in storage)
- `LZCrossChainBridge` (`_enable()` is a no-op)
- `CCIPBurnMintTokenPool`, `Heart`, `ConvertibleDepositFacility`, `DepositManager`, `DepositRedemptionVault`, `LimitOrders`, `Burner`, `ReserveWrapper`, `CoolerTreasuryBorrower`

**Params required on enable** — `_enable(bytes)` carries semantically meaningful parameters that go stale:

- `EmissionManager` — rates, backing, tick sizes, restart window
- `ConvertibleDepositAuctioneer` — auction target, tick size, min price, tick step
- `V1Migrator` — remaining mint approval

The first group should support `restart()` indefinitely. The second group can opt into a **bounded grace window** during which restart replays the parameters captured at `disable()` time; once the window expires, OCG `enable(bytes)` is the only path back.

## Design

### Core mechanism

`restart()` is a new entry point alongside `enable()` / `disable()`:

1. **No `bytes` payload** — re-enables from stored state only; does not call `_enable(bytes)`. Optional payloads were rejected as a future-ambiguity hazard.
2. **Auth**:
  - `PolicyEnabler` — `MANAGER_ROLE` (DAO MS), with admin also allowed so OCG never loses the ability.
  - `PeripheryEnabler` — `owner` (typically the MS for periphery deployments).
3. **Grace window** — `restart()` is permitted only within `restartWindow()` seconds of the most recent `disable()`. Default is `type(uint48).max` (unbounded). Consumers whose `_enable(bytes)` carries semantically meaningful parameters override to a short value (e.g. 1 day).
4. **Two extension points**:
  - `_restart()` — internal hook for symmetric side-effects (mirrors `_enable`/`_disable`).
  - `restart()` itself is `virtual` — consumers that should *never* support auto-restart override it to revert outright. Putting the opt-out at the public-function level makes refusal visible at the function signature, not buried in a hook.

### PolicyEnabler

```solidity
abstract contract PolicyEnabler is IEnabler, PolicyAdmin {
    bool public isEnabled;
    uint48 public disabledAt;

    error RestartWindowExpired();
    error RestartDisallowed();

    function enable(bytes calldata data_) public onlyAdminRole onlyDisabled {
        _enable(data_);
        isEnabled = true;
        emit Enabled();
    }

    function disable(bytes calldata data_) public onlyEmergencyOrAdminRole onlyEnabled {
        _disable(data_);
        isEnabled = false;
        disabledAt = uint48(block.timestamp);
        emit Disabled();
    }

    /// @notice Re-enable from stored state. `_enable()` is NOT called.
    /// @dev    Permitted only within `restartWindow()` seconds of the most recent `disable()`.
    ///         Consumers that never want auto-restart override this to revert.
    function restart() external virtual onlyManagerOrAdminRole onlyDisabled {
        if (block.timestamp > disabledAt + restartWindow())
            revert RestartWindowExpired();

        _restart();
        isEnabled = true;
        emit Enabled();
    }

    /// @notice Per-contract grace window after `disable()` during which `restart()` is permitted.
    /// @dev    Default = `type(uint48).max` (unbounded). Consumers whose `_enable(bytes)` carries
    ///         semantically meaningful parameters should override to a short value (e.g. 1 day).
    function restartWindow() public view virtual returns (uint48) {
        return type(uint48).max;
    }

    /// @notice Hook for symmetric side-effects on restart (mirrors `_enable`/`_disable`).
    function _restart() internal virtual {}
}
```

### PeripheryEnabler

`PeripheryEnabler` has no ROLES module; auth stays on `owner` (already the typical MS for periphery deployments):

```solidity
function restart() external virtual onlyOwner onlyDisabled {
    if (block.timestamp > disabledAt + restartWindow())
        revert RestartWindowExpired();

    _restart();
    isEnabled = true;
    emit Enabled();
}

function restartWindow() public view virtual returns (uint48) { return type(uint48).max; }
function _restart() internal virtual {}
```

`disabledAt` is stamped inside `disable()` the same way as `PolicyEnabler`.

### IEnabler

Add `restart()`, `disabledAt()`, and `restartWindow()` so external consumers (UIs, monitoring) can detect support and read the deadline.

## Consumer integrations

### LZBridgeGateway (motivating consumer)

- Inherit `PolicyEnabler`.
- Override `_restart()` to mirror `_enable()`:
  ```solidity
  function _restart() internal override {
      if (!isReceiveEnabled) _setIsReceiveEnabled(true);
  }
  ```
- Keep `_enable()` and `_disable()` as-is — they already only manage `isReceiveEnabled`.
- Leave `restartWindow()` at the default (unbounded) — no params on enable, no stale state.
- All config setters (`setPeer`, `setEnforcedOptions`, `setRateLimits`, `setDelegate`, …) stay `onlyAdminRole` / `onlyBridgeAdminOrAdmin`. Restart only flips `isEnabled` and `isReceiveEnabled`. **This is the load-bearing security boundary**: DAO MS gets resume authority, not configuration authority.

### LZCrossChainBridge

- Inherit `PeripheryEnabler`.
- `_restart()` is a no-op (mirrors the no-op `_enable()`).
- `restart()` is gated by `owner`.
- Leave `restartWindow()` at the default.

### Bounded-window example: ConvertibleDepositAuctioneer (future)

```solidity
function restartWindow() public pure override returns (uint48) {
    return 1 days; // auction params go stale within ~1 day
}
```

After `disable()`, DAO MS has 1 day to `restart()` if the pause was a false alarm and the snapshotted auction target / tick size / min price / tick step are still acceptable. Past 1 day, `restart()` reverts and OCG must call `enable(bytes)` with fresh parameters.

### Production flow

1. `disable()` — by Emergency MS, as today.
2. Within `restartWindow()`: `restart()` — by DAO MS (manager role) or `owner` for periphery, no OCG.

`disabledAt = 0` on first deploy is harmless: `onlyDisabled` blocks restart on a never-disabled contract anyway.

