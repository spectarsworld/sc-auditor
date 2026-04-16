# Security Audit Report: VulnerableVault

| Field | Detail |
|-------|--------|
| **Contract** | VulnerableVault.sol |
| **Auditor** | [spectarsworld](https://github.com/spectarsworld) |
| **Date** | 2026-04-10 |
| **Solidity** | ^0.8.19 |
| **Methods** | Manual review, Slither, SC-Auditor |

## Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 1 |
| 🟠 High | 2 |
| 🟡 Medium | 1 |
| 🔵 Low | 1 |

---

## Findings

### [C-01] Reentrancy in `withdraw()`

**Severity:** Critical

The `withdraw` function sends ETH to the caller before updating the internal balance. A malicious contract can re-enter `withdraw` and drain the vault.

```solidity
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");

    // ❌ External call before state update
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");

    balances[msg.sender] -= amount;
}
```

**Impact:** Complete loss of funds. An attacker deploys a contract whose `receive()` function calls `withdraw()` again. Each re-entrant call passes the balance check because `balances[msg.sender]` hasn't been decremented yet. The vault is drained in a single transaction.

**Recommendation:**

Apply checks-effects-interactions. Update state before the external call:

```solidity
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");

    balances[msg.sender] -= amount;

    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}
```

Additionally, consider adding a reentrancy guard (`nonReentrant` modifier from OpenZeppelin).

---

### [H-01] Missing Access Control on `setAdmin()`

**Severity:** High

```solidity
function setAdmin(address _newAdmin) external {
    admin = _newAdmin;
}
```

Anyone can call `setAdmin` and take over administrative privileges. There is no `onlyAdmin` or ownership check.

**Impact:** An attacker sets themselves as admin, gaining access to privileged functions (fee changes, pausing, emergency withdrawal). Full protocol takeover without needing to touch user funds directly.

**Recommendation:**

```solidity
function setAdmin(address _newAdmin) external {
    require(msg.sender == admin, "Not admin");
    require(_newAdmin != address(0), "Zero address");
    admin = _newAdmin;
}
```

Consider using OpenZeppelin's `Ownable2Step` for a two-step transfer pattern that prevents accidental transfers to wrong addresses.

---

### [H-02] Unchecked ERC-20 `transfer` Return Value

**Severity:** High

```solidity
function withdrawToken(address token, uint256 amount) external {
    require(tokenBalances[msg.sender][token] >= amount, "Insufficient");
    tokenBalances[msg.sender][token] -= amount;
    IERC20(token).transfer(msg.sender, amount);  // ❌ return value ignored
}
```

Some ERC-20 tokens (like USDT) don't revert on failure — they return `false`. Ignoring the return value means the internal balance is decremented but tokens never actually leave the contract. The user loses their accounting balance while funds stay locked.

**Impact:** Users permanently lose access to deposited tokens for any ERC-20 that returns `false` on failure instead of reverting.

**Recommendation:**

Use OpenZeppelin's `SafeERC20`:

```solidity
using SafeERC20 for IERC20;

function withdrawToken(address token, uint256 amount) external {
    require(tokenBalances[msg.sender][token] >= amount, "Insufficient");
    tokenBalances[msg.sender][token] -= amount;
    IERC20(token).safeTransfer(msg.sender, amount);
}
```

---

### [M-01] Timestamp Dependence in Reward Calculation

**Severity:** Medium

```solidity
function claimRewards() external {
    uint256 elapsed = block.timestamp - lastClaim[msg.sender];
    uint256 reward = balances[msg.sender] * rewardRate * elapsed / 1e18;
    lastClaim[msg.sender] = block.timestamp;
    _mint(msg.sender, reward);
}
```

`block.timestamp` can be manipulated by validators within a ~15 second window. For high-value positions, a validator who is also a depositor could influence their reward payout by choosing favorable timestamps.

**Impact:** Minor reward inflation per block, but it compounds. In practice, the risk scales with the reward rate and deposit size. Unlikely to be profitable for most users, but worth addressing if the vault holds significant TVL.

**Recommendation:**

Use block numbers instead of timestamps for elapsed time, or accept the small variance and document it as a known limitation. If using timestamps, ensure the reward rate is low enough that a 15-second manipulation window doesn't produce meaningful gain.

---

### [L-01] Missing Events on State Changes

**Severity:** Low

The following state-changing functions emit no events:

- `withdraw()`
- `deposit()`
- `setAdmin()`
- `setRewardRate()`

**Impact:** Off-chain monitoring and indexing tools (The Graph, block explorers, internal dashboards) can't track contract activity. Incident response is harder because there's no event trail to query. This is a best-practice issue, not a direct security risk.

**Recommendation:**

```solidity
event Deposit(address indexed user, uint256 amount);
event Withdrawal(address indexed user, uint256 amount);
event AdminChanged(address indexed oldAdmin, address indexed newAdmin);
event RewardRateUpdated(uint256 oldRate, uint256 newRate);
```

Emit these in each respective function.

---

## Disclaimer

This report represents a point-in-time review based on the source code provided. It does not guarantee the absence of vulnerabilities. Smart contract security is an ongoing process — changes to the codebase after this review may introduce new issues.
