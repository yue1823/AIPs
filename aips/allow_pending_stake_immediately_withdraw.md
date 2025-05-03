---
aip: X
title: Allow Pending Stake Immediately Withdraw
author: Alex Yue
Status: Draft
discussions-to: 
type: Core, Framework
created: 04/25/2025
---

# AIP X: Allow Pending Stake Immediately Withdraw

## Summary
   This AIP proposes a modification to the Aptos Framework's staking module (stake.move) to allow delegators to immediately withdraw APT coins that are in the pending_active state within a StakePool. Currently, stake added during an epoch only becomes active and starts earning rewards at the beginning of the next epoch, and it cannot be withdrawn until it transitions out of the pending_active state (typically by becoming active then later pending_inactive). This restriction hinders the capital efficiency and responsiveness of protocols built on top of Aptos staking, particularly Liquid Staking Tokens (LSTs). This proposal introduces a mechanism to withdraw pending_active stake within the same epoch it was added, improving LST stability and overall staking flexibility.
## Motivation
   The current Aptos staking mechanism introduces a delay for newly staked APT. When a user delegates APT to a validator via `0x1::stake::add_stake`, the amount is added to the pending_active field of the corresponding StakePool. This stake does not earn rewards and cannot be withdrawn until the next epoch boundary when it transitions to the active state.
   This delay presents challenges, especially for LST protocols:
   Reduced Capital Efficiency: Funds deposited into an LST protocol, which are then staked, are effectively idle for up to one epoch before generating yield. If a user needs to redeem their LST for APT shortly after depositing, the protocol cannot efficiently access the underlying pending_active APT.
   LST Peg Stability Risk: LSTs aim to represent staked APT while providing liquidity. Ideally, an LST (e.g., xAPT) should trade close to the value of the underlying staked APT plus accrued rewards. If xAPT temporarily trades below its underlying value (de-pegs), arbitrageurs should be able to buy cheap xAPT, redeem it for APT via the LST protocol, and sell the APT for a profit, pushing the xAPT price back towards its fair value. However, if the LST protocol's recently added stake is locked as pending_active, it cannot fulfill redemption requests immediately using that capital. It must either use its active stake (losing potential rewards for long-term holders) or wait for the epoch change, slowing down the arbitrage process and potentially allowing the de-peg to persist longer.
   Suboptimal Redemption Logic: When an LST protocol needs to fulfill a redemption request, it's most efficient to use stake that isn't currently earning rewards. If a protocol holds both active stake (earning rewards) and pending_active stake (not earning rewards yet), it would strongly prefer to redeem the pending_active portion first. The current system prevents this within the same epoch the stake was added.
   By allowing the immediate withdrawal of pending_active stake, this AIP directly addresses these issues. It enables LST protocols (and potentially other applications) to:
   Offer near-instant redemption paths for recently deposited funds.
   Facilitate faster and more efficient arbitrage, strengthening the LST peg stability.
   Optimize redemption logic by using non-yielding capital first.

   Example Scenario:
   Alice deposits APT into an LST protocol, minting xAPT. The protocol adds her APT to a validator's StakePool, placing it in pending_active. Later in the same epoch, the price of xAPT drops significantly below its intrinsic value on a DEX. Bob sees this arbitrage opportunity. He wants to buy cheap xAPT and redeem it immediately for APT via the LST protocol. If the protocol can instantly withdraw the pending_active APT (from Alice's recent deposit or others), it can fulfill Bob's redemption request quickly. Bob's actions help restore the xAPT peg. Without this AIP, the protocol might have to wait until the next epoch, delaying the arbitrage and potentially harming user confidence in xAPT.

## Specification

```move

    const EINSUFFICIENT_PENDING_ACTIVE_STAKE: u64 = 1234;

    /// Withdraws stake that is currently in the `pending_active` state for the given stake pool.
    /// Can only be called by the owner of the stake pool.
    /// This operation can only be performed within the same epoch the stake was added.
    /// Aborts if the caller is not the owner, if the stake pool doesn't exist,
    /// or if the amount requested exceeds the available `pending_active` stake.
    public entry fun withdraw_pending_active_stake(
        owner: &signer,
        operator_address: address,
        amount: u64
    ) {
        let stake_pool = borrow_global_mut<StakePool>(signer::address_of(owner));
        // 1. Assert that the stake pool is associated with the specified operator.
        assert!(address_of(owner) == stake_pool.operator_address,E_not_operator);
        // 2. Check for sufficient pending_active stake.
        assert!(stake_pool.pending_active >= amount, Errors::invalid_argument(EINSUFFICIENT_PENDING_ACTIVE_STAKE));
        // 3. Decrease the pending_active balance.
        stake_pool.pending_active = stake_pool.pending_active - amount;
        // 4. Move the APT coins back to the owner.
        //    (Requires the staking module/StakePool itself to hold the APT, or access to it.
        let coins = coin::extract<AptosCoin>(&mut stake_pool.pending_active, amount);
        coin::deposit<AptosCoin>(address_of(owner), coins);
    }

```
Key Considerations:

- Scope: This function only operates on pending_active stake. It does not affect active, pending_inactive, or locked_stake.
- Epoch Boundary: This withdrawal is only possible before the epoch transition occurs that would move the stake from pending_active to active. Once the epoch transition happens, this function can no longer withdraw that specific amount. The system implicitly handles this, as the stake won't be in pending_active after the transition.
- Permissions: Only the owner (delegator) of the StakePool can initiate this withdrawal. LST protocols would call this using the signer capability derived from their resource account holding the StakePool object.
- Fund Flow: The exact mechanism for returning the APT (coin::withdraw/deposit lines above) needs to align with how the stake.move module manages the underlying APT collateral.

Rationale
- Simplicity: Introducing a new, dedicated function withdraw_pending_active_stake is cleaner than overloading existing withdrawal functions (withdraw_stake, unlock_stake). It clearly signals the specific state being targeted and avoids complicating the logic for handling other stake states.
- Targeted Solution: It directly addresses the identified problem of locked pending_active stake without altering the core mechanics of active or inactive stake management and reward distribution.
- Efficiency Focus: The design prioritizes enabling LSTs and users to reclaim non-yielding capital quickly, enhancing market efficiency.

Backwards Compatibility
- This change is backwards compatible:
- It introduces a new function, not modifying existing ones. Existing contracts and users interacting with the stake.move module are unaffected.
- No data structures are fundamentally changed in a way that breaks existing reads or writes (only the value of pending_active is potentially modified by the new function).

## Impacts

#### Positive:
   - Improved LST Stability: Enables faster arbitrage, helping LSTs maintain their peg.
   - Increased Capital Efficiency: Reduces the time capital is unproductive in the staking system.
   - Better User Experience: Allows faster redemptions for LST users under specific conditions.
   - Potential for New Products: May enable novel financial instruments built on Aptos staking.
#### Neutral/Negative:
   - Increased Framework Complexity: Adds a new function and logic path to the core staking module.
   - Potential Minor Gas Overhead: Slightly more complex state to manage, though likely negligible impact on existing operations.
## Reference Implementation

A prototype implementation of the withdraw_pending_active_stake function as described in Section 3 should be developed and tested within the Aptos Framework codebase. This includes adding the necessary error code and event modifications.

## Risks and Drawbacks
- Authorization: The primary security check is ensuring only the legitimate owner of the StakePool resource can call this function for their stake. This relies on Move's standard ownership and capability model via the signer.
- State Management: Ensure atomicity. The decrement of pending_active and the transfer of APT coins must occur together or not at all.
- Epoch Transition Race Conditions: The function must correctly read the pending_active state. If called during an epoch transition, the system should handle it gracefully – either the stake is still pending_active and withdrawable, or it has already transitioned to active and the assert! for sufficient pending_active stake will correctly fail. This needs careful review during implementation.
- Economic Exploits: The ability to withdraw immediately might slightly alter incentive calculations for validators or delegators, but since it only applies to non-reward-earning stake, major economic shifts are unlikely. The primary impact is on LST mechanics, which is the intended goal.
## Timeline

